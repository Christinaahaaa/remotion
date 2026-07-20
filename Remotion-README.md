# Remotion 项目说明

Remotion 是一个使用 React 代码创建视频和动态图像的开源工具集。

它把 React 组件当作画面，把时间轴表示为帧，并允许开发者使用 JavaScript、TypeScript、CSS 和浏览器能力控制动画。视频可以在 Remotion Studio 中预览，也可以通过命令行、Node.js、服务器或云端批量渲染。

## Remotion 能做什么

- 使用 React 组件编写视频画面
- 根据帧数精确控制动画、转场和时间轴
- 在 Remotion Studio 中交互式预览 Composition
- 将 React 画面渲染为 MP4、WebM、GIF 或静态图片
- 使用数据批量生成结构一致、内容不同的视频
- 在网页中嵌入 Remotion Player
- 在 Node.js、AWS Lambda、Cloud Run 等环境中自动渲染
- 组合字幕、字体、音频、Three.js、Lottie、Canvas 和 WebCodecs 等能力

## 核心概念

### Composition

Composition 是一个可渲染的视频入口，主要定义：

- React 组件
- 唯一 ID
- 视频宽度和高度
- 帧率（fps）
- 总帧数
- 输入参数

### Frame

Remotion 使用当前帧决定画面状态。组件通过 `useCurrentFrame()` 获取帧数，再配合 `interpolate()`、`spring()` 等函数计算透明度、位置、缩放和其他动画属性。

### Sequence

`Sequence` 用于安排组件在时间轴中的出现时间和持续时间。复杂视频通常由多个 Sequence、音频、字幕和转场组合而成。

### Render

Remotion 会在浏览器环境中逐帧渲染 React 画面，再编码为最终媒体文件。渲染可以在本机执行，也可以部署到服务器或云端。

## 普通使用者：创建视频项目

如果目标是使用 Remotion 制作视频，请在仓库之外运行：

```bash
npx create-video@latest
```

按照终端提示选择模板并进入项目后，通常可以使用以下命令：

```bash
npm install
npm run dev
```

项目启动后，可以在 Remotion Studio 中选择 Composition、调整输入参数并逐帧预览。

具体命令以所选模板生成的 `package.json` 和 README 为准。

## 使用千问 API 生成视频内容

如果项目需要通过大模型生成视频脚本、分镜、字幕、标题或配色，统一使用千问 API。Remotion 只负责把结构化数据渲染成视频，不在 Composition 的逐帧执行过程中调用模型。

### 技术边界

- 大模型服务：千问
- 接口协议：阿里云百炼提供的 OpenAI 兼容接口
- 内容格式：JSON
- 视频渲染：Remotion
- API 调用位置：独立 Node.js 脚本或业务服务端
- API Key：只通过服务端环境变量读取，不进入浏览器代码、React props 或 Git 仓库

推荐的数据链路：

```text
用户主题或业务数据
        ↓
千问 API 生成结构化 JSON
        ↓
服务端校验并保存 JSON
        ↓
Remotion Composition 读取数据
        ↓
本地、服务器或云端渲染
        ↓
MP4 / WebM / GIF / 图片
```

> 不要在 `useCurrentFrame()`、React 组件渲染函数或每一帧中请求千问。Remotion 渲染会多次执行组件代码，这会造成重复计费、结果不一致和渲染失败。

### 环境变量

```bash
export DASHSCOPE_API_KEY="sk-your-api-key"
export DASHSCOPE_BASE_URL="https://YOUR_WORKSPACE_ID.cn-beijing.maas.aliyuncs.com/compatible-mode/v1"
export QWEN_MODEL="qwen-plus"
```

- `DASHSCOPE_API_KEY`：阿里云百炼 API Key，必填。
- `DASHSCOPE_BASE_URL`：对应账号、地域和工作空间的兼容接口地址，必填。
- `QWEN_MODEL`：千问模型名称，可根据账号可用模型调整。

### 建议的数据协议

千问应返回固定结构，业务代码必须在写入文件或触发渲染前完成校验：

```json
{
  "title": "视频标题",
  "subtitle": "视频副标题",
  "scenes": [
    {
      "text": "第一段画面文案",
      "durationInFrames": 90,
      "backgroundColor": "#111827"
    }
  ]
}
```

工程师可以根据具体视频模板扩展 `imageUrl`、`audioUrl`、`captions`、`transition`、`font` 等字段，但应先定义稳定的数据结构，再编写 Composition。

### 最小千问调用示例

以下脚本应放在独立的视频项目或服务端中，不应添加到 Remotion 核心包：

```js
const response = await fetch(
  `${process.env.DASHSCOPE_BASE_URL}/chat/completions`,
  {
    method: 'POST',
    headers: {
      Authorization: `Bearer ${process.env.DASHSCOPE_API_KEY}`,
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({
      model: process.env.QWEN_MODEL ?? 'qwen-plus',
      response_format: {type: 'json_object'},
      messages: [
        {
          role: 'system',
          content: '你是视频脚本生成器，只输出合法 JSON。',
        },
        {
          role: 'user',
          content: '生成一个包含标题和分镜数组的短视频脚本，使用 JSON 格式。',
        },
      ],
    }),
  },
);

if (!response.ok) {
  throw new Error(`Qwen request failed: ${response.status}`);
}

const payload = await response.json();
const videoData = JSON.parse(payload.choices[0].message.content);
```

启用 `response_format: {type: 'json_object'}` 时，提示词中也必须明确要求输出 JSON。拿到结果后仍需校验字段类型、数组数量、帧数、颜色和媒体 URL，不能直接信任模型输出。

### Remotion 接入方式

千问生成的数据可以通过以下任一方式传给 Remotion：

1. 生成本地 JSON 文件，由 Composition 导入。
2. 在调用 Remotion Node.js 渲染 API 时通过 `inputProps` 传入。
3. 将数据保存到数据库，由渲染任务读取后再传入 Composition。

无论采用哪种方式，同一次渲染都应固定输入数据。重新生成脚本和重新渲染视频应当是两个独立步骤，便于重试、审计和控制 API 成本。

## 本仓库：开发 Remotion 源码

本仓库使用 Bun Workspaces 和 Turborepo 管理多个包。仓库要求的包管理器版本记录在根目录 `package.json` 的 `packageManager` 字段中。

### 环境要求

- Git
- Bun 1.3.3
- 支持当前项目依赖的操作系统和开发工具

### 安装依赖

```bash
bun install
```

### 构建全部包

```bash
bun run build
```

也可以直接调用 Turborepo：

```bash
bunx turbo run make
```

### 启动开发用 Remotion Studio

首次启动前应先完成根目录构建：

```bash
bun run build
cd packages/example
bun run dev
```

默认开发地址为：

```text
http://localhost:3000
```

### 运行测试和代码检查

```bash
bun run test
bun run stylecheck
```

或者直接运行对应的 Turbo 任务：

```bash
bunx turbo run lint test
```

### 构建单个包

```bash
bunx turbo run make --filter='<package-name>'
```

例如：

```bash
bunx turbo run make --filter='remotion'
bunx turbo run make --filter='@remotion/player'
bunx turbo run make --filter='@remotion/cli'
```

### 渲染测试视频

在 `packages/example` 中执行：

```bash
cd packages/example
bunx remotion compositions
bunx remotion render <composition-id> --output ../../out/video.mp4
bunx remotion still <composition-id> --output ../../out/still.png
```

## 仓库结构

```text
remotion/
├── packages/                 # Remotion 的核心包、扩展、模板和示例
├── scripts/                  # 仓库维护和自动化脚本
├── .github/                  # GitHub Actions 和项目配置
├── .githooks/                # Git hooks
├── AGENTS.md                 # 编程代理和开发环境说明
├── CONTRIBUTING.md           # 贡献入口
├── LICENSE.md                # Remotion 许可证
├── package.json              # 工作区、脚本和依赖目录
├── turbo.json                # Turborepo 任务配置
└── bun.lock                  # Bun 锁定文件
```

## 主要包

| 目录 | 作用 |
| --- | --- |
| `packages/core` | `remotion` 核心包，包含 Composition、时间轴和动画 API |
| `packages/cli` | Remotion 命令行工具 |
| `packages/renderer` | 浏览器启动、逐帧渲染和媒体输出能力 |
| `packages/bundler` | 打包 Remotion 视频项目 |
| `packages/studio` | Remotion Studio 前端界面 |
| `packages/studio-server` | Studio 的本地服务端 |
| `packages/player` | 在 React 页面中嵌入和播放 Composition |
| `packages/lambda` | AWS Lambda 渲染能力 |
| `packages/serverless` | 通用服务端和无服务器渲染能力 |
| `packages/media` | 视频、音频和媒体处理组件 |
| `packages/media-parser` | 媒体文件解析能力 |
| `packages/webcodecs` | 基于 WebCodecs 的媒体处理能力 |
| `packages/transitions` | 视频转场组件 |
| `packages/effects` | 视觉效果组件 |
| `packages/captions` | 字幕处理能力 |
| `packages/fonts` | 字体加载能力 |
| `packages/three` | Three.js 集成 |
| `packages/lottie` | Lottie 动画集成 |
| `packages/docs` | 官方文档站点源码 |
| `packages/example` | 核心开发和测试使用的示例项目 |
| `packages/create-video` | `create-video` 项目脚手架 |
| `packages/template-*` | 不同用途的视频项目模板 |

仓库中还有平台相关的 compositor、云渲染客户端、媒体工具、测试工程和集成示例。修改前应先确认功能所属的包，避免在无关模块中重复实现。

## 常见开发流程

1. 确定需要修改的 package。
2. 为该 package 安装并构建依赖。
3. 在 `packages/example` 或对应示例中复现和验证行为。
4. 运行目标 package 的测试和 lint。
5. 提交前运行根目录构建与 `bun run stylecheck`。

提交 Pull Request 前，请先阅读 [贡献指南](https://www.remotion.dev/docs/contributing)。

## 相关链接

- [官方网站](https://www.remotion.dev/)
- [官方文档](https://www.remotion.dev/docs/)
- [API Reference](https://www.remotion.dev/api/)
- [项目模板](https://www.remotion.dev/templates)
- [GitHub Issues](https://github.com/remotion-dev/remotion/issues)
- [Discord 社区](https://www.remotion.dev/discord)
