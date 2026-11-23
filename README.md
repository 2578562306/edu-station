# 数树 · 教师工作台（edu-station）

[下载完整源码（ZIP）](https://github.com/2578562306/edu-station/releases/download/handoff-2026-11-23/edu-station-public-20261123.zip)。解压后进入 `edu-station/`，下文代码路径都从这个目录开始。

数树记录学生作业、原卷、错题和学情，帮助老师看清谁在哪一课反复出错、哪些内容可以一起讲。网页和微信小程序共用后端。**先跑本机 Mock 演示，见到实际界面后再决定改哪里。**

## 把这段话发给能操作电脑的 AI

> 请读完这个仓库的 README.md 和 AGENTS.md，按我的系统从零启动数树 H2 + Mock 演示。先检查 JDK、Maven、Node 和端口，使用新的独立目录、临时老师账号和随机演示密码，只监听本机。已有老师工作台、学生库、作业、凭据和模型服务全部留在原处。打开网页登录，创建一个虚构学生，生成并录入一页合成作业，确认能查看原卷和模拟判定。验收登录状态和 `/api/config`，告诉我已启动的本机网址、演示进度，以及如果要改功能该看哪些文件。密码只在本机登录时使用，勿输出到聊天、日志、截图或提交里。依赖或端口有问题时先说明实测结果，勿把没跑通的步骤写成通过。

## 本机快速启动

准备 **JDK 21、Maven 3.9、Node.js 22.13 或更新的 22.x**。演示使用一份新建 H2 数据库、Mock 判定和关闭的新归类任务，整个演示无需模型账号、API Key 或真实作业。下列命令默认使用后端 `8080` 和前端 `5173`。先确认端口空闲。已有服务时可使用下文的 Docker 方式，或为演示另选本机端口并在独立工作副本同步调整 `frontend/vite.config.ts` 的前端端口和 `/api` 代理目标。保留已有进程。

### macOS 或 Linux

在仓库根目录打开终端。以下为 Bash 命令，命令报错时先排查该步。后端保持运行，演示账号为 `teacher`。随机密码写入一个仅当前用户可读的本地文件，终端只显示该文件位置。

```bash
npm ci --prefix frontend
mvn -f backend/pom.xml -DskipTests package

umask 077
mkdir -p .local-dev
DEMO_DATA_DIR="$(mktemp -d "$PWD/.local-dev/demo.XXXXXX")"
export SPRING_PROFILES_ACTIVE=h2
export SPRING_DATASOURCE_URL="jdbc:h2:file:$DEMO_DATA_DIR/eduinsight;MODE=MySQL;DB_CLOSE_ON_EXIT=FALSE"
export DATA_DIR="$DEMO_DATA_DIR/images"
export SOL_QUEUE_DIR="$DEMO_DATA_DIR/sol-jobs"
export SERVER_ADDRESS=127.0.0.1 SERVER_PORT=8080
export GRADING_PROVIDER=mock CLASSIFICATION_ENABLED=false
export ADMIN_USER=teacher
ADMIN_PASSWORD="$(node -e 'process.stdout.write(require("node:crypto").randomBytes(24).toString("base64url"))')"
export ADMIN_PASSWORD
printf '%s\n' "$ADMIN_PASSWORD" > "$DEMO_DATA_DIR/login.txt"
printf '演示密码文件：%s\n' "$DEMO_DATA_DIR/login.txt"
java -jar backend/target/edu-insight-1.0.0-SNAPSHOT.jar
```

### Windows

在仓库根目录打开 PowerShell。命令报错时先排查该步。`Read-Host` 不显示输入内容。为这次演示输入一个从未在其他服务使用的密码，网页登录时用同一个。能执行命令的 AI 也可以在启动进程内生成随机临时密码，再通过本机浏览器登录。

```powershell
npm ci --prefix frontend
mvn -f backend/pom.xml -DskipTests package

$demoPath = Join-Path (Get-Location).Path ('.local-dev\demo-' + [guid]::NewGuid().ToString('N'))
New-Item -ItemType Directory -Path $demoPath -Force | Out-Null
$demoDataDir = $demoPath.Replace('\', '/')
$env:SPRING_PROFILES_ACTIVE = 'h2'
$env:SPRING_DATASOURCE_URL = "jdbc:h2:file:${demoDataDir}/eduinsight;MODE=MySQL;DB_CLOSE_ON_EXIT=FALSE"
$env:DATA_DIR = "$demoDataDir/images"
$env:SOL_QUEUE_DIR = "$demoDataDir/sol-jobs"
$env:SERVER_ADDRESS = '127.0.0.1'
$env:SERVER_PORT = '8080'
$env:GRADING_PROVIDER = 'mock'
$env:CLASSIFICATION_ENABLED = 'false'
$env:ADMIN_USER = 'teacher'
$demoPassword = Read-Host '输入本机演示密码' -AsSecureString
$env:ADMIN_PASSWORD = [System.Net.NetworkCredential]::new('', $demoPassword).Password
Remove-Variable demoPassword
java -jar backend/target/edu-insight-1.0.0-SNAPSHOT.jar
```

### 启动网页并验证

后端启动后，在第二个终端从仓库根目录执行：

```sh
npm run dev --prefix frontend
```

1. 打开 <http://127.0.0.1:5173/>，用 `teacher` 和这次演示的密码登录。开发服务器把 `/api` 转发到本机后端。
2. 进入「学生管理」→「新增学生」，创建虚构的「演示学生 A」。回工作台点「录入作业」，选该学生和数学，上传一页自行制作的假作业。下方命令可生成一页合成 PDF，里面只有虚构的算式。
3. 在原卷页点击「开始分析」（如果已经自动开始，等结果即可），确认能看到页面和 Mock 模拟判定，再从工作台或学生档案找回这份录入。**Mock 返回固定的示例区域和对错，可验证操作流程。判定和图上的算式没有对应关系。**
4. 登录后在同一个浏览器分别打开 `/api/auth/status` 和 `/api/config`。预期是 `data.authenticated=true`、`data.grading="mock"`、`data.ready=true`。登录不通过先看启动终端的报错和正在使用的数据库目录。

在任一终端从仓库根目录执行以下命令生成合成 PDF（前端依赖须已安装）。文件写到 `.local-dev/demo-worksheet.pdf`。

```sh
cd frontend
node --input-type=module -e "import { PDFDocument, StandardFonts } from 'pdf-lib'; import { mkdir, writeFile } from 'node:fs/promises'; const doc = await PDFDocument.create(); const page = doc.addPage([595, 842]); const font = await doc.embedFont(StandardFonts.Helvetica); page.drawText('Synthetic demo worksheet', { x: 48, y: 786, size: 19, font }); page.drawText('1. 3 + 5 = 8', { x: 48, y: 724, size: 14, font }); page.drawText('2. 7 x 4 = 24', { x: 48, y: 679, size: 14, font }); await mkdir('../.local-dev', { recursive: true }); await writeFile('../.local-dev/demo-worksheet.pdf', await doc.save());"
```

首次创建负责人账号必须显式配置非空 `ADMIN_PASSWORD`。使用同一数据库和 `ADMIN_USER` 重启时，会继续使用数据库里已设的密码。密钥、真实学生和录屏凭据放在私有环境中，演示目录与个人资料分开，录前确认没有真实学生、密码或令牌出现在画面里。

## 这个版本能做什么

- 网页工作台、学生和机构管理、作业录入、错题复核、学情统计与错题练习卷导出。浏览和导出已有结果无需再次调用模型。
- 图片和 PDF 可录入单份作业。单张图片最大 10 MB，单份 PDF 最大 50 MB，一份作业最多 20 页。PDF 批量录入支持按学号匹配，每批最多 60 份。详细规则见 PDF 批量录入（`docs/pdf-batch-upload.md`）。
- 工作台按学生、课时和**实际作业日期**观察反复出错的情况。当前人教版七年级上册数学有教材课时映射，其他范围按已有知识点名称呈现。教师可以补记作业日期、纠正判定和查看原卷。见 工作台口径（`docs/product-refactor-20261116.md`） 和 学情统计口径（`docs/knowledge-analytics.md`）。
- 默认真实识别使用独立 Sol 后台，需要使用者自己授权 ChatGPT 订阅并具备 `gpt-6.1-sol` 的权限。豆包、火山方舟和通用视觉模型 API Key 接入尚未实现。

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

macOS 安装器需要 Python 3 和可用的 Swift 编译器，支持 `python3 tools/sol/install_service.py --jobs-dir /path/to/sol-jobs`。Windows 使用本仓库的 `start-sol.cmd` 或 Windows 运行器。Linux 可以运行 H2 + Mock 和容器，仓库尚无 Linux 的正式 Sol 订阅授权流程。后端 `SOL_QUEUE_DIR` 与 Sol 进程读取的队列必须指向同一个目录。安装器的参数是 `--jobs-dir`，直接运行 `service.mjs` 时对应参数是 `--jobs`。

默认授权页是 <http://127.0.0.1:8768/>。由实际使用者登录并授权自己的 ChatGPT 订阅。Sol 由独立后台运行，可以在关闭 Claude、Codex 等开发工具后继续处理任务。授权页确认模型可用，工作台登录后的 `/api/config` 显示 `grading=sol` 和 `ready=true` 后，再用合成作业验证上传。Sol 断开、未授权或额度不足时会提示原因，后端不会自动切换识别服务商。

## Docker、机构账号和小程序

- **Docker Compose。** 将 `.env.example` 复制为 `.env`，填写管理员和数据库密码。只演示时在 `.env` 中设 `GRADING_PROVIDER=mock`、`CLASSIFICATION_ENABLED=false`。执行 `docker compose up --build -d` 后访问 <http://127.0.0.1:8088/>。这一组容器启动网站和 MySQL，Sol 模式还需另启主机 Sol 后台。`SOL_QUEUE_HOST_DIR` 是主机目录，挂载为容器的 `/runtime/sol-jobs`。
- **数据库与权限。** 新建 MySQL 使用 初始化 SQL（`database/init.sql`），已有库按 数据库说明（`docs/database.md`） 选择迁移。平台管理员、机构负责人和教师的权限不同。账号激活、分配和离职交接见 机构与账号说明（`docs/institution-accounts.md`）。
- **微信小程序。** 在 `miniapp/` 执行 `npm ci` 和 `npm run build`，导入到微信开发者工具。使用自己的 AppID，复制 `config.example.json` 为 `config.local.json` 并配置 `apiBase`。AppSecret 配在后端。真机需要可达的后端，体验版和正式版需要 HTTPS 和合法域名配置。见 小程序开发（`miniapp/README.md`）。Apple 芯片 Mac 的小程序编译器可能需要 Rosetta。

## 想改什么，就从哪里下手

| 修改目标 | 入口 |
| --- | --- |
| 工作台、学生、作业上传和原卷 | Vue 3 + TypeScript，`frontend/src/views/` 的 `Workbench.vue`、`Students.vue`、`Upload.vue`、`Review.vue`。 |
| 作业、账户、统计、教学知识结构 | Java 21 + Spring Boot，从 `Application.java`（`backend/src/main/java/com/eduinsight/Application.java`） 启动。按业务找 `backend/src/main/java/com/eduinsight/` 中的 `paper/`、`auth/`、`insight/`、`analytics/` 或 `curriculum/`。 |
| 图像识别、模型或提示词 | `tools/sol/inference.mjs` 组装模型请求和校验识别结果，`prompt.txt`（`tools/sol/prompt.txt`） 是判断提示词，`output-schema.json`（`tools/sol/output-schema.json`） 定义结果结构。Java 端 `SolGradingProvider.java`（`backend/src/main/java/com/eduinsight/grading/SolGradingProvider.java`） 对接队列、验证模型结果。 |
| 知识点归类、授权与队列 | `tools/sol/taxonomy.mjs` 管归类，`service.mjs`（`tools/sol/service.mjs`） 管队列和回执，`auth.mjs`（`tools/sol/auth.mjs`） 管订阅授权。Java 端见 `classification/`（`backend/src/main/java/com/eduinsight/classification/`）。 |
| 微信端页面和上传流程 | 原生微信小程序 + TypeScript，`miniapp/src/pages/` 和 `miniapp/src/core/`，沿用同一 API。 |
| 数据库、接口和业务规则 | `database/` 中的初始化和迁移，架构（`docs/architecture.md`）、API（`docs/api.md`）、知识目录（`docs/knowledge-management.md`）和 工作台口径（`docs/product-refactor-20261116.md`）。 |

网页或小程序将作业交给 Java API。Sol 模式下，Java 将任务和页面图片写入共享目录，Node 后台调用模型并回写状态、回执和结果，Java 校验后再落库。知识点归类也使用独立文件任务。没有原卷、无法确认对错、重复上传和重启恢复是正常业务分支，修改时需要保留这些行为。

## 检查代码与接手开发

在仓库根目录执行：

```sh
npm ci --prefix frontend
npm ci --prefix tools/sol
mvn -f backend/pom.xml test
npm run build --prefix frontend
npm run test:insight --prefix frontend
npm run test:students --prefix frontend
npm test --prefix tools/sol
```

普通后端测试使用隔离 H2 和测试替身。MySQL 和真实原件专项需要独立库或样本路径。前端浏览器专项使用无头 Chrome / Edge，详见 `frontend/package.json`。小程序用 `npm test --prefix miniapp` 执行构建及单元测试。通过软件测试后，真实模型仍须另测识别准确率、响应时间、未知题比例和费用。

让 AI 接手时，先读 [AGENTS.md](AGENTS.md)，再选一个具体目标。若要支持豆包或其他视觉 API，需要实现接口适配，并同时覆盖整卷识别和知识点归类；`result.json`、结果校验、UNKNOWN 策略和教师更正流程须保持兼容。仓库已记录验收条件，暂未提供通用视觉 API 配置开关。

## 数据与隐私

将 `.env`、本地数据库、学生姓名和账号、作业原图、模型请求样本及授权文件留在使用者自己的环境中。演示和测试优先使用合成材料。真实作业可能含未成年人的姓名和字迹，启用任何第三方识别或归类服务前，使用者应确认授权和数据会发往哪里。源码交接不包含原工作台的数据、账号、运行实例或订阅授权。
