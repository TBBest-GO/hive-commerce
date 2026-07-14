# 绘梦 Agent 实施进度记录

**更新日期：** 2026-07-10  
**状态：** 图片发送链路已跑通，Skills 和图片生成待完成

---

## 一、已完成 ✅

### 阶段一：创建 Hermes Profile

| 步骤 | 内容 | 状态 | 备注 |
|------|------|------|------|
| 步骤 1 | 创建 `huimeng` profile | ✅ | `hermes profile create huimeng --clone` |
| 步骤 2 | 配置 SOUL.md | ✅ | 已写入绘梦人格设定 |
| 步骤 3 | 修改 config.yaml | ✅ | 已添加 `feishu` platform_toolsets |
| 步骤 4 | 配置 .env | ✅ | 已配置飞书凭证 `` |

### 阶段二：飞书应用

| 步骤 | 内容 | 状态 | 备注 |
|------|------|------|------|
| 步骤 5 | 创建飞书应用 | ✅ | 用户自行完成 |
| 步骤 6 | 添加到 HiveCommerce 群 | ✅ | 用户自行完成 |

### 阶段五：启动与测试

| 步骤 | 内容 | 状态 | 备注 |
|------|------|------|------|
| 步骤 11 | 启动 gateway | ✅ | `hermes -p huimeng gateway` |
| 步骤 12 | 基础对话测试 | ✅ | 飞书群 @绘梦 可正常回复 |
| 图片发送验证 | MEDIA: 标签发送图片到飞书 | ✅ | 路径：`/Users/wxc/.hermes/profiles/huimeng/workspace/test.png` |

### 新增完成项

| 内容 | 状态 | 备注 |
|------|------|------|
| workspace 目录结构创建 | ✅ | `workspace/`, `workspace/images/`, `workspace/reference/` |
| docker_volumes 挂载配置（方案 A） | ✅ | 同路径挂载：`/Users/wxc/.hermes/profiles/huimeng/workspace` |
| test.png 复制测试 | ✅ | 验证图片发送链路通过 |

---

## 二、未完成 ⚪

### 阶段三：Skills

| 步骤 | 内容 | 状态 | 说明 |
|------|------|------|------|
| 步骤 7 | 安装 dreamina-cli skill | ⚪ | 需从 `~/.dreamina_cli/dreamina/SKILL.md` 复制到 `~/.hermes/profiles/huimeng/skills/dreamina-cli/`（CLI 已安装，但 SKILL.md 未复制到 profile） |
| 步骤 8 | 创建 product-prompts skill | ⚪ | 需创建 `~/.hermes/profiles/huimeng/skills/product-prompts/SKILL.md`（通用中文模板 + 风格库） |

### 阶段四：初始化项目文件

| 步骤 | 内容 | 状态 | 说明 |
|------|------|------|------|
| 步骤 9 | 创建 prompt-templates 目录 |  | 需创建 `pendants/` `clothing/` `beauty/` `home/` 子目录（注意：与 workspace/ 不同，这是提示词库目录） |
| 步骤 10 | 初始化 prompt-log.md | ⚪ | 提示词生成记录文件（Markdown 格式） |

### 阶段五：测试（剩余）

| 步骤 | 内容 | 状态 | 说明 |
|------|------|------|------|
| 步骤 12 | 文生图测试（仿真挂件） |  | 依赖 dreamina-cli skill |
| 步骤 12 | 文生图测试（新品类） | ⚪ | 依赖 dreamina-cli skill |
| 步骤 12 | 多风格批量生成 | ⚪ | 依赖 dreamina-cli skill |
| 步骤 12 | 图生图测试 | ⚪ | 需要 vision 能力 + reference/ 目录 |
| 步骤 13 | 图片发送排查 | ✅ | 已通过 test.png 验证发送链路 |

---

## 三、待修复 / 注意事项

### 3.1 FEISHU_ALLOWED_USERS ✅ 已更新

用户已手动将 `.env` 中的 `FEISHU_ALLOWED_USERS` 更新为绘梦应用对应的 open_id。

### 3.2 platform_toolsets 不完整 ⚠️

当前 feishu 只配了 `hermes-feishu`，缺少 `terminal`、`file`、`vision`。
图片发送已跑通（通过 MEDIA: 标签），但后续调用即梦 CLI 生成图片时需要补上：

```yaml
platform_toolsets:
  feishu:
    - hermes-feishu
    - terminal    # 调用 dreamina CLI
    - file        # 读写提示词记录
    - vision      # 图生图（同时需从 disabled_toolsets 中移除 vision）
```

### 3.3 vision 仍在 disabled_toolsets

方案要求移除 `vision`（图生图场景需要视觉分析），当前仍被禁用。后续需要图生图功能时再启用（与 3.2 一起处理）。

### 3.4 提示词语言 ✅ 已确认

提示词使用**中文**（不再用英文）。即梦对中文支持好，且便于复盘和修改。

### 3.5 排障经验

- 飞书 WebSocket 连接会被本地代理（`127.0.0.1:15732`）干扰，启动 gateway 时需设置 `NO_PROXY='*'`
- 不同飞书应用的 open_id 不同，不能直接复制默认 profile 的 allowlist
- Agent 运行在宿主机 gateway 进程中，terminal 命令运行在 Docker 容器内
- MEDIA: 标签中的路径必须是宿主机上存在的路径（方案 A 同路径挂载解决了这个问题）

---

## 四、关键配置备忘

```
Profile: ~/.hermes/profiles/huimeng/
飞书应用 ID: 
飞书应用 Secret: 
群聊 ID:    (HiveCommerce)
启动命令: NO_PROXY='*' no_proxy='*' hermes -p huimeng gateway
工作区路径: /Users/wxc/.hermes/profiles/huimeng/workspace/
容器挂载: /Users/wxc/.hermes/profiles/huimeng/workspace → /Users/wxc/.hermes/profiles/huimeng/workspace (同路径)
图片发送: MEDIA:/Users/wxc/.hermes/profiles/huimeng/workspace/xxx.png
提示词语言: 中文
```

---

## 五、下一步计划

| 优先级 | 内容 | 前置条件 |
|--------|------|----------|
|  高 | 补全 platform_toolsets（terminal + file） | 需要重启 gateway |
| 🔴 高 | 安装 dreamina-cli skill（步骤 7） | 一条 cp 命令 |
| 🔴 高 | 创建 product-prompts skill（步骤 8） | 中文模板 |
| 🟡 中 | 创建 prompt-templates 目录 + prompt-log.md（步骤 9-10） | — |
|  中 | 文生图端到端测试 | dreamina-cli skill 就绪 |
| 🟢 低 | 启用 vision + 图生图测试 | platform_toolsets 补全 |
