# Remotion 项目中文说明

Remotion 是一个使用 React 和 TypeScript 编程生成视频的工具集。React 组件负责画面，帧数负责时间轴，最终可渲染为 MP4、WebM、GIF 或静态图片。

本仓库是 Remotion 官方源码 monorepo，不是业务视频项目。业务开发应通过脚手架创建独立项目，除非需要修改 Remotion 本身，否则不要改动核心包。

## 主要能力

- 使用 React、CSS 和 TypeScript 编写视频画面
- 精确控制帧、动画、转场、字幕、音频和素材
- 在 Remotion Studio 中逐帧预览
- 通过命令行或 Node.js 自动渲染
- 支持本地、服务器和云端批量生成视频
- 提供 Player、Three.js、Lottie、字幕及媒体处理等扩展

## 创建业务视频项目

```bash
npx create-video@latest
cd <project-name>
npm install
npm run dev
```

视频入口称为 Composition，定义组件、尺寸、帧率和总帧数。动画组件通常使用：

- `useCurrentFrame()`：获取当前帧
- `interpolate()`：映射动画数值
- `spring()`：生成弹性动画
- `Sequence`：安排组件的出现时间

## 开发本仓库

本仓库使用 Bun 1.3.3、Bun Workspaces 和 Turborepo。

```bash
# 安装并构建
bun install
bun run build

# 启动开发示例
cd packages/example
bun run dev

# 检查和测试（在仓库根目录执行）
bun run test
bun run stylecheck
```

渲染测试 Composition：

```bash
cd packages/example
bunx remotion compositions
bunx remotion render <composition-id> --output ../../out/video.mp4
bunx remotion still <composition-id> --output ../../out/still.png
```

构建单个包：

```bash
bunx turbo run make --filter='<package-name>'
```

## 核心目录

| 目录 | 作用 |
| --- | --- |
| `packages/core` | Composition、时间轴和动画 API |
| `packages/cli` | 命令行工具 |
| `packages/renderer` | 浏览器启动、逐帧渲染和媒体输出 |
| `packages/studio` | Remotion Studio 界面 |
| `packages/player` | 在 React 页面中播放 Composition |
| `packages/media` | 视频和音频组件 |
| `packages/lambda` | AWS Lambda 渲染 |
| `packages/example` | 源码开发与测试项目 |
| `packages/create-video` | 视频项目脚手架 |
| `packages/template-*` | 官方项目模板 |

Remotion 发布版本记录在 `packages/core/src/version.ts`，根目录 `package.json` 的版本仅为工作区占位值。

## 千问 API 接入规范

需要通过大模型生成视频脚本、分镜、字幕、标题或配色时，统一调用千问。Remotion 只读取结构化结果并渲染视频。

```text
业务输入 → 千问 API → 校验 JSON → Remotion Composition → 渲染视频
```

### 基本要求

- 在独立 Node.js 脚本或服务端调用千问。
- API Key 只从服务端环境变量读取，不得提交到 Git。
- 千问必须返回 JSON，服务端校验后再交给 Remotion。
- 同一次渲染必须固定输入数据，保证结果可重现。
- 不得在 React 组件或 `useCurrentFrame()` 中调用 API，否则会重复请求和计费。

### 环境变量

```bash
export DASHSCOPE_API_KEY="sk-your-api-key"
export DASHSCOPE_BASE_URL="https://YOUR_WORKSPACE_ID.cn-beijing.maas.aliyuncs.com/compatible-mode/v1"
export QWEN_MODEL="qwen-plus"
```

接口地址、地域和模型名称以阿里云百炼工作空间配置为准。

### 最小调用示例

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
        {role: 'system', content: '只输出合法 JSON。'},
        {role: 'user', content: '用 JSON 生成包含标题和分镜的视频脚本。'},
      ],
    }),
  },
);

if (!response.ok) throw new Error(`Qwen request failed: ${response.status}`);

const payload = await response.json();
const videoData = JSON.parse(payload.choices[0].message.content);
```

使用 `json_object` 时，提示词中必须明确出现 JSON 要求。返回结果仍需校验字段类型、数组长度、帧数、颜色和媒体 URL。

### 建议数据结构

```json
{
  "title": "视频标题",
  "scenes": [
    {
      "text": "画面文案",
      "durationInFrames": 90,
      "backgroundColor": "#111827"
    }
  ]
}
```

数据可以保存为 JSON 文件，也可以通过 Remotion Node.js API 的 `inputProps` 传入 Composition。生成脚本和渲染视频应为两个独立步骤，便于重试、审计和控制成本。

## 许可证

Remotion 使用分级许可证。个人、非营利组织及员工不超过 3 人的营利组织通常可免费使用；超出免费范围的营利组织需要 Company License。实际使用前请以 [LICENSE.md](./LICENSE.md) 为准。

## 相关链接

- [官方文档](https://www.remotion.dev/docs/)
- [API Reference](https://www.remotion.dev/api/)
- [贡献指南](https://www.remotion.dev/docs/contributing)
