我需要用 golang 写一个 sqlite 数据管理程序，这个程序通过提供执行 sqliite3 命令来管理一个 sqlite 数据库，只需要一个http API 就是执行发送过来的 sql 语句，收到语句后执行 `echo "sql" | sqlite3 -json /tmp/data.db`  这样的命令, 然后 返回 sql 执行结果就行。
程序名： sqlitem 

先帮我规划一下项目文件结构，对应实现功能，简介。 用环境变量做配置，要有统一的 json 返回格式, 主要支持展示所有表格的查询，以及具体表格的CURD，给出 API 请求例子。


我先看看，先不要实现代码。

-------------
好的，我们来规划一下这个名为 `sqlitem` 的 Golang SQLite 数据管理程序。

---

## 项目规划：`sqlitem` - SQLite HTTP 命令执行器

### 1. 简介

`sqlitem` 是一个轻量级的 Golang HTTP 服务，它提供一个简单的 API 接口来执行用户提交的 SQLite SQL 语句。它不直接使用 Go 的 `database/sql` 驱动，而是通过调用外部 `sqlite3` 命令（使用 `echo "sql" | sqlite3 -json <database_path>`）来与 SQLite 数据库交互，并将 `sqlite3 -json` 的输出作为统一 JSON 格式的一部分返回给客户端。

### 2. 核心功能

*   **SQL 命令执行：** 接收任意 SQL 语句并使用 `sqlite3` 命令行工具执行。
*   **统一 JSON 响应：** 成功和失败都以统一的 JSON 格式返回结果。
*   **环境配置：** 通过环境变量配置监听地址和数据库文件路径。
*   **支持所有 SQLite 命令：** 由于直接调用 `sqlite3` 命令，理论上支持 `sqlite3` 支持的所有 SQL 语法和命令。
*   **特定场景支持：** 方便查询所有表格，以及对具体表格进行 CURD 操作（通过发送相应的 SQL 语句）。

### 3. 项目文件结构规划

```
sqlitem/
├── main.go                       # 程序入口，HTTP 服务启动，路由设置
├── config/                       # 配置模块，负责加载环境变量
│   └── config.go                 # 定义配置结构体和加载函数
├── handler/                      # HTTP 请求处理模块
│   └── handler.go                # 定义唯一的 SQL 执行 HTTP Handler
├── service/                      # 业务逻辑模块，负责实际的 SQL 命令执行
│   └── executor.go               # 封装 `os/exec` 调用 `sqlite3` 命令的逻辑
├── common/                       # 通用工具和类型定义
│   ├── response.go               # 定义统一的 JSON 响应结构体和辅助函数
│   └── request.go                # 定义请求体结构体
├── .env.example                  # 环境变量配置示例文件
├── go.mod                        # Go 模块文件
└── go.sum                        # Go 模块校验文件
```

### 4. 文件功能对应

*   **`main.go`**
    *   从 `config` 包加载配置。
    *   初始化 HTTP 路由器 (例如 `net/http` 默认 mux)。
    *   注册 `/execute` 路由到 `handler.ExecuteSQLHandler`。
    *   启动 HTTP 服务器。

*   **`config/config.go`**
    *   定义 `Config` struct，包含 `ListenAddr` (string) 和 `DatabasePath` (string) 字段。
    *   实现 `LoadConfig()` 函数，从环境变量 (`SQLITEM_LISTEN_ADDR`, `SQLITEM_DATABASE_PATH`) 读取配置并返回 `Config` 实例。提供默认值或错误处理。

*   **`handler/handler.go`**
    *   定义 `ExecuteSQLHandler(w http.ResponseWriter, r *http.Request)` 函数。
    *   解析传入的 JSON 请求体（期望 `{"sql": "YOUR_SQL_STATEMENT"}`）。
    *   调用 `service.ExecuteSQL` 函数执行 SQL 语句。
    *   根据 `service.ExecuteSQL` 的结果，使用 `common.WriteJSONSuccess` 或 `common.WriteJSONError` 返回统一的 JSON 响应。

*   **`service/executor.go`**
    *   定义 `ExecuteSQL(dbPath string, sqlStatement string) ([]byte, error)` 函数。
    *   使用 `os/exec` 包构建并执行命令：`echo "sqlStatement" | sqlite3 -json dbPath`。
    *   捕获 `stdout` 和 `stderr`。
    *   如果命令执行成功，返回 `stdout` 的内容。
    *   如果命令执行失败（非零退出码或`stderr`有内容），返回相应的错误信息。

*   **`common/response.go`**
    *   定义 `APIResponse` struct，包含 `Status` (string), `Message` (string, optional), `Data` (interface{}, optional) 字段。
    *   定义 `WriteJSONSuccess(w http.ResponseWriter, data interface{})` 函数，封装成功响应。
    *   定义 `WriteJSONError(w http.ResponseWriter, message string, details string, statusCode int)` 函数，封装错误响应。

*   **`common/request.go`**
    *   定义 `SQLRequest` struct，包含 `SQL` (string) 字段，用于解析请求体。

### 5. 环境变量配置示例 (`.env.example`)

```
# HTTP 服务监听地址
SQLITEM_LISTEN_ADDR=:8080

# SQLite 数据库文件路径
# 确保该路径可写且程序有权限访问
SQLITEM_DATABASE_PATH=/var/lib/sqlitem/data.db
```

### 6. API 接口规范

`sqlitem` 只有一个 HTTP API 接口：

*   **Method:** `POST`
*   **Path:** `/execute`
*   **Content-Type:** `application/json`
*   本接口支持跨域请求（CORS），允许所有来源。

#### 6.1. 请求示例

发送一个包含 SQL 语句的 JSON 对象。

```json
{
    "sql": "SELECT * FROM users WHERE age > 30;"
}
```

#### 6.2. 成功响应示例

`sqlite3 -json` 的输出将直接作为 `data` 字段的值。

**查询示例 (e.g., `SELECT * FROM users;`)**

```json
{
    "status": "success",
    "data": [
        {"id": 1, "name": "Alice", "email": "alice@example.com"},
        {"id": 2, "name": "Bob", "email": "bob@example.com"},
        {"id": 3, "name": "Charlie", "email": "charlie@example.com"}
    ]
}
```

**非查询操作示例 (e.g., `INSERT`, `UPDATE`, `DELETE`, `CREATE TABLE`)**

对于这些操作，`sqlite3 -json` 命令通常不会返回数据，`data` 字段将为空数组或 `null`。

```json
{
    "status": "success",
    "data": []
}
```
或者
```json
{
    "status": "success",
    "data": null
}
```
（具体取决于 `sqlite3 -json` 在无数据输出时是返回空数组还是空字符串/null，通常是空数组 `[]`）。

#### 6.3. 错误响应示例

当 SQL 语句执行失败（例如语法错误，或数据库文件问题）时。

```json
{
    "status": "error",
    "message": "SQL execution failed",
    "details": "Error: near \"FROMM\": syntax error" // 来源于 sqlite3 的 stderr 输出
}
```

### 7. API 请求例子（如何实现具体功能）

所有的功能都通过向 `/execute` 接口发送不同的 SQL 语句来实现。

#### 7.1. 查询所有表格

**SQL:** `SELECT name FROM sqlite_master WHERE type='table';`

**请求:**

```bash
curl -X POST -H "Content-Type: application/json" -d '{"sql": "SELECT name FROM sqlite_master WHERE type=\'table\';"}' http://localhost:8080/execute
```

**响应 (示例):**

```json
{
    "status": "success",
    "data": [
        {"name": "users"},
        {"name": "products"},
        {"name": "orders"}
    ]
}
```

#### 7.2. 具体表格的 CURD 操作

假设我们有一个 `users` 表，包含 `id` (INTEGER PRIMARY KEY), `name` (TEXT), `email` (TEXT) 字段。

**a. 创建表格 (Create Table)**

**SQL:** `CREATE TABLE IF NOT EXISTS users (id INTEGER PRIMARY KEY AUTOINCREMENT, name TEXT NOT NULL, email TEXT UNIQUE);`

**请求:**

```bash
curl -X POST -H "Content-Type: application/json" -d '{"sql": "CREATE TABLE IF NOT EXISTS users (id INTEGER PRIMARY KEY AUTOINCREMENT, name TEXT NOT NULL, email TEXT UNIQUE);"}' http://localhost:8080/execute
```

**b. 插入数据 (Insert)**

**SQL:** `INSERT INTO users (name, email) VALUES ('Alice', 'alice@example.com');`

**请求:**

```bash
curl -X POST -H "Content-Type: application/json" -d '{"sql": "INSERT INTO users (name, email) VALUES (\'Alice\', \'alice@example.com\');"}' http://localhost:8080/execute
```

**c. 查询数据 (Select)**

**SQL:** `SELECT id, name, email FROM users WHERE name = 'Alice';`

**请求:**

```bash
curl -X POST -H "Content-Type: application/json" -d '{"sql": "SELECT id, name, email FROM users WHERE name = \'Alice\';"}' http://localhost:8080/execute
```

**响应 (示例):**

```json
{
    "status": "success",
    "data": [
        {"id": 1, "name": "Alice", "email": "alice@example.com"}
    ]
}
```

**d. 更新数据 (Update)**

**SQL:** `UPDATE users SET email = 'alice.new@example.com' WHERE id = 1;`

**请求:**

```bash
curl -X POST -H "Content-Type: application/json" -d '{"sql": "UPDATE users SET email = \'alice.new@example.com\' WHERE id = 1;"}' http://localhost:8080/execute
```

**e. 删除数据 (Delete)**

**SQL:** `DELETE FROM users WHERE id = 1;`

**请求:**

```bash
curl -X POST -H "Content-Type: application/json" -d '{"sql": "DELETE FROM users WHERE id = 1;"}' http://localhost:8080/execute
```

---

这个规划涵盖了 `sqlitem` 的核心需求，提供了清晰的项目结构和 API 定义，并给出了详细的请求和响应示例。下一步就可以根据这个规划开始实现代码了。