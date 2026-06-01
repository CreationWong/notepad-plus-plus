# Notepad++ 完整离线审计报告

生成日期: 2026-06-01

范围:
- 运行时代码: `PowerEditor/src`
- 打包与安装程序: `PowerEditor/installer`
- CI 与发布工作流: `.github/workflows`, `appveyor.yml`
- 含有大量 URL 字面量的参考/文档树: `scintilla`, `lexilla`, `README.md`, `BUILD.md`

## 1. 执行摘要

关键结论:

1. 在 `PowerEditor/src`、`lexilla` 或 `scintilla` 中，未发现直接使用 WinInet / WinHTTP / `URLDownloadToFile` / Winsock 客户端 API。
2. 从该仓库来看，Notepad++ 主进程似乎没有实现自己的 HTTP 客户端。
3. 实际的在线行为主要由以下途径引入:
   - 通过 `ShellExecute` 启动外部网页 URL
   - 启动外部更新程序 `GUP.exe`
   - Plugin Admin 将安装/更新工作委托给 `GUP.exe`
4. 该仓库在文档、注释、测试、翻译和随附示例中包含大量 URL 字符串。这些属于噪声命中，不等同于运行时网络行为。

grep 得到的原始 URL 命中量:

- `PowerEditor/src`: 约 825 处类似 URL 的命中
- `PowerEditor/installer`: 约 253 处类似 URL 的命中
- `.github` + `appveyor.yml`: 约 15 处类似 URL 的命中
- `scintilla` + `lexilla` + 顶层文档: 约 3130 处类似 URL 的命中

重要说明:

- `GUP.exe` 以及更新器真实的网络实现不在本源码树中。
- 该仓库只包含针对 `GUP.exe` 的集成点和打包引用。
- 因此，如果从产品层面要求“完全离线”，仍需审计随产品分发的更新器二进制文件和插件列表制品。

## 2. 高优先级运行时在线入口点

### P0. 通过外部 `GUP.exe` 实现自动更新

证据:

- `PowerEditor/src/Parameters.h:813`
- `PowerEditor/src/Parameters.h:816`
- `PowerEditor/src/winmain.cpp:380`
- `PowerEditor/src/winmain.cpp:793`
- `PowerEditor/src/winmain.cpp:866`
- `PowerEditor/src/Parameters.cpp:8907`
- `PowerEditor/src/resource.h:33`
- `PowerEditor/src/resource.h:34`

其行为:

- 自动更新模式默认值为 `autoupdate_on_startup`。
- 在启动或退出时，Notepad++ 可能会在满足以下条件时启动 `GUP.exe`:
  - 更新器存在
  - 操作系统新于 XP
  - 签名校验通过
  - 自动更新已启用
- 更新器参数包括:
  - `INFO_URL = https://notepad-plus-plus.org/update/getDownloadUrl.php`
  - `FORCED_DOWNLOAD_DOMAIN = https://github.com/notepad-plus-plus/notepad-plus-plus/`
- 在更新器执行抓取/下载前，会先配置证书检查。

离线影响:

- 这是最重要的运行时在线路径。
- 即使主 EXE 本身没有实现 HTTP，它仍会显式启动一个具备联网能力的外部更新器。

离线化措施:

1. 移除更新器的打包与随产品分发。
2. 强制将自动更新模式设为禁用。
3. 移除更新菜单项以及更新器代理配置项。
4. 从启动与退出流程中移除更新器启动路径。

### P0. Plugin Admin 安装/更新/删除路径

证据:

- `PowerEditor/src/WinControls/PluginsAdmin/pluginsAdmin.cpp:260`
- `PowerEditor/src/WinControls/PluginsAdmin/pluginsAdmin.cpp:287`
- `PowerEditor/src/WinControls/PluginsAdmin/pluginsAdmin.cpp:362`
- `PowerEditor/src/WinControls/PluginsAdmin/pluginsAdmin.cpp:368`
- `PowerEditor/src/WinControls/PluginsAdmin/pluginsAdmin.cpp:371`
- `PowerEditor/src/WinControls/PluginsAdmin/pluginsAdmin.cpp:658`
- `PowerEditor/src/WinControls/PluginsAdmin/pluginsAdmin.cpp:691`

其行为:

- Plugin Admin 同时依赖以下两项:
  - 插件列表制品: `nppPluginList.dll` 或调试版 `nppPluginList.json`
  - 更新器二进制文件: `GUP.exe`
- 安装/更新操作会准备包含插件目录、仓库和 ID 的参数，然后退出 Notepad++ 并将工作交给 `GUP.exe`。
- 插件元数据包含仓库和主页字段。

离线影响:

- 插件安装与更新路径并不适合离线场景。
- 即使插件列表是本地的，实际的插件获取/更新仍然委托给更新器路径处理。

离线化措施:

1. 在离线构建中移除 Plugin Admin 菜单和对话框。
2. 停止为离线版本打包插件列表制品。
3. 移除插件操作对 `GUP.exe` 的依赖。

## 3. 中优先级运行时在线入口点

### P1. 手动“Search on Internet”

证据:

- `PowerEditor/src/NppCommands.cpp:820`
- `PowerEditor/src/NppCommands.cpp:838`
- `PowerEditor/src/NppCommands.cpp:843`
- `PowerEditor/src/NppCommands.cpp:847`
- `PowerEditor/src/NppCommands.cpp:851`
- `PowerEditor/src/NppCommands.cpp:855`
- `PowerEditor/src/Parameters.h:857`

其行为:

- 菜单命令 `Search on Internet` 会用选中文本拼接 URL。
- 默认搜索引擎是 Google。
- 其他内置选项包括 DuckDuckGo、Yahoo 和 Stack Overflow。
- 允许自定义搜索引擎 URL。

离线影响:

- 这是一个由用户触发的在线浏览入口点。

离线化措施:

1. 移除 `IDM_EDIT_SEARCHONINTERNET`。
2. 如果要求严格离线行为，则移除搜索引擎偏好设置 UI 及其配置处理逻辑。

### P1. 编辑器文本中检测到的可点击 URL

证据:

- `PowerEditor/src/NppNotification.cpp:392`
- `PowerEditor/src/Parameters.h:782`
- `PowerEditor/src/Parameters.h:783`
- `PowerEditor/src/Parameters.h:699`
- `PowerEditor/src/ScintillaComponent/Buffer.cpp:784`

其行为:

- 双击检测到的 URL 会触发 `ShellExecute(..., "open", url, ...)`。
- 支持的 scheme 范围不仅限于 HTTP，还包括 `ssh://`、`sftp://`、`slack://`、`steam://`、`spotify:` 等。
- 大文件限制下会设置 `_allowClickableLink = false`，但这只在大文件限制模式启用时生效。
- 普通文件仍然可以打开外部 URL 或应用处理器。

离线影响:

- 这不会让 Notepad++ 自身变成网络客户端。
- 但它仍是一个可直接启动浏览器或其他具备联网能力处理器的入口点。

离线化措施:

1. 禁用可点击链接的热点行为。
2. 可选地清空或收窄 `_uriSchemes`。
3. 从通知处理路径中移除 URL 打开逻辑。

### P1. About / Help / Website 菜单项

证据:

- `PowerEditor/src/NppCommands.cpp:3746`
- `PowerEditor/src/NppCommands.cpp:3751`
- `PowerEditor/src/NppCommands.cpp:3757`
- `PowerEditor/src/NppCommands.cpp:3769`
- `PowerEditor/src/NppCommands.cpp:3775`
- `PowerEditor/src/Notepad_plus.rc:1350`
- `PowerEditor/src/Notepad_plus.rc:1351`
- `PowerEditor/src/Notepad_plus.rc:1352`
- `PowerEditor/src/Notepad_plus.rc:1353`
- `PowerEditor/src/Notepad_plus.rc:1355`
- `PowerEditor/src/Notepad_plus.rc:1356`

其行为:

- 会打开:
  - Notepad++ 首页
  - 项目 GitHub 页面
  - 在线用户手册
  - 社区论坛
  - 更新入口
  - 更新器代理配置

离线影响:

- 这些是主菜单中的用户触发在线入口点。

离线化措施:

1. 在离线构建中移除在线帮助和网站菜单项。
2. 完全移除更新/代理项，而不是仅在更新器不存在时隐藏。

## 4. 低优先级运行时在线入口点

### P2. User Defined Language 在线帮助

证据:

- `PowerEditor/src/ScintillaComponent/UserDefineDialog.cpp:181`

其行为:

- UDL 对话框中包含一个硬编码的在线手册链接。

离线化措施:

- 替换为本地帮助，或直接移除链接。

### P2. Plugin Admin 仓库链接

证据:

- `PowerEditor/src/WinControls/PluginsAdmin/pluginsAdmin.cpp:228`

其行为:

- 打开 GitHub 上的插件列表仓库页面。

离线化措施:

- 移除该链接，或替换为本地元数据路径。

### P2. 面向互联网的默认用户命令

证据:

- `PowerEditor/src/MISC/Common/NppConstants.h:444`
- `PowerEditor/src/MISC/Common/NppConstants.h:445`
- `PowerEditor/src/MISC/Common/NppConstants.h:448`
- `PowerEditor/src/MISC/Common/NppConstants.h:449`

其行为:

- 默认用户自定义命令模板包含:
  - 在互联网上查看 PHP 帮助
  - 搜索 Wikipedia

离线化措施:

- 从离线版本模板中移除这些默认项。

## 5. 打包与安装程序中的在线内容

### P1. 更新器被显式打包

证据:

- `PowerEditor/installer/msi/wingup.wxs:4`
- `PowerEditor/installer/msi/wingup.wxs:8`
- `PowerEditor/installer/nsisInclude/binariesComponents.nsh:69`
- `PowerEditor/installer/nsisInclude/binariesComponents.nsh:79`
- `PowerEditor/installer/nsisInclude/binariesComponents.nsh:85`
- `PowerEditor/installer/nsisInclude/binariesComponents.nsh:91`
- `PowerEditor/installer/packageAll.bat:343`
- `PowerEditor/installer/packageAll.bat:359`
- `PowerEditor/installer/packageAll.bat:375`

其行为:

- MSI 和 NSIS 都会打包更新器载荷。
- 便携版打包也会复制更新器文件。

离线化措施:

- 从 MSI、NSIS 和便携版打包定义中移除更新器组件。

### P1. 安装程序在不支持的平台场景下会打开网站

证据:

- `PowerEditor/installer/nsisInclude/tools.nsh:161`
- `PowerEditor/installer/nsisInclude/tools.nsh:169`
- `PowerEditor/installer/nsisInclude/tools.nsh:181`

其行为:

- 当操作系统/架构不受支持时，安装程序可能会打开 Notepad++ 下载页面。

离线化措施:

- 改为仅显示离线提示信息。
- 不要从安装程序启动浏览器。

### P2. 可通过配置制品禁用自动更新，但更新器仍会随产品分发

证据:

- `PowerEditor/installer/xml4Config/disableNppAutoUpdate.xml:2`
- `PowerEditor/installer/xml4Config/disableNppAutoUpdate.xml:3`

含义:

- 当前打包流程已经识别了一个“禁用自动更新”开关。
- 但这还不足以实现完全离线化，因为更新器及相关 UI 仍然存在。

## 6. 构建 / CI / 发布流水线的在线依赖

这些不是终端用户运行时网络行为，但如果你希望实现完全离线的构建/发布流水线，它们仍然重要。

### P1. CI 访问 PyPI 并安装 Python 包

证据:

- `.github/workflows/CI_build.yml:89`
- `.github/workflows/CI_build.yml:90`
- `.github/workflows/CI_build.yml:91`
- `.github/workflows/CI_build.yml:92`
- `.github/workflows/CI_build.yml:112`
- `.github/workflows/CI_build.yml:130`

其行为:

- 查询 PyPI 的版本元数据。
- 通过 `pip` 安装 `requests`、`rfc3987`、`pywin32`、`lxml`。

离线化措施:

1. 将这些 Python 依赖 vendoring 到仓库中，或在内部建立镜像。
2. 从工作流中移除实时版本探测。

### P1. 发布通知器调用 GitHub API

证据:

- `.github/workflows/release-notifier.yml:55`
- `.github/workflows/release-notifier.yml:56`
- `.github/workflows/release-notifier.yml:61`

其行为:

- 使用 `curl` 调用外部仓库中的 GitHub workflow dispatch API。

离线化措施:

- 对离线流水线移除该步骤，或替换为内部事件机制。

### P2. 打包步骤使用外部时间戳服务

证据:

- `PowerEditor/installer/packageAll.bat:26`

其行为:

- 签名命令使用 `http://timestamp.globalsign.com/...`

离线化措施:

- 替换为内部时间戳服务，或采用离线签名策略。

## 7. 看起来像在线行为但大多只是噪声的内容

以下区域包含大量 URL 字符串，但大多并不是运行时网络行为:

1. `scintilla/doc`
   - 大量文档页面和外部链接
2. `lexilla/*`
   - 注释、语法参考、词法分析器示例
3. `PowerEditor/installer/nativeLang/*`
   - 翻译文本和示例搜索引擎字符串
4. `PowerEditor/Test/*`
   - URL 检测测试
5. `README.md`, `BUILD.md`, `.github/ISSUE_TEMPLATE*`
   - 文档与 GitHub 元数据
6. 类似 `json.hpp` 的随附头文件
   - 含外部引用的源码注释

建议:

- 不要将原始 URL 数量直接视为运行时在线数量。
- 对离线产品工作而言，应优先处理第 2 到第 6 节。

## 8. 已确认的否定性发现

在已扫描的代码库区域中，未发现以下直接网络模式:

- `InternetOpen`
- `WinHttp`
- `URLDownloadToFile`
- `WSAStartup`

解读:

- 主源码树似乎并未直接执行原始 HTTP/socket 操作。
- 在线行为主要基于浏览器启动，或委托给随产品分发的外部组件。

## 9. 建议的离线化顺序

### Stage 1. 停止真实的更新和插件联网

1. 移除 `GUP.exe` 的集成与打包。
2. 通过代码禁用自动更新，而不仅仅依赖配置文件。
3. 在离线构建中移除 Plugin Admin。

### Stage 2. 移除浏览器启动入口点

1. 移除网站/帮助/论坛/项目菜单项。
2. 移除 `Search on Internet`。
3. 移除 UDL 在线帮助和 Plugin Admin 仓库链接。
4. 移除默认的互联网导向用户命令。

### Stage 3. 移除被动 URL 启动行为

1. 禁用从编辑器文本打开可点击 URL。
2. 可选地减少支持的 URI scheme。

### Stage 4. 清理安装程序和流水线

1. 移除安装程序中的浏览器重定向。
2. 从 MSI/NSIS/便携版中移除更新器载荷。
3. 如果要求完整离线构建，则替换 CI 和发布流水线中的互联网依赖。

## 10. 本次审计后仍然存在的缺口

本次审计基于源码树。如果“完全离线”意味着随产品分发的二进制文件必须具备零在线能力，则以下内容仍需单独检查:

1. 已构建的 `GUP.exe`
2. 已构建的 `nppPluginList.dll`
3. 本源码树之外任何随产品分发的插件二进制文件
4. 最终安装程序二进制文件和便携包内容

如有需要，下一步应当是:

1. 产出一个补丁清单，用于移除所有 P0/P1 级运行时在线路径
2. 生成第二份报告，将每个菜单/对话框/配置项映射到所需的精确源码修改
