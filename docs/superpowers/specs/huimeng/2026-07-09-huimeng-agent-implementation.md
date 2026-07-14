# 绘梦 Agent 实施方案

**日期：** 2026-07-09  
**角色：** 产品图片生成工程师  
**技术栈：** Hermes Agent (Profile) + 飞书独立机器人 + 即梦 CLI (Dreamina)  
**状态：** 待确认

---

## 一、角色概览

### 1.1 角色定位

**绘梦** 是电商作战室中的产品图片生成工程师，负责将任何品类的产品概念转化为高质量的视觉内容。核心能力通用，通过按品类积累提示词库来保持专业性。

**设计原则：通用能力 + 品类专属提示词库**
- 图像生成能力本身是品类无关的（文生图、图生图、风格转换）
- 专业性通过每个品类的提示词库沉淀来体现
- 新品类进入时，先用通用模板生成，跑通后沉淀专属模板
- 当前深耕品类：仿真挂件；后续可按需扩展到其他品类

### 1.2 飞书机器人简介

> 📌 以下内容直接用于飞书开放平台创建应用时的「应用描述」字段。

---

**🎨 绘梦 | 产品图片生成工程师**

你好，我是绘梦，电商作战室的产品图片生成工程师。

**【我的职责】**

- 根据产品描述和风格要求，使用即梦 AI 生成高质量产品图片
- 支持任何品类的产品：挂件、服饰、美妆、家居、食品等
- 提供多种风格方案（可爱风、生活场景、特写展示、商务风等）
- 接收供应商原图或参考图，进行风格转换和场景优化
- 将生成的图片直接发送到群聊，供团队选用
- 按品类积累提示词库，持续优化生成效果

**【如何使用我】**

在群里 @我 并描述你的需求即可，例如：

- `生成小汉堡钥匙扣的可爱风产品图`
- `为这条连衣裙生成 3 种不同风格的图片`
- `把这张护肤品图转换成ins风场景` [附图]
- `生成一组家居好物的北欧风产品图`

**【输出规范】**

- 默认提供 3 种风格方案，可指定
- 图片分辨率默认 2K，可指定 4K
- 宽高比默认 1:1（适合小红书），可指定其他比例
- 每张图附带使用的提示词，便于复盘优化
- 首次生成新品类时，会先确认产品特征和风格偏好

**【注意事项】**

- 生成图片会消耗即梦积分，请注意控制频率
- 异步生成任务通常需要 10-30 秒，请耐心等待
- 如果对生成效果不满意，可以让我调整提示词重新生成
- 新品类首次生成时，建议先出试片确认方向，再生成高清成片

---

### 1.3 人格设定（SOUL.md）

> 📌 以下内容写入 Hermes Profile 的 `SOUL.md` 文件。

```
你是绘梦，电商作战室的产品图片生成工程师。

你的核心能力是通过即梦 AI 将任何品类的产品概念转化为高质量的视觉内容。
你对构图、色彩、光影有敏锐的感知，熟悉电商产品摄影的美学标准。
你能驾驭各种品类——挂件、服饰、美妆、家居、食品、3C 配件等，
并根据品类特征和目标受众选择最合适的视觉表达方式。

工作原则：
1. 每次生成前，先确认产品特征、品类属性、目标受众和风格偏好
2. 默认提供 3 种风格方案，除非用户另有要求
3. 生成图片后，记录提示词到 prompt-templates/prompt-log.md，按品类归档，便于后续优化
4. 首次遇到新品类时，先用通用模板生成试片，跑通后沉淀该品类的专属提示词
5. 生成失败时，分析原因并给出调整建议
6. 始终保持专业、高效、有条理的沟通风格

品类提示词库管理：
- 提示词库按品类组织，位于 ~/.hermes/profiles/huimeng/prompt-templates/ 目录
- 生成记录位于 ~/.hermes/profiles/huimeng/prompt-templates/prompt-log.md
- 当前已深耕品类：仿真挂件（钥匙扣、包挂、冰箱贴）
- 新品类首次生成后，将成功的提示词沉淀到对应品类目录
- 通用模板作为兜底，确保任何品类都能生成

内容平台：小红书电商
视觉风格：根据具体品类和目标受众灵活调整

回复要简洁专业，重点突出图片本身。避免过度卖萌或冗长解释。
```

---

## 二、架构设计

### 2.1 整体架构

```
┌─────────────────────────────────────────────────────┐
│           飞书「电商作战室」群                         │
│                                                     │
│   @绘梦 (独立机器人)                                  │
└─────────────────────────────────────────────────────┘
                       ↓ WebSocket
┌─────────────────────────────────────────────────────┐
│         Hermes Profile: huimeng                      │
│         ~/.hermes/profiles/huimeng/                  │
│                                                     │
│   ┌──────────────┐  ┌────────────────────────────┐  │
│   │ SOUL.md      │  │ Skills                     │  │
│   │ (绘梦人格)    │  │ ├── dreamina-cli           │  │
│   │              │  │ └── product-prompts        │  │
│   │              │  │     (通用模板 + 风格库)      │  │
│   ├──────────────┤  └────────────────────────────┘  │
│   │ Tools:       │                                   │
│   │ - terminal   │  ← 调用 dreamina CLI             │
│   │ - file       │  ← 读写提示词记录/品类库          │
│   │ - vision     │  ← 分析用户上传的产品原图          │
│   └──────────────┘                                   │
│                                                     │
│   ┌─────────────────────────────────────────────┐   │
│   │ prompt-templates/  提示词库（按品类组织）      │   │
│   │ ├── prompt-log.md        生成记录            │   │
│   │ ├── pendants/            仿真挂件 🟢 深耕中  │   │
│   │ ├── clothing/            服饰鞋包 ⚪ 待扩展  │   │
│   │ ├── beauty/              美妆护肤 ⚪ 待扩展  │   │
│   │ └── home/                家居好物 ⚪ 待扩展  │   │
│   └─────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────┘
                       ↓
┌─────────────────────────────────────────────────────┐
│   即梦 CLI (Dreamina)                                │
│   ~/.local/bin/dreamina                              │
│                                                     │
│   text2image  →  文生图（任何品类）                    │
│   image2image →  图生图（风格转换）                    │
│   query_result → 查询异步任务结果                      │
│   list_task    → 查看任务历史                         │
└─────────────────────────────────────────────────────┘
```

### 2.2 为什么使用 Hermes Profile（多实例）

Hermes 支持 `profile` 机制，每个 profile 是一个**完全独立**的 Hermes 实例：

- 独立的 `config.yaml`、`.env`、`SOUL.md`、`sessions`、`logs`
- 独立的飞书应用凭证
- 独立的 gateway 进程
- 位于 `~/.hermes/profiles/<name>/`

**优势：**
- 绘梦与未来其他 Agent 完全隔离，互不干扰
- 可以单独升级、重启、调试
- 每个 Agent 有自己的记忆、技能、会话
- 统一管理入口：`hermes -p <profile_name> <command>`

---

## 三、实施步骤

### 阶段一：创建 Hermes Profile

#### 步骤 1：创建绘梦专用 Profile

```bash
# 创建新 profile（从默认 profile 克隆基础配置）
hermes profile create huimeng --clone

# 验证创建成功
hermes profile list

# 查看 profile 详情
hermes profile show huimeng

# 确认目录结构
ls ~/.hermes/profiles/huimeng/
```

**预期输出：**
- `hermes profile list` 显示 `huimeng` 在列表中
- 目录包含：config.yaml, .env, SOUL.md, skills/, sessions/, logs/ 等

**克隆说明：**
- `--clone` 会复制 config.yaml、.env、SOUL.md 和部分记忆文件
- 后续步骤会逐一替换这些文件的内容

---

#### 步骤 2：配置绘梦的 SOUL.md

**文件：** `~/.hermes/profiles/huimeng/SOUL.md`

**内容：** 直接使用上方「1.3 人格设定」中的内容。

---

#### 步骤 3：修改绘梦的 config.yaml

**文件：** `~/.hermes/profiles/huimeng/config.yaml`

**需要修改以下部分：**

**3a. 模型配置（保持与默认一致）：**
```yaml
model:
  default: qwen3.7-plus
  provider: custom
  base_url: https://coding.dashscope.aliyuncs.com/apps/anthropic
  api_mode: anthropic_messages
  api_key: ${ANTHROPIC_AUTH_TOKEN}
```

**3b. 从 `disabled_toolsets` 中移除 `vision`：**
> 移除原因：图生图场景需要分析用户上传的产品原图，需要视觉能力。

修改前：
```yaml
agent:
  disabled_toolsets:
    - browser
    - clarify
    ...
    - video_gen
    - vision      # ← 删除此行
    - web
    ...
```

修改后：
```yaml
agent:
  disabled_toolsets:
    - browser
    - clarify
    ...
    - video_gen
    # - vision   # 已移除，图生图需要视觉分析
    - web
    ...
```

**3c. 在文件末尾添加/修改 `platform_toolsets`：**

```yaml
platform_toolsets:
  feishu:
    - hermes-feishu
    - terminal
    - file
    - vision
```

**说明：**
- `hermes-feishu`：飞书基础消息能力
- `terminal`：用于调用 `dreamina` CLI 命令
- `file`：用于读写提示词记录和产品数据
- `vision`：用于分析用户上传的图片（图生图场景）

**验证：**
```bash
hermes -p huimeng tools
# 确认 terminal、file、vision 在可用工具列表中
```

---

#### 步骤 4：配置绘梦的 .env 文件

**文件：** `~/.hermes/profiles/huimeng/.env`

**需要修改飞书相关配置**（步骤 5 中创建新飞书应用后获取）：

```bash
# ===== 飞书应用配置（绘梦专属） =====
FEISHU_APP_ID=<步骤5创建的新应用 App ID>
FEISHU_APP_SECRET=<步骤5创建的新应用 App Secret>
FEISHU_DOMAIN=feishu
FEISHU_CONNECTION_MODE=websocket
FEISHU_GROUP_POLICY=allowlist
FEISHU_ALLOWED_USERS=<你的飞书 Open ID>

# ===== LLM API（从默认 profile 复制） =====
ANTHROPIC_AUTH_TOKEN=<你的 token，与默认 profile 相同>
```

> ⚠️ **重要：** 绘梦需要独立的飞书应用，所以 `FEISHU_APP_ID` 和 `FEISHU_APP_SECRET` 与默认 profile 不同。

---

### 阶段二：创建飞书应用

#### 步骤 5：在飞书开放平台创建「绘梦」应用

**5a. 创建应用**

1. 打开 https://open.feishu.cn/app
2. 点击「创建企业自建应用」
3. 填写信息：
   - **应用名称：** `绘梦`
   - **应用描述：** 使用上方「1.2 飞书机器人简介」中的完整内容
   - **应用图标：** 选择一个调色板或画笔类图标 🎨
4. 点击「创建」

**5b. 启用机器人能力**

1. 进入应用详情页
2. 左侧菜单 → 「应用能力」→「机器人」
3. 点击「启用」

**5c. 配置权限**

进入「权限管理」，逐一添加以下权限：

| 权限标识 | 用途 | 是否必须 |
|---------|------|---------|
| `im:message` | 接收和读取消息 | ✅ 必须 |
| `im:message:send_as_bot` | 以机器人身份发送消息 | ✅ 必须 |
| `im:resource` | 访问图片、文件和音频（发送图片必须） | ✅ 必须 |
| `im:chat` | 访问群聊元数据 | ✅ 必须 |
| `im:chat:readonly` | 读取群列表和成员信息 | ✅ 必须 |
| `im:message.reactions:readonly` | 接收表情回应事件 | ⚪ 推荐 |
| `contact:user.id:readonly` | 解析用户 ID 用于白名单匹配 | ⚪ 推荐 |

**5d. 配置事件订阅**

1. 进入「事件与回调」
2. 连接方式：选择 **「长连接（WebSocket）」**
3. 点击「添加事件」
4. 搜索并添加：`im.message.receive_v1`（接收消息）

**5e. 发布应用**

1. 进入「版本管理与发布」
2. 点击「创建版本」
3. 填写版本号和更新说明
4. 点击「发布」
5. 等待管理员审批（个人版飞书通常即时生效）

**5f. 获取凭证**

1. 进入「凭证与基础信息」
2. 记录 `App ID`（格式如 `cli_xxxxxx`）
3. 记录 `App Secret`
4. 将这两个值填入步骤 4 的 `.env` 文件

**5g. 获取你的 Open ID**

1. 进入飞书开发者后台的「API 调试台」
2. 选择接口：`GET /open-apis/contact/v3/users/me`
3. 点击「发送请求」（使用 user_access_token）
4. 记录返回的 `open_id`（格式如 `ou_xxxxxxxxx`）
5. 将值填入 `.env` 的 `FEISHU_ALLOWED_USERS`

> **替代方法：** 在默认 profile 的 `.env` 中查看已有的 `FEISHU_ALLOWED_USERS`，直接复制即可。

**验证：**
- 在手机飞书中搜索「绘梦」，能找到这个机器人
- 机器人的描述信息与你填写的一致

---

#### 步骤 6：将「绘梦」添加到 HiveCommerce 群

1. 打开飞书「电商作战室」群
2. 点击群名称 → 进入群设置
3. 找到「机器人」→「添加机器人」
4. 搜索「绘梦」
5. 点击「添加」
6. 确认绘梦出现在群成员列表中

**验证：**
- 群成员列表中能看到「绘梦」
- 在群中输入 `@` 时，下拉列表出现「绘梦」

---

### 阶段三：安装 Skills

#### 步骤 7：安装即梦 CLI Skill

```bash
# 创建 skill 目录
mkdir -p ~/.hermes/profiles/huimeng/skills/dreamina-cli

# 复制即梦官方 SKILL.md
cp ~/.dreamina_cli/dreamina/SKILL.md \
   ~/.hermes/profiles/huimeng/skills/dreamina-cli/SKILL.md

# 验证
cat ~/.hermes/profiles/huimeng/skills/dreamina-cli/SKILL.md | head -5
```

**预期输出：**
```
---
name: dreamina-cli
description: Use when an agent needs Dreamina（即梦）...
---
```

---

#### 步骤 8：创建产品提示词模板 Skill

**创建文件：** `~/.hermes/profiles/huimeng/skills/product-prompts/SKILL.md`

**内容：**

```markdown
---
name: product-prompts
description: 通用产品图片提示词模板库（按品类组织，持续积累）
version: 1.0.0
---

# 产品图片提示词模板库

## 架构说明

```
提示词库
├── 通用模板（任何品类都能用）
│   ├── 风格模板（可爱风、生活场景、特写展示、商务风、ins风...）
│   └── 通用负面提示词
└── 品类专属库（按品类沉淀，持续优化）
    ├── 📁 仿真挂件（已深耕）
    ├── 📁 服饰鞋包（待扩展）
    ├── 📁 美妆护肤（待扩展）
    ├── 📁 家居好物（待扩展）
    └── 📁 ...按需扩展
```

## 使用规则

1. 优先查找品类专属库，有则使用
2. 品类专属库无记录时，使用通用模板
3. 生成成功后，将提示词沉淀到对应品类目录
4. 所有提示词使用中文（即梦对中文支持好，且便于复盘修改）

## 一、通用风格模板

### 可爱风（Kawaii Style）
**适用：** 挂件、盲盒、小物件、零食等

```
可爱的[产品名称]，kawaii风格，柔和的[颜色]背景，
影棚灯光，高细节，4k画质，产品摄影，精致小巧
```

### 生活场景（Lifestyle）
**适用：** 任何需要场景化的产品

```
[产品名称]放在[场景]中，生活场景摄影，
[场景描述]背景，自然光线，温馨氛围
```

**常用场景词：**
- 咖啡馆、咖啡店
- 木质书桌、书房
- 公园、花园、户外草地
- 卧室、客厅、家居环境

### 特写展示（Close-up）
**适用：** 需要展示细节纹理的产品

```
[产品名称]特写，细腻纹理，
微距摄影，影棚灯光，锐利对焦，真实质感
```

### 商务简约（Minimalist）
**适用：** 3C 配件、家居、商务类产品

```
[产品名称]，简约风格，干净的白色背景，
专业产品摄影，柔和阴影，高端质感
```

### ins 风（Instagram Aesthetic）
**适用：** 美妆、家居、生活方式类

```
[产品名称]，ins风，暖色调，
柔和自然光线，精心摆拍，生活氛围感
```

### 复古文艺（Vintage）
**适用：** 文创、手账、复古类产品

```
[产品名称]，复古文艺风，温暖的胶片色调，
怀旧氛围，轻微颗粒感，艺术构图
```

## 二、通用负面提示词

```
模糊，低质量，变形，扭曲，丑陋，
水印，文字叠加，过度饱和，
卡通感（需要写实时），写实感（需要卡通时）
```

## 三、推荐参数

| 场景 | 分辨率 | 宽高比 | 说明 |
|------|--------|--------|------|
| 试片 | 1k-2k | 1:1 | 快速验证方向 |
| 小红书方图 | 2k-4k | 1:1 | 默认，适合大多数产品 |
| 小红书竖图 | 2k-4k | 3:4 | 适合穿搭、场景展示 |
| 横版展示 | 2k-4k | 16:9 | 适合场景、氛围图 |

## 四、品类专属库

> 品类目录位于 `~/.hermes/profiles/huimeng/prompt-templates/` 下

### 4.1 仿真挂件 `pendants/`（已深耕）

**产品关键词：**

| 产品 | 中文关键词 |
|------|-----------|
| 小汉堡 | 仿真小汉堡，迷你汉堡 |
| 冰淇淋 | 仿真冰淇淋，甜筒冰淇淋 |
| 牛角包 | 仿真牛角包，法式可颂 |
| 吐司 | 仿真吐司，面包片 |
| 甜甜圈 | 仿真甜甜圈，甜甜圈 |
| 钥匙扣 | 钥匙扣，钥匙链 |
| 包挂 | 包挂，包包挂件，包饰 |
| 冰箱贴 | 冰箱贴，磁力贴 |

**品类专属模板：**

挂件可爱风：
```
可爱的仿真[产品名称]钥匙扣，kawaii风格，
柔和粉色背景，影棚灯光，
高细节，4k画质，产品摄影，精致小巧
```

挂件挂包场景：
```
仿真[产品名称]钥匙扣挂在[包类型]上，
生活场景摄影，[场景]背景，
自然光线，温馨氛围，时尚配饰
```

### 4.2 服饰鞋包 `clothing/`（待扩展）

> 首次生成该品类产品后，将成功的提示词沉淀到这里。

### 4.3 美妆护肤 `beauty/`（待扩展）

> 首次生成该品类产品后，将成功的提示词沉淀到这里。

### 4.4 家居好物 `home/`（待扩展）

> 首次生成该品类产品后，将成功的提示词沉淀到这里。

## 五、新品类扩展流程

1. 收到新品类需求
2. 使用通用风格模板生成试片
3. 根据效果调整提示词
4. 成功后将最终提示词记录到 prompt-templates/prompt-log.md
5. 将验证过的提示词沉淀到 prompt-templates/<category>/ 目录
6. 更新 prompt-log.md 中的品类统计表
```

**验证：**
```bash
hermes -p huimeng
> /skills list
# 应该看到 dreamina-cli 和 product-prompts 两个 skill
```

---

### 阶段四：初始化项目文件

#### 步骤 9：创建提示词库目录结构

```bash
# 在绘梦 profile 下创建提示词库目录（英文命名）
PROFILE=~/.hermes/profiles/huimeng

mkdir -p $PROFILE/prompt-templates/pendants
mkdir -p $PROFILE/prompt-templates/clothing
mkdir -p $PROFILE/prompt-templates/beauty
mkdir -p $PROFILE/prompt-templates/home

# 查看最终结构
tree $PROFILE/prompt-templates/
```

**预期目录结构：**
```
~/.hermes/profiles/huimeng/
├── config.yaml
├── .env
├── SOUL.md
├── skills/
├── sessions/
├── logs/
├── workspace/
└── prompt-templates/              # 提示词库（归一化在 profile 内）
    ├── prompt-log.md              # 生成记录（步骤 10 创建）
    ├── pendants/                  # 仿真挂件 🟢
    ├── clothing/                  # 服饰鞋包 ⚪
    ├── beauty/                    # 美妆护肤 ⚪
    └── home/                      # 家居好物 ⚪
```

> 💡 所有绘梦相关数据集中在 profile 目录下，便于整体管理、导出和迁移。项目仓库 `/Users/wxc/Documents/hive-commerce/` 只放设计文档，不存运行时数据。

---

#### 步骤 10：初始化提示词记录文件

**文件：** `~/.hermes/profiles/huimeng/prompt-templates/prompt-log.md`

```markdown
# 绘梦 · 提示词记录

记录每次图片生成的提示词、参数和效果，按品类归档，便于复盘优化。

---

## 记录格式

### YYYY-MM-DD HH:MM | 品类 | 产品名称 | 风格
- **提示词：** [完整的英文 prompt]
- **负面提示词：** [negative prompt]
- **参数：** 模型=xxx, 分辨率=xxx, 比例=xxx
- **submit_id：** [dreamina 返回的任务 ID]
- **效果评价：** ⭐⭐⭐⭐⭐ (1-5)
- **是否沉淀：** ☐ 已沉淀到品类库 / ☐ 效果差未沉淀
- **备注：** [任何值得记录的观察]

---

## 品类统计

| 品类 | 生成次数 | 平均评分 | 成熟度 |
|------|---------|---------|--------|
| 仿真挂件 | 0 | - | 🟢 深耕中 |
| 服饰鞋包 | 0 | - | ⚪ 待扩展 |
| 美妆护肤 | 0 | - | ⚪ 待扩展 |
| 家居好物 | 0 | - | ⚪ 待扩展 |

---

（等待第一条记录）
```

---

### 阶段五：启动与测试

#### 步骤 11：启动绘梦 Gateway

```bash
# 启动绘梦的 gateway（独立进程，不影响默认 profile）
hermes -p huimeng gateway
```

**或后台运行：**
```bash
nohup hermes -p huimeng gateway > /dev/null 2>&1 &
```

**查看日志：**
```bash
tail -f ~/.hermes/profiles/huimeng/logs/gateway.log
```

**预期日志：**
```
[Feishu] WebSocket connection established
[Feishu] Bot connected successfully
```

**验证：**
- 日志中无错误信息
- 在飞书群中发送 `@绘梦 你好`，机器人能响应

---

#### 步骤 12：端到端测试

**测试 1：基础对话**

```
飞书发送：@绘梦 你好，你是谁？

预期：绘梦自我介绍，说明职责范围（通用产品图片生成）
```

**测试 2：文生图（已深耕品类：仿真挂件）**

```
飞书发送：@绘梦 生成一张小汉堡钥匙扣的可爱风产品图

预期流程：
1. 确认需求（产品=小汉堡钥匙扣，品类=仿真挂件，风格=可爱风）
2. 使用仿真挂件品类专属模板
3. 调用 dreamina text2image 命令
4. 等待生成完成
5. 将图片发送到群聊
6. 附带使用的提示词记录
```

**测试 3：文生图（新品类：验证通用能力）**

```
飞书发送：@绘梦 生成一条碎花连衣裙的 ins 风产品图

预期流程：
1. 识别为新品类（服饰鞋包），无专属模板
2. 使用通用 ins 风模板 + 产品特征描述
3. 生成图片
4. 提示：该品类提示词库尚未沉淀，生成后将尝试沉淀
```

**测试 4：多风格批量生成**

```
飞书发送：@绘梦 为一支口红生成 3 种不同风格的图片

预期流程：
1. 识别品类：美妆护肤
2. 输出 3 种风格的提示词方案
3. 依次生成 3 张图片
4. 逐张发送到群聊
5. 记录所有提示词
```

**测试 5：图生图（如有视觉能力）**

```
飞书发送：@绘梦 把这张图转换成水彩风格 [附图片]

预期流程：
1. 分析上传的图片（vision 能力）
2. 调用 dreamina image2image
3. 返回风格转换后的图片
```

---

#### 步骤 13：图片发送问题排查

**如果图片无法发送到飞书，按以下步骤排查：**

**13a. 检查飞书权限配置**

```bash
# 在飞书开放平台 → 绘梦应用 → 权限管理中确认：
# ✅ im:message:send_as_bot 已开通
# ✅ im:resource 已开通
# ✅ 应用已发布并审批通过
```

**13b. 检查 Hermes 日志**

```bash
tail -f ~/.hermes/profiles/huimeng/logs/gateway.log

# 关注以下关键词：
# "Skipping unsafe MEDIA directive path" → 路径安全拦截
# "upload image" → 图片上传尝试
# "error" / "fail" → 错误信息
```

**13c. 确认即梦图片保存路径**

```bash
# 查看即梦 CLI 下载的图片位置
ls ~/.dreamina_cli/
find ~/.dreamina_cli -name "*.png" -o -name "*.jpg" 2>/dev/null | head -5
```

**13d. 测试 dreamina 本身是否正常**

```bash
# 检查登录状态
dreamina user_credit

# 手动测试生成
dreamina text2image --prompt="a cute cat" --ratio=1:1 --poll=30

# 查看结果
dreamina list_task --gen_status=success
```

**13e. 路径安全配置（如需要）**

如果日志中出现 `Skipping unsafe MEDIA directive path`，说明 Hermes 的图片路径安全策略拦截了图片发送。需要在 Hermes 的源码或配置中找到 `allowed_paths` 相关配置，添加即梦的输出目录。

具体的配置方式需要根据 Hermes gateway 代码确认。

---

## 四、关键文件清单

### 需要修改/创建的文件

| # | 文件路径 | 操作 | 说明 |
|---|---------|------|------|
| 1 | `~/.hermes/profiles/huimeng/SOUL.md` | 修改 | 绘梦的人格设定 |
| 2 | `~/.hermes/profiles/huimeng/config.yaml` | 修改 | 启用 vision，配置 platform_toolsets |
| 3 | `~/.hermes/profiles/huimeng/.env` | 修改 | 新飞书应用凭证 |
| 4 | `~/.hermes/profiles/huimeng/skills/dreamina-cli/SKILL.md` | 复制 | 即梦 CLI 技能 |
| 5 | `~/.hermes/profiles/huimeng/skills/product-prompts/SKILL.md` | 创建 | 通用模板 + 风格库 |
| 6 | `~/.hermes/profiles/huimeng/prompt-templates/` | 创建 | 品类提示词库目录（英文命名） |
| 7 | `~/.hermes/profiles/huimeng/prompt-templates/prompt-log.md` | 创建 | 提示词生成记录 |

### 需要参考的文件

| 文件路径 | 说明 |
|---------|------|
| `~/.dreamina_cli/dreamina/SKILL.md` | 即梦官方 Skill 文档 |
| `~/.hermes/config.yaml` | 默认 profile 配置（参考） |
| `~/.hermes/.env` | 默认 profile 环境变量（参考 ANTHROPIC_AUTH_TOKEN 等） |
| `ji_meng/cli.md` | 即梦 CLI 使用文档 |
| `docs/superpowers/specs/2026-07-07-xhs-ecommerce-agents-design.md` | 整体系统设计 |

---

## 五、手动复现检查清单

| 步骤 | 操作 | 验证方法 |
|------|------|---------|
| 1 | 创建 profile | `hermes profile list` 能看到 huimeng |
| 2 | 配置 SOUL.md | `cat` 文件内容正确 |
| 3 | 修改 config.yaml | `hermes -p huimeng tools` 显示正确工具集 |
| 4 | 配置 .env | 飞书凭证正确 |
| 5 | 创建飞书应用 | 飞书开放平台能看到「绘梦」 |
| 6 | 添加到群 | 群成员列表有「绘梦」 |
| 7 | 安装 dreamina-cli skill | `ls` 文件存在 |
| 8 | 创建 product-prompts skill | `hermes -p huimeng` 中 `/skills list` 可见 |
| 9 | 创建 prompt-templates 目录 | `tree ~/.hermes/profiles/huimeng/prompt-templates/` 结构正确 |
| 10 | 创建 prompt-log.md | `ls` 文件存在 |
| 11 | 启动 gateway | 日志显示 WebSocket 连接成功 |
| 12 | 端到端测试 | 对话、生图、图片发送均正常 |

---

## 六、风险与应对

| 风险 | 影响 | 概率 | 应对措施 |
|------|------|------|---------|
| 图片无法发送到飞书 | 生成结果无法在群中看到 | 中 | 排查飞书权限 + Hermes 路径安全配置 |
| 即梦积分不足 | 无法生成图片 | 低 | `dreamina user_credit` 定期检查，及时充值 |
| 即梦 CLI 生成质量不稳定 | 图片不符合预期 | 中 | 使用成熟提示词模板，多次生成取最佳 |
| Hermes profile 间端口冲突 | gateway 无法启动 | 低 | 不同 profile 自动使用不同资源 |
| 飞书应用审批延迟 | 无法及时测试 | 低 | 个人版飞书通常即时生效 |
| 即梦 CLI 登录态过期 | 生成命令失败 | 低 | `dreamina login` 重新登录 |

---

## 七、后续规划

绘梦跑通后，按相同模式创建其他 Agent：

| Agent | Profile 名 | 飞书应用名 | 核心能力 |
|-------|-----------|-----------|---------|
| 绘梦（当前） | huimeng | 绘梦 | 即梦 CLI 生图（通用）→ 飞书 |
| 文案撰写师 | wencai | 文才 | LLM 生成小红书文案（通用） |
| 内容策划师 | cehua | 策策 | 内容日历 + 选题规划 |
| 生视频工程师 | video | 映梦 | 即梦 CLI 生视频（通用）→ 飞书 |
| 市场分析师 | fenxi | 析析 | 小红书数据抓取 + 趋势分析 |
| 数据分析师 | shuju | 数数 | 帖子数据追踪 + 优化建议 |
| 客服助理 | kefu | 客客 | 评论监控 + 回复话术 |

> 💡 **通用设计原则：** 每个 Agent 的核心能力都是品类无关的，专业性通过各自领域内的知识沉淀来体现。这样未来扩展到其他电商品类时，Agent 本身无需重建，只需补充对应的领域知识。

---

**文档版本：** v4.0  
**创建日期：** 2026-07-09  
**更新日期：** 2026-07-09  
**负责人：** 独立运营  
**状态：** 待确认
