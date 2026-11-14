# GitHub Actions 设计笔记

本项目 CI workflow 的设计决策和踩坑记录。

## 架构

```
check job → 比较版本 → build job → 构建 + 发布
```

- **check job**：读本仓库当前 release tag + 上游 Helium 最新 tag + 上游 Chrome++ 最新 tag，三个值决定要不要出包
- **build job**：checkout（取配置）→ 下载 Helium 安装包与 Chrome++ 包 → 解压 → 组装便携包 → 创建 release

## 关键设计决策

### Chrome++：二进制跟上游，配置用仓库里的

`version.dll` 每次构建从上游 `Bush2021/chrome_plus` 下载；`chrome++.ini` 固定用
`vendor/chrome_plus/chrome++.ini`，覆盖上游包里的同名文件。

```
vendor/chrome_plus/
└── chrome++.ini        # UTF-16LE，与上游 1.20.0 只差一行 command_line
config/
└── initial_preferences # 首启种子：搜索引擎列表 / 默认 Bing / 关闭自动更新
```

Chrome++ 包在便携包里的落点只有三处：`App/chrome++.ini`、`App/version.dll`
（根目录的 `Cache/`、`Data/` 是空目录）。空目录 git 存不了，改由构建脚本 `mkdir -p` 生成。

- 为什么不用上游自带的 ini：我们要带自己的 `command_line`（`--no-first-run`、
  `--disable-component-update`、关掉 `OutdatedBuildDetector` 等）。
- 配置漂移：仓库里这份是 1.20.0 的**快照**。上游若新增配置项，我们缺失的键会被
  Chrome++ 用默认值补上（不会报错）。构建时因此加了一步比对：把两份 ini 的
  `section|key` 集合求差，上游多出来的键打 `::warning::`，只告警不阻断。
- `vendor/chrome_plus/**` 在 `.gitattributes` 里声明 `-text`，原因见下方踩坑记录。

### 历史：2026-09 上游账号被冻结

`Bush2021` 账号曾被平台冻结，**整个 owner 变 404**（不只是单个仓库）。当时把 Chrome++ 的
`version.dll` 也一并 vendored 进仓库应急，2026-11 上游恢复（1.20.0）后改回跟随上游。

留下的一条经验：上游取值在 `shell: bash`（`-e -o pipefail`）下失败会直接终止 job，
表现为「release not found / exit code 1」；但单纯加 `|| echo ""` 会让空值被判成
「版本相同」而静默跳过构建——必须「静默捕获 + 显式判空报错」。

### 首启配置走 initial_preferences，不改 Data 里的 Preferences

想把「默认 Bing + 只留五个搜索引擎 + 关自动更新」预置给用户，**不能**直接往包里塞一个
配好的 `Data/Default/Preferences`：Chromium 对受保护偏好（搜索引擎、主页、启动页、扩展）
存了 HMAC-SHA256，且**绑定本机设备标识**，跨机拷贝后会被判定篡改并重置——现象就是
「搜索引擎不生效、设置还原」。这正是 `xrgzs/CBPrefs` 要解决的事（它靠脚本在本机重算 MAC）。

我们改用官方通道：**`App/initial_preferences`**（老名字 `master_preferences`）。
浏览器首次运行时读取它、**自己**把值写进 `Preferences` 并标记为「已播种」
（`MarkPrefsSeededFromInitialPrefs`），所以 MAC 天然正确，不需要任何脚本或管理员权限。

- 只在**首次运行**生效：`Data/` 为空时才会触发；用户把 `Data/` 删掉会重新播种。
- 文件必须放在 `chrome.exe` 同级目录（`App/`），无扩展名。
- **`command_line` 里绝对不能有 `--no-first-run`**。源码三处互锁：
  `ChromeFeatureListCreator` 里 `if (!first_run::IsChromeFirstRun()) return;` 才加载种子，
  而 `IsChromeFirstRun()` = `force_first_run || (!has_sentinel && !no_first_run)`。
  带上这个开关等于宣告「不是首次运行」，种子文件会被**完全忽略且没有任何报错**。
  这是本次最容易踩的坑：`--no-first-run` 原本就在配置里，看起来毫无关联。
  （`Data/` 空 → 无哨兵 → 首次运行 → 播种，之后哨兵生效，不会重复。）
- 用到的三个键（都来自源码核对，不是猜的）：
  - `search_provider_overrides`：**列表非空时会整表替换**内置预置引擎表
    （`GetPrepopulatedEngines` 先读它）。元素必填 `name/keyword/search_url/favicon_url/encoding/id`，
    可选 `suggest_url` 等。Helium 默认给的是 DDG/Ecosia/Bing/Qwant/Kagi/Startpage/Brave，
    且把区域映射清空了——所以国内用户一个中文引擎都没有，这正是本次要修的点。
  - `default_search_provider_data.template_url_data`：显式指定默认引擎。它按
    `prepopulate_id` 回查已有引擎（`FindPrepopulatedTemplateURL`），所以写 `prepopulate_id=3`
    会和列表里的 Bing 合成同一个，不会多出一个重复项。
  - `helium.services.browser_updates`：Helium 自己的 WinSparkle 更新器总开关
    （`ShouldAccessUpdateService` 与它做 AND）。置 `false` 后连初始化都不做。
    Helium 不是 Google Update，光靠 `--disable-component-update` 关不掉它。
- 联想接口只在 `suggest_url` 返回 OpenSearch 数组（`["q",["s1",...]]`）时才有用：
  Bing / Google 是这种格式；百度 `sugrec`、搜狗 `ajajjson`、360 `suggest` 都是自家 JSON
  结构，Chromium 的通用解析器读不懂，所以这三个**故意不配 `suggest_url`**，省掉无用请求。

### 只跟踪 Helium 与 Chrome++ 的版本变化

任一上游发新版本都会触发重建，因为 tag 里同时带两个版本号。

### 用 gh CLI 替代第三方 Actions

版本获取和 release 发布全部使用 `gh` CLI，不用第三方 action（如 `softprops/action-gh-release`、`pozetroninc/github-action-get-latest-release`）。

理由：
- 避免 Node.js 版本废弃警告（第三方 action 引用 `@master` 时不可控）
- `+` 号在文件名中会被 `softprops/action-gh-release` 的 glob 引擎错误展开，导致目录内所有文件被上传为独立 asset
- `gh` 预装在 GitHub runners，无额外依赖

### 只发 Release，不传 artifact

产物直接作为 Release 附件（先 `7z a` 再传文件，GitHub Releases API 只接受文件不接受目录）。
不再额外上传 `upload-artifact`：产物已在附件里，再存一份纯属重复，而 artifact 会吃
Actions 存储配额（本账户曾因此把上传全堵死）。构建失败要排查时看 Actions 日志。

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
