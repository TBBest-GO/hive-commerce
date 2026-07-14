# 容器内使用即梦 CLI 方案

**创建日期：** 2026-07-14  
**状态：** 已实施  
**适用范围：** Hermes Agent 在 Docker 容器内使用即梦 CLI

---

## 一、问题背景

### 1.1 架构挑战

Hermes Agent 的 terminal 工具在 Docker 容器（Linux）内执行命令，但即梦 CLI 在宿主机（macOS）上安装的是 macOS 原生二进制（Mach-O 格式），无法在 Linux 容器内运行。

```
宿主机 (macOS arm64)
┌─────────────────────────────┐
│ dreamina: Mach-O arm64      │ ← macOS 原生二进制
│ ~/.local/bin/dreamina       │
└─────────────────────────────┘
           ↓ 无法直接调用
┌─────────────────────────────┐
│ Docker 容器 (Debian Linux)  │
│ 只能运行 ELF 格式二进制      │
└─────────────────────────────┘
```

### 1.2 解决方案概述

**核心思路：** 在宿主机保存 Linux 版 dreamina 二进制，通过 Docker volume 挂载到容器内使用。

**关键设计：**
- Linux 版 dreamina 统一存放在 `/Users/wxc/.hermes/dreamina-linux/`
- 通过 `docker_volumes` 挂载到容器的 `/opt/dreamina`（自定义目录，避免覆盖系统命令）
- **使用绝对路径 `/opt/dreamina/dreamina` 调用命令**（不依赖 PATH 环境变量）
- 登录态 `~/.dreamina_cli` 挂载，但**需要在容器内手动完成一次认证**（macOS Keychain 无法跨平台）
- 一次下载，长期使用

---

## 二、实施步骤

### 2.1 下载 Linux 版 dreamina

使用临时 Docker 容器下载 Linux 版二进制到宿主机：

```bash
# 1. 创建目标目录
mkdir -p /Users/wxc/.hermes/dreamina-linux

# 2. 用 Debian 容器下载 Linux 版 dreamina
docker run --rm \
  -v /Users/wxc/.hermes/dreamina-linux:/output \
  debian:trixie \
  bash -c "curl -fsSL https://jimeng.jianying.com/cli | bash && cp ~/.local/bin/dreamina /output/dreamina"

# 3. 验证下载成功
ls -lh /Users/wxc/.hermes/dreamina-linux/dreamina
file /Users/wxc/.hermes/dreamina-linux/dreamina
# 预期输出：ELF 64-bit LSB executable, ARM aarch64

# 4. 设置可执行权限
chmod +x /Users/wxc/.hermes/dreamina-linux/dreamina
```

### 2.2 修改 config.yaml

在 Agent 的 `config.yaml` 中配置 volume 挂载和环境变量：

```yaml
terminal:
  backend: docker
  # ... 其他配置保持不变 ...
  docker_volumes:
    # workspace 挂载（图片输出）
    - /Users/wxc/.hermes/profiles/huimeng/workspace:/Users/wxc/.hermes/profiles/huimeng/workspace
    # Linux 版 dreamina 二进制（挂载到自定义目录，避免覆盖系统命令）
    - /Users/wxc/.hermes/dreamina-linux:/opt/dreamina
    # 即梦登录态（OAuth token、任务记录）
    - /Users/wxc/.dreamina_cli:/Users/wxc/.dreamina_cli
```

**重要说明：**
- ⚠️ 不要挂载到 `/usr/local/bin`，会覆盖容器内原有的系统命令
- ✅ 使用 `/opt/dreamina` 自定义目录，**所有命令使用绝对路径 `/opt/dreamina/dreamina`**

### 2.3 重启 Gateway

```bash
# 1. 删除旧容器（配置变更后必须重建）
docker rm -f $(docker ps -a --filter "name=hermes-" -q)

# 2. 重启 Gateway
NO_PROXY='*' no_proxy='*' hermes -p huimeng gateway
```

### 2.4 容器内认证（首次使用必需）

由于 macOS Keychain 无法在 Linux 容器内访问，**需要在容器内手动完成一次认证**：

**在飞书群发送：**
```
@绘梦 请执行以下命令并返回输出：
/opt/dreamina/dreamina login --headless
```

**Agent 会返回：**
```
verification_uri: https://jimeng.jianying.com/...
user_code: ABCD-1234
device_code: xxxxx
```

**完成认证：**
1. 在浏览器打开 `verification_uri`
2. 输入 `user_code` 并授权
3. 让 Agent 执行：
   ```
   @绘梦 请执行：/opt/dreamina/dreamina login checklogin --device_code=<device_code> --poll=30
   ```

**验证认证成功：**
```
@绘梦 请执行：/opt/dreamina/dreamina user_credit
```

预期返回账户积分信息。

**重要说明：**
- ✅ 认证完成后，登录态保存在 `~/.dreamina_cli` 目录（已挂载到容器）
- ✅ 只要容器不重建，认证一直有效
- ⚠️ 容器重建后，需要重新执行认证流程

### 2.5 验证安装

在飞书群中测试：

```
@绘梦 请执行以下命令验证：
1. /opt/dreamina/dreamina -h
2. /opt/dreamina/dreamina user_credit
```

预期结果：
- `/opt/dreamina/dreamina -h` 显示帮助信息
- `/opt/dreamina/dreamina user_credit` 返回账户积分信息

---

## 三、端到端测试

### 3.1 文生图测试

```
@绘梦 生成一张小汉堡钥匙扣的可爱风产品图

预期流程：
1. Agent 调用 terminal 执行 dreamina text2image
2. 使用 --download_dir 指定输出到 workspace
3. 图片保存到 workspace/images/2026-07-14/001_小汉堡_可爱风.png
4. Agent 回复包含 MEDIA: 标签
5. 图片发送到飞书群 ✅
```

**命令示例：**
```bash
/opt/dreamina/dreamina text2image \
  --prompt="可爱的仿真小汉堡钥匙扣，kawaii风格，柔和粉色背景，影棚灯光，高细节，4k画质，产品摄影，精致小巧" \
  --ratio=1:1 \
  --resolution_type=2k \
  --download_dir=/Users/wxc/.hermes/profiles/huimeng/workspace/images/2026-07-14/ \
  --poll=30
```

### 3.2 图片发送验证

确认图片成功发送到飞书群，检查日志：

```bash
tail -f ~/.hermes/profiles/huimeng/logs/gateway.log
# 应看到 send_image_file 成功记录
```

---

## 四、维护与更新

### 4.1 更新 dreamina CLI

当即梦发布新版本时：

```bash
# 1. 重新下载 Linux 版
docker run --rm \
  -v /Users/wxc/.hermes/dreamina-linux:/output \
  debian:trixie \
  bash -c "curl -fsSL https://jimeng.jianying.com/cli | bash && cp ~/.local/bin/dreamina /output/dreamina"

# 2. 验证版本
docker run --rm \
  -v /Users/wxc/.hermes/dreamina-linux:/opt/dreamina \
  debian:trixie \
  /opt/dreamina/dreamina version

# 3. 重启 Gateway
```

### 4.2 登录态过期处理

如果 `/opt/dreamina/dreamina user_credit` 返回未登录：

**在容器内重新认证：**
```
@绘梦 请执行：/opt/dreamina/dreamina login --headless
```

然后按照步骤 2.4 的流程完成认证。

**注意：** 由于认证信息存储在 macOS Keychain 中，无法通过 volume 挂载共享，容器内需要独立认证。

---

## 五、排障指南

### 5.1 dreamina 命令找不到

```bash
# 检查容器内是否有 dreamina
docker exec <container_id> ls -lh /opt/dreamina/dreamina
# 预期：-rwxr-xr-x 文件存在

# 检查 volume 挂载
docker inspect <container_id> | grep -A 10 "Mounts"

# 使用绝对路径执行
docker exec <container_id> /opt/dreamina/dreamina -h
```

### 5.2 登录态无效

```bash
# 在容器内检查登录状态
docker exec <container_id> /opt/dreamina/dreamina user_credit

# 如果返回未登录，需要在容器内重新认证
# 参考步骤 2.4 的认证流程
```

### 5.3 权限问题

```bash
# 确保二进制可执行
chmod +x /Users/wxc/.hermes/dreamina-linux/dreamina

# 检查容器内执行权限
docker exec <container_id> ls -lh /opt/dreamina/dreamina
```

### 5.4 认证问题

**问题：** 宿主机已登录，但容器内无法使用

**原因：** macOS Keychain 无法在 Linux 容器内访问

**解决：** 在容器内独立执行认证流程（参考步骤 2.4）

---

## 六、关键配置备忘

```
Linux 版 dreamina 路径: /Users/wxc/.hermes/dreamina-linux/dreamina
容器内挂载点: /opt/dreamina/dreamina
命令调用方式: 使用绝对路径 /opt/dreamina/dreamina <command>
登录态目录: ~/.dreamina_cli/（仅保存任务记录，认证需容器内独立完成）
认证机制: 容器内首次使用需手动执行 login --headless 流程
共享机制: Docker volume 挂载
```

---

## 七、常见问题 FAQ

### Q1: 为什么不使用 PATH 环境变量？

**A:** `docker_env` 配置在 Hermes 中可能不生效，使用绝对路径更可靠。虽然命令较长，但稳定性更好。

### Q2: 为什么需要在容器内重新认证？

**A:** 宿主机（macOS）的认证信息存储在系统 Keychain 中，Linux 容器无法访问。因此需要在容器内独立完成一次 OAuth 认证流程。

### Q3: 容器重建后需要重新认证吗？

**A:** 是的。容器重建后，之前的认证信息会丢失，需要重新执行步骤 2.4 的认证流程。

### Q4: 如何避免频繁重新认证？

**A:** 
- 避免频繁删除容器（`container_persistent: true` 已配置）
- 只在配置变更或出现问题时才重建容器
- 日常使用只需重启 Gateway，不需要删除容器

---

**文档版本：** v1.0  
**创建日期：** 2026-07-14  
**维护者：** 独立运营
