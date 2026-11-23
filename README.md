# 数树 · 教师工作台（edu-station）

## 下载完整源码

[下载 edu-station-github-20261123.zip](https://github.com/2578562306/edu-station/releases/download/handoff-2026-11-23/edu-station-github-20261123.zip)。解压后进入 `edu-station/`，从包内的 `README.md` 和 `AGENTS.md` 开始。下文提到的源码和文档路径都在解压目录内。

数树把作业照片和 PDF 变成可复核的学情记录。老师录入作业后，可以查看系统识别出的题目、作答和对错，回到原卷核对，再看哪些学生在同一课时反复出错、哪些内容适合一起讲。明确判错的题目可以进入错题本，无法判断的题目保留给老师复核。

**第一次接手，从本页的「本机快速启动」跑通空白工作台。** 需要改代码时再读 [AGENTS.md](AGENTS.md)。这个版本提供源码、界面资源、测试和启动说明，使用者自行建立账号和数据库。真实作业识别需要另外配置服务。

## 这个版本能做什么

- 网页工作台、学生和机构管理、作业录入、错题复核、学情统计与错题练习卷导出。浏览和导出已有结果无需再次调用模型。微信小程序与网页共用后端、账号和数据。
- 图片和 PDF 可录入单份作业。单张图片最大 10 MB，单份 PDF 最大 50 MB，一份作业最多 20 页。PDF 批量录入支持按学号匹配，每批最多 60 份。详细规则见 PDF 批量录入（`docs/pdf-batch-upload.md`）。
- 工作台按学生、课时和**实际作业日期**观察反复出错的情况。当前人教版七年级上册数学有教材课时映射，其他范围按已有知识点名称呈现。旧作业可补实际日期，教师更正和原卷来源应得到保留。见 工作台口径（`docs/product-refactor-20261116.md`） 和 学情统计口径（`docs/knowledge-analytics.md`）。
- 默认真实识别使用独立的 Sol 后台，目前走用户自己的 ChatGPT 订阅授权，并要求该授权提供 `gpt-6.1-sol` 权限。没有共享订阅或预置授权。豆包、火山方舟和通用视觉模型 API Key 接入尚未实现。可用 Mock 模式在没有模型账号时验证录入流程。

## 本机快速启动

准备 **JDK 21、Maven 3.9、Node.js 22.13 或更新的 22.x**。下面是 macOS / Linux shell 的 H2 演示命令，在新检出目录执行。Windows 可使用同样的环境变量和打包命令，IDE 配置见 Windows 本地开发（`tools/dev/README.md`）。

在仓库根目录安装和打包：

```sh
npm ci --prefix frontend
(cd backend && mvn -DskipTests package)
```

在第一个终端启动后端。请先将 `ADMIN_PASSWORD` 的示例文字换成本机演示专用密码。数据库和图片放在当前项目的 `.local-dev/demo/` 下，首次启动创建空库和负责人账号。

```sh
DEMO_DATA_DIR="$PWD/.local-dev/demo"
mkdir -p "$DEMO_DATA_DIR"
export SPRING_PROFILES_ACTIVE=h2
export SPRING_DATASOURCE_URL="jdbc:h2:file:$DEMO_DATA_DIR/eduinsight;MODE=MySQL;DB_CLOSE_ON_EXIT=FALSE"
export DATA_DIR="$DEMO_DATA_DIR/images"
export SOL_QUEUE_DIR="$DEMO_DATA_DIR/sol-jobs"
export SERVER_ADDRESS=127.0.0.1
export ADMIN_USER=teacher
export ADMIN_PASSWORD='replace-with-your-local-demo-password'
export GRADING_PROVIDER=mock
export CLASSIFICATION_ENABLED=false
java -jar backend/target/edu-insight-1.0.0-SNAPSHOT.jar
```

在第二个终端从仓库根目录启动前端：

```sh
npm run dev --prefix frontend
```

打开 <http://127.0.0.1:5173/>，用刚配置的账号密码登录。从学生管理添加演示学生，用自行生成的样张走一遍录入。网页开发服务器将 `/api` 转发到 `127.0.0.1:8080`。Mock 返回的是固定的合成区域和对错结果，不识别图片内容，这个演示也关闭了新的知识点归类。

新库创建账号时使用 `ADMIN_USER` 和 `ADMIN_PASSWORD`。同一个库里已经存在该账号时，改环境变量不会重置密码。已有工作台请使用新的演示目录，避免读写原业务库。如果 8080 或 5173 已被占用，可使用下文的 Docker 方案，或另设后端 `SERVER_PORT`，同时更新 `frontend/vite.config.ts` 的代理目标和前端端口。

## 配置真实作业识别

| 模式 | 配置及用途 |
| --- | --- |
| Sol（默认） | `GRADING_PROVIDER=sol`、`CLASSIFICATION_PROVIDER=sol`、`CLASSIFICATION_ENABLED=true`。本机 Sol 后台负责整卷识别和新知识点归类。 |
| Mock | `GRADING_PROVIDER=mock`、`CLASSIFICATION_ENABLED=false`。用于开发和演示，结果是合成示例。 |
| 腾讯 / DeepSeek（兼容） | `GRADING_PROVIDER=tencent` 用于识别，`CLASSIFICATION_PROVIDER=deepseek` 用于归类。需要分别配置凭据，变量见 `.env.example`。腾讯识别需要配置数据授权。 |

使用 Sol 时先安装 `tools/sol` 的 Node 依赖，再按 Sol 订阅运行说明（`docs/sol-local-subscription.md`） 或 Windows 本地开发（`tools/dev/README.md`） 启动后台：

```sh
npm ci --prefix tools/sol
```

macOS 安装器支持 `python3 tools/sol/install_service.py --jobs-dir /path/to/sol-jobs`，Windows 使用本仓库的 `start-sol.cmd` 或 Windows 运行器。后端 `SOL_QUEUE_DIR` 与 Sol 进程读取的队列必须指向同一个目录。安装器的参数是 `--jobs-dir`，直接运行 `service.mjs` 时对应参数是 `--jobs`。

默认授权页是 <http://127.0.0.1:8768/>。由实际使用者登录并授权自己的 ChatGPT 订阅。Sol 由独立后台运行，可以在关闭 Claude、Codex 等开发工具后继续处理任务。授权页确认模型可用，工作台登录后的 `/api/config` 显示 `grading=sol` 和 `ready=true` 后，再用合成作业验证上传。Sol 断开、未授权或额度不足时会提示原因，后端不会自动切换识别服务商。

## Docker、机构账号和小程序

- **Docker Compose。** 将 `.env.example` 复制为 `.env`，填写管理员和数据库密码。只演示时在 `.env` 中设 `GRADING_PROVIDER=mock`、`CLASSIFICATION_ENABLED=false`。执行 `docker compose up --build -d` 后访问 <http://127.0.0.1:8088/>。这一组容器启动网站和 MySQL，Sol 模式还需另启主机 Sol 后台。`SOL_QUEUE_HOST_DIR` 是主机目录，挂载为容器的 `/runtime/sol-jobs`。
- **数据库与权限。** 新建 MySQL 使用 初始化 SQL（`database/init.sql`），已有库按 数据库说明（`docs/database.md`） 选择迁移。平台管理员、机构负责人和教师的权限不同。账号激活、分配和离职交接见 机构与账号说明（`docs/institution-accounts.md`）。
- **微信小程序。** 在 `miniapp/` 执行 `npm ci` 和 `npm run build`，导入到微信开发者工具。使用自己的 AppID，复制 `config.example.json` 为 `config.local.json` 并配置 `apiBase`。AppSecret 配在后端。真机需要可达的后端，体验版和正式版需要 HTTPS 和合法域名配置。见 小程序开发（`miniapp/README.md`）。Apple 芯片 Mac 的小程序编译器可能需要 Rosetta。

## 代码入口和数据流

| 目录 | 职责和入口 |
| --- | --- |
| `frontend/` | Vue 3 + TypeScript 教师网页。开发入口是 `frontend/package.json`，页面在 `frontend/src/views/`。 |
| `backend/` | Java 21 + Spring Boot API。入口是 `com.eduinsight.Application`，业务包包括 `paper`、`grading`、`classification`、`curriculum` 和 `auth`。 |
| `tools/sol/` | Node 后台。`service.mjs` 扫描本机任务目录，`auth.mjs` 处理授权，`inference.mjs` 请求模型和校验结果，`taxonomy.mjs` 负责知识点归类。 |
| `miniapp/` | 原生微信小程序 + TypeScript，沿用同一 API。 |
| `database/` | MySQL 初始化和迁移入口。H2 用于本机开发。 |
| `docs/` | 架构（`docs/architecture.md`）、API（`docs/api.md`）、知识目录（`docs/knowledge-management.md`）和业务口径。 |

网页或小程序将作业交给 Java API。Sol 模式下，Java 将任务和页面图片写入共享目录，Node 后台调用模型并回写状态、回执和结果，Java 校验后再落库。知识点归类也使用独立文件任务。没有原卷、无法确认对错、重复上传和重启恢复是正常业务分支，修改时需要保留这些行为。

## 检查代码与接手开发

在仓库根目录执行：

```sh
npm ci --prefix frontend
npm ci --prefix tools/sol
(cd backend && mvn test)
npm run build --prefix frontend
npm run test:insight --prefix frontend
npm run test:students --prefix frontend
npm test --prefix tools/sol
```

普通后端测试使用隔离 H2 和测试替身。MySQL 和真实原件专项需要独立库或样本路径。前端浏览器专项使用无头 Chrome / Edge，详见 `frontend/package.json`；小程序用 `npm test --prefix miniapp` 执行构建及单元测试。通过软件测试后，真实模型仍须另测识别准确率、响应时间、未知题比例和费用。

让 AI 接手时，先读 [AGENTS.md](AGENTS.md)，再选一个具体目标。若要支持豆包或其他视觉 API，需要实现接口适配，并同时覆盖整卷识别和知识点归类；`result.json`、结果校验、UNKNOWN 策略和教师更正流程须保持兼容。仓库已记录验收条件，暂未提供通用视觉 API 配置开关。

## 数据与隐私

将 `.env`、本地数据库、学生姓名和账号、作业原图、模型请求样本及授权文件留在使用者自己的环境中。演示和测试优先使用合成材料。真实作业可能含未成年人的姓名和字迹，启用任何第三方识别或归类服务前，使用者应确认授权和数据会发往哪里。源码交接不包含原工作台的数据、账号、运行实例或订阅授权。
