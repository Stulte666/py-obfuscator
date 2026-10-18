# py-obfuscator 设计规范

**日期：** 2026-10-18  
**状态：** 待实施（实施范围以方案评审结论为准）  
**版本：** 3.0.0

**对应评审材料：** `2026-10-18-py-obfuscator-decision.md`。本文件描述自研路线（评审材料中的方案 A / C / D，含加载器的两种形态）的具体设计；候选方案之间的取舍由评审决定，本文不做选型。

> **本文档不含以下内容**：攻击成本的时间估算、方案的星级评分、"多少比例攻击者会放弃"这类预期。理由是这些量在 AI 辅助分析下已失去区分度，且缺少 Python 环境的实测支撑（外部评测均基于 C 程序与通用原生二进制，只可作参考量级）。涉及效果的地方一律标注为待实测。

---

## 1. 项目概述

### 1.1 项目目标

为纯离线分发的 Python 软件提供代码保护手段，替代当前"仅发布 `.pyc`"的形态。

目标设定为**抬高"提取并复用代码逻辑"的门槛**，而不是"防止反编译"——后者在代码必须于目标机执行的前提下不可实现。门槛具体体现在三处：产物不可直接复用、批量处理需要额外工作、语义重建需要人工投入。

不设"拖延多少天"之类的指标。在 AI 辅助分析下，时间维度是否仍有区分度取决于实测结果。

### 1.2 核心约束

**硬约束（作用于目标机运行）：**

| 约束 | 说明 | 不满足的后果 |
|------|------|------|
| 目标机离线运行 | 无需网络连接即可运行 | 方案不可用 |
| 可安装到任意机器 | 不做硬件绑定 / 机器指纹 | 方案不可用 |
| 不依赖外部服务 | 无云端校验、无在线授权 | 方案不可用 |

**非硬约束：**

| 项 | 口径 |
|------|------|
| 构建期联网 | 取决于构建环境实际条件，不构成排除项；本项目按构建环境可联网设计 |
| 第三方依赖 | 对依赖没有硬性限制。各方案所需组件与是否随交付物分发运行时，作为成本项记录 |

**待澄清的约束：**

| 约束 | 初值 | 待确认 |
|------|------|------|
| 运行时性能开销 | ≤ 20% | 分母是整机还是关键路径；是否允许对热点代码降低强度；基准测试方法与数据集 |
| 目标 Python 版本 | 未定 | 决定 `.pyc` / marshal / C 扩展绑定成本，也决定现状反编译难度 |

**结构性事实（由交付模式直接推出，不依赖实验）：**

| 事实 | 含义 |
|------|------|
| 密钥必然随软件分发 | 任何依赖"保密密钥"的保护，前提都不成立 |
| 运行时内存中存在明文 | 无论何种方案，运行时代码提取无法防御 |
| `.pyc` 可被反编译 | 成立范围取决于目标版本。常被引用的"95%+ 恢复"来自 uncompyle6 在 3.9 及以下的经验，**对 3.10 及以上不适用**，需按目标版本实测 |

### 1.3 核心功能

- **代码混淆：** 标识符重命名、控制流扁平化、垃圾代码插入、字符串加密、不透明谓词
- **二进制加密：** 逐模块 AEAD 加密，产出 `.enc`
- **加载器两种形态：** 纯 Python，或自研 C 扩展
- **日志行号保留：** 混淆后日志仍能显示原始源码行号
- **多级预设：** 基础、中等、高级三档，可按模块覆盖
- **批量处理：** 支持单文件和整个项目目录
- **可验证性：** 语义等价校验、性能基准、还原测试集

### 1.4 目标用户

需要保护Python代码知识产权的离线软件开发者，包括：
- 商业软件开发者
- 算法/模型保护需求者
- 需要分发Python应用的团队

---

## 2. 安全模型

### 2.1 离线场景的现实

```
┌─────────────────────────────────────────────────────────────┐
│  离线软件必须满足：                                          │
│  1. 能安装到任意机器 ✓                                       │
│  2. 无需网络连接 ✓                                           │
│  3. 能正常运行 ✓                                             │
│                                                             │
│  但同时：                                                    │
│  4. 无法防止密钥提取（软件必须包含密钥）                      │
│  5. 无法防止运行时代码被读取（明文必然出现在内存）            │
│  6. 无法防止代码分析（静态产物与动态行为都可被观察）          │
└─────────────────────────────────────────────────────────────┘
```

### 2.2 工具能做到的与做不到的

|  | 内容 | 说明 |
|---|---|---|
| ✅ 做到 | 使产物不可直接复用 | 拿到 `.enc` / 混淆产物无法原样跑起来或转售 |
| ✅ 做到 | 抬高批量提取的处理成本 | 每多一道变换，自动化脚本就多一层要攻克的门槛 |
| ✅ 做到 | 提供"已采取保护措施"的客观事实 | 与强度无关，是否构成法律意义上的合理保密措施需法务判断 |
| ❌ 做不到 | 防止代码逻辑被分析 | 代码最终必须在目标机执行 |
| ❌ 做不到 | 防止密钥被恢复 | 密钥随软件分发（结构性事实） |
| ❌ 做不到 | 防止运行时内存被读取 | 与实现语言无关，原生扩展同样防不住 |
| ❌ 做不到 | 可靠检测 AI 在分析代码 | 本地静态分析无法与使用者自身执行区分 |

对 AI 辅助分析的含义：有公开实验显示，具备工具链的自主 agent 面对多层保护时**不逐层破解，而是绕开**——静态读不出的秘密，就提取代码片段移植执行、观察输出。据此本工具遵循一条设计准则：**任何"静态读出 + 移植执行"即可获得的秘密，都不应假设混淆层能守住**；混淆层的定位是提高阅读与重建语义的成本，不是阻断。

各技术对 LLM 还原的实际抗性缺少 Python 环境数据（现有评测基于 C 程序，只作参考量级），须通过 §11 的还原测试集实测。

### 2.3 保护形态的构成关系

本工具覆盖评审材料中的三个自研候选，它们的构件关系如下（效果排序无实测支撑，不给星级）：

| 形态 | 构成 | 对应候选 | 关键性质 |
|------|------|:---:|------|
| 混淆 + `.py` | 纯静态变换 | A | 可直接阅读，仅符号失去语义 |
| 混淆 + `.pyc` | A + 编译为字节码 | C | 反编译是确定性的；防护力 ≈ 混淆成分 |
| 混淆 + `.enc` | A + 逐模块 AEAD + 加载器 | D | 攻击重心移向密钥提取与加载器 |
| └ 加载器形态 | 纯 Python 或 C 扩展 | D | C 扩展把密钥提取门槛从脚本级抬到原生逆向级 |

加密有效的前提是密钥无法被静态提取，而离线分发不满足该前提；因此加密层的实际价值在于**抬高批量处理的成本**（每个模块都要先解决密钥与加载器），不在于阻断。白盒密码学（评审材料方案 E）不在本工具的实施范围——它依赖外部实现且性能代价大，是否引入由评审决定。

### 2.4 硬件绑定：无意义

**原因：** 离线安装包的设计目的就是能安装到任意机器上。

```python
# ❌ 错误：硬件绑定
def get_machine_key():
    mac = uuid.getnode()
    return hashlib.sha256(str(mac).encode()).digest()

# 问题：软件无法在其他机器上运行！
```

---

## 3. 架构设计

### 3.1 整体架构

```
┌─────────────────────────────────────────────────────────────┐
│                     CLI Interface (cli.py)                  │
└─────────────────────────────────────────────────────────────┘
                               │
         ┌─────────────────────┼─────────────────────┐
         ▼                     ▼                     ▼
┌───────────────┐   ┌───────────────┐   ┌───────────────────┐
│   Analyzer    │   │  Obfuscator   │   │     Encryptor     │
│  AST 代码分析  │   │  混淆引擎      │   │  加密 + 加载器生成  │
└───────────────┘   └───────────────┘   └───────────────────┘
         │                     │                     │
         └─────────────────────┼─────────────────────┘
                               ▼
┌─────────────────────────────────────────────────────────────┐
│                Module Registry（变换模块注册中心）             │
│  Rename │ ControlFlow │ JunkCode │ StringEncrypt │ Opaque   │
│  Constant │ LineMap │ Bytecode │ Binary                     │
└─────────────────────────────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────┐
│   Loader 生成器：纯 Python 形态  /  C 扩展形态（appsec）      │
├─────────────────────────────────────────────────────────────┤
│   Verify：语义等价校验   Benchmark：性能基准   RestoreTest：  │
│   还原测试集（LLM / 脚本 / 人工三条路径）                     │
└─────────────────────────────────────────────────────────────┘
```

新增的三个非变换组件对应 2.2 的设计准则：效果未经实测前不能假定，因此实现阶段必须自带度量手段（等价校验、性能基准、还原测试）。

### 3.2 目录结构

```
D:\Desktop\code\ai\py-obfuscator/
├── pyproject.toml              # 项目配置(PEP 621)
├── README.md                   # 项目说明文档
├── LICENSE                     # 许可证
├── config/
│   └── presets.ini             # 预设配置文件
├── src/
│   └── py_obfuscator/
│       ├── __init__.py         # 包入口，导出API
│       ├── cli.py              # CLI命令行入口 (argparse)
│       ├── core/
│       │   ├── __init__.py
│       │   ├── analyzer.py     # AST代码分析器
│       │   ├── obfuscator.py   # 混淆引擎(策略调度)
│       │   ├── encryptor.py    # 加密引擎(字节码/二进制)
│       │   ├── line_mapper.py  # 行号映射器(日志保留)
│       │   ├── compat.py       # 动态特性兼容：preserve清单解析与生效
│       │   └── verify.py       # 语义等价校验（输出/副作用比对）
│       ├── modules/
│       │   ├── __init__.py     # 模块注册中心
│       │   ├── base.py         # 模块基类定义
│       │   ├── rename.py       # 标识符重命名模块
│       │   ├── control_flow.py # 控制流扁平化模块
│       │   ├── junk_code.py    # 垃圾代码插入模块
│       │   ├── string_encrypt.py # 字符串加密模块
│       │   ├── opaque.py       # 不透明谓词模块
│       │   ├── constant.py     # 常量混淆模块
│       │   ├── bytecode.py     # 字节码编译模块
│       │   └── binary.py       # 二进制加密模块
│       └── loader/
│           ├── py_impl/        # 纯 Python 加载器模板
│           │   ├── hook.py
│           │   └── keyring.py
│           └── cext/           # C 扩展加载器（appsec）源码
│               ├── appsec.c
│               ├── hook.c
│               ├── encfile.c
│               ├── crypto.c    # ChaCha20-Poly1305
│               ├── keyring.c
│               └── build/      # MSVC 与 gcc/容器构建脚本
├── bench/
│   ├── benchmarks/             # 基准测试集
│   └── restore/                # 还原测试集：混淆样本 + 原始代码
└── tests/
    ├── __init__.py
    ├── conftest.py             # pytest配置
    ├── test_analyzer.py
    ├── test_obfuscator.py
    ├── test_encryptor.py
    ├── test_line_mapper.py
    ├── test_compat.py
    ├── test_verify.py
    └── test_modules/
        ├── test_rename.py
        ├── test_control_flow.py
        ├── test_opaque.py
        └── test_integration.py
```

---

## 4. 核心模块设计

### 4.1 CLI接口层 (cli.py)

**职责：** 解析命令行参数，调用核心引擎处理

**实现：** 使用Python内置 `argparse` 模块。本工具自身不引入第三方依赖；`cryptography` 等作为可选依赖（纯 Python 加载器受标准库限制时才需要）

**命令设计：**

```bash
# 混淆命令
py-obfuscator obfuscate <input> [options]
  -o, --output          输出文件/目录路径
  -l, --level           混淆等级 (basic/medium/advanced)
  -e, --exclude         排除文件模式
  -c, --config          自定义配置文件路径
  -v, --verbose         详细输出
  --preserve-line-numbers 保留原始行号(用于日志)
  --preserve            动态特性保护清单（见 4.9）
  --hot-path            热点代码清单，用于降低施加强度（见 4.10）

# 加密命令
py-obfuscator encrypt <input> [options]
  -o, --output          输出文件/目录路径
  -m, --method          加密方式 (bytecode/binary)
  --loader              加载器形态 (python/cext)，默认 python
  -k, --key             加密密钥(binary模式，可选)

# 完整保护命令(混淆+加密)
py-obfuscator protect <input> [options]
  -o, --output          输出文件/目录路径
  -l, --level           混淆等级
  -e, --encrypt         加密方式
  --loader              加载器形态 (python/cext)
  --preserve-line-numbers 保留原始行号(用于日志)
  --preserve            动态特性保护清单
  --hot-path            热点代码清单

# 校验与度量命令
py-obfuscator verify <original> <protected>   # 语义等价校验（输入/输出/副作用比对）
py-obfuscator bench <dir>                     # 性能基准，报告相对未保护版的开销
```

**代码示例：**

```python
import argparse

def create_parser():
    parser = argparse.ArgumentParser(
        prog='py-obfuscator',
        description='Python代码混淆与加密工具'
    )
    
    subparsers = parser.add_subparsers(dest='command', help='可用命令')
    
    # obfuscate 子命令
    obfuscate_parser = subparsers.add_parser('obfuscate', help='混淆代码')
    obfuscate_parser.add_argument('input', help='输入文件或目录')
    obfuscate_parser.add_argument('-o', '--output', required=True, help='输出文件或目录')
    obfuscate_parser.add_argument('-l', '--level', choices=['basic', 'medium', 'advanced'], 
                                   default='basic', help='混淆等级')
    obfuscate_parser.add_argument('-e', '--exclude', nargs='+', help='排除文件模式')
    obfuscate_parser.add_argument('-c', '--config', help='自定义配置文件路径')
    obfuscate_parser.add_argument('-v', '--verbose', action='store_true', help='详细输出')
    obfuscate_parser.add_argument('--preserve-line-numbers', action='store_true', 
                                   help='保留原始行号(用于日志)')
    
    # encrypt 子命令
    encrypt_parser = subparsers.add_parser('encrypt', help='加密代码')
    encrypt_parser.add_argument('input', help='输入文件或目录')
    encrypt_parser.add_argument('-o', '--output', required=True, help='输出文件或目录')
    encrypt_parser.add_argument('-m', '--method', choices=['bytecode', 'binary'], 
                                 default='bytecode', help='加密方式')
    encrypt_parser.add_argument('--loader', choices=['python', 'cext'],
                                 default='python', help='加载器形态')
    encrypt_parser.add_argument('-k', '--key', help='加密密钥(binary模式)')
    
    # protect 子命令
    protect_parser = subparsers.add_parser('protect', help='完整保护(混淆+加密)')
    protect_parser.add_argument('input', help='输入文件或目录')
    protect_parser.add_argument('-o', '--output', required=True, help='输出文件或目录')
    protect_parser.add_argument('-l', '--level', choices=['basic', 'medium', 'advanced'], 
                                 default='advanced', help='混淆等级')
    protect_parser.add_argument('-e', '--encrypt', choices=['bytecode', 'binary'], 
                                 default='bytecode', help='加密方式')
    protect_parser.add_argument('--loader', choices=['python', 'cext'],
                                 default='python', help='加载器形态')
    protect_parser.add_argument('--preserve-line-numbers', action='store_true', 
                                 help='保留原始行号(用于日志)')
    protect_parser.add_argument('--preserve', nargs='+',
                                 help='动态特性保护清单(见 4.9)')
    protect_parser.add_argument('--hot-path', nargs='+',
                                 help='热点代码清单，降低施加强度(见 4.10)')
    
    # verify / bench 子命令
    verify_parser = subparsers.add_parser('verify', help='语义等价校验')
    verify_parser.add_argument('original', help='原始代码')
    verify_parser.add_argument('protected', help='保护后产物')
    
    bench_parser = subparsers.add_parser('bench', help='性能基准')
    bench_parser.add_argument('dir', help='基准测试集目录')
    
    return parser
```

---

### 4.2 代码分析器 (analyzer.py)

**职责：** 使用Python `ast` 模块解析源代码，提取符号信息和行号映射

**核心功能：**
- 解析Python源代码为AST抽象语法树
- 提取符号表（变量名、函数名、类名、模块名）
- 分析import依赖关系
- 检测代码结构（函数、类、循环、条件语句）
- **生成行号映射表（用于日志保留）**
- **检测动态特性用法**（`getattr` / `vars` / `__dict__` / `eval` / `pickle` / `dataclasses` 等按名称依赖代码的机制），输出给兼容层（4.9）作为保留依据

**输出结构：**

```python
@dataclass
class CodeAnalysis:
    ast_tree: ast.Module           # AST树
    symbols: SymbolTable           # 符号表
    imports: List[ImportInfo]      # 导入信息
    functions: List[FunctionInfo]  # 函数信息
    classes: List[ClassInfo]       # 类信息
    globals: List[str]             # 全局变量
    line_map: Dict[int, int]       # 行号映射 {混淆后行号: 原始行号}
    dynamic_refs: List[str]        # 按名称依赖代码的机制清单
```

---

### 4.3 行号映射器 (line_mapper.py)

**职责：** 保留原始行号信息，用于日志调试

**问题：**

```python
# 源码
def process_order(order):
    logger.info("开始处理订单")  # 第2行
    logger.debug(f"订单ID: {order.id}")  # 第3行
    result = calculate_price(order)  # 第4行
    logger.info(f"处理完成，价格: {result}")  # 第5行

# 混淆后
def a(b):
    c.info("开始处理订单")  # 第1行（行号变了！）
    d.debug(f"订单ID: {b.id}")  # 第2行
    e = f(b)  # 第3行
    c.info(f"处理完成，价格: {e}")  # 第4行

# 日志输出：
# 2024-01-15 10:30:00 - INFO - 开始处理订单 (obfuscated.py:1)  ← 无意义！
```

**解决方案：**

```python
# 1. 混淆时生成行号映射
line_map = {
    1: 2,  # 混淆后第1行 → 源码第2行
    2: 3,  # 混淆后第2行 → 源码第3行
    3: 4,  # 混淆后第3行 → 源码第4行
    4: 5,  # 混淆后第4行 → 源码第5行
}

# 2. 保存行号映射
import json
with open('line_map.json', 'w') as f:
    json.dump(line_map, f)

# 3. 自定义日志格式化器
import logging

class SourceLineFormatter(logging.Formatter):
    def __init__(self, line_map):
        super().__init__()
        self.line_map = line_map
    
    def format(self, record):
        # 转换行号
        if hasattr(record, 'lineno'):
            original_line = self.line_map.get(record.lineno, record.lineno)
            record.lineno = original_line
            record.filename = 'source.py'  # 恢复原始文件名
        return super().format(record)

# 4. 使用
line_map = load_line_map('line_map.json')
formatter = SourceLineFormatter(line_map)
handler.setFormatter(formatter)
```

**效果：**

```
# 日志输出：
# 2024-01-15 10:30:00 - INFO - 开始处理订单 (source.py:2)  ← 正确！
# 2024-01-15 10:30:00 - DEBUG - 订单ID: 12345 (source.py:3)  ← 正确！
# 2024-01-15 10:30:00 - INFO - 处理完成，价格: 100 (source.py:5)  ← 正确！
```

---

### 4.4 混淆引擎 (obfuscator.py)

**职责：** 根据配置策略调度各混淆模块

**设计模式：** 策略模式 + 管道模式

**混淆等级定义：**

| 等级 | 模块 | 描述 |
|------|------|------|
| basic | rename | 标识符重命名 |
| basic | remove_comments | 删除注释 |
| basic | remove_docstrings | 删除文档字符串 |
| medium | junk_code | 插入垃圾代码(ratio=0.1) |
| medium | string_encrypt | 字符串加密 |
| advanced | control_flow | 控制流扁平化(intensity=high) |
| advanced | opaque | 不透明谓词 |
| advanced | constant | 常量混淆 |

> 各等级不再标注"攻击成本"。对 LLM 而言，重命名/垃圾代码/字符串加密缺少公开数据，控制流扁平化有 C 语言参考量级，不透明谓词是参考数据中抗性最高的一档；这些差异在 Python 上是否成立需实测。

**处理流程：**

```python
def obfuscate(source: str, config: ObfuscateConfig) -> Tuple[str, Dict[int, int]]:
    # 1. 分析代码（含动态特性检测）
    analysis = analyzer.analyze(source)

    # 2. 根据等级选择模块，并套用 preserve 清单与热点豁免
    modules = select_modules(config.level)
    scope = compat.build_scope(analysis, config.preserve, config.hot_path)

    # 3. 管道处理
    result = source
    line_map = {}  # 行号映射

    for module in modules:
        result, module_line_map = module.process(result, analysis, scope)
        line_map.update(module_line_map)

    return result, line_map
```

---

### 4.5 加密引擎 (encryptor.py)

**职责：** 提供字节码编译和二进制加密两种方式

**字节码模式：**
- 使用 `py_compile.compile()` 编译为 `.pyc` 文件
- 可选优化级别（0/1/2）
- 输出为 `.pyc` 文件，Python可直接运行
- **注意：** `.pyc` 可被反编译工具还原，成功率取决于目标 Python 版本（3.10 及以上缺少可靠数据）。反编译是确定性过程，本模式的价值来自混淆成分，不来自编译本身

**二进制模式：**
- 逐模块 AEAD 加密（纯 Python 形态受标准库限制，无 AES，用 `hashlib` 构造流加密或引入可选依赖；C 扩展形态内嵌 ChaCha20-Poly1305）
- 输出为自定义二进制格式
- 运行时需要解密加载器
- **注意：** 密钥随软件分发，无法假设不可提取。加密层的价值在于抬高批量处理成本，不在阻断

**密钥管理原则：**

```python
# ❌ 错误：硬编码明文密钥
KEY = b'my_secret_key_12345'

# ❌ 错误：硬件绑定（离线场景无意义）
def get_key():
    mac = uuid.getnode()
    return hashlib.sha256(str(mac).encode()).digest()

# ✅ 正确：主密钥分片存放 + 按模块 salt 派生子密钥
#    注意：这只是抬高静态提取门槛，不改变"密钥必然随软件分发"的前提。
#    纯 Python 形态下拼装逻辑反编译即可还原；C 扩展形态需原生逆向。
def get_key():
    part1 = b'\x6d\x79\x5f'  # "my_"
    part2 = b'\x73\x65\x63\x72'  # "secr"
    part3 = b'\x65\x74\x5f\x6b\x65\x79'  # "et_key"
    return part1 + part2 + part3
```

---

### 4.6 模块接口 (modules/base.py)

**统一接口定义：**

```python
class ObfuscationModule(ABC):
    """混淆模块基类"""
    
    @property
    @abstractmethod
    def name(self) -> str:
        """模块名称"""
        pass
    
    @property
    @abstractmethod
    def description(self) -> str:
        """模块描述"""
        pass
    
    @abstractmethod
    def process(self, source: str, analysis: CodeAnalysis, scope: PreserveScope) -> Tuple[str, Dict[int, int]]:
        """处理源代码，返回处理后的代码和行号映射。
        scope 指明本模块可变换的符号/语句范围（兼容层与热点豁免的结果）"""
        pass
    
    @abstractmethod
    def validate_config(self, config: dict) -> bool:
        """验证配置"""
        pass
```

---

### 4.7 加载器：两种形态

加密模式（4.5 二进制模式）需要一个解密加载器。加载器有两种实现路径，成本与防护效果差异明显，工具两种都支持，由 `--loader python|cext` 选择。

#### 4.7.1 两种形态的取舍

| 维度 | 纯 Python 加载器 | C 扩展加载器 |
|---|---|---|
| **密钥保护** | 弱。密钥拼装逻辑以字节码形式交付，反编译即可还原，LLM 可加速 | 较强。需原生逆向 + 动态跟踪才能定位密钥 |
| **批量解密门槛** | 低。解密接口在 Python 层暴露，可被脚本直接批量调用 | 较高。接口不暴露解密函数，无法作为黑盒复用 |
| **加密算法选择** | 受标准库限制。标准库无 AES，只能用 `hashlib` 构造流加密 | 可内嵌任意算法（ChaCha20-Poly1305），无外部依赖 |
| **构建成本** | 低。一次编写，全平台通用，无需 C 工具链 | 高。按 OS × 架构 × Python 小版本分别编译，并需持续回归 |

**实质差异只有两处**：密钥能否被反编译得到，解密能力能否被脚本批量复用。C 扩展在这两点上更强，代价是构建矩阵与维护成本。其余方面两者一致——都不改变"运行时内存可读"这一边界，也都需要随软件分发密钥材料。

**纯 Python 形态的构成**：一个 `sys.meta_path` finder + loader（Python 类），负责读 `.enc`、解析头、派生子密钥、解密、`exec`。密钥分片与拼装逻辑以 Python 代码存在。适用于快速验证加密流水线，以及构建环境不具备 C 工具链的场景。

#### 4.7.2 C 扩展形态（appsec）

加密、解密与密钥管理全部收敛到原生扩展中，Python 层不持有解密能力。

**交付结构：**

```
app/
  main.pyc                              ← 入口，仅一行引导: import appsec; appsec.install("app.pkg")
  appsec.cp311-win_amd64.pyd            ← Windows构建
  appsec.cpython-311-x86_64-linux-gnu.so ← Linux构建
  pkg/
    core.enc                            ← 混淆+编译+加密的字节码（全平台通用）
    util.enc
```

`.pyd`/`.so`按平台+Python小版本命名，`import appsec`时解释器自动选择当前平台的二进制。Python层只保留一行明文引导代码，逻辑做到最小（不含业务信息）。

**C扩展内部模块划分：**

```
appsec.c            PyInit_appsec：版本校验 + 暴露唯一接口 install(prefix)
├── hook.c          注册sys.meta_path finder（用PyCFunction实现，非Python类）
├── encfile.c       .enc头解析、完整性tag校验
├── crypto.c        ChaCha20-Poly1305（内嵌公有领域纯C实现，无外部依赖）
└── keyring.c       密钥分片拼装 + 按模块salt派生子密钥
```

**核心执行路径（exec_module）：**

```c
static PyObject* loader_exec(PyObject* self, PyObject* module) {
    uint8_t* blob; size_t len;
    if (read_file(path, &blob, &len) < 0) return NULL;       // 1) 读.enc，读完即关句柄

    enc_header hdr = parse_header(blob);                      // 2) 校验magic/版本，取salt/nonce/tag

    uint8_t master[32]; keyring_assemble(master);             // 3) 分片拼装主密钥（.data段混淆存储）
    uint8_t sub[32];    kdf(master, hdr.salt, sub);           //    派生模块子密钥

    uint8_t* plain = malloc(len);                             // 4) 内存中解密，明文不落盘、不经PyBytes
    if (aead_decrypt(sub, hdr.nonce, blob + hdr.size, len - hdr.size, plain, hdr.tag) < 0) {
        secure_bzero(plain, len); free(plain);
        return_error("integrity check failed");               //    tag不符=文件被篡改，直接终止
    }

    PyObject* code = PyMarshal_ReadObjectFromString((char*)plain, plain_len);
    secure_bzero(plain, len); free(plain);                    // 5) marshal还原后立即覆零解密缓冲
    if (!code) return NULL;

    PyObject* dict = PyObject_GetAttrString(module, "__dict__");
    PyObject* r = PyEval_EvalCode(code, dict, dict);          // 6) 执行，全程无明文文件
    Py_XDECREF(r); Py_DECREF(code); Py_DECREF(dict);
    return r ? Py_None : NULL;
}
```

要点：
- 加密端流水线：`混淆后源码 → py_compile → 剥16字节.pyc头 → marshal数据段 → AEAD加密 → .enc`
- `.enc`布局：`magic(4B) | 版本(1B) | 算法ID(1B) | salt(16B) | nonce(12B) | tag(16B) | 密文`；头部逐字段`memcpy`解析（不直接memcpy结构体），保证跨字节序安全
- `secure_bzero`：Windows用`SecureZeroMemory`，Linux用`explicit_bzero`（老glibc回退volatile手写循环），防编译器优化掉清零
- 只使用公开稳定C-API（`PyEval_EvalCode`等），不用私有符号，保证Linux下"不链接libpython、符号由宿主解释器解析"的标准扩展模式可用
- code object的`co_filename`写虚拟路径，行号映射链（§8）在加载器方案下继续有效

**密钥策略：**

- 密钥分片存放于`.data`段并做运行时变形拼装，任一静态片段不构成可用密钥
- 主密钥派生每模块子密钥（salt取自.enc头），单模块密钥泄露不影响其他模块
- 每个发行版本重新生成密钥并重跑加密流水线，密钥提取只对当前版本有效

**版本锁定（硬约束）：**

marshal格式绑定CPython小版本，加密端与运行端必须一致：

- 一个二进制只配一个Python小版本，`PyInit_appsec`校验`PY_VERSION_HEX`不匹配即抛`ImportError`（而不是运行到marshal处崩溃）
- 加密产物记录目标版本，发行包说明兼容范围

**跨平台构建：**

核心C代码平台无关，差异收敛到构建脚本与少量宏。加密一次、多平台编译二进制：

```cmake
find_package(Python 3.11 EXACT REQUIRED COMPONENTS Development)
add_library(appsec MODULE appsec.c hook.c encfile.c crypto.c keyring.c)
set_target_properties(appsec PROPERTIES PREFIX "" PythonExtension True)  # 自动产出正确的平台命名
if (WIN32)
    target_compile_options(appsec PRIVATE /O2 /GS /guard:cf)
else()
    target_compile_options(appsec PRIVATE -O2 -fvisibility=hidden -fstack-protector-strong)
endif()
```

发布流水线：

```
encrypt一次（任一平台） → pkg/*.enc
    ├─ Windows: MSVC编译 → .pyd
    └─ Linux:   manylinux_2_17容器 + gcc编译 → .so   # 容器构建解决glibc版本下限问题
汇总 → 分平台发行包（共享同一份.enc）
```

补充说明：
- `.enc`与OS/CPU无关（marshal产物跨平台一致），差异仅在Python小版本
- 路径拼接沿用 Python 侧的 `os.path`（由 C 层调用，避免自己实现分隔符规则）；C 层只接收完整路径字符串，文件读写走`fopen`/`fread`+UTF-8，两边行为一致
- 构建容器/工具链属构建期依赖，不进入交付物；扩展本身自研，不引入第三方运行库

**威胁模型定位：** C扩展把"提取密钥""批量解密"的门槛从脚本级抬到原生逆向级；但"运行时内存dump code object"仍是无法防御的固有边界（Linux上`gdb`/`/proc/pid/mem`更易得手），这一点不因实现语言改变，防护目标仍是分层拦截而非防止。

---

### 4.8 语义等价校验 (core/verify.py)

**定位：** 混淆与加密都要求语义不变，但正确性目前只能靠人工测试。本模块提供机器可执行的等价校验，把"变换没改行为"从信念变成检查项。

**校验层次（由弱到强）：**

| 层次 | 方法 | 能发现什么 |
|------|------|-----------|
| L1 可导入 | `import` 产物，捕获异常 | 语法错误、动态特性被破坏导致的启动失败 |
| L2 结构比对 | AST 归一化后比对（忽略标识符名、注释、docstring、垃圾语句） | 意外改动了控制流或常量 |
| L3 行为比对 | 对给定输入集，分别执行原始版与保护版，比较返回值、异常类型、stdout、副作用快照 | 逻辑变化、字符串/常量解密错误 |
| L4 差分回归 | 接入项目已有测试套件，对保护版运行同一套测试 | 集成层面的破坏 |

L3 需要测试输入集；无输入集时降级到 L1+L2+L4，并在报告中明确标注校验强度，避免把"没报错"读成"等价"。

**输出：** 校验报告（通过项、差异样本、覆盖到的模块清单），随构建产物归档。

---

### 4.9 动态特性兼容层 (core/compat.py)

**定位：** 混淆可能破坏依赖名称/结构的 Python 机制。本层把"哪些代码不许改名、不许扁平化"变成显式配置与自动检测的并集，传给各变换模块作为作用域。

**可能受影响的机制：**

| 机制 | 破坏方式 | 处理 |
|------|---------|------|
| `getattr` / `setattr` / `hasattr` 字符串查找 | 名称被改后查不到 | 收集字面量参数中的名称，纳入保留清单 |
| `vars()` / `__dict__` / 序列化（`pickle`、`json.dumps` 字段名） | 字段名改变导致数据不兼容 | 整个类保留属性名 |
| `dataclasses` / `NamedTuple` / `TypedDict` | 字段名是对外契约 | 保留字段名 |
| 装饰器与元类反射 | 依赖 `__name__` / `__qualname__` | 保留相关函数名，或改写为装饰器可识别形式 |
| `eval` / `exec` 中的字符串代码 | 内部名称与外部改名不一致 | 检测后默认不改名，并给出告警 |
| 模块级对外 API（`__all__`、包入口） | 改名破坏调用方 | 白名单保留 |
| 异常信息 / 日志中的类名函数名 | 改名后难以定位 | 由行号映射（4.3）配合，或保留可诊断名 |

**配置：**

```ini
[preserve]
; 显式清单：模块/类/函数名模式，命中即跳过改名与扁平化
patterns = api.*, *.handle_*, __all__
; 自动检测结果是否强制生效
auto = true
; 检测到高风险用法时的行为：warn | skip | fail
on_dynamic = warn
```

`--preserve` 命令行清单与配置文件合并，自动检测结果作为兜底：即使命中项未出现在清单里，也不会被改名。

---

### 4.10 热点代码豁免（强度分级）

**定位：** 混淆开销随强度非线性上升，控制流扁平化施加于计算密集代码可达数倍开销。在 20% 预算口径确定前，实现层面先预留分级施加的能力。

**三条识别路径（择一或组合，待评审确认）：**

| 路径 | 做法 | 代价 |
|------|------|------|
| 自动分析 | 以 AST 统计循环嵌套深度、语句密度、被调用频次（需运行时采样） | 需要 profiling 基础设施 |
| 静态启发式 | 按函数体大小与循环深度阈值判定 | 误判率高，但零额外依赖 |
| 人工标注 | 由开发者在配置中列出热点模块/函数 | 准确但难以长期维护 |

**作用方式：** 命中项降低施加强度（跳过控制流扁平化、保留重命名与字符串加密），而非整体跳过保护。分级结果写入构建报告，使"哪些代码只受弱保护"可被审计。

**与预算的关系：** 分母口径（整机端到端 / 关键路径）未定前，本节只提供机制，不给出具体阈值。

---

## 5. 混淆模块详细设计

### 5.1 重命名模块 (modules/rename.py)

**功能：** 将变量名、函数名、类名重命名为无意义的名称

**策略：**
- 生成无意义名称（`a`, `b`, `c` 或 `var_1`, `func_2`）
- 保留特殊方法（`__init__`, `__str__` 等）
- 可配置排除列表
- **作用域受兼容层约束**（4.9）：命中 `preserve` 清单或被自动检测出动态名称依赖的符号，不参与改名

**示例：**

```python
# 输入
def calculate_total_price(items):
    total = 0
    for item in items:
        total += item.price
    return total

# 输出
def a(b):
    c = 0
    for d in b:
        c += d.price
    return c
```

---

### 5.2 控制流混淆模块 (modules/control_flow.py)

**功能：** 打乱控制流结构，增加反编译难度

**技术：**
- 控制流扁平化
- 循环展开/折叠

（不透明谓词独立为 5.5，避免与扁平化耦合在同一模块中配置）

**示例：**

```python
# 输入
if x > 0:
    print("positive")
else:
    print("negative")

# 输出（扁平化后）
control_var = 0
while True:
    if control_var == 0:
        if x > 0:
            control_var = 1
        else:
            control_var = 2
    elif control_var == 1:
        print("positive")
        break
    elif control_var == 2:
        print("negative")
        break
```

---

### 5.3 垃圾代码模块 (modules/junk_code.py)

**功能：** 插入无副作用的垃圾代码

**策略：**
- 插入无用变量声明
- 插入恒假分支（形式复杂、实际永不成立）
- 插入无用循环

**配置：**
- `ratio`: 垃圾代码比例（0.0-1.0）。分母定义与 20% 预算口径绑定，口径未定前该值仅用于内部一致性比较

---

### 5.4 字符串加密模块 (modules/string_encrypt.py)

**功能：** 加密字符串常量，运行时解密

**技术：**
- 每个字符串独立密钥或主密钥派生，运行时通过解密函数还原
- 可选是否缓存解密结果——在循环内重复解密同一字符串会显著放大开销

**示例：**

```python
# 输入
message = "Hello, World!"

# 输出
message = _d("SGVsbG8sIFdvcmxkIQ==", _k)  # 运行时解密
```

> 敏感字面量（接口地址、密钥、内部标识）在明文 `.py` 形态下可被文本扫描直接定位，本模块针对的就是这一路径。

---

### 5.5 不透明谓词模块 (modules/opaque.py)

**功能：** 注入恒真或恒假、但形式上复杂的条件，使分支可达性分析失效

**策略：**
- 基于数论性质（如 `x*(x-1) % 2 == 0` 恒真）
- 基于变量取值范围不变式
- 生成的谓词不得依赖可能被外部修改的状态，否则可能破坏语义

**优先级说明：** 在现有的公开评测（170 段 C 程序，LLM 语义还原）中，不透明谓词的还原难度是四种变换里最高的一档，而其他三类技术（重命名、垃圾代码、字符串加密）缺少公开数据、控制流扁平化的效果低于传统预期。因此本模块作为 advanced 档的默认项之一。

> 上述结论来自 C 语言实验，在 Python 上是否成立属未验证假设（见 2.2 的设计准则）。是否真正有效以 11.2 的还原测试集实测为准。

---

### 5.6 常量混淆模块 (modules/constant.py)

**功能：** 将字面量常量改写为等价的表达式形式

**示例：**

```python
# 输入
timeout = 3600

# 输出
timeout = (_a * _b) + _c  # 值相同，形式不再是字面量
```

**作用边界：** 阻止对常量表的文本扫描；对决心执行等价计算的攻击者（人或脚本）不构成障碍。

---

## 6. 配置文件设计

### 6.1 预设配置 (config/presets.ini)

```ini
; 基础混淆
[basic]
rename = true
remove_comments = true
remove_docstrings = true
junk_code = false
string_encrypt = false
control_flow = false
opaque = false

; 中等混淆
[medium]
rename = true
remove_comments = true
remove_docstrings = true
junk_code_ratio = 0.1
junk_code_max_statements = 5
string_encrypt = true
control_flow = false
opaque = false

; 高级混淆
[advanced]
rename_min_length = 3
rename_excluded = __init__,__str__,__repr__
remove_comments = true
remove_docstrings = true
junk_code_ratio = 0.2
junk_code_max_statements = 10
string_encrypt = true
control_flow_intensity = high
opaque = true
constant_obfuscation = true

; 动态特性保护（4.9）
[preserve]
patterns = api.*, *.handle_*, __all__
auto = true
on_dynamic = warn          ; warn | skip | fail

; 热点豁免（4.10）。识别路径待评审确认，机制先就位
[hotpath]
mode = manual              ; auto | heuristic | manual
functions =                ; 命中项降低强度：跳过扁平化，保留改名与字符串加密

; 加密与加载器
[encrypt]
method = binary            ; bytecode | binary
loader = python            ; python | cext
target_python = 3.11       ; cext / bytecode 模式下的小版本绑定

[verify]
level = L3                 ; L1 | L2 | L3 | L4，无输入集时自动降级并标注
inputs = tests/fixtures/inputs
```

> `string_encrypt` 不再配置算法名：纯 Python 产物内标准库无 AES，统一使用 `hashlib` 构造的流加密（密钥派生见 4.5）；cext 形态由加载器内 ChaCha20-Poly1305 承担。混淆等级与加密算法是两个正交维度，避免在预设里混写。

**Python配置读取示例：**

```python
import configparser

def load_presets(config_path='config/presets.ini'):
    config = configparser.ConfigParser()
    config.read(config_path)
    
    presets = {}
    for section in config.sections():
        presets[section] = dict(config.items(section))
    
    return presets
```

---

## 7. 运行流程

### 7.1 代码混淆

```
源码 (source.py)
    ↓ 混淆处理
混淆后代码 (obfuscated.py)
    ↓ 直接运行
python obfuscated.py
```

**与源码的区别：**

| 方面 | 源码 | 混淆后代码 |
|------|------|-----------|
| 可读性 | 变量名有意义，有注释 | 变量名无意义（a, b, c），无注释 |
| 功能 | 完全相同 | 完全相同 |
| 文件格式 | .py 文本文件 | .py 文本文件 |
| 运行方式 | `python source.py` | `python obfuscated.py` |
| 性能 | 正常 | 可能略慢（多了垃圾代码） |

---

### 7.2 字节码编译

```
源码 (source.py)
    ↓ py_compile.compile()
字节码文件 (source.pyc)
    ↓ 直接运行
python source.pyc
```

**与源码的区别：**

| 方面 | 源码 | 字节码文件 |
|------|------|-----------|
| 可读性 | 可读的Python代码 | 二进制字节码，不可读 |
| 功能 | 完全相同 | 完全相同 |
| 文件格式 | .py 文本文件 | .pyc 二进制文件 |
| 运行方式 | `python source.py` | `python source.pyc` |
| 反编译 | 不涉及 | ⚠️ 可被 decompyle3 / pycdc 等还原；成功率取决于目标 Python 版本，3.10+ 缺可靠数据 |
| 版本绑定 | 无 | 绑定 CPython 小版本，构建端与运行端必须一致 |

> `.pyc` 这一层对 LLM 不构成障碍——还原是确定性的。本形态（对应候选 C）的实际防护力来自混淆成分，不来自编译步骤。

---

### 7.3 二进制加密

```
字节码 (source.pyc，剥去文件头)
    ↓ AEAD 加密（逐模块 salt / nonce）
加密文件 (source.enc)
    ↓ 运行时由加载器解密
加载器（纯 Python 或 C 扩展，见 4.7）
```

**与源码的区别：**

| 方面 | 源码 | 加密文件 |
|------|------|---------|
| 可读性 | 可读的Python代码 | 密文，静态不可读 |
| 功能 | 完全相同 | 完全相同 |
| 文件格式 | .py 文本文件 | .enc 二进制文件 |
| 运行方式 | `python source.py` | 由加载器接管 import |
| 密钥安全 | 不涉及 | ⚠️ 密钥随软件分发，无法假设不可提取 |
| 额外开销 | 无 | 一次性解密 + 常驻 import hook |

---

### 7.4 完整保护（对应候选 D）

```
构建期：
  源文件 .py
    ↓ 混淆（按等级 + preserve 清单 + 热点豁免）
  混淆后 .py
    ↓ py_compile → 剥 16 字节 .pyc 头
  marshal 数据段
    ↓ 逐模块 salt/nonce → AEAD 加密
  pkg/*.enc + 加载器（python 或 cext）
    ↓ verify（等价校验）+ bench（性能基准）
  分平台发行包

目标机：
  import appsec; appsec.install("app.pkg")   ← 唯一明文引导
    ↓ 注册 sys.meta_path Finder
  业务 import 命中 → 读 .enc → 派生子密钥 → 校验 tag → 内存解密
    ↓ marshal 还原 code object → PyEval_EvalCode
  执行，全程无明文文件落盘
```

**攻击面（不做时间估算）：**

- 静态取不到明文；重心移向密钥提取与加载器分析。
- 若攻破密钥，得到的仍是混淆代码，仍需重建语义——这是相对候选 C 的额外一层。
- 批量处理需先解决密钥与加载器，单次成本被放大到每个模块。这是加密层的主要实际价值。
- 运行时内存读取不在防御范围内（见 2.2）。

---

## 8. 日志行号保留

### 8.1 问题

混淆后代码行号会变化，日志中的行号就失去了意义。

```python
# 源码
logger.info("开始处理订单")  # 第2行

# 混淆后
c.info("开始处理订单")  # 第1行（行号变了！）

# 日志输出：
# 2024-01-15 10:30:00 - INFO - 开始处理订单 (obfuscated.py:1)  ← 无意义！
```

### 8.2 解决方案

**方案1：行号映射（推荐）**

```python
# 混淆时生成行号映射
line_map = {
    1: 2,  # 混淆后第1行 → 源码第2行
    2: 3,  # 混淆后第2行 → 源码第3行
}

# 自定义日志格式化器
class SourceLineFormatter(logging.Formatter):
    def format(self, record):
        record.lineno = line_map.get(record.lineno, record.lineno)
        return super().format(record)
```

**方案2：逻辑行号**

```python
# 不用物理行号，用函数内相对行号
logger.info("[process_order:1] 开始处理订单")
logger.info("[process_order:2] 订单ID: 12345")
```

**方案3：编译时保留行号**

```python
# 编译时保留行号信息
python -m py_compile -O obfuscated.py
# .pyc文件会保留行号，日志能正确显示
```

### 8.3 映射表的处置（待评审）

行号映射本身是一份"混淆名 → 原始名"的对照表，属于敏感材料。需要明确：

| 问题 | 选项 | 影响 |
|---|---|---|
| 是否随软件分发 | 分发 / 只留内部 | 分发则等于提供还原线索；不留内部则线上排障受限 |
| 历史版本归档 | 保留哪些版本 | 旧版本崩溃栈需要旧映射表才能解析 |
| 加载器形态下的差异 | `.pyc` / `.enc` 中 `co_filename` 写虚拟路径 | 映射链在两种形态下都需保持有效（见 4.7.2） |

---

## 9. 安全设计要点

### 9.1 实现层面的原则

| 原则 | 说明 |
|------|------|
| 不假设密钥安全 | 密钥必然随软件分发。加密层的价值在抬高批量处理成本，不在阻断 |
| 不假设混淆层能守住秘密 | 遵循 2.2 的准则：可"静态读出 + 移植执行"获得的秘密，混淆不视为防线 |
| 分层而非单点 | 静态不可读（加密）、语义难读（混淆）、批量难复用（加载器）各自负责不同环节 |
| 不做硬件绑定 | 与离线安装矛盾（2.4） |
| 不以纯 `.pyc` 为终点 | 反编译确定性可还原，价值仅来自混淆成分 |
| 效果以实测为准 | 等价校验（4.8）、性能基准（bench）、还原测试集（11.2）三项产出决定各档位是否值得保留 |

### 9.2 密钥管理原则

1. **永远不要硬编码明文密钥**
2. **主密钥分片存放 + 按模块 salt 派生子密钥**：单模块泄露不影响其他模块；每个发行版本重新生成密钥并重跑流水线，使提取结果只对当前版本有效
3. **不依赖密钥安全性**：其前提是"密钥无法被静态提取"，本项目的交付模式不满足
4. **解密能力不对外暴露**：cext 形态的接口只暴露 `install(prefix)`，不提供可供脚本批量调用的解密函数

### 9.3 与其他候选的关系

本工具实现的是候选 A / C / D（含加载器两种形态）。候选 F（商业工具）、G（Nuitka）、H（Cython）、E（白盒）不在实现范围内，是否引入由方案评审结论决定。评审未定之前，本工具的构件按可组合方式设计：混淆流水线可单独产出 `.py` 或 `.pyc`，也可作为加密的前置阶段，加密层可替换加载器形态。

---

## 10. 依赖项

### 10.1 工具自身（构建期）

- Python 3.8+（建议与目标运行版本一致）
- 内置模块：argparse、configparser、ast、py_compile、hashlib、importlib
- cext 形态的构建需要 C 工具链：MSVC（Windows）、gcc + manylinux 容器（Linux）。属构建期依赖，不进入交付物
- 可选：`pytest`（测试）、`cryptography`（纯 Python 加载器需要标准库之外的 AEAD 时）

### 10.2 交付物（运行期）

| 产物形态 | 运行期依赖 | 版本 / 平台绑定 |
|---|---|---|
| `.py`（仅混淆） | 无 | 无 |
| `.pyc`（混淆 + 编译） | 宿主 CPython | 绑定小版本 |
| `.enc` + 纯 Python 加载器 | 宿主 CPython | 绑定小版本（字节码模式） |
| `.enc` + cext 加载器 | 宿主 CPython（不链接 libpython，符号由宿主解析） | 绑定小版本 + OS + CPU 架构 |

> 依赖没有硬性约束（1.2）。上表按成本项列出，供评审衡量各形态的部署代价。

---

## 11. 验收标准

### 11.1 功能验收

- [ ] 支持单文件和目录处理
- [ ] 实现 3 种混淆等级（basic/medium/advanced），含不透明谓词与常量混淆
- [ ] 实现 2 种加密方式（bytecode/binary）与 2 种加载器形态（python/cext）
- [ ] `preserve` 清单与动态特性自动检测生效，命中项不被改名/扁平化
- [ ] 热点豁免机制可用，分级结果写入构建报告
- [ ] 日志行号保留功能，且映射表处置策略可配置
- [ ] `verify` 达到所声明的校验等级（无输入集时自动降级并在报告中标注）
- [ ] `bench` 可输出相对未保护版的开销
- [ ] cext 形态在目标平台全部构建通过，Python 小版本不匹配时抛 `ImportError` 而非中途崩溃
- [ ] CLI 命令可用，文档完整，测试通过

### 11.2 效果验证（实施后必做，产出数据供评审回填）

| # | 项目 | 方法 | 现状 |
|---|------|------|------|
| 1 | **LLM 对候选技术的还原能力** | 取代表性代码分别施加各类技术，交 LLM 做语义还原，记录成功率与耗时 | 现有数据全来自 C 程序，Python 上无数据（最大缺口） |
| 2 | 目标版本下 `.pyc` 反编译成功率与产物可读性 | 按目标 Python 版本实测 decompyle3 / pycdc | "95%+"仅适用于 3.9 及以下 |
| 3 | 各等级实际性能开销 | 基准测试集，按 1.2 口径确定分母 | 目前仅有量级估计 |
| 4 | 垃圾代码插入、字符串加密、不透明谓词的 AI 抗性 | 并入第 1 项 | 无数据 |
| 5 | 相对候选 C，"多一道编译"是否带来额外防护 | A 与 C 产物分别做还原测试 | 不应凭直觉假定 |
| 6 | cext 形态的实际密钥提取门槛 | 由具备原生逆向能力者尝试，或评估工具链成本 | 无数据 |

第 1、2 项决定各技术档位与 `.pyc` 形态的取舍，属于验收的一部分而非事后补充。

---

## 12. 风险与缓解

| 风险 | 影响 | 缓解措施 |
|------|------|----------|
| 混淆破坏动态特性导致运行失败 | 高 | 兼容层（4.9）：自动检测 + `preserve` 白名单 + `on_dynamic` 策略；`verify` L3/L4 兜底 |
| 性能开销超 20% 预算 | 中 | 热点豁免（4.10）分级施加；`bench` 前置到构建流程；预算口径待评审明确 |
| 语义被意外改变 | 高 | 等价校验（4.8），差异样本进构建报告 |
| 加密被误当作阻断手段 | 中 | 文档与设计准则明确：加密价值在批量成本，密钥假设可被提取（2.2 / 9.1） |
| 基于 C 语言实验的数据被过度外推 | 中 | 各处标注证据边界；效果以 11.2 实测为准，不以参考量级作决策依据 |
| 日志行号丢失导致线上不可排障 | 中 | 行号映射（第 8 章）+ 映射表处置策略（8.3） |
| cext 构建矩阵维护成本 | 中 | 平台差异收敛到构建脚本与少量宏；小版本不匹配提前 fail-fast |
| 跨版本 marshal 不兼容 | 高 | 加密产物记录目标版本；`PyInit_appsec` 校验 `PY_VERSION_HEX` |

---

## 13. 附录：还原路径与已知边界

### 13.1 反编译工具与支持范围

| 工具 | 支持的 Python 版本 | 说明 |
|------|---------|-----------|
| uncompyle6 | 2.7, 3.3–3.9 | 不支持 3.10 及以上 |
| decompyle3 | 3.8–3.11 | |
| pycdc / pycdas | 2.7–3.x | 高版本支持取决于具体构建 |

> 常被引用的"95%+ 成功率"来源为 uncompyle6 在 3.9 及以下的经验，**对 3.10 及以上不适用**。当前发布形态的实际还原难度需按目标版本实测（11.2 第 2 项）。

### 13.2 各形态的攻击路径（不做成本估算）

| 形态 | 攻击者需要完成的步骤 | 关键变量 |
|------|---------------------|---------|
| 纯 `.pyc`（现状） | 直接反编译 | 目标 Python 版本的工具支持度 |
| A 混淆 `.py` | 阅读并重建语义 | 所选混淆技术对 LLM 的实际抗性 |
| C 混淆 `.pyc` | 反编译 → 重建语义 | 同上；"多一道编译"是否增加门槛待实测 |
| D 混淆 `.enc` + Python 加载器 | 反编译加载器 → 取密钥 → 批量解密 → 重建语义 | 加载器可反编译，密钥提取为脚本级 |
| D 混淆 `.enc` + cext 加载器 | 原生逆向定位密钥 → 复现解密 → 重建语义 | 技能栈从还原字节码变为逆向机器码 |

### 13.3 结论性表述

**离线软件无法做到"绝对安全"。** 本工具的目标是抬高"提取并复用代码逻辑"的门槛，具体贡献为：产物不可直接复用、批量处理需要额外工作、语义重建需要人工投入。

各档位的实际效果缺少 Python 环境数据，须由 11.2 的实测结果支撑后再回填本文件；在此之前不给出成本时间、成功率或强度评分。
