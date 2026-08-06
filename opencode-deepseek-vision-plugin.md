# 给 DeepSeek 装双眼睛：纯文本模型在 opencode 里看懂截图的折腾实录

我用 opencode 当日常的主力 AI 编辑器已经有一阵子了。这玩意儿生态不错，插件机制、各种 hook 一应俱全，用起来很顺手。唯一的痛点是：我默认挂在背后的模型是 deepseek-v4-flash，一个纯文本模型。

纯文本模型意味着什么？意味着我没法直接把一张截图甩给它说"帮我看看这个报错"。

但日常干活哪有不截图的时候——GCP 控制台报错、BigQuery 查询结果、某个同事发来的架构图，全都得靠截图沟通。于是这个需求就变成了：**怎么让一个不吃图片的模型，也能"看懂"图片？**

## 思路：找个翻译官，而不是换模型

换个大模型当然最省事，但成本高，而且没必要——我的文本模型已经够用，缺的只是一个"图片翻译"环节。

所以方案是绕道走：

```
用户贴图 → 视觉模型把图片翻译成文字描述 → 文字描述喂给主模型 → 主模型"读"描述来回答
```

翻译官我选了阿里的 DashScope，用 `qwen3-vl-flash` 这个视觉模型。便宜、响应快，关键是 DashScope 有 OpenAI 兼容的接口，代码写起来几乎零成本。

opencode 提供了插件机制，可以在消息进入 LLM 之前做手脚。于是计划很清晰：写一个插件，拦截用户消息里的图片，调 DashScope 分析，把结果转成文字，替换掉原图。

听起来很简单对吧？我当时的想法也是"一下午就能搞定"。结果这一下下午，变成了两天。踩了三个坑，每一个都值得单独拿出来说。

## 坑一：插件目录名，单数和复数不是一个世界

第一个版本写得很快，`chat.message` hook、`getApiKey` 读环境变量、调 DashScope、替换 part，一气呵成。

然后重启 opencode，贴图，等回复。

……没反应。一点动静都没有，好像插件根本不存在。

查日志、查配置、查文档，折腾半天，最后在 opencode 官方文档里看到了这样一句话：

> 自动加载的插件目录只有两个：项目级 `.opencode/plugins/` 和全局级 `~/.config/opencode/plugins/`。

注意，是 `plugins`，复数。

我把插件放到了 `~/.config/opencode/plugin/`，单数。opencode 压根没扫这个目录。

就这么一个字母的差别，我的插件连被加载的资格都没有。改目录，重启，插件终于活了。

**教训：插件机制这种东西，目录名一定以官方文档为准，别凭感觉猜。**

## 坑二：非 async 函数里用 await，JS 直接当场去世

目录改对了，插件加载了，但启动就报错。看日志，`SyntaxError`。

定位到 `getApiKey` 这个函数：

```ts
function getApiKey(): string | undefined {
  if (process.env.QWEN_API_KEY || process.env.DASHSCOPE_API_KEY) {
    return process.env.QWEN_API_KEY || process.env.DASHSCOPE_API_KEY;
  }
  const fs = await import("node:fs"); // ← 这里
  // ...
}
```

我原本想偷懒用动态 `import` 去读 `.env.local` 文件，但写的时候函数忘了加 `async`。JS 的规则是：**`await` 只能出现在 async 函数里**。非 async 函数里写 `await`，直接 SyntaxError，整个文件加载失败。

修复也简单：把 `node:fs`、`node:os`、`node:path` 挪到文件顶部静态导入，`getApiKey` 恢复成普通同步函数。

**教训：写完代码先 `node --experimental-strip-types --check` 过一遍语法，别急着跑。这种低级错误浪费了我大半个晚上。**

## 坑三：消息"丢了"——其实是 hook 把管道堵死了

到这里插件总算能用了，但主人反馈了一个更诡异的现象：**贴图发消息，第一次发不出去，重发才成功。** 而且不是偶发，是每次都这样。

我一开始以为是消息真的丢了，翻日志、查数据库，把消息时间线全捞出来对。结果发现了一个惊人的规律——每条图片消息，在数据库里都有两三条几乎一模一样的记录，时间只差十几秒：

| 时间 | 内容 | 有没有回复 |
|---|---|---|
| 20:07:36 | `[Image] 解析一下截图` | ❌ 没回复 |
| 20:07:52 | `[Image] 解析一下截图`（重发） | ✅ 有回复 |
| 20:08:42 | `[Image] 里面是什么别乱说` | ❌ 没回复 |
| 20:08:48 | `[Image] 里面是什么别乱说`（重发） | ✅ 有回复 |

消息根本没丢，是**第一次发送时整个界面卡住了**。

原因在于 `chat.message` 这个 hook 的工作方式：它是同步等待的。我贴的截图一般比较大，DashScope 分析一张大图要 10~20 秒。在这十几秒里，opencode 的消息处理管道就卡在那儿等 hook 返回，界面表现为"发送中"却毫无反应。主人以为没发出去，重发一次，第二条消息才触发正常的处理流程。

说白了，是"假丢失"——**hook 同步阻塞 + 视觉模型响应慢 = 看起来像消息没了。**

## 双保险：超时熔断 + 换 hook

解决思路有两个，我最后两个都上了：

**第一个：给 DashScope 调用加超时熔断。**

用 `AbortController`，设置 60 秒超时。分析超时就快速降级，返回一段占位文本，绝不让网络请求无限期阻塞消息管道：

```ts
const controller = new AbortController();
const timer = setTimeout(() => controller.abort(), 60_000);

try {
  const resp = await fetch(`${BASE_URL}/chat/completions`, {
    method: "POST",
    headers: { "Content-Type": "application/json", Authorization: `Bearer ${apiKey}` },
    body: JSON.stringify({ model: "qwen3-vl-flash", messages: [...] }),
    signal: controller.signal,
  });
  // ...
} finally {
  clearTimeout(timer);
}
```

**第二个：把 hook 从 `chat.message` 换成 `experimental.chat.messages.transform`。**

这个 hook 的语义完全不同：它是在消息**即将发给 LLM 之前**才执行转换。也就是说，消息的入库、显示、发送流程完全不受影响，图片分析发生在"回复生成前"这一步。就算分析超时降级，消息本身也已经在对话里躺得好好的了，绝不丢：

```ts
"experimental.chat.messages.transform": async (_input, output) => {
  for (const msg of output.messages) {
    if (msg?.parts && Array.isArray(msg.parts)) {
      await translateParts(msg.parts);
    }
  }
},
```

改完之后体验彻底变了：贴图，消息立刻显示在界面里，然后慢个几秒等视觉模型分析完，再正常出回复。再也没有"发不出去"的错觉。

## 一些收尾的细节

- **API Key 不硬编码**：走环境变量 `QWEN_API_KEY` / `DASHSCOPE_API_KEY`，回退到 `~/.config/opencode/.env.local` 这个被 gitignore 保护的文件，插件本身干干净净，随便传 GitHub 都不怕。
- **图片描述做了缓存**：同一张图重复发，直接命中缓存，不重复花钱调 API。
- **降级策略**：图片分析失败（网络问题、Key 失效、超时）时，把图片替换成 `[图片解析失败：xxx]` 的占位文本，对话还能继续，不会因为一张图把整个 session 卡死。

## 现在回头看

这套方案的本质，其实是**用"翻译"代替"升级"**。模型不够强的地方，用一个便宜的外部能力去补齐，成本低、见效快、可替换。视觉模型今天用 qwen，明天换别的，插件逻辑一行都不用改，换模型名就行。

而且它治好了我最大的一个坏习惯——以前截图发不出去，我总怀疑是 opencode 的 bug，来回重启、清缓存。现在才知道，八成是自己写的插件把管道堵死了。

**在怪工具之前，先想想是不是自己写的东西在里面捣乱。**

如果哪天你也遇到了"消息发不出去"的诡异现象，记得先检查一下自己的插件 hook 是不是同步阻塞了消息管道——这个坑，我已经替你踩过了。

## 附：完整插件源码

插件整个就一个文件 `vision-translator.ts`，放到 `~/.config/opencode/plugins/`（注意是复数！）重启 opencode 就能用。API Key 优先读环境变量 `QWEN_API_KEY` / `DASHSCOPE_API_KEY`，没有的话会回退到 `~/.config/opencode/.env.local` 这个被 gitignore 保护的文件里读，格式就是一行 `QWEN_API_KEY=sk-xxx`：

```ts
/**
 * vision-translator plugin
 * --------------------------
 * 为纯文本模型（如 deepseek-v4-flash）提供"图片预分析"能力：
 * 用户发送图片 → 调用 DashScope qwen3-vl-flash 视觉模型 → 转成文字描述 → 主模型"读"描述。
 *
 * API Key 读取优先级（绝不硬编码）：
 *   1. 环境变量 QWEN_API_KEY
 *   2. 环境变量 DASHSCOPE_API_KEY
 *   3. 回退到 gitignored 安全文件 ~/.config/opencode/.env.local
 *   若均缺失 → 保留原图片 part，不阻塞用户。
 *
 * ⚠️ 2026-08-06 修复记录：
 *   - 目录：~/.config/opencode/plugin/（单数，opencode 不识别）→ plugins/（复数，官方要求）
 *   - Bug：getApiKey 曾用 `await import()` 但函数未声明 async，导致 SyntaxError 加载失败
 *          现改为顶部静态导入 node:fs/os/path，getApiKey 恢复同步函数。
 *   - 方案A+B（消息丢失修复）：
 *     A. DashScope 调用加 AbortController 超时熔断（60s），超时快速降级，绝不阻塞消息管道；
 *     B. hook 从 chat.message 改为 experimental.chat.messages.transform
 *        （在消息即将发给 LLM 前才转换图片），避免同步 await 网络请求卡死消息处理。
 */

import { existsSync, readFileSync } from "node:fs";
import { homedir } from "node:os";
import { join } from "node:path";
import type { Plugin } from "@opencode-ai/plugin";

const DASHSCOPE_BASE_URL = "https://dashscope.aliyuncs.com/compatible-mode/v1";
const VISION_MODEL = "qwen3-vl-flash";
const DASHSCOPE_TIMEOUT_MS = 60_000;

function getApiKey(): string | undefined {
  // 1. 环境变量优先
  if (process.env.QWEN_API_KEY || process.env.DASHSCOPE_API_KEY) {
    return process.env.QWEN_API_KEY || process.env.DASHSCOPE_API_KEY;
  }
  // 2. 回退到本地 gitignored 安全文件（~/.config/opencode/.env.local）
  try {
    const envFile = join(homedir(), ".config/opencode/.env.local");
    if (existsSync(envFile)) {
      const content = readFileSync(envFile, "utf8");
      const m = content.match(/^QWEN_API_KEY=(.+)$/m);
      if (m) return m[1].trim();
    }
  } catch {
    // 文件读取失败则忽略，交给上层处理
  }
  return undefined;
}

/** 图片描述缓存：避免同一图片重复调用视觉 API */
const descriptionCache = new Map<string, string>();

/**
 * 调用 DashScope 视觉模型分析图片。
 * 带 60s 超时熔断：DashScope 响应慢时立即 abort，快速降级返回错误信息，
 * 避免网络请求无限阻塞 opencode 的消息处理管道。
 */
async function analyzeImage(dataUrl: string): Promise<string> {
  const cached = descriptionCache.get(dataUrl);
  if (cached) return cached;

  const apiKey = getApiKey();
  if (!apiKey) {
    throw new Error("QWEN_API_KEY / DASHSCOPE_API_KEY 未配置");
  }

  // 超时熔断器
  const controller = new AbortController();
  const timer = setTimeout(() => controller.abort(), DASHSCOPE_TIMEOUT_MS);

  try {
    const resp = await fetch(`${DASHSCOPE_BASE_URL}/chat/completions`, {
      method: "POST",
      headers: {
        "Content-Type": "application/json",
        Authorization: `Bearer ${apiKey}`,
      },
      body: JSON.stringify({
        model: VISION_MODEL,
        messages: [
          {
            role: "user",
            content: [
              { type: "image_url", image_url: { url: dataUrl } },
              {
                type: "text",
                text: "请详细描述这张图片的内容，包括画面主体、文字、图表、界面元素、颜色布局等所有可辨识的细节，用中文回答。",
              },
            ],
          },
        ],
      }),
      signal: controller.signal,
    });

    if (!resp.ok) {
      const errBody = await resp.text();
      throw new Error(`DashScope API ${resp.status}: ${errBody.slice(0, 300)}`);
    }

    const data = (await resp.json()) as {
      choices?: Array<{ message?: { content?: string } }>;
    };
    const description =
      data?.choices?.[0]?.message?.content || "(图片描述为空)";
    descriptionCache.set(dataUrl, description);
    return description;
  } catch (e) {
    if (controller.signal.aborted) {
      throw new Error(`DashScope 分析超时（>${DASHSCOPE_TIMEOUT_MS / 1000}s）`);
    }
    throw e;
  } finally {
    clearTimeout(timer);
  }
}

/** 转换单条消息里的图片 part 为文字描述（替换而非追加，模型只"读"文字） */
async function translateParts(parts: Array<Record<string, any>>): Promise<void> {
  for (let i = 0; i < parts.length; i++) {
    const part = parts[i];
    // 只处理"图片"类型的文件 part（mime 以 image/ 开头且为 data URL）
    if (
      part?.type === "file" &&
      part.mime?.startsWith("image/") &&
      part.url?.startsWith("data:")
    ) {
      try {
        const description = await analyzeImage(part.url);
        parts[i] = {
          id: part.id,
          sessionID: part.sessionID,
          messageID: part.messageID,
          type: "text",
          synthetic: true,
          text: `[图片预分析 - ${part.filename || "image"}] ${description}`,
        };
      } catch (e) {
        console.error("[vision-translator] 图片解析失败:", e);
        // 解析失败时降级为文本占位，绝不让原始图片卡住纯文本模型
        parts[i] = {
          id: part.id,
          sessionID: part.sessionID,
          messageID: part.messageID,
          type: "text",
          synthetic: true,
          text: `[图片解析失败：${(e as Error).message}]（图片已降级为占位文本）`,
        };
      }
    }
  }
}

export const VisionTranslatorPlugin: Plugin = async () => {
  return {
    /**
     * 在消息即将发送给 LLM 前转换图片 → 文字。
     * 相比 chat.message：此处转换不影响消息的入库/显示/发送流程，
     * 只会让"回复生成"稍等分析完成；即使超时也走降级路径，消息绝不丢失。
     */
    "experimental.chat.messages.transform": async (_input, output) => {
      for (const msg of output.messages) {
        if (msg?.parts && Array.isArray(msg.parts)) {
          await translateParts(msg.parts);
        }
      }
    },
  };
};
```

<p align="center">—— 完 ——</p>
