# CheckDependency

## 简介

仅支持 Windows。

CheckDependency 用于检查一个 exe 或 dll 文件的依赖关系，类似于 `dumpbin /DEPENDENTS` 的功能。它会递归扫描目标文件所依赖的所有 dll，并分析：

- 每个 dll 的完整路径和平台（x64 / x86）
- 每个 dll 导出的全部函数（导出表）
- 当前文件实际使用了每个 dll 的哪些函数（导入表）
- 有问题的 dll：找不到文件、或缺失被依赖的函数

这在部署程序、排查运行时 "找不到 dll" 或 "找不到函数入口点" 错误时非常有用。

## 使用方法

只需将 `CheckDependency.h` 加入工程，它是一个 header-only 的类。依赖 Windows API 和 `imagehlp.lib`（已在头文件中通过 `#pragma comment(lib)` 自动链接）。

### 基本用法

```c++
#include "CheckDependency.h"
#include <iostream>

int main()
{
    // 方式一：构造时直接传入文件名
    CheckDependency dep("myself.dll");
    auto problem_dlls = dep.DllsNotGood();

    // 方式二：先构造，再调用 Check
    // CheckDependency dep;
    // auto problem_dlls = dep.Check("myself.dll");

    if (problem_dlls.empty())
    {
        std::cout << "All dependencies are OK." << std::endl;
        return 0;
    }

    std::cout << "Problem DLLs:" << std::endl;
    for (auto& [dll_name, info] : problem_dlls)
    {
        if (info.isNotFoundDll())
        {
            std::cout << "  " << dll_name << " - NOT FOUND" << std::endl;
        }
        else
        {
            std::cout << "  " << dll_name
                      << " (" << info.machine << ")"
                      << " - missing " << info.lost_functions.size() << " functions:" << std::endl;
            for (auto& func : info.lost_functions)
            {
                std::cout << "    " << func << std::endl;
            }
        }
    }
    return 0;
}
```

### 查看导入表和导出表

`Check` 方法执行后，可以通过 `ImportTable()` 和 `ExportTable()` 获取完整的导入/导出信息：

```c++
CheckDependency dep;
dep.Check("myself.dll");

// 导入表：当前文件依赖的所有 dll 及其使用的函数
std::cout << "Import Table:" << std::endl;
for (auto& [dll_name, info] : dep.ImportTable())
{
    std::cout << "  " << dll_name
              << " (" << info.machine << ")"
              << " path: " << info.full_path
              << " used: " << info.used_functions.size() << " functions" << std::endl;
}

// 导出表：所有被扫描到的 dll 导出的函数
std::cout << "Export Table:" << std::endl;
for (auto& [dll_name, funcs] : dep.ExportTable())
{
    std::cout << "  " << dll_name << " exports " << funcs.size() << " functions" << std::endl;
}
```

## API 参考

### 构造函数

| 签名 | 说明 |
|------|------|
| `CheckDependency()` | 默认构造，不做任何检查 |
| `CheckDependency(const std::string& file)` | 构造时立即执行 `Check(file)` |

### 方法

| 方法 | 返回值 | 说明 |
|------|--------|------|
| `Check(const std::string& file)` | `std::map<std::string, NotGoodInfo>` | 递归扫描 `file` 的所有依赖，返回有问题的 dll 映射。无问题时 map 为空 |
| `DllsNotGood()` | `std::map<std::string, NotGoodInfo>` | 返回上次 `Check` 的结果 |
| `ImportTable()` | `std::map<std::string, ImportInfo>` | 返回完整的导入表 |
| `ExportTable()` | `std::map<std::string, std::map<std::string, int>>` | 返回完整的导出表 |

### 数据结构

#### ImportInfo

| 字段 | 类型 | 说明 |
|------|------|------|
| `full_path` | `std::string` | dll 的完整路径（由 `SearchPath` 查找）|
| `machine` | `std::string` | 平台类型：`"x64"`, `"x86"`, `"unknown"` |
| `used_functions` | `std::vector<std::string>` | 被依赖的函数名列表 |

#### NotGoodInfo

| 字段 | 类型 | 说明 |
|------|------|------|
| `full_path` | `std::string` | dll 的完整路径，空表示文件未找到 |
| `machine` | `std::string` | 平台类型 |
| `lost_functions` | `std::vector<std::string>` | 缺失的函数名列表 |
| `isNotFoundDll()` | `bool` | `full_path` 为空时返回 `true`，表示 dll 文件未找到 |

## 注意事项

- 仅支持 Windows 平台，依赖 `ImageHlp.h` 和 `imagehlp.lib`。
- 扫描会自动跳过 `api-ms-win*` 和 `KERNEL32` 等系统 API Set dll。
- 如果某个 dll 的路径和平台都正确但有缺失函数，通常说明该 dll 的版本不对。
- 递归扫描依赖树，因此会自动处理间接依赖（A → B → C）。
