# DS9（SAOImageDS9）编译 + 汉化 · 经验备忘

> **用途**：给未来的自己读。遇到 DS9 相关任务先看这份文档，避免重踩同一个坑。
> **最后更新**：2026-10-14（文档中文化完成日）
> **项目根**：`F:\openAI\SAOImageDS9`（从官方 GitHub 克隆的源码树，版本 8.8）

---

## 0. 一句话总结

- **编译**：DS9 在 Windows 上用 MSYS2（ucrt64 gcc）成功编译，产物 `bin\SAOImageDS9\ds9.exe` 可运行。官方只支持 Cygwin，MSYS2 需要绕开 pacman（完全不可用）手动装依赖，并修复若干 Makefile/链接问题。
- **汉化**：DS9 用 Tcl `msgcat` 国际化，界面文字全是 `[msgcat::mc {…}]` 外置语言包。**重写 `msgs\zh.msg`（1136 条）+ 修好 win32 语言检测**（读注册表 LocaleName）→ 全中文界面。没有改任何英文硬编码，可随时切回英文。

---

## 1. 项目概况

| 项 | 值 |
|---|---|
| 可运行程序 | `F:\openAI\SAOImageDS9\bin\SAOImageDS9\ds9.exe`（约 23.5 MB，双击即启动） |
| 界面脚本目录 | `bin\SAOImageDS9\library\`（750 个 .tcl） |
| 消息语言包目录 | `bin\SAOImageDS9\msgs\`（cs/da/de/es/fr/ja/pt/zh.msg） |
| 构建日志 | `build17/18/20/21.log`（build21 为 ds9-only 成功轮），位置见当时工作目录 |
| 用户环境 | Windows + MSYS2 `C:\msys64`，ucrt64 gcc 15.2.0；中文系统（zh-CN） |

---

## 2. 构建环境（重要前提）

- DS9 官方只支持 Cygwin。本机没有 Cygwin，改用 **MSYS2**。
- **`pacman` 完全不可用**（进程空转卡死，无论什么命令）——**这是死路，不要浪费时间重试**。
- 装依赖的唯一有效通道：从 `https://mirror.msys2.org` 手动下载 `xxx.pkg.tar.zst`，再用：
  ```bash
  bsdtar -xf xxx.pkg.tar.zst -C C:/msys64
  ```
- 这样装过的包（按需补充）：`make-4.4.1-2`、`diffutils`、`zip`、`unzip`、`pkgconf`×2、`libxml2`、`libxslt`、交叉前缀 `binutils`、`pkg-config`。

---

## 3. 编译踩坑清单（按解决顺序）

### 3.1 内存/虚拟内存崩溃（现象最吓人的一类）

**现象**：构建反复报 `Win32 error 1455` / `error 299` / `Resource temporarily unavailable`，严重时**系统死锁数小时**（连鼠标都卡）。Tcl/Tk 顶层 Makefile 内部**硬编码 `-j $(JOBS)` 并行**，fork 风暴把提交内存顶满。

**解决**：
- 去掉/调低并行（Tcl/Tk 的 Makefile 里把 `-j` 相关逻辑去掉，用单线程 make 编译这些子项目）。
- 用户把 **C 盘页面文件关闭、全部转到 F 盘**（`F:\pagefile.sys`，初始 32 GB）后，内存类崩溃大幅减少。
- 编译大工程时**预留充足虚拟内存**，别在页面文件很小的情况下跑。

### 3.2 MSYS2 路径转换 / 转义问题

**现象**：`OBJPATH` 之类的路径含 `\o` 等无效转义（MSYS2 会把反斜杠路径转义吃掉），编译/链接找不到库。

**解决**：改成转义引号 + 正斜杠：
```makefile
OBJPATH = "F:/openAI/SAOImageDS9/lib:."
```
涉及文件：`funtools/filter/Makefile`、`funtools/fitsy/Makefile`（**每次重新 configure/clean 后要重新检查**）。

### 3.3 Tcl/Tk 的 install-doc 反复重装崩溃

**现象**：`install-doc` 是 phony 目标，make 每次都会重装文档，装到一半崩溃/重复装。

**解决**：从 `tcl9.0/win/Makefile` 和 `tk9.0/win/Makefile` 的 `install` 目标里**剔除 install-doc**。

### 3.4 链接失败（决定性修复，务必记住）

**现象**：ds9 链接报 `hist.C undefined reference`，缺 `ft_headinit / FilterOpen / FilterClose / FilterEvents`。

**根因**：`funtools` 的 `filter`（13 个 .o）与 `fitsy`（23 个 .o）两个子目录的对象**从未被打包进主库 `libfuntools.a`**——make 认为库已最新、跳过了 ar。

**解决**（手动把对象归档进库）：
```bash
cd funtools/filter && ar cruv ../libfuntools.a *.o
cd funtools/fitsy && ar cruv ../libfuntools.a *.o
ranlib ../libfuntools.a
# 成功后库应有 87 个成员、约 4.26 MB
Copy-Item funtools/libfuntools.a lib/libfuntools.a -Force   # 同步到 lib/
```
**⚠️ 教训**：这种"库对象缺失"问题 make 永远不会自己发现（认为库最新）。**每次重建 funtools 后都必须重做上面的 ar + 拷贝**，否则 ds9 又链接失败。

### 3.5 DLL 部署

**现象**：按 Cygwin 路径复制 DLL 失败。

**解决**：`ds9/win/Makefile` 的 `DLLDIR` 改为 `/c/msys64/ucrt64/bin`；文件名用**实际存在**的（`libxml2-16.dll`、`libiconv-2.dll`），删掉不需要的 `liblzma-5.dll`。

---

## 4. 汉化：方案与踩坑（本次主任务）

### 4.1 DS9 的国际化机制（为什么能不动代码汉化）

- 界面文字全部包在 `[msgcat::mc {File}]` 这类调用里，英文是默认值（fallback）。
- 运行时 `msgcat::mc` 查"当前语言"的翻译表：命中返回翻译，未命中返回英文原文。
- 语言包 = `msgs\<lang>.msg`，格式（Tcl 9）：
  ```tcl
  ::msgcat::mcset zh {File} {文件}
  ```
- 原版有 8 种语言：cs/da/de/es/fr/ja/pt/zh。**原版 zh.msg 质量极差**：约一半没翻译、一半是 big5 繁体乱码，可用条目不足 200 → 必须整体重写。

### 4.2 汉化三步

1. **提取全部键**：从 750 个 .tcl 里正则提取 `msgcat::mc {…}` → 1178 条唯一键（`ds9_keys.txt`）。
2. **手写翻译字典**：`C:\Users\wang\AppData\Local\Temp\ds9_zh_dict.py`（Python 字典，1178 键全覆盖，中文值），生成脚本 `gen_zh_msg.py` → 输出 `zh.msg`（1136 条，UTF-8，`\n` 换行）。
3. **修语言检测**（见 4.4）。**原 zh.msg 备份为 `zh.msg.orig`**。

**验证语言包本身可用**（不依赖 GUI）：
```bash
cd F:/openAI/SAOImageDS9/bin && ./tclsh90s.exe test_zh.tcl
# test_zh.tcl 里 msgcat::mclocale zh; msgcat::mcload {F:/openAI/SAOImageDS9/bin/SAOImageDS9/msgs}; 再查 mc {File} 等
```

### 4.3 语言切换失败的根因（两个，都别忘）

**根因 1：DS9 进程内 `msgcat::mclocale` 返回 `c`**
- GUI 程序没有 `LANG`/`LC_ALL` 环境变量，Tcl 拿到的系统 locale 是 `c` → `SetLanguage c` 失败 → 回退 en。
- 在 `tclsh90s.exe` 里测试却返回 `zh_cn`（有终端环境），**所以别用 tclsh 的结果推断 ds9 进程**。

**根因 2：`package require registry` 在 ds9.exe 里不可用**
- 我曾试图用 Tcl registry 扩展读注册表，但 **registry 扩展没有编译进 ds9.exe**（`info loaded` 里没有），静默 catch 失败 → 又回退 en。
- tclsh90s.exe 有 registry 扩展，**再次证明不能用 tclsh 推断 ds9**。

### 4.4 最终修复（现在生效的版本）

`bin\SAOImageDS9\library\ds9.tcl` 的 win32 语言初始化分支（`switch $pds9(language)` → locale → win32）：

```tcl
win32 {
    # use the Windows display language (LocaleName, e.g. zh-CN -> zh)
    set ll en
    catch {
        set out [exec reg query {HKEY_CURRENT_USER\Control Panel\International} -v LocaleName]
        foreach line [split $out "\n"] {
            if {[regexp {REG_SZ\s+(\S+)} $line -> loc]} {
                set ll [string tolower [string range $loc 0 1]]
            }
        }
    }
    if {![SetLanguage $ll]} {
        SetLanguage en
    }
}
```

**原理**：调用系统 `reg.exe`（不依赖 Tcl 扩展）读 `LocaleName`（中文系统返回 `zh-CN`）→ 取前两位 `zh` → `SetLanguage zh` → 加载 `msgs/zh.msg`。

### 4.5 调试技巧（GUI 程序 stdout 会丢）

- **GUI/Tk 程序启动后，`puts stdout`/`puts stderr` 会被 Tk 接管而丢失**——我在 ds9.tcl 里插的调试 puts 大多看不见。
- **可靠方法：写文件探针**：
  ```tcl
  catch {set f [open {F:/openAI/SAOImageDS9/probe.txt} w]; puts $f "……"; close $f}
  ```
  用文件是否存在/内容判断代码执行到哪一行。本次就是靠 probe1/2/3 定位到"win32 分支执行了但 SetLanguage 失败"。
- 启动诊断：
  ```bash
  cd /f/openAI/SAOImageDS9/bin/SAOImageDS9 && ./ds9.exe > /tmp/ds9_stdout.log 2>&1 &
  # 日志在 C:\msys64\tmp\ds9_stdout.log
  ```
- 杀进程：`Stop-Process -Name ds9 -Force`（进程名就是 `ds9`）。
- **验证界面是否汉化**：PrintWindow 按句柄截图（全屏截图若窗口不在前台会截到壁纸），再用 OCR 读菜单文字。

---

## 5. 关键文件与备份清单

| 路径 | 说明 |
|---|---|
| `bin\SAOImageDS9\ds9.exe` | 编译成品（可运行） |
| `bin\SAOImageDS9\msgs\zh.msg` | **新版中文包（1136 条，UTF-8）** |
| `bin\SAOImageDS9\msgs\zh.msg.orig` | 原版中文包备份 |
| `bin\SAOImageDS9\library\ds9.tcl` | 唯一改过的 tcl：win32 语言检测分支 |
| `ds9\msgs\zh.msg`（源码树） | 已同步新版中文包；原版备份 `zh.msg.orig`——**重新 make/install 不再覆盖汉化** |
| `funtools\libfuntools.a` / `lib\libfuntools.a` | 87 成员 4.26 MB（已手工 ar 补全） |
| `C:\Users\wang\AppData\Local\Temp\ds9_zh_dict.py` 等 | 翻译字典/生成脚本/验证脚本（可重新生成，不必长期保留） |
| `%APPDATA%\.ds9\ds9.8.8.init` | 用户偏好文件（language 设置在此；默认没生成） |

---

## 6. 验证方法（交付前必做）

1. **启动**：双击 `ds9.exe` 或命令行启动，进程持续运行、窗口标题 "SAOImage ds9"。
2. **汉化**：窗口菜单应为 文件/编辑/视图/帧/像素合并/缩放/灰度拉伸/颜色/区域/WCS/插图/分析/帮助；按钮 打开/保存/打印/退出。
3. **语言包独立验证**：tclsh90s + test_zh.tcl（见 4.2）。
4. **切换回英文**：编辑 → 首选项 → 常规 → 字体 → 语言 → English → 保存 → 重启（确认汉化没有破坏原功能）。

---

## 7. 未来注意事项 / 死路清单

- ❌ **pacman 任何命令都别试**（死路）。
- ⚠️ **重新 make/install DS9 前**：确认 `funtools` 是否重建过——若重建，必须重做 3.4 的 `ar + Copy-Item`，否则链接必失败。
- ⚠️ **重新 configure 后**：检查 `funtools/filter`、`funtools/fitsy` 的 `OBJPATH` 转义（3.2）与 `ds9/win/Makefile` 的 DLLDIR（3.5）是否还在。
- ⚠️ **重新构建会覆盖安装副本里的汉化**：`bin\SAOImageDS9\msgs\zh.msg` 和 `library\ds9.tcl` 是安装时从源码树拷过去的；源码树已同步汉化（见第 5 节），正常重装即可保留。
- 📌 **汉化原理**：不依赖改英文硬编码；`zh.msg` + win32 语言检测 = 全部。想改某条翻译，直接改 `zh.msg` 里对应 `mcset zh {键} {值}`，重启生效。
- 📌 DS9.exe 内嵌了 Tcl zipfs 虚拟文件系统（PE 里 PK 头在 offset 6795633），但 **win32 实际从文件系统加载 library**（`info script` 证实），不用管 zipfs。
- 📌 `%APPDATA%\.ds9\ds9.8.8.init` 里可手动改 `language`，命令行也能 `ds9.exe -language zh`。

---

## 8. 帮助文档中文化（2026-10-14 完成）

**范围**（用户选定）：**用户手册 + FAQ + 参考手册**（`doc/ref`，24 个 html，中英对照，2026-10-14 晚完成）；版本说明（`doc/release`）不翻。参考手册译文同时同步到安装副本 `bin\SAOImageDS9\doc\ref\` 与源码树 `ds9\doc\ref\`（重装不丢）。

| 项 | 值 |
|---|---|
| 翻译方式 | **中英对照**：英文原版一字不删，译文紧跟其后，用 `.zh` 类样式区分 |
| 涉及文件 | `doc\user\` 下 7 个 index.html（主页/gui/binning/rgb/catalogs/contour/scripts）+ 根 `doc\faq.html` + **`doc\ref\` 参考手册全部 24 个 html** |
| 英文原版备份 | `F:\openAI\SAOImageDS9\doc_en_backup\`（全部 61 个 html，含 ref/release，保险起见都备了） |
| 译文位置 | `bin\SAOImageDS9\doc\` 对应路径 + **源码树 `ds9\doc\ref\` 已同步**（重装不丢；用户手册/FAQ 当时只改了安装副本） |

**参考手册（doc/ref）验收结论（2026-10-14 晚）**：24/24 文件 `charset=utf-8` 无乱码；标签配对无翻译引入问题（command/xpa/samp/index/cloud 的少量不平衡为**英文原版固有**）；英文文本节点 **0 缺失**（xpa.html 做过"移除译文后与备份逐字节一致"的最强验证）；说明段中文全覆盖，命令语法块按规范不译；安装副本与源码树 `ds9\doc\ref\` **逐字节一致**。catalog.html 曾发现 `&nbsp;` 对齐数量与原文不符，已从备份原样提取修正。

**样式规范**（继续翻新文件时照抄，保持统一）：
- `<head>` 里 charset 改为 `utf-8`（原 `windows-1252` / `us-ascii` **必须改**，否则中文乱码）；
- `<title>` 附中文，如 `<title>Contours - DS9（等高线）</title>`；
- 每个英文 `<p>` 后紧跟 `<p class="zh">中文译文</p>`；标题 `<h1/h2/h3>` 后加 `<span class="zh">中文</span>`；列表项用 `<div class="zh">` 内嵌；
- `<head>` 内置统一 CSS：`.zh { color:#333; border-left:3px solid #ccc; padding-left:8px; font-family:"Microsoft YaHei","PingFang SC",sans-serif; }`
- 命令块（`<pre>`）/表格关键字**不翻译**，必要时在块后补一句中文说明。

**验证（已完成）**：
1. 8 个文件 charset 全部 UTF-8 ✓
2. 标签配对完整性脚本通过 ✓
3. **英文原版内容级保留**：脚本逐条核对备份英文节点（共 1007 条），全部以子串形式存在于翻译版 ✓

**注意**：DS9 帮助菜单→用户手册/常见问题，实际打开的就是 `doc\user\index.html` / `doc\faq.html`，路径没变，直接生效；重新 make/install 会从源码树覆盖 `doc\` 安装副本，把已翻译文件**再拷回** `doc_en_backup` 对照即可（或先同步回源码树）。

---

## 9. 附：无关项目插曲（REA）

- GitHub 热门逆向工程项目 **rea-agents**（用户要求克隆到 `F:\openAI\rea` 并按 README 装 Node/rea-agents 6.4.0）。
- CLI 可用，但 **Windows 原生组件被上游 bug 阻塞**（issue #1701 P1、#2196），**是上游 Windows bug 不是平台不支持**。用户当时一句"白忙活了是吧"已澄清为部分进展。若日后继续，先查这两个 issue 是否修复。

---

*本备忘的验证基准：2026-10-14，DS9 8.8 中文界面截图确认（菜单全中文）。*

## 术语修订（2026-10-15，最终定稿，参考手册 doc/ref 24 页）

术语策略定为：**按上下文语境灵活翻译，天文软件优先使用天文标准术语，不与界面菜单词机械统一**。
先执行过"对齐界面语言包"（方案B），后按用户意见回退修正，最终状态：
- magnifier -> 放大镜（修正原"放大器"误译；Amplifier 坐标系"放大器坐标"3 处保留不动：file.html ATV/ATM、framecmd.html amplifier）
- crosshair -> 十字丝（天文标准术语，原"十字丝/十字准线/十字线"混用，统一为天文术语"十字丝"，不迁就界面"十字线"）
- binning -> 分箱（天文数据处理标准术语，不迁就界面 Bin 菜单"像素合并"）
- panner -> 全景导航器（UI 控件名，与界面 zh.msg 一致，便于用户对照操作；"导览器"欠准）
- FITS/数据 header -> 文件头（天文软件标准说法）；HTTP header -> 报头（cloud.html 1 处）
- 保留不动：catalog.html"表头"3 处（CSV/星表列头，非 FITS header）；colorbar 本就是统一"颜色条"（原统计"色标/色条"是"颜色标记/颜色条"子串误报）
- 最终共 85 处中文替换（放大镜 11、十字丝 16、全景导航器 11、分箱 47、文件头 15、报头 1，含一次回退修正 63 处）
- 安装副本 + 源码树 ds9/doc/ref 双份逐字节一致（SHA256 24/24 相同）；英文原文 0 改动；编码 UTF-8 无 BOM 不变
- 可复现脚本：项目根 term_unify.py（需按最终映射调整后重跑）