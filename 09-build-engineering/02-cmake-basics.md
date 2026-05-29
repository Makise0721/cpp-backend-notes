# 02 — CMake 基础

> 关联篇目：[01 编译链接](./01-compilation-link-process.md)

---

## 1. CMakeLists.txt 基本结构

```cmake
cmake_minimum_required(VERSION 3.15)
project(MyApp VERSION 1.0.0 LANGUAGES CXX)

# 设置 C++ 标准
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# 添加可执行文件
add_executable(my_app main.cpp utils.cpp)

# 添加静态库
add_library(mylib STATIC src/lib.cpp)

# 添加动态库
add_library(mylib_shared SHARED src/lib.cpp)

# 添加仅头文件的库（INTERFACE）
add_library(header_only INTERFACE)
```

---

## 2. 核心 Target 命令

### 2.1 `target_include_directories`——头文件搜索路径

```cmake
# PUBLIC: 自己需要，链接此库的目标也需要
target_include_directories(mylib PUBLIC include/)

# PRIVATE: 只有自己需要
target_include_directories(mylib PRIVATE src/internal/)

# INTERFACE: 自己不需要，但链接此库的目标需要（纯头文件库）
target_include_directories(header_only INTERFACE include/)
```

### 2.2 `target_link_libraries`——链接依赖

```cmake
target_link_libraries(my_app
    PRIVATE mylib           # my_app 用 mylib，不暴露给 my_app 的使用者
    PUBLIC  other_lib       # my_app 用 other_lib，也暴露给 my_app 的使用者
)

# PUBLIC / PRIVATE / INTERFACE 传递规则:
#
# app → libB → libA
# libB 对 libA 是 PRIVATE:  app 不知道 libA 的存在
# libB 对 libA 是 PUBLIC:   app 自动能 include libA 的头文件 + 链接 libA
# libB 对 libA 是 INTERFACE: libB 自己不用 libA，但 app 需要
```

### 2.3 关键字传递一览

| 关键字 | 自己用 | 使用者用 | 典型场景 |
|--------|:---:|:---:|----------|
| `PRIVATE` | ✅ | ❌ | 内部实现细节 |
| `PUBLIC` | ✅ | ✅ | 头文件暴露了该依赖的类型 |
| `INTERFACE` | ❌ | ✅ | 仅头文件库 |

---

## 3. `find_package`——查找外部库

```cmake
# 查找已安装的库（需要对应的 FindXXX.cmake 或 XXXConfig.cmake）
find_package(OpenSSL REQUIRED)

target_link_libraries(my_app PRIVATE OpenSSL::SSL OpenSSL::Crypto)

# 常见可 find_package 的库:
# Boost, OpenSSL, Protobuf, gRPC, Threads, ZLIB, Eigen3, OpenCV...

# 自定义查找路径
find_package(MyLib REQUIRED PATHS /opt/mylib)

# 可选依赖
find_package(OptionalLib QUIET)
if(OptionalLib_FOUND)
    target_link_libraries(my_app PRIVATE OptionalLib::OptionalLib)
endif()
```

---

## 4. 编译选项与构建类型

```cmake
# 设置构建类型
set(CMAKE_BUILD_TYPE Debug)       # -g -O0
set(CMAKE_BUILD_TYPE Release)     # -O2 -DNDEBUG
set(CMAKE_BUILD_TYPE RelWithDebInfo)  # -O2 -g

# 编译选项
target_compile_options(my_app PRIVATE -Wall -Wextra -Wpedantic)

# 按构建类型设置选项
target_compile_options(my_app PRIVATE
    $<$<CONFIG:Debug>:-O0 -g>
    $<$<CONFIG:Release>:-O2 -DNDEBUG>
)

# Macro 定义
target_compile_definitions(my_app PRIVATE VERSION_MAJOR=1)
# 等价于在代码中 #define VERSION_MAJOR 1
```

---

## 5. 常用 CMake 模式

### 5.1 子目录

```cmake
# 顶层 CMakeLists.txt
add_subdirectory(src)
add_subdirectory(tests)

# src/CMakeLists.txt
add_library(core STATIC core.cpp)

# tests/CMakeLists.txt
add_executable(test_core test.cpp)
target_link_libraries(test_core PRIVATE core GTest::gtest)
```

### 5.2 安装规则

```cmake
install(TARGETS mylib
    EXPORT MyLibTargets
    ARCHIVE DESTINATION lib
    LIBRARY DESTINATION lib
    RUNTIME DESTINATION bin
)
install(DIRECTORY include/ DESTINATION include)
```

### 5.3 条件编译

```cmake
# 根据平台
if(WIN32)
    target_sources(my_app PRIVATE platform_win.cpp)
elseif(APPLE)
    target_sources(my_app PRIVATE platform_mac.cpp)
else()
    target_sources(my_app PRIVATE platform_linux.cpp)
endif()

# 根据选项
option(ENABLE_LOGGING "Enable logging" ON)
if(ENABLE_LOGGING)
    target_compile_definitions(my_app PRIVATE ENABLE_LOGGING=1)
endif()
```

---

## 6. 构建流程

```bash
# 创建构建目录（out-of-source build）
mkdir build && cd build

# 配置
cmake .. -DCMAKE_BUILD_TYPE=Release

# 构建
cmake --build . -j$(nproc)

# 安装
cmake --install . --prefix /usr/local
```

---

## 小结

| 命令 | 用途 |
|------|------|
| `add_executable` | 可执行文件 |
| `add_library` | STATIC/SHARED/INTERFACE 库 |
| `target_include_directories` | 头文件路径 (PUBLIC/PRIVATE/INTERFACE) |
| `target_link_libraries` | 链接依赖 |
| `find_package` | 查找外部库 |
| `target_compile_options` | 编译选项 |

下一篇 [03 Git 工作流](./03-git-workflow.md)。
