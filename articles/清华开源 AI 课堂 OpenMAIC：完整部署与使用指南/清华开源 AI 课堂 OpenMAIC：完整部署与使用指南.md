---
title: "清华开源 AI 课堂 OpenMAIC：完整部署与使用指南"
author: "夏风的屋"
date: "2026-03-30 09:05"
source: "https://mp.weixin.qq.com/s/5N09-UcoHm_4iyXN-B_1sw"
---

# 清华开源 AI 课堂 OpenMAIC：完整部署与使用指南

> 公众号: 夏风的屋
> 发布时间: 2026-03-30 09:05
> 原文链接: https://mp.weixin.qq.com/s/5N09-UcoHm_4iyXN-B_1sw

---
输入一个主题，30 秒生成一节有 AI 教师、AI 同学、互动测验的完整课堂。
本文从零讲清楚怎么把它跑起来。

### 目录

1. OpenMAIC 是什么，能做什么
2. 部署前准备：环境和 API Key
3. 三种部署方式详解（Vercel / 本地 / Docker）
4. 环境变量完整配置说明
5. 功能使用指南
6. 常见问题与解决方法

---

## 一、OpenMAIC 是什么，能做什么

OpenMAIC 是清华大学开源的多智能体互动课堂平台（Open Multi-Agent Interactive Classroom）。研究成果已发表于《计算机科学技术学报》（JCST 2026），经过 700 多名真实学生两年课堂验证。

![Image](images/img_001.png)

它能做的核心几件事：

- **主题生成课程**：输入任意学习主题（"教我读懂财务报表"、"Python 基础"），自动生成带语音讲解的完整课程
- **PDF 转课程**：上传论文或文档，AI 直接拆解讲解，可随时打断提问
- **互动课堂**：AI 教师 + AI 同学共同参与，支持课堂讨论、圆桌辩论、自由问答三种模式
- **HTML 模拟实验**：物理模拟、算法可视化，直接在浏览器里运行，不需安装额外软件
- **导出 PPT**：课程内容可导出为 PowerPoint 或 HTML 文件
- **飞书 / Slack 集成**：通过 OpenClaw 接入，在聊天工具里直接生成课程

项目地址：**github.com/THU-MAIC/OpenMAIC**　协议：AGPL-3.0

---

## 二、部署前准备

### 2.1 环境要求

| 依赖 | 版本要求 | 说明 |
| --- | --- | --- |
| Node.js | ≥ 20 | 建议用 LTS 版本 |
| pnpm | ≥ 10 | 包管理器，需单独安装 |
| Git | 任意版本 | 用于克隆仓库 |
| LLM API Key | — | 至少配置一个，见下方清单 |

如果还没装 pnpm，先跑一下：

```
npm install -g pnpm
```

### 2.2 获取 API Key

OpenMAIC 本身免费，生成课程时会调用大语言模型的 API，费用由你自己承担。以下是支持的模型和推荐选择：

| 模型 | 环境变量名 | 费用参考 | 推荐程度 |
| --- | --- | --- | --- |
| DeepSeek | `DEEPSEEK_API_KEY` | 轻度使用约 ¥5/月 | 国内推荐 |
| 通义千问（阿里） | `DASHSCOPE_API_KEY` | 有免费额度 | 国内推荐 |
| 智谱 GLM | `GLM_API_KEY` | 有免费额度 | 国内可用 |
| 豆包（字节） | `DOUBAO_API_KEY` | 有免费额度 | 国内可用 |
| Kimi（Moonshot） | `KIMI_API_KEY` | 有免费额度 | 国内可用 |
| OpenAI GPT-4o | `OPENAI_API_KEY` | 较高 | 需翻墙 |
| Anthropic Claude | `ANTHROPIC_API_KEY` | 较高 | 需翻墙 |
| Google Gemini | `GOOGLE_GENERATIVE_AI_API_KEY` | 有免费额度 | 需翻墙 |

**国内用户推荐：** 先用 DeepSeek，去 platform.deepseek.com 注册，充值 ¥10 够用很久。通义千问新用户有免费 token，也可以先拿来测试。

---

## 三、三种部署方式

方式一：Vercel 部署 推荐

方式二：本地运行

方式三：Docker

### 方式一：一键部署到 Vercel（最快，约 2 分钟）

适合不想折腾本地环境、只想快速用起来的人。Vercel 免费套餐对个人使用完全够用。

1、打开 GitHub 仓库：`github.com/THU-MAIC/OpenMAIC`

点击页面上的 **"Deploy with Vercel"** 按钮（README 里有，蓝色按钮）。

2、用 GitHub 账号登录 Vercel，它会自动 fork 仓库并开始配置。

3、在 Vercel 的 **Environment Variables** 页面，添加你的 API Key，比如：

```
DEEPSEEK_API_KEY=sk-xxxxxxxxxxxxxxxx
```

4、点击 Deploy，等待 1-2 分钟构建完成。Vercel 会给你一个访问链接，类似 `openmaic-xxx.vercel.app`，直接打开就能用。

**注意：** Vercel 免费版有函数执行时间限制（60 秒），生成较复杂的课程可能超时。如果遇到这个问题，换本地部署或 Docker 部署。

### 方式二：本地运行（最灵活）

适合开发调试、或者不想把 API Key 放在第三方平台的用户。

1、克隆仓库到本地：

```
git clone https://github.com/THU-MAIC/OpenMAIC.git
cd OpenMAIC
```

2、安装依赖（首次运行需要几分钟，国内可能需要切换 npm 镜像）：

```
# 如果下载慢，先设置镜像源
pnpm config set registry https://registry.npmmirror.com

pnpm install
```

3、复制环境变量模板，然后编辑它：

```
# Windows PowerShell
cp .env.example .env.local

# 用记事本或 VS Code 打开 .env.local
notepad .env.local
```

在文件里找到对应的 Key 名称，填入你的 API Key：

```
DEEPSEEK_API_KEY=sk-你的密钥填在这里
```

4、启动开发服务器：

```
pnpm dev
```

看到 `Local: http://localhost:3000` 就说明启动成功了。打开浏览器访问 `http://localhost:3000` 即可。

**如果要正式使用（而不是开发）：** 用 `pnpm build && pnpm start` 代替 `pnpm dev`，性能更好，稳定性更高。

### 方式三：Docker 部署（适合服务器）

适合部署在 VPS 或 NAS 上，长期稳定运行。

1、先克隆仓库（同方式二第 1 步），然后创建 `.env.local` 文件并填入 API Key（同方式二第 3 步）。

2、构建 Docker 镜像：

```
docker build -t openmaic .
```

首次构建需要几分钟，会下载 Node.js 基础镜像并编译项目。

3、运行容器：

```
docker run -d \
  -p 3000:3000 \
  --env-file .env.local \
  --name openmaic \
  openmaic
```

加了 `-d` 参数后台运行，服务器重启后需要手动 `docker start openmaic`，或者加上 `--restart unless-stopped` 参数让它自动重启。

---

## 四、环境变量完整说明

以下是 `.env.local` 里所有可配置的变量，**必填项**只需要选一个 LLM，其余都是可选的增强功能。

### 4.1 大语言模型（必须至少配一个）

```
# 以下选一个或多个填入，OpenMAIC 会自动识别
DEEPSEEK_API_KEY=sk-...
DASHSCOPE_API_KEY=sk-...# 通义千问
GLM_API_KEY=...# 智谱 GLM
DOUBAO_API_KEY=...# 豆包
KIMI_API_KEY=...# Kimi
MINIMAX_API_KEY=...
SILICONFLOW_API_KEY=...
OPENAI_API_KEY=sk-...
ANTHROPIC_API_KEY=sk-ant-...
GOOGLE_GENERATIVE_AI_API_KEY=AIza...
```

### 4.2 语音功能（可选）

配置后 AI 教师可以开口说话，课堂体验更真实。

```
# 语音合成（TTS）— 支持 OpenAI / Azure / 通义千问 / 智谱
OPENAI_API_KEY=sk-...# 如果用 OpenAI TTS
# 或者直接用上面配的通义/GLM，它们自带 TTS 支持

# 语音识别（ASR）— 支持 OpenAI Whisper / 通义千问
# 配置后可以用麦克风直接向 AI 教师提问
```

### 4.3 网络搜索（可选）

开启后生成课程时可以联网检索最新资料，适合时效性强的内容。

```
TAVILY_API_KEY=tvly-...# 去 tavily.com 免费注册获取
```

### 4.4 图片生成（可选）

```
# 课程内可以自动生成配图
# 支持 SeedDream、通义万相、Nano Banana
```

### 4.5 PDF 增强解析（可选）

```
# 配置 MinerU 后，可以处理扫描版 PDF、复杂排版（含公式、图表）
# 普通 PDF 不需要配，内置解析器即可
```

---

## 五、功能使用指南

### 5.1 生成第一节课

打开 `localhost:3000` 后，界面很简单，主要操作就是在输入框里描述你想学的内容。

输入方式有两种：

- **直接描述主题**：比如「30 分钟教我看懂 Python 列表推导式」「给我讲一下什么是 RAG 检索增强生成」
- **上传文档**：支持 PDF，上传后系统自动解析并生成课程，适合消化论文、报告、合同等长文档

**描述越具体，生成质量越高。** 与其说"教我 Python"，不如说"用类比的方式帮我理解 Python 装饰器，我有一年 JavaScript 基础"。

### 5.2 三种课堂互动模式

| 模式 | 适用场景 | 参与方式 |
| --- | --- | --- |
| **课堂讨论** | 学习新知识，需要 AI 教师讲解 | 可以随时举手打断，提问或要求换个角度解释 |
| **圆桌辩论** | 理解有争议的话题，形成多元视角 | 多个 AI 同学分别持不同立场辩论，你可以加入任意一方 |
| **自由问答** | 快速查阅、深入某个细节 | 直接和 AI 教师一对一对话 |

### 5.3 互动测验

课程中 AI 会自动生成测验题，回答后给出详细的对错解析，不是简单的"正确/错误"，而是解释为什么。

测验结束后可以看到学习报告，显示哪些知识点掌握了，哪些还需要复习。

### 5.4 HTML 模拟实验

这个功能对理工科内容特别有用。当课程涉及需要动手验证的内容（比如排序算法、物理定律、数学函数图像），AI 会直接生成一个可以交互的实验，在浏览器里运行，不用安装任何东西。

### 5.5 导出课程

课程生成后可以导出为两种格式：

- **PowerPoint (.pptx)**：直接下载，可在 PPT 里编辑，适合二次加工或分享给别人
- **HTML 页面**：独立的网页文件，保留全部互动功能，发给别人可以直接在浏览器打开

### 5.6 集成到飞书 / Slack

通过 OpenClaw 集成，可以在聊天工具里直接触发课程生成。配置方法参考官方文档的 Integrations 章节（github.com/THU-MAIC/OpenMAIC/wiki）。

---

## 六、常见问题

#### Q：安装依赖时报错 / 速度极慢

大概率是 npm 源的问题。运行以下命令切换到国内镜像：

```
pnpm config set registry https://registry.npmmirror.com
```

#### Q：启动后报 API Key 错误

检查 `.env.local` 文件：

- 变量名拼写是否正确（区分大小写）
- Key 值前后有没有多余的空格或引号
- 文件是否保存了（有些编辑器自动保存不稳定）

配置改动后需要重启服务（`Ctrl+C` 停止，再 `pnpm dev` 重启）。

#### Q：课程生成到一半卡住了

可能是 API 响应超时。几个方向排查：

- 换一个响应更快的模型（DeepSeek 速度通常很快）
- Vercel 部署的话，尝试换本地部署，Vercel 免费版有 60 秒执行时间限制
- 检查网络，调用海外 API（OpenAI、Claude）时确认代理设置正常

#### Q：PDF 上传后提取效果不好（公式识别错误、图表内容丢失）

内置解析器对普通文字 PDF 效果良好，但对扫描版 PDF 和含大量公式/图表的文档支持有限。可以配置 MinerU（环境变量 `MINERU_API_KEY`）来增强解析能力，官方文档里有配置步骤。

#### Q：Node.js 版本不对怎么升级

```
# 推荐用 nvm 管理 Node 版本

# Windows 下载 nvm-windows：github.com/coreybutler/nvm-windows
# 安装后运行：
nvm install 20
nvm use 20
```

#### Q：如何更新到最新版

```
# 本地部署：
git pull origin main
pnpm install
pnpm build

# Vercel 部署：
# 在 GitHub 上 sync fork，Vercel 会自动重新部署
```

---

本文信息基于 OpenMAIC 2026 年 3 月版本。如有更新请以官方 GitHub README 为准。
项目地址：github.com/THU-MAIC/OpenMAIC


