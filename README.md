<div align="center">
  <img src="assets/logo-readme.png" alt="Oh My Prompt - AI 提示词管理工具" style="max-width: 750px; width: 100%;">
  <h1>Oh My Prompt</h1>
  <h3>AI 提示词管理工具</h3>
  <p><strong>告别复制粘贴，一键插入你的提示词，无需离开创造界面</strong></p>
  
  [![License](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Chrome Extension](https://img.shields.io/badge/Chrome-Manifest%20V3-green.svg)]()
[![Made for Prompt](https://img.shields.io/badge/Made%20for-Prompt-purple.svg)]()

  🌐 [官方网站](https://oh-my-prompt.com/) | 📦 [下载安装](https://github.com/wk240/oh-my-prompt/releases) | 🇺🇸 [English](README_EN.md) | 🚗 [车辆管理系统榜单](https://github.com/wk240/awesome-vehicle-management) | 🚙 [车辆管理系统](https://qiguanche.com)
</div>

---

## 项目结构

本项目采用 **Monorepo** 架构：

```
packages/
├── extension/      # Chrome Extension（开源）
│   ├── src/        # Extension 源码
│   └── dist/       # 构建产物（Chrome 加载此目录）
│
├── shared/         # 共享类型定义（开源）
│   ├── types/      # TypeScript 类型
│   └── constants/  # 常量定义
│
└── web-app/        # Web App（私有，官网与云同步服务）
    ├── app/        # Next.js App Router
    ├── lib/        # API 客户端
    └── supabase/   # Supabase 配置
```

---

## ✨ 核心功能

**Oh My Prompt** 是一款专为 AI 设计平台打造的 Chrome 浏览器扩展，帮助你高效管理和使用提示词。

| 功能 | 说明 |
|------|------|
| 🚀 **一键插入** | 保存常用提示词，下次创作时一键插入，无需重复输入 |
| 🖼️ **图片转提示词** | 鼠标悬停任意图片，一键生成双语提示词（需配置 API） |
| 🤖 **Prompt Agent** | 输入简单想法，AI 自动生成结构完整的专业提示词（需配置 API） |
| 🎨 **资源库** | 内置优质提示词模板，一键使用社区精选内容 |

**一句话说清楚：** 把常用提示词保存起来，下次创作时一键插入，不再重复输入相同内容。

---

## 🎯 解决什么痛点？

每次在 Lovart、星流、ChatGPT 等设计平台创作时，你是否也在重复输入：
- ✅ 自己积累的优质提示词模板
- ✅ 常用的风格描述：「扁平化设计」「赛博朋克风格」「水彩插画」
- ✅ 技术参数：「高清渲染」「4K分辨率」「光影细腻」
- ✅ 网络收集的提示词模板

**一次输入，下次还得再输。Oh My Prompt 解决这个问题。**

---

## 📦 安装指南

### 方式一：下载安装包（推荐）

适合大多数用户，无需编译：

1. 前往 [Releases 页面](https://github.com/wk240/oh-my-prompt/releases) 下载最新版本的 `oh-my-prompt-v*.zip`
2. 解压到任意文件夹
3. 打开 Chrome，访问 `chrome://extensions/`
4. 启用「开发者模式」
5. 点击「加载已解压的扩展程序」，选择解压后的文件夹

### 方式二：从源码构建

适合开发者或需要自定义的用户：

**前提条件**：Node.js 18+ 环境

```bash
# 克隆项目
git clone https://github.com/wk240/oh-my-prompt.git
cd oh-my-prompt

# 安装依赖
npm install

# 构建（生产版本）
npm run build

# 或开发模式（带热重载，推荐开发时使用）
npm run dev
```

**在 Chrome 加载扩展**：

1. 打开 `chrome://extensions/`
2. 启用「开发者模式」（右上角开关）
3. 点击「加载已解压的扩展程序」
4. **选择 `packages/extension/dist` 目录**（不是项目根目录或 src 目录）

> ⚠️ **注意**：开发模式下每次修改源码会自动重新构建，但需要在 Chrome 扩展页面点击刷新按钮才能生效。

---

## 📖 使用教程

### 1、一键插入提示词

在 Lovart 的输入框旁，你会看到一个闪电图标按钮：

1. 点击闪电图标 → 打开下拉菜单
2. 选择提示词 → 内容自动插入输入框
3. 继续选择 → 可组合多个提示词

![示例：下拉菜单插入提示词](assets/eg1.gif)

### 2、图片转提示词

**前提条件**：需先配置 Vision API（在扩展设置页面配置 API Key）

使用步骤：
1. 在任意网站浏览图片
2. 鼠标悬停在图片上，出现 ✨ 按钮
3. 点击按钮 → 弹出分析窗口
4. 等待分析完成 → 查看生成的提示词
5. 可切换语言（中/EN）和格式（自然语言/JSON）
6. 提示词自动保存到临时库

![示例：图片转提示词功能演示](assets/eg2.gif)

### 3、Prompt Agent

**前提条件**：需先配置 Vision API（在扩展设置页面配置 API Key）

Prompt Agent 可以根据你的简单描述，结合专业模板，自动生成结构完整、细节丰富的提示词。

使用步骤：
1. 点击浏览器工具栏的扩展图标，打开侧边栏面板
2. 进入「Agent」视图，选择电商套图、海报、插画、Logo、UI、3D 等模板分类
3. 输入你想生成的内容描述，也可以上传参考图
4. 点击生成 → 获得可直接复制或保存的专业提示词

![示例：Prompt Agent 使用演示](assets/eg3.gif)

### 4、资源库

点击浏览器工具栏的扩展图标，打开侧边栏面板。在左侧导航点击「资源库」：

- **内置模板**：社区精选优质提示词，按用途分类
- **一键使用**：选中模板即可保存到个人提示词库，创作时快速调用
- **收藏管理**：喜欢的模板可添加到个人收藏

---

## ❓ 常见问题 FAQ

<details>
<summary><strong>Q: 安装时出现 "Invalid script mime type" 错误怎么办？</strong></summary>

这个错误说明选择了错误的目录。请按以下步骤重新安装：

1. 移除当前扩展
2. 确认选择的是 **`packages/extension/dist` 目录**（不是项目根目录、src 目录或整个 packages 目录）
3. 重新加载扩展

![安装错误示例](assets/qa1.png)
</details>

<details>
<summary><strong>Q: 为什么在其他网站看不到闪电图标？</strong></summary>

扩展在 Lovart、ChatGPT、Claude.ai、Gemini、LibLib、即梦、Kimi、星流等支持的平台上激活。如在其他网站看不到图标，说明该平台暂未支持，后续版本可能添加。
</details>

<details>
<summary><strong>Q: 如何备份我的提示词？</strong></summary>

有两种方式：
- **本地同步**：开启同步功能，自动备份到本地文件夹，保留历史版本
- **导入导出**：管理界面点击导出图标，下载 JSON 文件
</details>

<details>
<summary><strong>Q: 提示词插入后平台没反应？</strong></summary>

确保输入框处于聚焦状态。如有问题，可手动输入几个字符后再插入。
</details>

<details>
<summary><strong>Q: 资源库的内容从哪里来？</strong></summary>

来自社区贡献者分享的优质提示词，每条都标注了原作者信息。
</details>

<details>
<summary><strong>Q: 如何更新扩展？</strong></summary>

更新步骤如下：

1. 扩展会自动检测新版本并提示，或点击管理界面的「检查更新」按钮
2. 点击提示，前往 Releases 页面下载新版本安装包
3. 解压后，在 `chrome://extensions/` 点击扩展的「重新加载」按钮
</details>

<details>
<summary><strong>Q: 图片转提示词功能如何配置 API？</strong></summary>

需要配置 Vision API 才能使用此功能：
1. 点击扩展图标打开侧边栏面板，点击右上角设置图标，选择「AI识图」标签页
2. 填写 API Base URL、API Key、模型名称
3. 选择 API 格式（OpenAI 格式或 Anthropic 格式）
4. 保存配置后即可使用

支持的服务：Claude API、OpenAI GPT-4V、或其他兼容服务。
</details>

<details>
<summary><strong>Q: 为什么有些图片看不到转提示词按钮？</strong></summary>

按钮只在满足以下条件时显示：
- 图片尺寸至少 100×100 像素
- 图片有有效的 URL（非 data URL）
- Vision 功能在设置中已启用（默认启用）
</details>

---

## 👤 作者

**Neo**（取自《黑客帝国》主角）—— Lovart AI 用户，为提升创作效率而开发。

社交媒体「Neo与AI」：公众号、小红书、抖音 | [GitHub](https://github.com/wk240)

---

## 📄 许可证

[MIT License](LICENSE) - Extension 和 Shared 包开源

---

## 💬 交流群

如果你需要进群，可以扫下方二维码：

<img src="assets/group-qrcode.jpg" alt="微信群二维码" width="200">

---

## 🤝 贡献

欢迎提交 Issue 和 Pull Request！

如果这个项目对你有帮助，请给个 ⭐ Star 支持一下！
