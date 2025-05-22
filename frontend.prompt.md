
参考下面的后端 API ，用 taildwindcss 规划一个美观可以运行的前端页面，用原生 js 写, 来管理这个sqlite 数据库， 假设后端口的接口是: `http://160.30.231.46:8080/execute` 

主要功能是：
- 打开页面后，要显示数据库中所有表的列表
- 有一个编辑区，可以输入 sql,  用鼠标选中里面的文本， 有一个按钮可以运行这部分 sql 语句，以表格展示结果使用前端分页 10 条一页 , 实现基本 CURD 功能选中一行后有下面有添加，删除按钮， 然后，结果是可以编辑， 编辑后点一个按钮可以保存到数据库。

先给出思路及 文件结构， 先不要实现代码，文件内的要实现的功能函数。 

````
### 6. API 接口规范

`sqlitem` 只有一个 HTTP API 接口：

*   **Method:** `POST`
*   **Path:** `/execute`
*   **Content-Type:** `application/json`

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


````

----
# SQLite 管理前端规划

## 1. 整体思路 (Overall Approach)

本前端应用旨在提供一个用户友好的界面，通过调用后端 API 来管理 SQLite 数据库。

**核心交互流程：**

1.  **页面加载 (Page Load):**
    * 应用启动后，立即向后端 API 发送请求 (`SELECT name FROM sqlite_master WHERE type='table';`) 获取所有数据库表的列表。
    * 将获取到的表名展示在页面的指定区域（例如：左侧边栏）。

2.  **SQL 编辑与执行 (SQL Editing & Execution):**
    * 用户可以在一个文本编辑区域（`<textarea>`）输入 SQL 语句。
    * 提供一个“运行选中 SQL ⚡️”按钮。当用户在编辑区选中一部分 SQL 文本并点击此按钮时，仅将选中的 SQL 语句发送到后端执行。
    * 提供一个“运行所有 SQL 🚀”按钮，执行编辑区内的所有 SQL。
    * 执行后，后端会返回 JSON 格式的结果。

3.  **结果展示 (Results Display):**
    * **查询结果 (SELECT):**
        * 如果执行的是查询语句，结果将以表格形式动态渲染到页面。
        * 表格数据需要支持前端分页，例如每页显示 10 条记录。
        * 表头应根据查询结果的字段动态生成。
    * **非查询操作 (INSERT, UPDATE, DELETE, CREATE TABLE, etc.):**
        * 对于这些操作，后端通常返回空数据或操作成功的状态。前端应显示相应的成功或失败消息。
        * 执行 DDL (如 `CREATE TABLE`) 或影响数据的 DML (如 `INSERT`, `DELETE`) 后，可能需要刷新表列表或当前表格数据。

4.  **数据操作 (CURD on Table Data):**
    * **选中行 (Select Row):** 用户可以点击结果表格中的某一行来选中它，以便进行后续操作。选中的行应有高亮显示。
    * **编辑数据 (Edit Data):**
        * 结果表格中的单元格应允许用户直接编辑。点击单元格后，可以将其转换为输入框，或使内容可编辑 (`contenteditable`)。
    * **添加行 (Add Row):**
        * 提供“添加行 ➕”按钮。点击后，可以在结果表格中（如果当前显示的是某个表的数据）添加一个新的空行（所有单元格可编辑）。
        * 用户填写完毕后，通过构造 `INSERT` 语句并执行来保存新行。
    * **删除行 (Delete Row):**
        * 提供“删除选中行 🗑️”按钮。用户选中一行后，点击此按钮将根据该行的一个唯一标识（如主键）构造 `DELETE` 语句并执行。
    * **保存更改 (Save Changes):**
        * 提供“保存更改 💾”按钮。当用户编辑了表格中的数据或添加了新行后，点击此按钮。
        * 前端需要识别哪些行被修改过或新添加，并为每一行构造相应的 `UPDATE` 或 `INSERT` 语句（需要基于主键或唯一标识）。
        * 逐条执行这些语句。

5.  **用户反馈 (User Feedback):**
    * 在页面的特定区域显示操作成功、失败或错误信息。
    * 在进行 API 调用时，可以显示加载指示。

6.  **SQL 执行历史 (SQL Execution History):**
    * 提供一个专门的区域（例如，只读的 `<textarea>`），用于记录通过“添加行”、“删除选中行”、“保存更改”等功能按钮自动生成的并已执行的 SQL 语句。
    * 每条历史记录会包含时间戳和操作类型，方便用户追溯。
    * 此区域与用户手动输入的 SQL 编辑区分开，保持编辑区的整洁。

**技术栈：**

* HTML
* TailwindCSS (通过 CDN)
* 原生 JavaScript (ES6+)

## 2. 文件结构 (File Structure)

/sqlite-web-manager/├── index.html               # 主HTML文件，包含页面布局骨架├── css/│   └── style.css            # 主要用于引入TailwindCSS指令，可包含少量自定义样式├── js/│   ├── app.js               # 应用主逻辑，事件绑定，协调各模块│   ├── api.js               # 封装与后端API的通信│   ├── ui.js                # 负责DOM操作，如渲染表格、列表、消息提示等│   └── utils.js             # 通用辅助函数，如SQL转义、数据处理等└── README.md                # 项目说明文档
## 3. 各文件主要功能及函数 (Key Functions per File)

#### `index.html`

* **主要内容：**
    * 引入 TailwindCSS。
    * 定义页面整体布局结构：
        * 左侧区域：用于显示数据库表列表 (`<div id="table-list-container">`)。
        * 右侧区域（主内容区，按从上到下顺序）：
            * SQL 编辑器 (`<section id="sql-editor-section">` 内含 `<textarea id="sql-editor"></textarea>`)。
            * 操作按钮区 (SQL 编辑器下方):
                * `<button id="run-selected-sql-btn">运行选中 SQL <span class="emoji">⚡️</span></button>`
                * `<button id="run-all-sql-btn">运行所有 SQL <span class="emoji">🚀</span></button>`
            * 消息提示区 (`<section id="message-area-container">` 内含 `<div id="message-area"></div>`)。
            * 结果展示区 (`<section id="results-section">`)，包含：
                * 表格 (`<div id="results-container">` 内含 `<table id="results-table">...</table>`)
                * 分页控件 (`<div id="pagination-controls"></div>`)
                * 数据行操作按钮区 (结果表格下方，居中):
                    * `<button id="add-row-btn">添加行 <span class="emoji">➕</span></button>`
                    * `<button id="delete-selected-row-btn">删除选中行 <span class="emoji">🗑️</span></button>`
                    * `<button id="save-changes-btn">保存更改 <span class="emoji">💾</span></button>`
            * SQL 执行历史区 (`<section id="sql-history-section">` 内含 `<textarea id="sql-history-log" readonly></textarea>`)。
    * 引入所有 JavaScript 文件 (`app.js`, `api.js`, `ui.js`, `utils.js`)，通常放在 `</body>` 前。

#### `css/style.css`

* **主要内容：**
    ```css
    @tailwind base;
    @tailwind components;
    @tailwind utilities;
    
    /* 可选的自定义样式 */
    .table-row-selected {
        background-color: theme('colors.blue.200'); /* 示例 Tailwind 颜色 */
    }
    .emoji { /* 用于调整 Emoji 的显示，如果需要 */
        font-size: 1.1em; 
    }
    #sql-history-log { /* SQL 历史记录区特定样式 */
        font-family: monospace;
    }
    ```

#### `js/api.js`

* `const API_BASE_URL = "http://160.30.231.46:8080/execute";`
* `async function executeSQL(sqlQueryString)`
    * **职责:** 向后端 `/execute` API 发送 POST 请求。
    * **参数:** `sqlQueryString` (string) - 要执行的 SQL 语句。
    * **返回:** `Promise` - 解析后的 JSON 响应对象。

#### `js/ui.js`

* **元素引用:** (除了之前的，新增)
    * `sqlHistoryLog: document.getElementById('sql-history-log')`
* `function displayTableList(tablesArray)` (同前)
* `function displayResultsTable(dataArray, currentPage, rowsPerPage, headers)` (之前是 `displayResultsTable(dataArray, currentPage, rowsPerPage)`)
    * **职责:** 将查询结果数据显示在 `#results-table` 中，并处理分页。
    * **参数:**
        * `dataArray` (Array of Objects) - 查询结果。
        * `currentPage` (number) - 当前页码。
        * `rowsPerPage` (number) - 每页显示的行数。
        * `headers` (Array of Strings) - 表头列名。
    * **实现:** (同前，但确保表头参数被使用)
* `function renderPaginationControls(totalItems, currentPage, rowsPerPage)` (同前)
* `function getSQLEditorSelection()` (同前)
* `function highlightSelectedRow(tableRowElement)` (同前)
* `function getSelectedRowData(tableRowElement, headers)` (同前)
* `function getEditedDataFromRow(tableRowElement, headers)` (同前)
* `function showMessage(messageText, type = 'info' | 'success' | 'error')` (同前)
* `function clearMessages()` (同前)
* `function createEditableCell(value, isPrimaryKeyCell = false)` (之前是 `createEditableCell(value)`)
    * **职责:** 创建一个可编辑的表格单元格 `<td>`。
    * **参数:** `value` (any), `isPrimaryKeyCell` (boolean) - 标记是否为主键单元格以应用不同样式或行为。
* `function createNewEditableRow(columns)` (同前)
* `function updateCurdButtonStates(isTableContextActive, isRowSelectedAndModifiable = false)`
    * **职责:** 根据当前上下文（是否有活动表、是否有选中行、是否有修改）更新 CURD 操作按钮（添加、删除、保存）的启用/禁用状态。
* `function logSqlToHistory(sqlStatement, operationType = "操作")`
    * **职责:** 将带有时间戳和操作类型的 SQL 语句添加到 `#sql-history-log` 文本区域的顶部。
    * **参数:** `sqlStatement` (string), `operationType` (string) - 例如 "删除行", "保存更改"。
    * **实现:** 获取当前时间，格式化日志条目，然后将其预置到 `sqlHistoryLog.value` 中，并滚动到顶部。

#### `js/utils.js`

* `function sanitizeSQLValue(value)` (同前)
* `function getPrimaryKeyCandidate(headers, dataSample)` (之前是 `getPrimaryKeyColumn`)
    * **职责:** 尝试根据表头和少量数据样本推断主键列名（例如，名为 'id' 或 'rowid' 的列）。

#### `js/app.js` (应用主控制器)

* **全局/模块级变量:** (同前)
    * `currentPrimaryKey: null` (新增，用于存储当前表推断出的主键名)
* `async function init()` (同前)
* `async function loadTableList()` (同前)
* `async function handleRunSQL(isSelectionOnly)`
    * **实现:** (基本同前)
        * 在成功执行 `SELECT` 后，会尝试从 SQL 语句中解析表名并存入 `app.currentEditingTable`。
        * 执行 DDL 或 DML 后，会相应地刷新表列表或当前表数据。
* `function changePage(newPage)`
    * **职责:** 处理分页逻辑，重新渲染表格的特定页面。
* `async function handleAddRow()` (同前)
* `async function handleDeleteSelectedRow()`
    * **实现:** (基本同前)
        * 构造 `DELETE` SQL 后，调用 `ui.logSqlToHistory(deleteSql, "删除行")` 将其记录到历史区，而不是主 SQL 编辑器。
* `async function handleSaveChanges()`
    * **实现:** (基本同前)
        * 为每个修改或新增的行构造 `INSERT` 或 `UPDATE` SQL。
        * 将所有执行的 SQL 语句收集起来，在所有操作完成后，通过 `ui.logSqlToHistory(allExecutedSql, "保存更改")` 一次性记录到历史区。
* **事件监听器设置 (Event Listener Setup):** (同前)

这个更新后的规划文档现在更准确地反映了应用的当前功能和结构。

