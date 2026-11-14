# Chrome++ 配置（仓库自带）

这里只放 **`chrome++.ini`**。`version.dll` 已经改回从上游 release 下载，不再入库。

## 为什么要自带这份 ini

上游 `Bush2021/chrome_plus` 的账号在 2026-09 曾被平台冻结（整个 owner 变 404），当时把
Chrome++ 载荷整体 vendored 进仓库应急。2026-11 上游恢复（1.20.0），CI 改回跟随上游版本，
但**配置文件仍用仓库里的这份**——因为它带着我们自己的启动参数。

## 与上游的差异

这份文件取自 Chrome++ 1.20.0 完整包，与上游 1.20.0 自带的 `chrome++.ini`
**逐行 diff 只有 `command_line` 这一行不同**：

```
command_line=--disable-component-update --disable-crash-reporter --metrics-recording-only
  --disable-logging
  --disable-features=PrintCompositorLPAC,ExtensionManifestV2Unsupported,ExtensionManifestV2Disabled,OutdatedBuildDetector
  --force-renderer-accessibility=basic --enable-features=WebMCP
```

（上游这里是空的。）其余 21 个配置项的名称与取值完全相同。

### 注意：这里**故意没有** `--no-first-run`

原来的配置里有 `--no-first-run`，已去掉。原因：Chromium 只有在「首次运行」时才会读取
`App/initial_preferences`（`ChromeFeatureListCreator` 里 `if (!first_run::IsChromeFirstRun()) return;`），
而 `IsChromeFirstRun()` 的判定是 `force_first_run || (!has_sentinel && !no_first_run)`——
带上 `--no-first-run` 就等于强制「不是首次运行」，预置的搜索引擎、更新开关会**全部静默失效**。

去掉后首次启动即完成播种，之后靠首启哨兵自然不再重复。**不要把它加回来。**

## 构建时的处理

1. 下载上游 `Chrome++_v<ver>_x86_x64_arm64.7z`，取 `{x64,arm64}/App/version.dll`。
2. 用本目录的 `chrome++.ini` **覆盖**上游同名文件。
3. CI 会比对两份 ini 的配置项集合，上游若新增了键会打 `::warning::`（只告警不阻断）。

`Cache/`、`Data/` 是空目录，git 存不了，由构建脚本 `mkdir -p` 生成。

## 校验值

| 文件 | 大小 | SHA256 |
|------|------|--------|
| `chrome++.ini` | 18814 | `fda720fcc521002c8577a7b400942a73e7a1525b12f1c809d02b300b07ea2227` |

## 升级方式

上游发新版时 CI 会自动跟随（版本号进入 tag `{helium}+{chrome++}`）。
只有当上游**新增配置项**（CI 会告警）时，才需要手动把新键补进这份 ini。

## 注意：不要改动编码

`chrome++.ini` 是 **UTF-16LE**（带 BOM `FF FE`，含 NUL 字节，行尾 `0D 00 0A 00`），
不是 UTF-8。`.gitattributes` 里已声明 `vendor/chrome_plus/** -text` 禁止 git 做换行符转换：

```gitattributes
vendor/chrome_plus/** -text
```

本机 `core.autocrlf=true`，如果不加这条，git 会把 `0A` 改写成 `0D 0A`，UTF-16 码元流被写坏；
而 Chrome++ 读配置是解析二进制流，坏了不会报错，只会悄悄退回默认配置——非常难查。
