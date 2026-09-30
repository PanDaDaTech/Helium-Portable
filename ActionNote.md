# GitHub Actions 设计笔记

本项目 CI workflow 的设计决策和踩坑记录。

## 架构

```
check job → 比较版本 → build job → 构建 + 发布
```

- **check job**：读本仓库当前 release tag + 上游 Helium 最新 tag + 仓库内 `vendor/chrome_plus/VERSION`，三个值决定要不要出包
- **build job**：checkout（取 `vendor/`）→ 下载 Helium 安装包 → 解压 → 组装便携包 → 上传 artifact + 创建 release

## 关键设计决策

### Chrome++ 改为仓库自带（vendored）

上游 `Bush2021/chrome_plus` 账号被平台冻结，**整个 owner 变 404**（不只是单个仓库）。
CI 里 `gh release view --repo Bush2021/chrome_plus` 必然失败，在 `shell: bash`（`-e -o pipefail`）
下未加保护的那行会直接让 check job 退出，表现为「release not found / exit code 1」。

处理方式：把 Chrome++ 的载荷提取出来提交进仓库，构建时只读本地文件，不再访问上游。

```
vendor/chrome_plus/
├── VERSION                              # Chrome++ 版本号，唯一来源
├── x64/App/{chrome++.ini,version.dll}
└── arm64/App/{chrome++.ini,version.dll}
```

- Chrome++ 包在便携包里的落点只有三处：`App/chrome++.ini`、`App/version.dll`
  （根目录的 `Cache/`、`Data/` 是空目录）。空目录 git 存不了，改由构建脚本 `mkdir -p` 生成。
- 版本号从「查上游」变成「读 `vendor/chrome_plus/VERSION`」。**升级 Chrome++ 必须手动改这个文件并提交**，
  push 后 CI 会在 check 阶段判定 `chrome_plus changed` 并重新出包（tag 里带 Chrome++ 版本，所以必须重建）。
- `vendor/chrome_plus/**` 在 `.gitattributes` 里声明 `-text`，原因见下方踩坑记录。

### 只跟踪 Helium Windows 更新

构建由 Helium Windows 发布新版本触发；Chrome++ 不再自动跟随上游（上游已不可访问），
改为仓库内固定版本，需要人工升级。

理由：Helium 是浏览器主体，Chrome++ 是增强插件且已停止更新。用户关心的是浏览器版本更新。

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

### 上游取值失效时不能「静默」

`shell: bash` 在 Actions 下带 `-e -o pipefail`，**命令替换失败会终止整个 step**。
`latest_helium="$(gh release view ...)"` 这种没加 `|| echo ""` 的写法，上游一 404 就整个 job 挂掉
（这就是 Chrome++ 账号被冻结后 CI 报 `release not found / exit code 1` 的直接原因）。

但反过来，把上游取值也一并 `|| echo ""` 同样危险：空值会被后面的 `[[ "$cur_helium" != "$latest_helium" ]]`
判成「版本相同」而**静默跳过构建**——CI 一片绿，其实再也不会出包。

正确做法是「静默捕获 + 显式判空报错」两步，见 main.yml 里对 `imputnet/helium-windows` 的处理：
先 `2>/dev/null || echo ""` 避免 `-e` 炸掉，再 `if [[ -z "$latest_helium" ]]; then echo "::error::..."; exit 1; fi`。

### vendored 的 ini 是 UTF-16LE，必须锁死换行符转换

`chrome++.ini` 不是 UTF-8 文本，而是 **UTF-16LE**（带 BOM `FF FE`，含 NUL 字节，行尾是 `0D 00 0A 00`）。
本机 `core.autocrlf=true`，一旦让 git 把它按文本处理，`0A` 会被改写成 `0D 0A`，
UTF-16 码元流被写坏；而 Chrome++ 解析配置是读二进制流，坏了也不报错、只会悄悄退回默认配置。

所以加了 `.gitattributes`：

```
vendor/chrome_plus/** -text
```

`version.dll` 是 PE 二进制同理。**以后新增任何 vendored 二进制或非 UTF-8 文件，都要一并纳入这条规则。**

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
