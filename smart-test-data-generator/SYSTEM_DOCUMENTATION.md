# SmartTestData Generator v2.0 - 系统功能详解

> 智能测试数据生成器 - 双 MCP Server 架构

---

## 🎯 系统概述

**SmartTestData Generator** 是一个智能测试数据生成系统，采用**双 MCP Server 架构**，通过 IDEA 灵码插件进行交互，支持代码分析和 Oracle 数据库查询。

### 核心特性

- ✅ **双 Server 架构**：代码分析与数据库操作分离
- ✅ **智能路径匹配**：支持中心名称、中文、接口名等多种输入
- ✅ **人工确认机制**：数据库查询前必须用户确认
- ✅ **中心冲突检测**：用户意图与接口名不一致时提示确认
- ✅ **Oracle 原生支持**：使用官方驱动

---

## 🏗️ 系统架构

### 架构总览

```
┌─────────────────────────────────────────────────────────┐
│                    IDEA 灵码插件                          │
│              (用户交互界面 - 聊天方式)                     │
└─────────────────────────────────────────────────────────┘
                          ↕ stdio
        ┌─────────────────┴─────────────────┐
        ↓                                   ↓
┌──────────────────┐            ┌────────────────────┐
│ mcp-code-analyzer │            │ mcp-database-client │
│   (代码分析 MCP)    │            │   (数据库 MCP)        │
│   4 个工具         │            │    4 个工具          │
└──────────────────┘            └────────────────────┘
        ↓                                   ↓
   源码文件解析                          Oracle 数据库
   (不需要数据库)                         (需要人工确认后)
```

### 执行流程说明

**两个 MCP Server 的调用时机**：

| 阶段 | MCP Server | 触发条件 |
|------|-----------|---------|
| **代码分析阶段** | mcp-code-analyzer | 用户提问后立即调用 |
| **人工确认阶段** | - | 灵码展示分析结果，等待用户确认 |
| **数据查询阶段** | mcp-database-client | ⭐ 用户确认后调用 |

**关键点**：
- ✅ mcp-code-analyzer：随时可用，不需要数据库连接
- ⏸️ mcp-database-client：**必须人工确认后才调用**
- ✅ 安全可控：数据库查询不会自动执行

---

### 完整工作流程（7 步）

```
Step 1: 用户提问
   "分析 order.createOrder 接口需要什么条件？"
   ↓
Step 2: 调用 mcp-code-analyzer.scan_source_directory
   输入："订单中心" 或 "order" 或 "去订单中心查"
   输出：源码文件列表
   ↓
Step 3: 调用 mcp-code-analyzer.analyze_interface
   输入：file_path, function_name
   输出：校验规则 + 调用链 + 建议查询条件
   ↓
Step 4: 灵码 LLM 理解（不需要 MCP）
   - 理解复杂业务逻辑
   - 补充隐含条件
   - 输出：完整条件列表
   ↓
Step 5: ⭐ 人工确认（关键步骤）
   灵码展示分析结果，等待用户确认
   用户输入："yes" / "no" / 修改条件
   ↓
Step 6: 调用 mcp-database-client.query_test_data
   输入：table, conditions[], limit
   输出：测试数据
   ↓
Step 7: 调用 mcp-database-client.build_test_data
   输入：interface_name, validations, db_data
   输出：最终测试数据（格式化）
```

---

## 📦 模块详解

### 模块 1️⃣：mcp-code-analyzer（代码分析 MCP Server）

**功能**：分析 Java/TypeScript 代码，提取接口校验规则和调用链

**位置**：`mcp/code-analyzer/`

**核心文件**：
- `src/server.ts` - MCP Server 主程序
- `src/code-parser.ts` - AST 代码解析器

**提供工具**（4 个）：

| 工具名 | 功能 | 参数 | 返回值 |
|--------|------|------|--------|
| `analyze_interface` | 分析接口代码，提取所有校验规则和条件 | `file_path`, `function_name` | 校验规则列表 + 调用链 + 建议查询条件 |
| `extract_validations` | 从单个文件中提取数据校验规则 | `file_path` | 校验规则数组 |
| `get_function_call_chain` | 获取函数的完整调用链（含嵌套层级） | `file_path`, `function_name` | 调用链树形结构 |
| `scan_source_directory` | 扫描源码目录，自动发现所有接口/服务类和方法 | `source_dir`, `file_type` | 文件列表 + 方法列表 |

**核心特性**：
- ✅ **智能路径解析**：支持中心名称、中文名称、接口名等多种输入方式
- ✅ **中心冲突检测**：用户指定中心与接口名解析不一致时提示确认
- ✅ **AST 解析**：使用 TypeScript AST 解析 Java/TypeScript 代码
- ✅ **调用链分析**：递归提取函数调用关系

---

### 模块 2️⃣：mcp-database-client（数据库 MCP Server）

**功能**：连接 Oracle 数据库，查询测试数据，获取表结构

**位置**：`mcp/database-client/`

**核心文件**：
- `src/server.ts` - MCP Server 主程序
- `src/database-client.ts` - Oracle 数据库客户端

**提供工具**（4 个）：

| 工具名 | 功能 | 参数 | 返回值 |
|--------|------|------|--------|
| `query_test_data` | 从数据库查询测试数据 | `table`, `conditions[]`, `limit` | 查询结果数组 |
| `get_table_schema` | 获取数据库表结构 | `table` | 字段列表 + 类型 + 约束 |
| `get_table_names` | 获取数据库所有表名 | 无参数 | 表名数组 |
| `build_test_data` | 组合最终测试数据（含人工确认提示） | `interface_name`, `validations`, `db_data` | 格式化测试数据 |

**核心特性**：
- ✅ **Oracle 原生驱动**：使用 `oracledb` 官方驱动
- ✅ **连接池**：支持数据库连接池配置
- ✅ **SQL 注入防护**：参数化查询
- ✅ **错误处理**：完善的异常捕获和错误提示

---

### 模块 3️⃣：配置文件模块

**位置**：`config/`

**配置文件**：

| 文件 | 功能 |
|------|------|
| `ide-mcp-config.json` | IDEA 灵码 MCP Server 配置（两个 Server 的路径和环境变量） |
| `project-paths.json` | 项目路径配置（7 个中心的源码目录映射） |
| `database.json` | 数据库连接配置（备选方案） |

**项目路径配置示例**：
```json
{
  "centers": {
    "order": "/Users/.../cmpak/src/order",
    "sec": "/Users/.../cmpak/src/sec",
    "cust": "/Users/.../cmpak/src/cust",
    "upc": "/Users/.../cmpak/src/upc",
    "channel": "/Users/.../cmpak/src/channel",
    "irsc": "/Users/.../cmpak/src/irsc",
    "acctmanm": "/Users/.../cmpak/src/acctmanm"
  }
}
```

---

## 🎨 智能路径匹配规则

### 支持的输入方式

| 输入类型 | 示例 | 匹配结果 |
|---------|------|---------|
| **英文中心名** | `order`, `sec`, `cust` | 直接映射到对应目录 |
| **中文中心名** | `订单中心`, `安全中心`, `客管中心` | 映射到对应目录 |
| **混合名称** | `upc 中心`, `channel 中心`, `irsc 中心` | 映射到对应目录 |
| **接口名** | `order.createOrder`, `sec.authenticate` | 提取前缀匹配中心 |
| **自然语言** | `去订单中心查 sec.design.query` | 提取"订单中心" |
| **完整路径** | `/Users/.../src/order` | 直接使用 |

### 中心名称映射表

| 中心 | 英文缩写 | 中文名称 | 别名 |
|------|---------|---------|------|
| 订单中心 | `order` | 订单中心 | - |
| 安全中心 | `sec` | 安全中心 | - |
| 客管中心 | `cust` | 客管中心，客户中心 | - |
| 产品中心 | `upc` | 产品中心 | upc 中心 |
| 渠道中心 | `channel` | 渠道中心 | channel 中心 |
| 资源中心 | `irsc` | 资源中心 | irsc 中心 |
| 账户中心 | `acctmanm` | 账户中心 | acct 中心 |

### 中心冲突检测

**触发条件**：用户指定中心 ≠ 接口名解析中心

**示例**：
```
输入：去订单中心查 sec.design.query
用户指定：订单中心 (order)
接口解析：sec
结果：⚠️ 冲突，提示用户确认

返回格式：
{
  "conflict": true,
  "userSpecifiedCenter": "order",
  "userSpecifiedCenterPath": "/src/order",
  "interfaceMatchCenter": "sec",
  "interfaceMatchCenterPath": "/src/sec",
  "message": "⚠️ 中心冲突：您说要去 order 中心查，但接口名 sec.design.query 看起来属于 sec 中心。请确认要去哪个中心搜索？"
}
```

---

## 🛠️ 技术栈

| 组件 | 技术 |
|------|------|
| **运行时** | Node.js 24.13.1 |
| **语言** | TypeScript 6.x |
| **MCP SDK** | `@modelcontextprotocol/sdk` |
| **数据库驱动** | `oracledb` 6.x (Oracle Instant Client) |
| **代码解析** | 自定义 AST 解析器 |
| **通信协议** | stdio (本地进程管道) |
| **配置管理** | `dotenv` |

---

## 📁 项目结构

```
smart-test-data-generator/
├── mcp/
│   ├── code-analyzer/          # 代码分析 MCP Server
│   │   ├── src/
│   │   │   ├── server.ts       # MCP Server (4 个工具)
│   │   │   └── code-parser.ts  # AST 解析器
│   │   ├── dist/               # 编译输出
│   │   └── package.json
│   │
│   └── database-client/        # 数据库 MCP Server
│       ├── src/
│       │   ├── server.ts       # MCP Server (4 个工具)
│       │   └── database-client.ts  # Oracle 客户端
│       ├── dist/               # 编译输出
│       └── package.json
│
├── config/
│   ├── ide-mcp-config.json     # IDEA 灵码配置
│   ├── project-paths.json      # 项目路径映射
│   └── database.json           # 数据库配置
│
├── .env                        # 环境变量（数据库连接）
├── package.json
├── README.md
└── SYSTEM_DOCUMENTATION.md     # 本文档
```

---

## 📊 核心功能实现状态

| 功能模块 | 状态 | 说明 |
|---------|------|------|
| **mcp-code-analyzer** | ✅ 完成 | 编译成功，4 个工具可用 |
| **mcp-database-client** | ✅ 完成 | 编译成功，4 个工具可用 |
| **智能路径匹配** | ✅ 完成 | 支持中心名、中文、接口名 |
| **中心冲突检测** | ✅ 完成 | 用户指定与接口解析不一致时提示 |
| **人工确认流程** | ✅ 完成 | 数据库查询前必须确认 |
| **IDEA 灵码配置** | ⏳ 待配置 | 需用户在 IDE 中配置 |
| **数据库连接** | ⏳ 待配置 | 需配置真实 Oracle 连接信息 |

---

## 🚀 使用示例

### 示例 1：分析接口

**用户提问**：
```
请分析订单中心的 createOrder 接口
```

**系统执行**：
1. `scan_source_directory("订单中心")` → 定位到 order 中心目录
2. `analyze_interface("OrderService.java", "createOrder")` → 提取校验规则
3. 灵码理解业务逻辑，补充隐含条件
4. **人工确认**：展示分析结果，等待用户确认
5. `query_test_data("orders", ["status='active'"], 1)` → 查询测试数据
6. `build_test_data(...)` → 输出最终测试数据

### 示例 2：探索数据库

**用户提问**：
```
有哪些表可以查询测试数据？
```

**系统执行**：
```
get_table_names() → 返回表名列表
```

### 示例 3：中心冲突处理

**用户提问**：
```
去订单中心查 sec.authenticate
```

**系统响应**：
```
⚠️ 中心冲突：您说要去 order 中心查，但接口名 sec.authenticate 看起来属于 sec 中心。

请确认要去哪个中心搜索？
- order 中心：/src/order
- sec 中心：/src/sec
```

---

## 🎯 核心优势

| 特性 | 说明 |
|------|------|
| **双 Server 架构** | 代码分析与数据库操作分离，便于维护和扩展 |
| **人工确认机制** | 数据库查询前必须用户确认，安全可控 |
| **智能路径匹配** | 支持多种输入方式，自动识别中心 |
| **中心冲突检测** | 用户意图与接口名不一致时提示确认 |
| **Oracle 原生支持** | 使用官方驱动，完整支持 Oracle 特性 |
| **stdio 通信** | 本地进程管道，无需网络配置 |

---

## 📝 配置步骤

### 1. 编译 MCP Server

```bash
cd smart-test-data-generator
npm run build
```

### 2. 配置数据库连接

```bash
cd mcp/database-client
cp .env.example .env
```

编辑 `.env` 文件：
```env
DB_USER=chenyg
DB_PASSWORD=your_password
DB_CONNECT_STRING=(DESCRIPTION=(ADDRESS=(PROTOCOL=TCP)(HOST=172.25.51.219)(PORT=6117))(CONNECT_DATA=(SID=ctest1)))
DB_POOL_SIZE=10
```

### 3. 配置 IDEA 灵码

在 IDEA 灵码插件设置中，添加 MCP Server 配置：

**mcp-code-analyzer**：
```json
{
  "command": "node",
  "args": ["/path/to/mcp/code-analyzer/dist/index.js"]
}
```

**mcp-database-client**：
```json
{
  "command": "node",
  "args": ["/path/to/mcp/database-client/dist/index.js"],
  "env": {
    "DB_USER": "chenyg",
    "DB_PASSWORD": "your_password",
    "DB_CONNECT_STRING": "..."
  }
}
```

### 4. 重启 IDEA 灵码插件

---

## ⚠️ 注意事项

1. **Oracle 客户端**：确保 Oracle Instant Client 已正确安装
2. **环境变量**：敏感信息通过 `.env` 文件传递
3. **人工确认**：在查询数据库前必须用户确认
4. **文件路径**：使用绝对路径
5. **中心匹配**：长关键字优先匹配（如"订单中心"优先于"order"）

---

## 📞 技术支持

- **项目地址**：`/Users/chenyigeng/Library/Application Support/winclaw/.openclaw/workspace/smart-test-data-generator/`
- **配置文件**：`config/ide-mcp-config.json`
- **项目路径**：`config/project-paths.json`

---

**文档版本**：v2.0  
**最后更新**：2026-04-19  
**状态**：功能完整，待配置使用
