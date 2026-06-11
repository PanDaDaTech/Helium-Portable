# GitHub Actions 设计笔记

本项目 CI workflow 的设计决策和踩坑记录。

## 架构

```
check job → 比较版本 → build job → 构建 + 发布
```

- **check job**：用 `gh release view` 获取本仓库当前 release tag 和上游最新 tag，比较 Helium 版本是否有更新
- **build job**：下载上游 release → 解压 → 组装便携包 → 上传 artifact + 创建 release

## 关键设计决策

### 只跟踪 Helium Windows 更新

只在 Helium Windows 发布新 release 时触发构建。Chrome++ 不单独触发——构建时自动拉取最新版本。

理由：Helium 是浏览器主体，Chrome++ 是增强插件。用户关心的是浏览器版本更新。

### 用 gh CLI 替代第三方 Actions

版本获取和 release 发布全部使用 `gh` CLI，不用第三方 action（如 `softprops/action-gh-release`、`pozetroninc/github-action-get-latest-release`）。

理由：
- 避免 Node.js 版本废弃警告（第三方 action 引用 `@master` 时不可控）
- `+` 号在文件名中会被 `softprops/action-gh-release` 的 glob 引擎错误展开，导致目录内所有文件被上传为独立 asset
- `gh` 预装在 GitHub runners，无额外依赖

### Artifact vs Release 上传策略

| 目标 | 方式 | 原因 |
|------|------|------|
| Artifact | 直接上传目录 | `upload-artifact` 支持目录 |
| Release | 先 zip 再上传文件 | GitHub Releases API 只接受文件，不接受目录 |

### 版本号格式

`{helium_version}+{chrome_plus_version}`，例如 `0.13.2.1+1.17.0`

从文件名解析版本号，去除 `v` 前缀。

## 踩坑记录

### glob 展开 bug

`softprops/action-gh-release` 的 `files` 参数使用 `@actions/glob`。文件名含 `+`（如 `helium_0.13.2.1+1.17.0_x64-windows.zip`）时 glob 引擎会错误展开，把同名目录下所有文件作为独立 asset 上传（33 个文件 + 1 个 zip）。

解决：改用 `gh release create` 直接指定文件路径。

### 无 release 时的处理

新仓库没有任何 release 时，`gh release view` 返回非零退出码。用 `2>/dev/null || echo ""` 静默处理，空值走首次构建逻辑。

### dev 分支 workflow 不生效

在 dev 分支修改 workflow 文件不会被 GitHub Actions 识别（schedule 和 workflow_dispatch 只看默认分支）。测试时需要直接在 main 分支操作。

## Bash 备忘

```bash
# 变量赋值不能有空格
var="value"  # 正确
var = "value"  # 错误

# 跨 step 传递变量
echo "key=value" >> "$GITHUB_OUTPUT"  # 同 job 跨 step
echo "key=value" >> "$GITHUB_ENV"     # 同 job 后续 step 环境变量

# 跨 job 传递
# job1 声明 outputs，job2 用 needs.job1.outputs.key 读取

# 字符串截取
ver="${tag#v}"        # 去掉 v 前缀
helium="${ver%%+*}"   # 取 + 号前的部分
plus="${ver#*+}"      # 取 + 号后的部分
```
