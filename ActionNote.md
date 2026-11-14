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
└── initial_preferences # 首启种子：默认 Bing / 关闭自动更新 / 跳过引导页
```

Chrome++ 包在便携包里的落点只有三处：`App/chrome++.ini`、`App/version.dll`
（根目录的 `Cache/`、`Data/` 是空目录）。空目录 git 存不了，改由构建脚本 `mkdir -p` 生成。

便携包结构（压缩包内的顶层目录固定叫 `Helium`，不带版本号；版本只体现在附件文件名里）：

```
Helium/
├── App/            # chrome.exe + Chrome++ + initial_preferences
├── Cache/          # 空
└── Data/
    └── Default/
        └── Preferences   # 与 App/initial_preferences 同内容，解压即已配置
```

`config/initial_preferences` 会**同时**投两份：`App/initial_preferences`（首启种子）和
`Data/Default/Preferences`（直接就是配置好的状态）。前者保证用户删掉 `Data/` 后重新首启
仍能播种；后者保证解压后不必等首启写入就已经是「默认 Bing + 关更新」。

- 为什么不用上游自带的 ini：我们要带自己的 `command_line`（`--disable-component-update`、
  关掉 `OutdatedBuildDetector` 等）。**注意里面没有 `--no-first-run`，也不要加**，理由见下。
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

想把「默认 Bing + 关自动更新 + 不弹引导页」预置给用户，**不能**直接往包里塞一个
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
- 用到的键（都来自源码核对 + 实机播种，不是猜的）：
  - `default_search_provider_data.template_url_data`：显式指定默认引擎。它按
    `prepopulate_id` 回查已有引擎（`FindPrepopulatedTemplateURL`），所以写 `prepopulate_id=3`
    会和内置的 Bing 合成同一个，不会多出一个重复项。实机播种后的 `Preferences` 里
    `default_search_provider.reset_occurred = false`、`mirrored_template_url_data.short_name`
    变成内置值 `Microsoft Bing`、`search_engines.engines` 仍为空 —— 三项都说明「合成了、没加新的、没被重置」。
  - `helium.services.browser_updates`：Helium 自己的 WinSparkle 更新器总开关
    （`ShouldAccessUpdateService` 与它做 AND）。置 `false` 后连初始化都不做。
    Helium 不是 Google Update，光靠 `--disable-component-update` 关不掉它。
  - `helium.completed_onboarding`：`helium://setup` 引导页的总开关。**这才是「首次启动页」的真凶**，
    它跟 `--no-first-run` 无关：`--no-first-run` 只作用于 Chromium 的首次运行判定，管不到 Helium 自己的引导页。
    置 `true` 即视为已完成引导。

### 为什么不能用 `search_provider_overrides` 增删引擎

原本的方案是「用 `search_provider_overrides` 整表替换内置引擎表」：
列表非空时 `GetPrepopulatedEngines` 会先读它，元素必填 `name/keyword/search_url/favicon_url/encoding/id`。
理论上可以做到「只留 Bing/百度/搜狗/360/Google」，但**实测会弹「重置了这些设置」**。

原因是这个键在 `kTrackedPrefs` 里，且标记为 `ENFORCE_ON_LOAD`：值必须是**本机**算出的 HMAC。
首启种子写入的值 MAC 对不上，浏览器就判定为篡改 → 丢弃该值 → 用户看到重置提示。
换句话说，它和 `Data/Default/Preferences` 里那堆受保护偏好是同一类东西，只能靠脚本在本机重算 MAC
（`xrgzs/CBPrefs` 干的就是这个），不适合放在便携包的种子里。

结论：**种子只能设默认引擎，不能删内置引擎、也不能加自定义引擎。**
`config/initial_preferences` 因此只剩 `default_search_provider_data` + `helium` 两块，
CI 里加了硬校验：出现 `search_provider_overrides` 直接 `exit 1`，防止以后有人「优化」回去。

顺带记一笔：Helium 的 `engine-defaults.patch` 清空了区域映射，默认给的是
DDG/Ecosia/Bing/Qwant/Kagi/Startpage/Brave，国内用户看不到中文引擎。
想让用户拿到百度/搜狗/360，只能引导他们在「设置 → 搜索引擎 → 管理搜索引擎」里自己添加。
联想接口也只有 `suggest_url` 返回 OpenSearch 数组（`["q",["s1",...]]`）时才有用，
Bing/Google 是这种格式，百度 `sugrec`、搜狗 `ajajjson`、360 `suggest` 都是自家 JSON，
Chromium 的通用解析器读不懂 —— 所以即使加引擎也不配 `suggest_url`，省掉无用请求。

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

### 两个「看起来是一件事、其实是两件事」的现象

1. **弹出「重置了这些设置」** → 种子里放了受跟踪偏好（`search_provider_overrides`），
   MAC 对不上被判定篡改。只保留 `default_search_provider_data` 后 `reset_occurred = false`，弹窗消失。
2. **首次启动还是打开引导页** → 不是 Chromium 的首次运行，而是 Helium 自己的 `helium://setup`，
   由 `helium.completed_onboarding` 控制，`--no-first-run` 完全管不到。把它置 `true` 才有效。

两个问题的共同教训：**先分清是 Chromium 的行为还是 Helium 的行为**，否则会一直调错开关。

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
