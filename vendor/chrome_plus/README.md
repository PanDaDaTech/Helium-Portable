# Chrome++ 配置 + 完整包备份

这里放两样东西：

| 文件 | 用途 |
|------|------|
| `chrome++.ini` | **构建时使用**，覆盖上游包里的同名文件（带我们自己的 `command_line`） |
| `Chrome++_v<ver>_x86_x64_arm64.7z` | 上游完整包的**备份**，CI 在上游资产下载失败时回退使用 |

`version.dll` 仍然从上游 release 下载，不单独入库——它就在备份的完整包里面。

## 为什么要备份完整包

上游 `Bush2021/chrome_plus` 的账号在 2026-09 曾被平台冻结（整个 owner 变 404），
当时是靠手工把 `version.dll` 也塞进仓库才救回来的。现在改成：

- 仓库常备一份**当前版本**的完整包（约 190 KB）；
- CI 里 `Refresh vendored Chrome++ backup` 会在上游出新版时自动替换并提交回 `main`；
- `Fall back to vendored Chrome++ payload` 会在上游资产 404/限流时回退到备份，
  但**只接受版本号完全匹配的备份**，不会拿旧 DLL 冒充新版本。

注意：如果上游 owner 再次整体 404，`check` job 读不到最新版本就会直接报错停下
（这种情况需要人工介入，备份能让手工恢复很快，但不能自动继续）。

## 目录里的 ini 与上游的差异

这份 `chrome++.ini` 取自 Chrome++ 1.20.0 完整包，与上游 1.20.0 自带的
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

1. 下载上游 `Chrome++_v<ver>_x86_x64_arm64.7z`，取 `{x64,arm64}/App/version.dll`；
   下载失败则回退到本目录同版本号的备份包。
2. 用本目录的 `chrome++.ini` **覆盖**上游同名文件。
3. CI 会比对两份 ini 的配置项集合，上游若新增了键会打 `::warning::`（只告警不阻断）。
4. 上游若出了比备份新的版本，CI 会自动把新包替换进本目录并提交回 `main`。

`Cache/`、`Data/` 是空目录，git 存不了，由构建脚本 `mkdir -p` 生成。

## 校验值

| 文件 | 大小 | SHA256 |
|------|------|--------|
| `chrome++.ini` | 18814 | `fda720fcc521002c8577a7b400942a73e7a1525b12f1c809d02b300b07ea2227` |
| `Chrome++_v1.20.0_x86_x64_arm64.7z` | 198331 | `483e0c78e99e4ae57556623445c01f69c9c872e549baf86750b0feb080388400` |

备份包换版本后，表格里的名字与 SHA256 可以这样重算：

```bash
sha256sum vendor/chrome_plus/Chrome++_*.7z
```

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
