# 绘梦 · 图片发送流程验证方案

**创建日期：** 2026-07-10  
**目标：** 验证绘梦能将图片发送到飞书群聊  
**状态：** 待执行

---

## 一、核心机制

Hermes 通过 `MEDIA:` 标签发送图片到飞书：

```
Agent 回复文本中包含: MEDIA:/path/to/image.png
       ↓
Gateway 解析 MEDIA: 标签（正则匹配）
       ↓
validate_media_delivery_path() 校验路径安全性（默认模式：非系统/凭证路径即可通过）
       ↓
调用 Feishu send_image_file() 上传图片到飞书
       ↓
用户在群聊中看到图片 ✅
```

---

## 二、目录结构

```
~/.hermes/profiles/huimeng/
├── config.yaml
├── .env
├── SOUL.md
├── skills/
├── sessions/
├── logs/
├── prompt-templates/
│   ├── prompt-log.md
│   ├── pendants/
│   ├── clothing/
│   ├── beauty/
│   └── home/
── workspace/                     # 工作区（新增）
    ├── images/                    # 生成的图片输出
    │   └── YYYY-MM-DD/           # 按日期分目录
    │       └── {序号}_{产品}_{风格}.png
    ├── reference/                 # 用户上传的参考图
    │   └── YYYY-MM-DD/
    │       └── {序号}_{来源描述}.{ext}
    └── test.png                   # 验证用测试图
```

### 图片命名规则

- 生成图片：`{序号}_{产品名称}_{风格}.png`，如 `001_小汉堡_可爱风.png`
- 参考图片：`{序号}_{来源描述}.{ext}`，如 `001_供应商原图_口红.jpg`
- 序号为当日三位数补零

---

## 三、容器挂载方案（方案 A：同路径挂载）

将宿主机目录挂载到容器内**同一路径**，确保 MEDIA: 标签中的路径在两边一致：

```
宿主机：/Users/wxc/.hermes/profiles/huimeng/workspace/
                              ↓ docker_volumes 挂载
容器内：/Users/wxc/.hermes/profiles/huimeng/workspace/
```

**优点：** 路径统一，Agent 写的 MEDIA: 路径 Gateway 在宿主机上能直接找到，无需路径转换。  
**与现有 `/workspace` 挂载不冲突。**

---

## 四、执行步骤

### 步骤 1：创建目录 + 复制测试图

```bash
mkdir -p ~/.hermes/profiles/huimeng/workspace/images
mkdir -p ~/.hermes/profiles/huimeng/workspace/reference
cp /Users/wxc/Documents/hive-commerce/test.png ~/.hermes/profiles/huimeng/workspace/test.png
```

### 步骤 2：修改 config.yaml

在 `terminal` 下添加 `docker_volumes`：

```yaml
terminal:
  backend: docker
  cwd: .
  timeout: 180
  home_mode: auto
  container_cpu: 1
  container_memory: 5120
  container_disk: 51200
  container_persistent: true
  docker_mount_cwd_to_workspace: false
  docker_volumes:
    - /Users/wxc/.hermes/profiles/huimeng/workspace:/Users/wxc/.hermes/profiles/huimeng/workspace
  lifetime_seconds: 300
```

### 步骤 3：删除旧容器 + 重启 Gateway

```bash
# 删除旧容器（配置变更后必须重建才能生效）
docker rm -f hermes-3c875322

# 停掉当前 gateway（Ctrl+C）
# 重新启动
NO_PROXY='*' no_proxy='*' hermes -p huimeng gateway
```

### 步骤 4：飞书群发送测试

```
@绘梦 请把这张图片发到群里：/Users/wxc/.hermes/profiles/huimeng/workspace/test.png
```

**预期：** 绘梦在回复中包含 `MEDIA:/Users/wxc/.hermes/profiles/huimeng/workspace/test.png`，图片出现在群聊中。

### 步骤 5：观察日志

```bash
tail -f ~/.hermes/profiles/huimeng/logs/gateway.log
```

| 日志关键词 | 含义 |
|-----------|------|
| `send_image_file` | 图片上传成功 ✅ |
| `Skipping unsafe MEDIA directive path` | 路径被安全策略拒绝 ❌ |
| `Media file not found` | 文件路径不存在 ❌ |
| `Feishu media send failed` | 飞书 API 调用失败 ❌ |

---

## 五、可能的问题与排查

| 问题 | 排查方法 |
|------|---------|
| 图片发送后群里看不到 | 检查日志是否有 `send_image_file` 成功记录 |
| 出现 `Skipping unsafe` | 检查路径是否在 denylist 中（/etc, ~/.ssh 等），test.png 路径应该没问题 |
| 出现 `Media file not found` | 确认容器重建成功，文件已复制到 workspace 目录 |
| Gateway 启动报错 | 检查 config.yaml 语法是否正确 |
| 容器没有重建 | 确认 `docker rm` 执行成功，terminal 调用会自动创建新容器 |

---

## 六、验证通过后

图片发送链路跑通后，后续步骤：
1. 安装 dreamina-cli skill（步骤 7）
2. 创建 product-prompts skill（步骤 8）
3. 初始化 prompt-log.md（步骤 10）
4. 用即梦实际生成图片并发送（端到端测试）

---

**文档版本：** v1.0  
**更新日期：** 2026-07-10
