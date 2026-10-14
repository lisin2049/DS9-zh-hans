# SAOImageDS9 简体中文汉化包

Simplified Chinese Localization Pack for SAOImageDS9

为天文软件 **SAOImageDS9** 提供的简体中文界面语言包与中英对照文档（基于 8.8 版本）。
A Simplified Chinese UI language pack and bilingual (English/Chinese) documentation for the astronomy software SAOImageDS9 (based on v8.8).

## 内容 / Contents

| 目录 | 说明 |
|---|---|
| `msgs/zh.msg` | 完整简体中文界面语言包（1136 条，纯简体 UTF-8） |
| `doc/` | 参考手册 24 页 + 用户手册 + FAQ **中英对照**（英文原文一字未删改） |
| `docs/` | 联网功能使用说明 8 篇 + 《DS9 编译汉化经验备忘》 |
| `patches/` | 语言菜单显示"简体中文"的修正补丁（util.tcl） |

## 安装方法 / Installation

**1. 界面语言包（zh.msg）**

1. 关闭 DS9；
2. 把 `msgs/zh.msg` 复制到 DS9 安装目录的 `msgs/` 文件夹（覆盖同名文件）；
3. 重启 DS9 → **首选项 → 常规 → 字体 → 语言** → 选择 **"简体中文"**。

**2. 中英对照文档**

- 方式一：直接用浏览器打开 `doc/index.html`（参考手册）或 `doc/ref/index.html` 在线阅读；
- 方式二：把 `doc/` 整个文件夹覆盖到 DS9 安装目录的 `doc/`，然后从 DS9 的 **帮助 → 参考手册** 菜单打开。

**3. 语言菜单补丁（可选）**

仅当你从源码自编译 DS9、且语言菜单未显示"简体中文"时需要：按 `patches/` 中的补丁修改 `library/util.tcl` 的 `LanguageToName` 过程。

## 说明 / Notes

- 术语遵循中文天文界标准用法：星表（catalog）、分箱（binning）、十字丝（crosshair）、颜色映射（colormap）、灰度拉伸（scale）等；
- 中英对照文档的英文原文与上游保持逐字一致（曾对照英文原版逐节点校验，英文 0 缺失）；
- 界面语言包与文档双份同步（安装副本 + 源码树），重新编译/安装不会丢失汉化。

## 许可 / License

GPL-3.0，与上游 SAOImageDS9 一致。本仓库内容为上游项目的衍生作品（翻译与文档）。
Derivative work of SAOImageDS9 under GPL-3.0.

## 上游 / Upstream

- 项目主页：https://ds9.si.edu/
- 源码仓库：https://github.com/SAOImageDS9/SAOImageDS9
