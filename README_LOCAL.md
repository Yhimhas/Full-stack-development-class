# 校园留言板：本地安装与运行

基于 Vue 3、Go/Gin 和 MySQL 的校园留言板，支持查看、发布留言，数据保存在本机 MySQL 中。本指南直接在电脑上安装依赖并启动服务，适用于 Windows PowerShell 和 macOS Terminal。

## 1. 准备运行环境

先安装下面三个软件，再安装项目依赖：

| 软件 | 版本要求 | 用途 | 下载入口 |
| --- | --- | --- | --- |
| Node.js（包含 npm） | 可使用项目原配置采用的 24.x；`package.json` 声明 `^20.19.0 || >=22.12.0` | 安装前端依赖、启动 Vite、构建页面 | [Node.js](https://nodejs.org/en/download) |
| Go | `go.mod` 声明最低 1.27.1；项目原配置采用 1.27.3 | 下载 Go 模块、编译和运行后端 | [Go](https://go.dev/dl/) |
| MySQL Community Server | 8.4.x，与项目原配置保持一致 | 保存留言数据 | [MySQL Community Server](https://dev.mysql.com/downloads/mysql/) |

上面的版本依据项目配置填写。`schema.sql` 使用 `utf8mb4_0900_ai_ci`，不要用 MySQL 5.7 或 MariaDB 直接替代。

### Windows

1. 安装对应电脑架构的 Node.js 和 Go 安装包，启用添加到 `PATH` 的选项。
2. 安装 MySQL **Server**，完成数据库初始化，设置并记住 `root` 管理员密码。仅安装 Workbench 无法提供数据库服务。
3. 在 MySQL 配置工具中启用 TCP/IP，端口使用 `3306`，将数据库注册为 Windows 服务并启动。
4. 将 MySQL 的 `bin` 目录加入 `PATH`，例如 `C:\Program Files\MySQL\MySQL Server 8.4\bin`；实际路径以安装位置为准。
5. 重新打开 PowerShell，让新的 `PATH` 生效。

### macOS

1. 下载并安装 Node.js、Go 对应的 macOS 安装包，Apple Silicon 选择 ARM64，Intel 选择 x64 / AMD64。
2. 下载并安装对应架构的 MySQL 8.4 安装包，完成初始化并设置 `root` 管理员密码。
3. 使用安装包提供的 MySQL 设置面板启动服务。
4. 如果 Terminal 找不到 `mysql`，可以在当前终端执行：

   ```bash
   export PATH="/usr/local/mysql/bin:$PATH"
   ```

   该路径适用于安装在 `/usr/local/mysql` 的官方安装包；其他安装位置使用实际的 `bin` 路径。需要长期生效时，将对应配置加入自己的 shell 配置文件，再打开新终端。

### 检查安装结果

Windows PowerShell：

```powershell
node --version
npm.cmd --version
go version
mysql --version
```

macOS Terminal：

```bash
node --version
npm --version
go version
mysql --version
```

四条命令都应输出版本号。`mysql --version` 只检查客户端是否安装，数据库服务是否可连接要通过后面的登录命令确认。

## 2. 进入项目目录

本文中的“项目根目录”指同时包含 `frontend/`、`backend/`、`scripts/` 的目录。本地这份代码的目录名为 `campus-message-board`。

Windows 示例：

```powershell
Set-Location 'C:\Users\lin\Desktop\全栈课程培训课\campus-message-board'
```

macOS 示例，按实际存放位置修改路径：

```bash
cd ~/projects/campus-message-board
```

后续需要同时打开两个终端，分别运行后端和前端。每个新终端都先进入项目根目录，再执行对应步骤。

## 3. 初始化本地数据库

确认 MySQL 服务已启动，然后在终端登录管理员账号：

```text
mysql --default-character-set=utf8mb4 -u root -p
```

看到密码提示后输入安装 MySQL 时设置的 `root` 密码。进入 `mysql>` 提示符后执行以下 SQL，先把密码占位文字替换为自己的项目密码；为方便直接使用本示例，密码不要包含单引号或反斜杠。

```sql
CREATE DATABASE IF NOT EXISTS campus_message_board
  CHARACTER SET utf8mb4
  COLLATE utf8mb4_0900_ai_ci;

CREATE USER IF NOT EXISTS 'campus_app'@'localhost'
  IDENTIFIED BY '替换为你自己的项目密码';

GRANT SELECT, INSERT, CREATE ON campus_message_board.*
  TO 'campus_app'@'localhost';

EXIT;
```

这里创建了数据库和应用账号，授予后端建表、查询、新增留言所需的权限。`root` 密码用于数据库管理，`campus_app` 的项目密码用于后端连接，两者可以不同。

`CREATE USER IF NOT EXISTS` 不会修改已有账号的密码。如果电脑中已经有这个账号，应使用原密码，并先确认现有数据库的用途；不要通过删除数据库或账号解决登录问题。

验证应用账号能够通过 TCP 连接，输入刚设置的项目密码：

```text
mysql --protocol=TCP -h 127.0.0.1 -P 3306 -u campus_app -p campus_message_board
```

进入 `mysql>` 后执行：

```sql
SELECT DATABASE(), CURRENT_USER();
EXIT;
```

数据库应为 `campus_message_board`。此时还没有 `messages` 表也正常，后端首次启动时会执行内嵌的 `backend/schema.sql`，自动创建缺失的表并保留已有数据，无需手动导入 SQL 文件。

## 4. 安装后端依赖并启动

Go 后端只读取**进程环境变量**，不会自动加载根目录的 `.env`。本地运行时，直接在启动后端的同一个终端里设置以下变量。

| 环境变量 | 本地配置 | 说明 |
| --- | --- | --- |
| `HOST` | `127.0.0.1` | 后端监听地址，默认也是此值 |
| `PORT` | `8080` | 后端端口，默认也是此值 |
| `DB_HOST` | `127.0.0.1` | 本机 MySQL 地址，默认也是此值 |
| `DB_PORT` | `3306` | MySQL 端口，默认也是此值 |
| `DB_NAME` | `campus_message_board` | 必填，前面创建的数据库 |
| `DB_USERNAME` | `campus_app` | 必填，前面创建的应用账号 |
| `DB_PASSWORD` | 你设置的项目密码 | 必填，必须与 MySQL 账号密码一致 |

### Windows PowerShell：终端一

从项目根目录执行：

```powershell
Set-Location backend
go mod download

$env:HOST = '127.0.0.1'
$env:PORT = '8080'
$env:DB_HOST = '127.0.0.1'
$env:DB_PORT = '3306'
$env:DB_NAME = 'campus_message_board'
$env:DB_USERNAME = 'campus_app'
$env:DB_PASSWORD = [System.Net.NetworkCredential]::new('', (Read-Host '输入 campus_app 的项目密码' -AsSecureString)).Password

go run .
```

### macOS Terminal：终端一

从项目根目录执行。下面的密码输入命令在 Bash 中运行，因此先进入 Bash；输入时屏幕不显示密码。

```bash
bash
cd backend
go mod download

export HOST=127.0.0.1
export PORT=8080
export DB_HOST=127.0.0.1
export DB_PORT=3306
export DB_NAME=campus_message_board
export DB_USERNAME=campus_app
read -r -s -p '输入 campus_app 的项目密码: ' DB_PASSWORD
printf '\n'
export DB_PASSWORD

go run .
```

首次启动需要下载依赖并编译。成功后日志中会出现 `Go/Gin 后端启动：http://127.0.0.1:8080`。保持这个终端运行。

浏览器访问 [后端 API](http://127.0.0.1:8080/api/messages)，应返回 JSON 数组；没有留言时为 `[]`。后端连接数据库、创建表成功后才会开放 HTTP 端口。

## 5. 安装前端依赖并启动

打开第二个终端，先进入项目根目录。前端无需配置数据库密码。

### Windows PowerShell：终端二

```powershell
Set-Location frontend
npm.cmd ci
npm.cmd run dev
```

### macOS Terminal：终端二

```bash
cd frontend
npm ci
npm run dev
```

`npm ci` 按 `package-lock.json` 安装依赖；它会重建现有 `node_modules`，不会更新锁文件。首次安装需要联网。

出现 Vite 启动信息后，打开 [校园留言板](http://127.0.0.1:5173)。前端的 `/api` 请求默认转发到 `http://127.0.0.1:8080`，不需要修改源码或额外配置跨域。

```text
浏览器 http://127.0.0.1:5173
  → 本机 Vite 开发服务
  → /api 转发到本机 Go/Gin 后端 :8080
  → Handler → Service → Repository
  → 本机 MySQL :3306 / campus_message_board
```

发布一条留言，再刷新页面，应仍然能看到该记录。也可以打开 [经过前端转发的 API](http://127.0.0.1:5173/api/messages)，与后端 API 的返回结果对照。

## 6. 验证数据库和数据保留

另开终端，用项目账号连接数据库：

```text
mysql --default-character-set=utf8mb4 --protocol=TCP -h 127.0.0.1 -P 3306 -u campus_app -p campus_message_board
```

在 `mysql>` 中执行：

```sql
SHOW TABLES;

SELECT id, nickname, content, created_at
FROM messages
ORDER BY id DESC
LIMIT 10;

EXIT;
```

对照页面、API 和 SQL 查询中的同一条留言。随后在后端终端按 `Ctrl+C`，再次执行 `go run .`，刷新页面并查询数据库，记录应保留。

留言存放在本机 MySQL 的数据目录中，停止前后端不会清空数据库。复制源码到另一台电脑不会自动带走已有留言。

## 7. 日常开发与检查

- 开始开发：先启动 MySQL，再按第 4、5 节分别启动后端和前端。
- 修改 Vue、CSS：Vite 自动热更新。
- 修改 Go 代码：在后端终端按 `Ctrl+C`，重新执行 `go run .`。
- 停止项目：在前后端终端分别按 `Ctrl+C`；如需停止 MySQL，使用其服务管理工具，留意本机其他项目是否也在使用它。
- 新开后端终端：重新设置第 4 节的环境变量。变量只对当前终端及其子进程生效，关闭终端后需要重新配置。

### 单元测试和构建

在 `backend/` 目录执行，Windows 和 macOS 通用：

```text
go mod verify
go test ./...
```

Go 单元测试不连接真实数据库。

在 `frontend/` 目录执行，Windows 用 `npm.cmd`，macOS 用 `npm`：

```text
npm test
npm run build
```

前端测试不访问真实 API 或数据库，构建结果输出到 `frontend/dist/`。`dist/` 是静态页面文件，单独打开其中的 `index.html` 不能替代完整服务；本指南日常运行使用 `npm run dev`。如需正式对外提供服务，还需要配置静态文件服务和 `/api` 反向代理。

### 真实 API 验收

前后端与 MySQL 都启动后，在**项目根目录**执行：

```text
node scripts/verify-api.mjs
```

脚本默认访问 `http://127.0.0.1:5173`，覆盖前端代理、读取、发布和非法输入检查。每次会在当前数据库中新增并保留 **3 条验收留言**；验收期间不要同时发布其他留言，以免影响数量校验。输出 `"result": "PASS"` 后，可用 SQL 核对 `createdIds` 对应的数据。

## 8. 常见问题

| 现象 | 处理方式 |
| --- | --- |
| 找不到 `node`、`npm`、`go` 或 `mysql` | 确认软件已安装且其可执行文件目录在 `PATH` 中，然后重新打开终端。MySQL Server 和客户端都需要安装。 |
| PowerShell 提示无法运行 `npm.ps1` | 使用本文的 `npm.cmd`，例如 `npm.cmd ci`，无需修改系统执行策略。 |
| Go 提示版本过低 | 安装满足 `backend/go.mod` 中 `go 1.27.1` 的工具链，不要只修改声明来绕过要求。 |
| 依赖下载失败 | 检查终端网络、系统代理以及 npm / Go 下载源是否可访问，恢复连接后重新执行 `npm ci` 或 `go mod download`。 |
| 后端提示必须设置 `DB_NAME` 等变量 | 在执行 `go run .` 的同一个终端设置环境变量；创建 `.env` 文件不会自动生效。 |
| MySQL 连接被拒绝 | 确认 MySQL 服务已启动，检查实际监听地址和端口，再用第 3 节的 TCP 登录命令验证。 |
| `Access denied for user` | 检查账号、密码和账号的 Host 是否匹配。先确认 TCP 登录成功；已有账号不会被 `CREATE USER IF NOT EXISTS` 重设密码。 |
| `Unknown database` | 先用管理员账号执行第 3 节的建库语句；后端只自动建表，不自动创建数据库。 |
| `Unknown collation: utf8mb4_0900_ai_ci` | 检查是否误用了旧版 MySQL 或 MariaDB，使用与本项目匹配的 MySQL 8.4。 |
| `8080` 端口被占用 | 检查占用程序。若改用其他后端端口，同步设置后端 `PORT` 和前端 `API_PROXY_TARGET`，然后重启两端。 |
| `5173` 端口被占用 | Vite 启用了 `strictPort`，不会自动换端口。关闭自己不再需要的前端实例，或用 `npm run dev -- --port 5174` 指定其他端口；Windows 使用 `npm.cmd`。 |
| 页面打开但加载留言失败 | 先检查 `http://127.0.0.1:8080/api/messages`，再检查 `http://127.0.0.1:5173/api/messages`，分别定位后端或前端代理问题。 |

例如后端改用 `8081` 时，后端终端在启动前设置 `PORT`，前端终端在启动前设置 `API_PROXY_TARGET`：

```powershell
# Windows：后端终端，保留已有数据库环境变量。
$env:PORT = '8081'
go run .

# Windows：前端终端，在 frontend 目录执行。
$env:API_PROXY_TARGET = 'http://127.0.0.1:8081'
npm.cmd run dev
```

```bash
# macOS：后端终端，保留已有数据库环境变量。
export PORT=8081
go run .

# macOS：前端终端，在 frontend 目录执行。
export API_PROXY_TARGET=http://127.0.0.1:8081
npm run dev
```

若前端改用 `5174`，浏览器也访问该端口。运行 API 验收脚本前，Windows 在验收终端设置 `$env:API_BASE = 'http://127.0.0.1:5174'`，macOS 设置 `export API_BASE=http://127.0.0.1:5174`。

## 9. 项目结构与接口

```text
campus-message-board/
├── README_LOCAL.md                本地安装与运行指南（本文）
├── frontend/
│   ├── package.json               前端依赖与命令
│   ├── package-lock.json          前端依赖锁文件
│   ├── vite.config.js             开发端口与 API 代理
│   └── src/                       Vue 页面、组件、样式与请求函数
├── backend/
│   ├── main.go                    连接数据库、初始化表、注册路由
│   ├── go.mod / go.sum            Go 模块与下载校验
│   ├── schema.sql                 messages 表结构
│   └── internal/
│       ├── config/                读取环境变量和生成数据库连接参数
│       ├── handler/               HTTP 请求与响应
│       ├── service/               输入校验与业务处理
│       ├── repository/            SQL 查询和写入
│       └── model/                 数据结构
└── scripts/verify-api.mjs          真实接口验收
```

| 方法 | 路径 | 成功响应 |
| --- | --- | --- |
| GET | `/api/messages` | `200`，按 id 倒序排列的留言数组，空库返回 `[]` |
| POST | `/api/messages` | `201`，刚创建的留言 |

发布接口使用 `Content-Type: application/json`，请求体示例：

```json
{"nickname":"小林","content":"周五有人一起去图书馆吗？"}
```

昵称去除首尾空白后为 1～30 个 Unicode code point；留言内容不能是纯空白，原文最多 200 个 code point。返回记录包含 `id`、`nickname`、`content`、`createdAt`，错误响应为 `{"message":"错误说明"}`。

代码分层和请求调用链可继续阅读 [教学说明](docs/TEACHING.md)。
