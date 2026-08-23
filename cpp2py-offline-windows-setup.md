# cpp2py-core Windows 离线环境准备指南

本文用于在一台断网的 Windows 新电脑上运行 `cpp2py-core`，完成单个 C++ translation unit 到 Python 的确定性转换，并按需执行 C++/Python Harness 验证。

## 1. 结论与选型

根据目标 C++ 项目的依赖选择工具链：

| C++ 项目情况 | 推荐工具链 |
|---|---|
| 普通、自包含、使用标准 C++20，且不依赖 MSVC ABI 的代码 | 完整的 **llvm-mingw ZIP** |
| 依赖 MSVC、Windows SDK、ATL/MFC、MSVC `.lib`、商业 SDK 或厂商专用工具链 | **Visual Studio Build Tools + LLVM/Clang** |

不要只复制一个 `clang++.exe`。完整使用 C++ 通常还需要标准库头文件、C/C++ 运行库、链接器和目标平台头文件。

无论选择哪套工具链，`cpp2py-core` 的确定性转换核心都不需要第三方 Python 运行时包。

## 2. 所有场景都需要准备的内容

- Python 3.11 或更高版本的离线安装包。
- 完整的 `cpp2py-core` 源码目录。
- 目标项目的 `.cpp`、`.h`、`.hpp` 和生成头文件。
- 目标项目使用的第三方头文件或 SDK。
- 与目标 `.cpp` 匹配的 `compile_commands.json`。
- 下文两种 C++ 工具链之一。

`compile_commands.json` 必须使用 `arguments` 数组，不能只有 shell 字符串形式的 `command` 字段。

## 3. 方案 A：完整 llvm-mingw，不安装 Visual Studio

### 3.1 适用范围

适用于：

- 普通 C++20 代码；
- 自包含的纯函数和值类型逻辑；
- 使用 `<string>`、`<optional>`、`<cstdint>` 等标准库头文件；
- 不需要链接 MSVC ABI 的 C++ 静态库；
- 不使用 ATL/MFC 或只能由特定商业 SDK 提供的 MSVC 库。

### 3.2 联网电脑上下载

从 llvm-mingw Releases 下载适用于 Windows x86-64 的原生 UCRT ZIP：

- 项目主页：<https://github.com/mstorsjo/llvm-mingw>
- Releases：<https://github.com/mstorsjo/llvm-mingw/releases>

文件名通常包含：

```text
llvm-mingw-<版本>-ucrt-x86_64.zip
```

下载后解压，完整复制整个目录到离线电脑，例如：

```text
D:\llvm-mingw\
```

不要只复制 `bin` 中的一个 EXE。完整目录包含 Clang、LLD、libc++、mingw-w64 头文件和运行库等组件。

### 3.3 离线电脑验证

```powershell
D:\llvm-mingw\bin\x86_64-w64-mingw32-clang++.exe --version
```

准备一个测试文件：

```cpp
#include <iostream>
#include <string>

int main() {
    std::string message = "toolchain-ready";
    std::cout << message << '\n';
    return 0;
}
```

编译并运行：

```powershell
D:\llvm-mingw\bin\x86_64-w64-mingw32-clang++.exe `
  -std=c++20 `
  .\toolchain-test.cpp `
  -o .\toolchain-test.exe

.\toolchain-test.exe
```

### 3.4 cpp2py 转换和验证

```powershell
cd D:\cpp2py-core
$env:PYTHONPATH = "$PWD\src"

python -m cpp2py.cli convert file D:\work\demo\sample.cpp `
  --compile-db D:\work\demo\compile_commands.json `
  --clang-path D:\llvm-mingw\bin\x86_64-w64-mingw32-clang++.exe `
  --output-dir D:\work\demo\conversion-output
```

需要 Harness 验证时：

```powershell
python -m cpp2py.cli verify function D:\work\demo\sample.cpp add `
  --cases D:\work\demo\cases.json `
  --compiler D:\llvm-mingw\bin\x86_64-w64-mingw32-clang++.exe `
  --compile-db D:\work\demo\compile_commands.json `
  --clang-path D:\llvm-mingw\bin\x86_64-w64-mingw32-clang++.exe `
  --output-report D:\work\demo\harness-report.json
```

## 4. 方案 B：Visual Studio Build Tools + LLVM

### 4.1 适用范围

遇到以下任一情况时，优先选择本方案：

- 原项目使用 MSVC 构建；
- 依赖 Windows SDK；
- 依赖 ATL、MFC、COM 或 Windows Runtime；
- 依赖 MSVC ABI 的 `.lib`、`.dll`；
- 依赖商业 SDK 或硬件厂商 SDK；
- 编译参数包含较多 MSVC 专用选项；
- 需要与现有 MSVC 二进制保持兼容。

### 4.2 联网电脑上准备离线布局

准备 Visual Studio Build Tools 离线安装文件，并至少包含：

- “使用 C++ 的桌面开发”工作负载；
- MSVC x64/x86 C++ 工具集；
- Windows 10 或 Windows 11 SDK；
- C++ CMake tools（需要生成 compile database 时可选）；
- LLVM/Clang for Windows 组件。

离线电脑安装完成后，典型工具包括：

```text
cl.exe
clang++.exe
VsDevCmd.bat
Windows SDK headers and libraries
```

不要单独复制 `cl.exe`。cpp2py Harness 使用 MSVC 时，需要从同一 Visual Studio 安装中找到 `Common7\Tools\VsDevCmd.bat`，以加载 `INCLUDE`、`LIB` 等开发环境。

### 4.3 验证工具链

打开 “Developer PowerShell for VS” 或执行对应的 `VsDevCmd.bat`，然后运行：

```powershell
cl.exe
clang++.exe --version
```

用 C++20 编译测试文件：

```powershell
cl.exe /nologo /EHsc /std:c++20 .\toolchain-test.cpp /Fe:.\toolchain-test.exe
.\toolchain-test.exe
```

### 4.4 cpp2py 转换和验证

```powershell
cd D:\cpp2py-core
$env:PYTHONPATH = "$PWD\src"

python -m cpp2py.cli convert file D:\work\demo\sample.cpp `
  --compile-db D:\work\demo\compile_commands.json `
  --clang-path "C:\Path\To\LLVM\bin\clang++.exe" `
  --output-dir D:\work\demo\conversion-output
```

Harness 可以使用完整路径指向 `cl.exe`：

```powershell
python -m cpp2py.cli verify function D:\work\demo\sample.cpp add `
  --cases D:\work\demo\cases.json `
  --compiler "C:\Path\To\MSVC\bin\Hostx64\x64\cl.exe" `
  --compile-db D:\work\demo\compile_commands.json `
  --clang-path "C:\Path\To\LLVM\bin\clang++.exe" `
  --output-report D:\work\demo\harness-report.json
```

## 5. Python 离线依赖

### 5.1 只运行转换或 Harness

不需要第三方 Python 包。项目核心只使用 Python 标准库。

直接从源码运行：

```powershell
cd D:\cpp2py-core
$env:PYTHONPATH = "$PWD\src"
python -m cpp2py.cli --help
```

这种方式不运行项目构建，也不需要：

- `setuptools`；
- `wheel`；
- `pytest`；
- `openai`；
- `pydantic`。

### 5.2 需要运行项目测试

直接依赖是：

```text
pytest>=8
```

在开发电脑下载适配 Windows x64、Python 3.11 的 wheel 及全部传递依赖：

```powershell
New-Item -ItemType Directory .\wheelhouse -Force

python -m pip download `
  --dest .\wheelhouse `
  --only-binary=:all: `
  --platform win_amd64 `
  --python-version 311 `
  --implementation cp `
  "pytest>=8"
```

将整个 `wheelhouse` 复制到离线电脑，然后：

```powershell
python -m pip install `
  --no-index `
  --find-links D:\wheelhouse `
  "pytest>=8"
```

运行测试：

```powershell
cd D:\cpp2py-core
python -m pytest -q
```

如果离线电脑使用 Python 3.12，把下载命令中的 `311` 改为 `312`。

### 5.3 需要从源码执行 pip 安装

只有执行下列命令时才需要构建依赖：

```powershell
python -m pip install -e .
```

对应直接依赖：

```text
setuptools>=69
wheel
```

建议离线场景优先使用 `PYTHONPATH=src`，从而完全避免源码构建和 editable installation。

### 5.4 OpenAI AI fallback

直接依赖是：

```text
openai>=2
pydantic>=2
```

开发电脑下载所有 wheel：

```powershell
python -m pip download `
  --dest .\wheelhouse `
  --only-binary=:all: `
  --platform win_amd64 `
  --python-version 311 `
  --implementation cp `
  "openai>=2" `
  "pydantic>=2"
```

离线安装：

```powershell
python -m pip install `
  --no-index `
  --find-links D:\wheelhouse `
  "openai>=2" `
  "pydantic>=2"
```

注意：真实 OpenAI provider 仍需要网络和 `OPENAI_API_KEY`。断网期间安装这些包也不能调用云端模型，因此一天离线使用时通常不必准备。

`pydantic-core` 包含平台相关二进制 wheel，所以下载时的操作系统架构和 Python 版本必须与离线电脑一致。

## 6. C++ 依赖需要复制哪些内容

### 6.1 只做 AST 转换

cpp2py 使用 Clang 的 `-fsyntax-only -Xclang -ast-dump=json` 解析源码，不执行链接。需要复制：

- 目标 `.cpp`；
- 项目自己的 `.h/.hpp`；
- 构建过程生成的头文件，例如 `config.h`；
- 第三方头文件；
- 对应工具链的 C++ 标准库头文件；
- 使用 Windows API 时所需的 Windows 头文件；
- `compile_commands.json`；
- 编译所需的 include 路径和宏定义。

只转换时通常不需要复制：

- `.lib`；
- `.a`；
- `.dll`；
- 链接参数。

### 6.2 运行 Harness

Harness 会生成并编译一个 C++ driver，然后运行 C++ 与 Python 两侧。因此除头文件外，还可能需要：

- 第三方静态库 `.a` 或 `.lib`；
- 第三方 DLL 及其传递运行时依赖；
- 匹配的 C/C++ 运行库；
- Windows 系统导入库；
- 与预编译库 ABI 兼容的编译器。

但当前 cpp2py Harness 主要支持自包含的纯函数和值类型。compile database 中的链接参数不会作为可靠的外部库链接配置保留，因此依赖外部符号的 translation unit 可能在 Harness 链接阶段被阻塞。

不要把 MSVC 编译的 C++ 静态库直接与 llvm-mingw 混用。二者通常使用不同的 C++ ABI、标准库和运行库。C ABI 的 DLL 边界有时可以互操作，但仍应单独验证内存分配、异常和资源所有权。

## 7. 手写 compile_commands.json

llvm-mingw 示例：

```json
[
  {
    "directory": "D:/work/demo",
    "file": "D:/work/demo/sample.cpp",
    "arguments": [
      "D:/llvm-mingw/bin/x86_64-w64-mingw32-clang++.exe",
      "-std=c++20",
      "-ID:/work/demo/include",
      "-ID:/third-party/example/include",
      "-DMY_FEATURE=1",
      "-c",
      "D:/work/demo/sample.cpp"
    ]
  }
]
```

要求：

- `directory`、`file` 和 include 目录必须在离线电脑真实存在；
- 使用 `arguments` 数组；
- JSON 中的 Windows 路径推荐写成 `/`；
- `-I`、`-D`、`-U` 和 `-std=c++20` 应与原项目一致；
- 目标项目有生成头文件时，必须提前生成并复制过去；
- 复制开发电脑生成的 compile database 时，需要修正其中的绝对路径。

## 8. 离线电脑最终检查清单

依次执行：

```powershell
python --version
```

使用 llvm-mingw：

```powershell
D:\llvm-mingw\bin\x86_64-w64-mingw32-clang++.exe --version
```

或使用 Visual Studio + LLVM：

```powershell
clang++.exe --version
cl.exe
```

确认 C++20 标准库程序能够编译并运行，然后：

```powershell
cd D:\cpp2py-core
$env:PYTHONPATH = "$PWD\src"
python -m cpp2py.cli --help
```

最后执行一次真实转换。需要确认：

- 状态不是 `blocked`；
- Clang 能找到全部头文件；
- `compile_commands.json` 路径正确；
- `conversion-output` 中生成了 `converted_v2.py`、`conversion_v2.json`、`typed-ir.json` 和 `source-map.json`。

## 9. 推荐的离线复制目录

```text
offline-package/
├─ Python-installer/
├─ cpp2py-core/
├─ toolchain/
│  ├─ llvm-mingw/                 方案 A
│  └─ visual-studio-offline/      方案 B
├─ wheelhouse/                    只有测试或 AI 依赖时需要
├─ target-cpp-project/
│  ├─ src/
│  ├─ include/
│  ├─ generated-include/
│  ├─ third-party/
│  └─ compile_commands.json
└─ checksums.txt                  建议记录安装包 SHA-256
```

## 10. 参考资料

- cpp2py-core 项目内 `docs/使用说明.md`
- LLVM Clang Toolchain：<https://clang.llvm.org/docs/Toolchain.html>
- LLVM Clang MSVC compatibility：<https://clang.llvm.org/docs/MSVCCompatibility.html>
- llvm-mingw：<https://github.com/mstorsjo/llvm-mingw>

