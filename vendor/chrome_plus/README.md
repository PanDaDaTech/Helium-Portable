# Chrome++ (vendored)

Chrome++ 的便携化载荷。放在这里是为了让 CI 不再依赖上游。

## 为什么 vendored

上游仓库 `Bush2021/chrome_plus` 的**账号已被平台冻结**，整个 owner 变 404，仓库与
release 都不可访问。原来的 CI 会在 check 阶段执行
`gh release view --repo Bush2021/chrome_plus`，上游一失效这个 job 就直接失败
（报 `release not found`，退出码 1）。

因此把 Chrome++ 的载荷提取出来提交进仓库，构建时只读本地文件。

## 文件来源

从本仓库**自己已发布的 release** 里反向提取，确保与线上产物完全一致：

- 来源 release：`0.18.1.1+1.18.2`（Helium `0.18.1.1` + Chrome++ `1.18.2`）
- 资产：`helium_0.18.1.1+1.18.2_x64-windows.7z` / `..._arm64-windows.7z`
- 提取路径：包内 `App/chrome++.ini`、`App/version.dll`

Chrome++ 的完整包在便携包里的落点只有这三处（`Cache/`、`Data/` 是空目录，git 存不了，
由构建脚本 `mkdir -p` 生成）。

## 校验值（SHA256）

| 文件 | 大小 | SHA256 |
|------|------|--------|
| `x64/App/chrome++.ini` | 16088 | `fc632b74ccbcc0a0fc9ea8da99e671f5802bd71929e1977b2c4da7d99c3620ed` |
| `x64/App/version.dll` | 185856 | `c5619037adb4222fe7b750e5c2463dbce6248a1e1e7381e0d8653d6e10732c60` |
| `arm64/App/chrome++.ini` | 16088 | `fc632b74ccbcc0a0fc9ea8da99e671f5802bd71929e1977b2c4da7d99c3620ed` |
| `arm64/App/version.dll` | 143872 | `9461473933c0e3036e4e2b92e98a20832964fcf229d58cbfff3077f602c80ed8` |

- `chrome++.ini` 是架构无关的配置文件，x64 / arm64 两份内容完全相同。
- `version.dll` 已确认是合法 PE 且架构正确：x64 `machine=0x8664`，arm64 `machine=0xaa64`。

## 升级方式

上游已停更，升级只能手动进行：

1. 拿到新版 Chrome++ 的 `App/chrome++.ini` 与 `App/version.dll`（x64 / arm64 各一份）。
2. 覆盖 `vendor/chrome_plus/<arch>/App/` 下对应文件。
3. 更新 `VERSION`（CI 的 tag 形如 `{helium}+{chrome++.版本}`，所以这个数字必须同步改）。
4. 提交并推送；CI 会判定 `chrome_plus changed` 并重新出包。

## 注意：不要改动这几个文件的编码

`chrome++.ini` 是 **UTF-16LE**（带 BOM `FF FE`，含 NUL 字节，行尾 `0D 00 0A 00`），
不是 UTF-8。`.gitattributes` 里已声明 `vendor/chrome_plus/** -text` 禁止 git 做换行符转换：

```gitattributes
vendor/chrome_plus/** -text
```

本机 `core.autocrlf=true`，如果不加这条，git 会把 `0A` 改写成 `0D 0A`，UTF-16 码元流被写坏；
而 Chrome++ 读配置是解析二进制流，坏了不会报错，只会悄悄退回默认配置——非常难查。

**新增任何 vendored 二进制或非 UTF-8 文件时，记得一并纳入这条规则。**
