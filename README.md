# Helium Browser Portable Edition

![GitHub Actions Workflow Status](https://img.shields.io/github/actions/workflow/status/Silverwolf-x/helium-plus/main.yml?event=schedule&label=daily%20build)
![GitHub Release](https://img.shields.io/github/v/release/Silverwolf-x/helium-plus?display_name=release&logo=googlechrome&label=Helium%2B%2B)
![GitHub Release](https://img.shields.io/github/v/release/Bush2021/chrome_plus?display_name=release&logo=github&label=Chrome%2B%2B)

基于 [Helium Browser](https://github.com/imputnet/helium-windows) 和 [Chrome++](https://github.com/Bush2021/chrome_plus) 的便携版浏览器。

Helium 是注重隐私的 Chromium 分支，完全兼容 Chrome 扩展。官方 Windows 构建不是便携模式，数据存储在 `C:\Users\[Name]\AppData\Local`。本项目通过集成 Chrome++（DLL 注入实现便携化和标签页增强），将 Helium 打包为真正的便携浏览器——所有数据存放在自身目录内。

**功能：**
- 便携配置，Data 和 Cache 目录可自定义
- 标签页增强（双击关闭、鼠标滚轮切换等）
- 快捷键重映射、老板键
- 无需安装，任意文件夹运行
- 仅保留 zh-CN 和 en-US 语言包，精简体积
- GitHub Actions 每日自动构建

**更新周期：**
- 每天 08:00（UTC+8）检查 [Helium Windows](https://github.com/imputnet/helium-windows) 是否有新 release
- 检测到新版本时自动构建并发布（同时拉取最新 Chrome++）
- 手动触发时构建为 draft release
- 版本号格式：`{helium_version}+{chrome_plus_version}`

---

## License

GPL-3.0. See [LICENSE](LICENSE).

Both source projects are GPL-3.0:
- [Helium Windows](https://github.com/imputnet/helium-windows)
- [Chrome++](https://github.com/Bush2021/chrome_plus)

