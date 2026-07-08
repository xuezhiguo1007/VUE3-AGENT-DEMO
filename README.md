# Vue3 Agent Demo

一个模拟 AI Agent 对话流程的 Vue 3 演示项目，展示思考、工具调用与流式回复的完整交互体验。

## 功能特性

- **多步对话流程** - 模拟 Agent 的思考过程和工具调用
- **流式文本输出** - 打字机效果展示回复内容
- **可视化步骤** - 展示思考、工具调用等中间步骤
- **演示模式** - 一键播放预设的完整对话场景
- **实时交互** - 支持用户输入并获得模拟回复
- **可中断操作** - 支持中止正在进行的对话流程

## 技术栈

| 技术 | 版本 | 说明 |
|------|------|------|
| Vue | 3.5.34 | 前端框架 |
| Vite | 8.0.12 | 构建工具 |
| Composition API | - | 使用 `<script setup>` 语法 |

## 项目结构

```
vue3-agent-demo/
├── public/                 # 静态资源
├── src/
│   ├── assets/            # 图片等资源
│   ├── components/
│   │   └── ConversationDemo.vue  # 主对话组件
│   ├── App.vue            # 入口组件
│   ├── main.js            # 应用入口
│   └── style.css          # 全局样式
├── index.html             # HTML 入口
├── package.json           # 项目配置
└── vite.config.js         # Vite 配置
```

## 快速开始

### 安装依赖

```bash
npm install
```

### 启动开发服务器

```bash
npm run dev
```

访问 http://localhost:5173 查看应用。

### 构建生产版本

```bash
npm run build
```

### 预览生产版本

```bash
npm run preview
```

## 使用说明

1. **播放演示** - 点击顶部「播放演示」按钮，自动展示完整的 Agent 对话流程
2. **发送消息** - 在底部输入框输入内容，按 Enter 或点击「发送」按钮
3. **清空对话** - 点击「清空」按钮重置所有消息

## 演示场景

项目预设了以下对话场景：

- 查询杭州明天天气，并推荐适合的外套
- 切换城市查询上海天气

展示了 Agent 的：
- 用户意图分析
- 工具调用（天气查询、商品搜索）
- 流式回复输出

## 开发说明

本项目为演示用途，对话内容为预设脚本模拟。如需接入真实 AI API，可修改 `ConversationDemo.vue` 中的 `sendMessage` 函数。

## License

MIT