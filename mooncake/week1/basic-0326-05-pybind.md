# Python Wheel 与 pybind11 编译打包原理

> 从 C++ 源码到 `pip install` 完整流程解析

---

## 1. Python Wheel 是什么？

### 1.1 概念

**Wheel**（.whl 文件）是 Python 的**预编译二进制包格式**，相当于：

| 语言 | 包格式 | 类比 |
|------|--------|------|
| Python | .whl | 免安装的"绿色软件" |
| Java | .jar | 可执行归档 |
| C/C++ | .deb/.rpm | 系统包管理器格式 |
| Windows | .exe/.msi | 安装程序 |

### 1.2 为什么需要 Wheel？

```
传统方式（源码安装）：
  pip install package.tar.gz
    ├── 下载源代码
    ├── 执行 setup.py build    ← 在用户机器上编译（慢！可能失败！）
    └── 执行 setup.py install

Wheel 方式（二进制安装）：
  pip install package.whl
    ├── 下载预编译的二进制
    └── 解压到 site-packages    ← 直接可用，无编译过程！
```

**优势**：
- **速度快**：跳过编译，直接解压
- **可靠性高**：编译环境问题由打包者解决
- **无依赖**：不依赖用户机器上有编译器（如 gcc）

### 1.3 Wheel 文件结构

```
mooncake-0.1.0-cp310-cp310-manylinux_2_31_x86_64.whl
│       │      │        │              │
│       │      │        │              └── 架构（x86_64/aarch64）
│       │      │        └── 平台和 glibc 版本
│       │      └── Python 版本（cp310 = CPython 3.10）
│       └── 包版本
└── 包名

解压后的内容：
mooncake-0.1.0.dist-info/      # 元数据
  ├── METADATA                 # 包信息
  ├── WHEEL                    # Wheel 格式版本
  └── RECORD                   # 文件哈希列表

mooncake/                      # 实际代码
  ├── __init__.py
  ├── store.cpython-310-x86_64-linux-gnu.so   ← pybind11 编译的模块！
  └── engine.cpython-310-x86_64-linux-gnu.so
```

---

## 2. pybind11 编译流程

### 2.1 完整流程图

```
┌─────────────────────────────────────────────────────────────────────┐
│                         pybind11 编译打包流程                        │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  1. 写 C++ 源码                                                       │
│     store_py.cpp ──┐                                                 │
│     transfer_engine_py.cpp ──┐                                       │
│                               │                                      │
│                               ▼                                      │
│  2. 配置编译系统                CMakeLists.txt                        │
│     ├─ 找 Python 头文件                                             │
│     ├─ 找 pybind11 头文件                                           │
│     └─ 定义编译目标                                                  │
│                               │                                      │
│                               ▼                                      │
│  3. 编译                      cmake .. && make                       │
│     ├─ 预处理（#include 展开）                                        │
│     ├─ 编译（.cpp → .o）                                             │
│     ├─ 链接（.o → .so）                                              │
│     │                                                              │
│     │   store.cpython-310-x86_64-linux-gnu.so                        │
│     │   engine.cpython-310-x86_64-linux-gnu.so                       │
│     │                                                              │
│                               │                                      │
│                               ▼                                      │
│  4. 打包                      python -m build                        │
│     ├─ setup.py 描述包信息                                           │
│     ├─ 复制 .so 文件到包目录                                          │
│     └─ 生成 .whl 文件                                                │
│                               │                                      │
│                               ▼                                      │
│  5. 发布/安装                 pip install mooncake.whl               │
│                               │                                      │
│                               ▼                                      │
│  6. Python 使用               import mooncake.store                  │
│                               MooncakeDistributedStore()             │
│                                                                      │
└─────────────────────────────────────────────────────────────────────┘
```

### 2.2 步骤详解：C++ 源码

```cpp
// store_py.cpp
#include <pybind11/pybind11.h>      // pybind11 核心头文件
#include <pybind11/stl.h>            // STL 容器自动转换

namespace py = pybind11;

// 你的 C++ 类
class MooncakeStorePyWrapper {
public:
    int put(const std::string& key, const std::string& value) {
        // ... 实际存储逻辑
        return 0;
    }
};

// 关键：PYBIND11_MODULE 宏定义 Python 模块
PYBIND11_MODULE(store, m) {          // "store" = Python 模块名
    m.doc() = "Mooncake Store Python Bindings";
    
    // 绑定类
    py::class_<MooncakeStorePyWrapper>(m, "MooncakeDistributedStore")
        .def(py::init<>())                                          // 构造函数
        .def("put", &MooncakeStorePyWrapper::put);                  // 方法
}
```

### 2.3 步骤详解：CMake 配置

```cmake
# CMakeLists.txt

cmake_minimum_required(VERSION 3.12)
project(mooncake_python)

# 1. 找 Python
find_package(Python REQUIRED COMPONENTS Interpreter Development)

# 2. 找 pybind11
find_package(pybind11 REQUIRED)

# 3. 定义编译目标
pybind11_add_module(store MODULE store_py.cpp)
#     │              │      │       │
#     │              │      │       └── 源文件
#     │              │      └────────── 类型：MODULE（动态库）
#     │              └───────────────── 输出名：store
#     └──────────────────────────────── pybind11 封装函数

# 4. 链接依赖
target_link_libraries(store PRIVATE mooncake_store_lib)
```

### 2.4 步骤详解：编译过程

```bash
# 创建构建目录
mkdir build && cd build

# 配置
cmake .. \
  -DPYTHON_EXECUTABLE=$(which python) \
  -DCMAKE_BUILD_TYPE=Release

# 编译
make -j$(nproc)

# 输出
cp310-cp310-linux-gnu/
└── store.cpython-310-x86_64-linux-gnu.so   ← 这就是 pybind11 模块！
```

**编译器做了什么？**

```cpp
// 你的代码
PYBIND11_MODULE(store, m) {
    py::class_<MooncakeStorePyWrapper>(m, "MooncakeDistributedStore")
        .def("put", &MooncakeStorePyWrapper::put);
}

// 编译器展开后（简化）：
extern "C" PyObject* PyInit_store() {           // Python 导入时调用的入口
    py::module m("store");
    
    // 创建类
    auto cls = py::class_<MooncakeStorePyWrapper>(m, "MooncakeDistributedStore");
    
    // 添加方法：使用模板元编程生成 Python C-API 代码
    cls.def("put", [](py::object self, py::str key, py::bytes value) {
        // 1. 从 Python 对象提取 C++ 类型
        std::string key_cpp = key.cast<std::string>();
        std::string value_cpp = value.cast<std::string>();
        
        // 2. 调用 C++ 方法
        auto* this_ptr = self.cast<MooncakeStorePyWrapper*>();
        int result = this_ptr->put(key_cpp, value_cpp);
        
        // 3. 转换返回值
        return py::int_(result);
    });
    
    return m.ptr();
}
```

### 2.5 步骤详解：打包 Wheel

```python
# setup.py
from setuptools import setup, find_packages
from setuptools.dist import Distribution

class BinaryDistribution(Distribution):
    """声明这是二进制包（有 .so 文件）"""
    def has_ext_modules(self):
        return True

setup(
    name='mooncake',
    version='0.1.0',
    packages=find_packages(),
    package_data={
        'mooncake': ['*.so'],        # ← 包含编译好的 .so 文件
    },
    distclass=BinaryDistribution,
)
```

```bash
# 打包命令
cd mooncake-wheel/
pip install build
python -m build

# 输出
dist/
├── mooncake-0.1.0-cp310-cp310-manylinux_2_31_x86_64.whl   ← 二进制 wheel
└── mooncake-0.1.0.tar.gz                                  ← 源码包（可选）
```

### 2.6 Wheel 内部结构

```bash
# 解压 .whl 看看
unzip -l mooncake-0.1.0-cp310-cp310-manylinux_2_31_x86_64.whl

# 输出
  Length      Date    Time    Name
---------  ---------- -----   ----
        0  2025-03-26 10:00   mooncake/
      234  2025-03-26 10:00   mooncake/__init__.py
  5242880  2025-03-26 10:00   mooncake/store.cpython-310-x86_64-linux-gnu.so
  3145728  2025-03-26 10:00   mooncake/engine.cpython-310-x86_64-linux-gnu.so
     2345  2025-03-26 10:00   mooncake-0.1.0.dist-info/METADATA
      100  2025-03-26 10:00   mooncake-0.1.0.dist-info/WHEEL
     4567  2025-03-26 10:00   mooncake-0.1.0.dist-info/RECORD
```

---

## 3. Python 导入机制

### 3.1 `import mooncake.store` 发生了什么？

```python
import mooncake.store
```

```
Python 内部流程：
1. sys.path 中找 mooncake/ 目录
   ↓
2. 执行 mooncake/__init__.py
   ↓
3. 加载 mooncake/store.cpython-310-x86_64-linux-gnu.so
   ↓
   3.1 dlopen() 打开 .so 文件
   3.2 调用 PyInit_store() 函数（pybind11 生成的入口）
   3.3 注册模块和方法到 Python 运行时
   ↓
4. 返回 module 对象，赋值给 mooncake.store
```

### 3.2 模块名规则

```
store.cpython-310-x86_64-linux-gnu.so
│     │      │        │           │
│     │      │        │           └── 平台（Linux）
│     │      │        └── 架构（x86_64）
│     │      └── Python 版本（3.10）
│     └── 解释器实现（cpython）
└── 模块名（import store）

Python 3.11 就无法导入这个模块（版本不匹配）
```

---

## 4. 完整示例：最小 pybind11 项目

### 4.1 目录结构

```
my_cpp_module/
├── CMakeLists.txt
├── setup.py
├── pyproject.toml
└── src/
    └── mymodule.cpp
```

### 4.2 C++ 源码

```cpp
// src/mymodule.cpp
#include <pybind11/pybind11.h>

int add(int a, int b) {
    return a + b;
}

PYBIND11_MODULE(mymodule, m) {
    m.def("add", &add, "Add two numbers");
}
```

### 4.3 CMake 配置

```cmake
# CMakeLists.txt
cmake_minimum_required(VERSION 3.12)
project(my_cpp_module)

find_package(pybind11 REQUIRED)
pybind11_add_module(mymodule src/mymodule.cpp)
```

### 4.4 打包配置

```python
# setup.py
from setuptools import setup
from setuptools.dist import Distribution

class BinaryDistribution(Distribution):
    def has_ext_modules(self):
        return True

setup(
    name='my-cpp-module',
    version='0.1.0',
    packages=['my_cpp_module'],
    package_data={'my_cpp_module': ['*.so']},
    distclass=BinaryDistribution,
)
```

```toml
# pyproject.toml
[build-system]
requires = ["setuptools", "wheel", "cmake", "pybind11"]
build-backend = "setuptools.build_meta"
```

### 4.5 构建命令

```bash
# 1. 编译 C++
mkdir build && cd build
cmake ..
make

# 2. 复制 .so 到包目录
cp mymodule*.so ../my_cpp_module/

# 3. 打包
python -m build

# 4. 安装测试
pip install dist/my_cpp_module-0.1.0-*.whl
python -c "import my_cpp_module; print(my_cpp_module.add(1, 2))"
```

---

## 5. 常见问题

### 5.1 为什么需要 pybind11？

```
不用 pybind11：
  手动写 Python C-API（500+ 行代码）
  内存管理容易出错
  类型转换繁琐

用 pybind11：
  20 行代码搞定
  自动内存管理
  STL 容器自动转换
  模板元编程生成胶水代码
```

### 5.2 Manylinux 是什么？

```
manylinux_2_31_x86_64.whl
│         │      │
│         │      └── 架构
│         └── glibc 版本（CentOS 7 约等于 2.17）
└── PEP 600 标准名称

为什么重要？
  Linux 各发行版的系统库版本不同
  manylinux 保证在足够老的系统上编译
  兼容性：能在新系统上跑，也能在老系统上跑
```

### 5.3 为什么 Mooncake 和 SGLang 分开？

```
Mooncake（独立项目）              SGLang（独立项目）
├─ C++ 存储引擎                   ├─ Python 推理框架
├─ Python 绑定（pybind11）        └─ 依赖：mooncake（pip install）
└─ pip install mooncake

好处：
  1. Mooncake 可被多个项目使用（SGLang、vLLM 等）
  2. 独立版本控制
  3. 独立测试和发布
```

---

## 6. 总结

| 概念 | 一句话解释 |
|------|-----------|
| **Wheel** | Python 的预编译二进制包格式，`pip install` 时直接解压使用 |
| **pybind11** | C++ 库，自动生成 Python C-API 代码，让 C++ 能被 Python 调用 |
| **编译** | `.cpp` + `CMake` + `make` → `.so` 动态链接库 |
| **打包** | `.so` + `setup.py` → `.whl` 分发包 |
| **安装** | `pip install .whl` → 解压到 `site-packages/` |
| **导入** | `import xxx` → Python 加载 `.so` 并调用 `PyInit_xxx()` |

**核心流程**：
```
C++ 源码 → pybind11 包装 → 编译成 .so → 打包成 .whl → pip 安装 → Python 导入使用
```
