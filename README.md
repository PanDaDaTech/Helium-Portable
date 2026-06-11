# Helium Browser Portable Edition

![GitHub Actions Workflow Status](https://img.shields.io/github/actions/workflow/status/Silverwolf-x/helium-plus/main.yml?event=schedule&label=UTC%2000%3A00%20schedule%20build)
![GitHub Release](https://img.shields.io/github/v/release/Silverwolf-x/helium-plus?include_prereleases&display_name=release&logo=googlechrome&label=Helium)
![GitHub Release](https://img.shields.io/github/v/release/Bush2021/chrome_plus?display_name=release&logo=github&label=Chrome%2B%2B)

Thanks to the contributions of the [Helium Browser](https://github.com/imputnet/helium-windows) and [Chrome++](https://github.com/Bush2021/chrome_plus) projects.

Helium is a privacy-focused Chromium-based browser that respects user freedom while maintaining full compatibility with Chrome extensions and web standards. However, the official Windows builds do not come in a portable format and store data in system folders like `C:\Users\[Name]\AppData\Local`.

By integrating [Chrome++](https://github.com/Bush2021/chrome_plus) — a DLL injection project that enables full portability and tab enhancements — this project packages Helium as a truly portable browser that stores all data in its own directory.

**Features:**
- Portable configuration with customizable data and cache directories
- Tab enhancements (double-click to close, mouse wheel switching, etc.)
- Hotkey remapping and boss key support
- No system installation required — runs from any folder
- Daily automated builds via GitHub Actions

**Update Cycle:**
* **Release** versions follow the update cycle of Chrome++ (stable enhancements)
* **Pre-releases** follow the update cycle of Helium Browser (latest Chromium features)
* The workflow runs daily at 08:00 (UTC+8); actual execution may be delayed depending on GitHub Actions queue

---

## License

This project is released under the GPL-3.0 license. See the [LICENSE](LICENSE) file for details.

Both source projects are licensed under GPL-3.0:
- [Helium Windows](https://github.com/imputnet/helium-windows) — GPL-3.0
- [Chrome++](https://github.com/Bush2021/chrome_plus) — GPL-3.0

This integration work preserves and respects those licenses.

