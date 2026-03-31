# xmJSON 开发管理文档

## 项目概述

### 项目名称
**xmJSON** - eXtreme Memory JSON Parser

### 项目定位
高性能、零拷贝、内存优化的JSON解析库，专为高频数据访问与网络分发场景设计。

### 核心特性
- **双层API设计**：构建器API + 只读访问器API
- **零拷贝访问**：通过mmap直接访问二进制文件
- **内存优化**：Arena分配器，一次分配，零碎片
- **UTF-8完整支持**：正确处理多字节字符
- **高性能**：解析速度 > 1GB/s，访问速度接近内存访问

### 技术指标

| 指标 | 目标值 | 备注 |
|------|--------|------|
| 解析速度 | ≥ 500 MB/s | JSON文本解析 |
| 加载时间 | < 1ms | 二进制文件mmap加载 |
| 内存占用 | ≤ 文件大小 + 10% | 零拷贝模式 |
| 支持JSON大小 | ≤ 2GB | 32位系统限制 |
| API响应时间 | < 100ns | 字段访问延迟 |

---

## 版本规划

### v0.1.0 - 原型验证 (2026-04-15)
- [x] UTF-8字符处理模块
- [x] 基础状态机解析器
- [x] 单元测试框架

### v0.2.0 - 核心解析器 (2026-05-01)
- [ ] JSON5语法完整支持
- [ ] 注释解析
- [ ] 转义字符处理
- [ ] 错误恢复机制

### v0.3.0 - 构建器API (2026-05-15)
- [ ] Arena内存分配器
- [ ] JSON文本解析器
- [ ] 节点树构建
- [ ] 编辑操作API

### v0.4.0 - 二进制格式 (2026-06-01)
- [ ] 二进制格式设计
- [ ] 序列化器实现
- [ ] 反序列化器实现
- [ ] 格式版本管理

### v0.5.0 - 只读访问器 (2026-06-15)
- [ ] mmap加载支持
- [ ] 零拷贝访问API
- [ ] 迭代器实现
- [ ] 性能统计

### v1.0.0 - 正式发布 (2026-07-01)
- [ ] 完整文档
- [ ] 性能测试报告
- [ ] 跨平台测试
- [ ] 安全审计

---

## 架构设计

### 模块划分

```
xmjson/
├── include/
│   ├── xmjson.h              # 公共API
│   ├── xmjson_builder.h      # 构建器API
│   ├── xmjson_reader.h       # 只读访问器API
│   └── xmjson_binary.h       # 二进制格式定义
├── src/
│   ├── core/
│   │   ├── utf8.c            # UTF-8处理
│   │   ├── parser.c          # 状态机解析器
│   │   ├── arena.c           # Arena分配器
│   │   └── error.c           # 错误处理
│   ├── builder/
│   │   ├── builder.c         # 构建器实现
│   │   ├── editor.c          # 编辑操作
│   │   └── serializer.c      # 序列化器
│   ├── reader/
│   │   ├── loader.c          # 文件加载
│   │   ├── accessor.c        # 访问器实现
│   │   └── iterator.c        # 迭代器
│   └── binary/
│       ├── format.c          # 格式处理
│       └── compat.c          # 兼容性处理
├── tests/
│   ├── unit/                 # 单元测试
│   ├── integration/          # 集成测试
│   ├── performance/          # 性能测试
│   └── fuzz/                 # 模糊测试
├── examples/
│   ├── build_config.c        # 构建示例
│   ├── load_config.c         # 加载示例
│   └── embed_data.c          # 嵌入数据示例
├── tools/
│   ├── xmjsonc               # 命令行工具
│   └── benchmark             # 性能基准
└── docs/
    ├── api/                  # API文档
    ├── format.md             # 二进制格式说明
    └── internals.md          # 内部实现文档
```

### 核心数据结构

```c
// UTF-8字符
typedef struct {
    const char* bytes;     // UTF-8字节（最多4字节）
    int pos;               // 字节位置
    int len;               // 字节长度
    uint32_t code;         // Unicode码点
} xmjson_utf8_char;

// 解析上下文（状态机）
typedef struct {
    const char* input;
    size_t pos;
    int line;
    int col;
    size_t len;
    int error_code;
    xmjson_utf8_char ch;
} xmjson_parse_ctx;

// 节点类型
typedef enum {
    XMJ_NULL,
    XMJ_FALSE,
    XMJ_TRUE,
    XMJ_NUMBER,
    XMJ_STRING,
    XMJ_ARRAY,
    XMJ_OBJECT
} xmjson_type;

// 二进制节点（紧凑布局）
typedef struct {
    uint8_t type;
    uint8_t flags;
    uint16_t padding;
    union {
        double number;
        struct {
            uint32_t offset;
            uint32_t length;
        } string;
        struct {
            uint32_t offset;
            uint32_t count;
        } container;
    };
} xmjson_binary_node;
```

---

## API设计

### 构建器API

```c
// 创建和销毁
xmjson_builder* xmjson_builder_create(void);
void xmjson_builder_free(xmjson_builder* builder);

// 解析
int xmjson_builder_parse(xmjson_builder* builder, const char* json, size_t len);
int xmjson_builder_parse_file(xmjson_builder* builder, const char* filename);

// 创建节点
xmjson_node* xmjson_create_null(xmjson_builder* builder);
xmjson_node* xmjson_create_boolean(xmjson_builder* builder, int value);
xmjson_node* xmjson_create_number(xmjson_builder* builder, double value);
xmjson_node* xmjson_create_string(xmjson_builder* builder, const char* value);
xmjson_node* xmjson_create_array(xmjson_builder* builder);
xmjson_node* xmjson_create_object(xmjson_builder* builder);

// 对象操作
int xmjson_object_set(xmjson_node* obj, const char* key, xmjson_node* value);
int xmjson_object_set_string(xmjson_node* obj, const char* key, const char* value);
int xmjson_object_set_number(xmjson_node* obj, const char* key, double value);
int xmjson_object_remove(xmjson_node* obj, const char* key);

// 数组操作
int xmjson_array_append(xmjson_node* arr, xmjson_node* value);
int xmjson_array_insert(xmjson_node* arr, int index, xmjson_node* value);
int xmjson_array_remove(xmjson_node* arr, int index);

// 序列化
int xmjson_serialize(xmjson_builder* builder, const char* filename);
int xmjson_serialize_to_memory(xmjson_builder* builder, uint8_t** out, size_t* size);
```

### 只读访问器API

```c
// 加载
xmjson_doc* xmjson_load(const char* filename);
xmjson_doc* xmjson_load_from_memory(const void* data, size_t size);
void xmjson_unload(xmjson_doc* doc);

// 根节点
xmjson_value* xmjson_root(xmjson_doc* doc);

// 类型检查
int xmjson_is_null(xmjson_value* val);
int xmjson_is_bool(xmjson_value* val);
int xmjson_is_number(xmjson_value* val);
int xmjson_is_string(xmjson_value* val);
int xmjson_is_array(xmjson_value* val);
int xmjson_is_object(xmjson_value* val);

// 值获取
double xmjson_get_number(xmjson_value* val);
const char* xmjson_get_string(xmjson_value* val, size_t* out_len);
int xmjson_get_bool(xmjson_value* val);

// 对象访问
xmjson_value* xmjson_object_get(xmjson_value* obj, const char* key);
const char* xmjson_object_get_string(xmjson_value* obj, const char* key, size_t* len);
int xmjson_object_size(xmjson_value* obj);
const char* xmjson_object_key(xmjson_value* obj, int index, size_t* len);

// 数组访问
int xmjson_array_size(xmjson_value* arr);
xmjson_value* xmjson_array_get(xmjson_value* arr, int index);

// 迭代器
typedef struct {
    int index;
    const char* key;
    size_t key_len;
    xmjson_value* value;
} xmjson_iter;

xmjson_iter xmjson_object_begin(xmjson_value* obj);
xmjson_iter xmjson_object_next(xmjson_iter iter);
```

---

## 二进制格式规范

### 文件布局

```
+-------------------+
| Header (64 bytes) |
+-------------------+
| String Table      |
| (variable)        |
+-------------------+
| Object/Array Table|
| (size * 16)       |
+-------------------+
| Data Section      |
| (variable)        |
+-------------------+
```

### 头部结构

```c
typedef struct {
    uint32_t magic;              // 0x584D4A42 "XMJB"
    uint32_t version;            // 当前版本 0x00010000
    uint32_t flags;              // 压缩标志、字节序等
    uint32_t reserved;
    
    uint64_t created_at;         // Unix时间戳
    uint64_t source_size;        // 原始JSON大小
    
    uint32_t header_size;        // 头部大小
    uint32_t string_table_offset;
    uint32_t string_table_size;
    uint32_t node_table_offset;
    uint32_t node_table_size;
    uint32_t data_offset;
    uint32_t data_size;
    uint32_t total_size;
    
    uint32_t object_count;
    uint32_t array_count;
    uint32_t string_count;
    uint32_t number_count;
    
    uint32_t crc32;              // 完整性校验
    uint32_t reserved2[4];
} xmjson_header;
```

### 版本兼容性

- **主版本号**：不兼容的格式变化
- **次版本号**：向后兼容的扩展
- **修订号**：内部优化

---

## 测试策略

### 单元测试覆盖率目标：≥ 85%

```c
// tests/unit/test_utf8.c
void test_utf8_char_len(void) {
    assert(utf8_char_len('A') == 1);
    assert(utf8_char_len(0xC2) == 2);
    assert(utf8_char_len(0xE4) == 3);
    assert(utf8_char_len(0xF0) == 4);
}

// tests/unit/test_parser.c
void test_parse_object(void) {
    const char* json = "{\"key\":\"value\"}";
    xmjson_builder* builder = xmjson_builder_create();
    assert(xmjson_builder_parse(builder, json, strlen(json)) == 0);
    // ...
}
```

### 性能基准测试

```c
// tests/performance/bench_parse.c
void bench_parse_large_file(void) {
    // 解析 10MB JSON 文件
    // 目标：< 20ms
}

void bench_mmap_load(void) {
    // mmap 加载 10MB 二进制文件
    // 目标：< 1ms
}
```

### 模糊测试

```bash
# 使用 AFL 进行模糊测试
afl-fuzz -i testcases/ -o findings/ ./xmjson_fuzzer @@
```

---

## 构建系统

### CMake配置

```cmake
cmake_minimum_required(VERSION 3.14)
project(xmjson VERSION 1.0.0)

option(XMJ_BUILD_TESTS "Build tests" ON)
option(XMJ_BUILD_BENCH "Build benchmarks" OFF)
option(XMJ_BUILD_TOOLS "Build tools" ON)

# 编译选项
set(CMAKE_C_STANDARD 11)
set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wall -Wextra -O3")

# 库目标
add_library(xmjson STATIC
    src/core/utf8.c
    src/core/parser.c
    src/core/arena.c
    src/builder/builder.c
    src/builder/editor.c
    src/builder/serializer.c
    src/reader/loader.c
    src/reader/accessor.c
    src/binary/format.c
)

# 安装
install(TARGETS xmjson DESTINATION lib)
install(FILES include/xmjson.h DESTINATION include)
```

### 依赖项

| 依赖 | 版本 | 用途 | 可选 |
|------|------|------|------|
| pthread | - | 线程局部存储 | 是 |
| mmap | - | 文件映射 | 是 |
| zlib | 1.2+ | 压缩支持 | 是 |

---

## 性能指标

### 目标性能

| 操作 | 大小 | 目标时间 | 测试平台 |
|------|------|----------|----------|
| JSON解析 | 1 MB | < 2 ms | Intel i7-9700K |
| JSON解析 | 10 MB | < 20 ms | Intel i7-9700K |
| 二进制加载 | 10 MB | < 1 ms | NVMe SSD |
| 字段访问 | - | < 100 ns | 缓存命中 |

### 内存占用

| 场景 | JSON大小 | 内存占用 |
|------|----------|----------|
| 文本解析 | 1 MB | ~2.5 MB |
| 二进制加载 | 1 MB | 1 MB + 开销 |
| 编辑模式 | 1 MB | ~3 MB |

---

## 错误处理

### 错误码定义

```c
typedef enum {
    XMJ_OK = 0,
    XMJ_ERR_INVALID_UTF8 = -1,
    XMJ_ERR_UNEXPECTED_EOF = -2,
    XMJ_ERR_INVALID_JSON = -3,
    XMJ_ERR_OUT_OF_MEMORY = -4,
    XMJ_ERR_FILE_NOT_FOUND = -5,
    XMJ_ERR_INVALID_FORMAT = -6,
    XMJ_ERR_VERSION_MISMATCH = -7,
    XMJ_ERR_CORRUPTED = -8,
} xmjson_error;
```

### 错误恢复策略

- **解析错误**：记录错误位置，继续解析（尽力模式）
- **内存不足**：回滚操作，返回错误码
- **文件损坏**：CRC校验失败，拒绝加载

---

## 安全考虑

### 输入验证

- UTF-8序列验证
- JSON深度限制（默认1000层）
- 字符串长度限制（默认64KB）
- 数值范围检查

### 内存安全

- 所有指针操作前检查边界
- Arena分配器防止内存泄漏
- 哨兵值防止空指针解引用

### 防御性编程

```c
// 边界检查示例
static inline int safe_memcpy(char* dst, const char* src, size_t n, size_t dst_size) {
    if (n > dst_size) return XMJ_ERR_BUFFER_TOO_SMALL;
    memcpy(dst, src, n);
    return XMJ_OK;
}
```

---

## 文档规范

### 代码注释

```c
/**
 * 解析JSON字符串并构建内部表示
 * 
 * @param builder 构建器实例
 * @param json JSON字符串（会被修改，建议传递副本）
 * @param len 字符串长度
 * @return 0成功，负数为错误码
 * 
 * @note 此函数会修改输入的JSON字符串（零拷贝优化）
 * @warning 输入字符串必须在builder生命周期内保持有效
 */
int xmjson_builder_parse(xmjson_builder* builder, char* json, size_t len);
```

### API文档格式

使用Doxygen生成，包含：
- 函数描述
- 参数说明
- 返回值
- 使用示例
- 注意事项

---

## 发布流程

### 版本号规则
- **主版本**：不兼容的API变更
- **次版本**：向后兼容的功能增加
- **修订号**：向后兼容的问题修复

### 发布前检查
- [ ] 所有单元测试通过
- [ ] 性能基准在目标范围内
- [ ] 文档更新
- [ ] 示例代码验证
- [ ] 跨平台测试（Linux/macOS/Windows）

### 发布后
- 创建Git标签
- 更新CHANGELOG.md
- 生成API文档
- 发布到GitHub Releases

---

## 贡献指南

### 代码风格
- 遵循Google编码风格
- 使用2空格缩进
- 函数名使用小写+下划线
- 类型名使用小写+下划线+_t

### 提交规范
```
<type>(<scope>): <subject>

<body>

<footer>
```

类型：
- feat: 新功能
- fix: 错误修复
- perf: 性能优化
- docs: 文档更新
- test: 测试相关
- refactor: 重构

### 代码审查要求
- 至少一位核心开发者批准
- 所有CI检查通过
- 测试覆盖率不降低

---

## 项目里程碑

### 2026 Q1
- [ ] 完成核心解析器
- [ ] 实现基础API
- [ ] 单元测试覆盖率 > 80%

### 2026 Q2
- [ ] 二进制格式稳定
- [ ] 性能优化
- [ ] 发布v1.0.0

### 2026 Q3
- [ ] 社区反馈收集
- [ ] 性能持续优化
- [ ] 增加压缩支持

---

## 风险与应对

| 风险 | 影响 | 概率 | 应对措施 |
|------|------|------|----------|
| UTF-8处理错误 | 高 | 低 | 完善单元测试，使用标准测试向量 |
| 内存安全问题 | 高 | 中 | 使用AddressSanitizer，模糊测试 |
| 性能未达标 | 中 | 低 | 持续性能分析，优化热点代码 |
| 格式兼容性问题 | 中 | 中 | 版本管理，向后兼容设计 |

---

## 联系方式

- **项目维护者**: [Hehaoslj]
- **GitHub**: https://github.com/LMiceOrg/xmjson
- **邮件列表**: hehaoslj@sina.com
- **问题追踪**: https://github.com/LMiceOrg/xmjson/issues

---
