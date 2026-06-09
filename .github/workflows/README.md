# GitHub Actions 构建说明

与上游 [mpv-winbuild-cmake](https://github.com/shinchiro/mpv-winbuild-cmake) 相同的两阶段流水线，已改为使用**当前仓库**（`github.repository`），不再指向 shinchiro / media-kit。

## 前置配置

在仓库 **Settings → Secrets and variables → Actions** 添加：

| Secret | 示例值 | 说明 |
|--------|--------|------|
| `CACHE_VERSION` | `v1` | 缓存命名前缀；依赖大改时改成 `v2` 可丢弃旧缓存 |

确保 **Settings → Actions → General** 中 Workflow permissions 为 **Read and write**（需要触发 workflow、创建 Release）。

## 工作流

### 1. `llvm toolchain`（先跑）

- 环境：Arch Linux 容器 `ghcr.io/shinchiro/archlinux:latest`
- 内容：PGO 训练 + 构建 LLVM/clang，并为 i686 / x86_64 / x86_64-v3 / aarch64 生成工具链
- 产物 Artifact：`toolchain`（`clang_root`）
- 完成后**自动触发**本仓库的 `mpv clang`

手动运行：**Actions → llvm toolchain → Run workflow**

可选输入 `command`：构建前执行的 shell 命令。

### 2. `mpv clang`（工具链完成后）

- 矩阵并行四架构：i686、x86_64、x86_64_v3、aarch64
- 内容：download → update → mpv → mpv-packaging
- 产物 Artifact：`mpv-<arch>`、`mpv-<arch>-debug`、日志

手动运行：**Actions → mpv clang → Run workflow**

| 输入 | 默认 | 说明 |
|------|------|------|
| `github_release` | false | true 时上传到**本仓库** GitHub Releases |
| `mpv_tarball` | false | true 时构建 mpv-release tarball 版 |
| `command` | 空 | 构建前自定义命令 |

## 推荐操作顺序

```
1. push 代码到 GitHub（含 workflow 与 CMake 修改）
2. 配置 CACHE_VERSION Secret
3. 运行 llvm toolchain（首次很慢，数小时；有缓存后会快很多）
4. 等待自动触发 mpv clang，或手动再跑 mpv clang
5. 在 Run 页面下载 Artifacts（*.7z）
```

发布到 Release：跑 `mpv clang` 时勾选 **github_release = true**。

从 Artifacts 中的 `mpv-*.7z` 解压即可得到 Windows 版 mpv / libmpv 文件。

## 常见问题

**Q: llvm 步骤失败 / 超时？**  
首次无缓存正常偏久；失败时下载 `logs` Artifact 查看。可重跑，Cache 会保留已完成部分。

**Q: mpv 未自动触发？**  
检查 llvm 工作流最后一步 `Run mpv_clang.yml` 是否成功，以及仓库 Actions 写权限。

**Q: Release 上传失败？**  
需 `contents: write` 权限；私有仓库注意 Token 范围。
