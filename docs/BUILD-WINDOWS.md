# Windows 构建说明（实测）

本文记录在一台 Windows 机器上从源码构建 Reader Windows 版的实测过程，只写验证过的事实，不涉及任何源码改动。

## 1. 环境

| 组件 | 实测版本 / 说明 |
|------|----------------|
| Flutter | 3.47.9（便携安装即可，无需写入系统 PATH） |
| Dart | 3.13.7（随 Flutter 附带） |
| Visual Studio | 2022 Build Tools，勾选「使用 C++ 的桌面开发」 |
| 操作系统 | Windows 10 / 11（x64） |

便携安装指把 Flutter SDK 解压到任意目录，只在当前终端把 `flutter\bin` 加入 `PATH`，不改系统环境变量。

## 2. 必须开启开发者模式

Windows 上 Flutter 构建带插件的项目需要创建符号链接，未开启时会在 `flutter pub get` / `build` 阶段直接失败：

```
Building with plugins requires symlink support
```

开启方式（二选一）：

- 图形界面：`设置 → 系统 → 开发者选项`，打开「开发人员模式」；
- 注册表（需管理员）：

```
HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\AppModelUnlock
  AllowDevelopmentWithoutDevLicense = 1 (DWORD)
```

开启后重新打开终端再继续。

## 3. 命令序列

```powershell
flutter config --no-analytics   # 关闭统计上报，避免首次运行的交互提示
flutter pub get
flutter analyze
flutter test                    # 实测 13 个用例全部通过
flutter build windows --release
```

产物位置：

```
build\windows\x64\runner\Release\reader.exe
```

实测 `reader.exe` 为 99,840 字节；整个 `Release` 目录共 21 个文件、约 33.6 MB（含 Flutter 引擎 DLL 与插件 DLL）。

> 若要分发便携包，还需把 `native\espeak-ng` 下的 DLL、`espeak-ng-data` 与语音模型一并拷入 `Release` 目录。

## 4. 构建环境已知坑（实测遇到）

1. **缺 `nuget.exe`**：CMake 配置阶段报找不到 NuGet 时，把 `nuget.exe` 补齐到 PATH 可见的目录即可继续。
2. **第三方头文件编码告警**：`uri_utils.hpp` 在中文代码页下触发 C4819（文件含无法在当前代码页表示的字符），在构建环境里设置 `CL=/utf-8` 可规避。
3. **`CMAKE_INSTALL_PREFIX` 误指**：CMake 3.31 下默认安装前缀会落到 `C:/Program Files/reader`，导致后续步骤写系统目录失败。处理方式是在构建目录的 `CMakeCache.txt` 中把 `CMAKE_INSTALL_PREFIX` 改成期望路径，然后重跑构建。

以上都属构建环境问题，不需要改仓库代码。

## 5. 网络提示

`flutter pub get` 在国内网络下建议使用镜像：

```powershell
$env:FLUTTER_STORAGE_BASE_URL = "https://storage.flutter-io.cn"
$env:PUB_HOSTED_URL = "https://pub.flutter-io.cn"
```

若终端里残留了指向已失效端口的代理变量（`HTTP_PROXY` / `HTTPS_PROXY`），先清空再执行，否则 `pub get` 会卡在连接阶段：

```powershell
Remove-Item Env:\HTTP_PROXY, Env:\HTTPS_PROXY -ErrorAction SilentlyContinue
```

## 6. 说明

本文所述的版本号、文件大小与报错文本均来自本机实测，属于构建环境事实，不涉及任何源码改动。
