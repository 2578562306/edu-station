# 数树代码接手指南

先看 [README.md](README.md) 的本机启动、识别模式和代码入口。这里的项目是教师工作台、Java API、微信小程序和一个独立的 Sol 模型后台。接手后先用全新的 H2 演示库和 Mock 走通，再修改目标功能。真实业务记录和模型凭据保存在使用者自己的环境。

## 找代码

- 网页看 `frontend/src/views/`，小程序看 `miniapp/src/`。两端使用同一份 Java API，接口见 [docs/api.md](docs/api.md)。
- 业务口径以 [docs/product-refactor-20261116.md](docs/product-refactor-20261116.md) 为入口。工作台按实际作业日期计算。教师更正、原卷页、跨页来源和复核结果须保留。
- 后端从 `backend/src/main/java/com/eduinsight/Application.java` 启动。重点模块有 `paper/`、`grading/`、`classification/`、`curriculum/` 和 `auth/`。
- 真实 Sol 识别在 `tools/sol/`。Java 写入 `request.json` 和图片，Node 返回 `status.json`、`result.json`、`receipt.json`。健康状态和实际模型信息写在 `worker.json`。`sol-*` 是整卷识别，`tax-*` 是目录归类。

## 加视觉 API 模式时

当前 Sol 识别和目录归类用个人 ChatGPT 订阅授权。一般视觉模型 API Key 接入（包括豆包／火山方舟）是待开发项。以下是该任务的要求。

1. 用受保护的环境变量或被 Git 忽略的本地文件读取接口地址、API Key、模型名。三项齐全后才启用新模式，保留订阅模式和已有的 Tencent、DeepSeek、Mock 兼容实现。`.env.example` 使用占位文字。
2. 先读目标厂商的官方文档，核对实际的多模态输入、结构化输出和错误格式。`tools/sol/auth.mjs` 处理订阅授权，`inference.mjs` 负责整卷识别和 `callStructured`，`taxonomy.mjs` 使用同一调用路径做归类。适配两个任务，并保留明确的鉴权失败、超时和格式失败处理。厂商没有严格 JSON Schema 输出时，在后台校验并有限重试。
3. 保持输出协议兼容。`tools/sol/inference.mjs` 的识别 Schema、`tools/sol/prompt.txt` 的判定约束、`SolGradingProvider.java` 的后端校验和 `UNKNOWN` 策略继续有效。处理重复投递、断线恢复和教师更正时不得重写已确认结果。
4. 后端和后台统一读取配置的模型名，默认仍为 `gpt-6.1-sol`。`worker.json`、回执和数据库中的结果来源记录真实模型。`service.mjs` 默认连接页端口为 `8768`，API Key 模式不应要求订阅授权。
5. 检查网页录入与复核、`miniapp/src/core/upload-job.ts`、`SolGradingProvider.java` 和 `classification/ClassificationSettings.java` 的错误提示。连接提示应符合实际模式。完成后同步更新 README、`.env.example` 和本页。

## 验收

- 运行 `npm test --prefix tools/sol`、`(cd backend && mvn test)`、`npm run build --prefix frontend`、`npm run test:insight --prefix frontend`。先执行对应目录的 `npm ci` 安装依赖。
- 改视觉 API 接入时，用本地替身覆盖成功、无效结构重试、错误 Key 和超时；人工复核已识别题目和未知题语义，不用真实学生资料做自动化夹具。
- 确认真实服务商接入时，用使用者授权的 Key 和合成作业跑一页。原卷须能查看题目、作答和判定，工作台须能看到新录入。记录模型、实测用时、未知题比例、费用估算和未验证项。真实学生作业仅在使用者确认资料授权和服务商后使用。
- 密钥和授权信息只在运行环境使用。提交前排除学生信息、原卷、数据库、运行日志、订阅授权以及临时私有配置。先在隔离环境验证数据库升级，已有库不可通过重新初始化来解决迁移问题。
