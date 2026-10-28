# 学成在线 xuecheng-plus

## 项目简介

黑马程序员「学成在线」微服务实战项目的后端实现，一个在线教育平台的分布式系统：课程内容管理、课程审核与发布（FreeMarker 静态化）、媒资（视频/图片）上传与管理（MinIO + 分块上传）、课程搜索（Elasticsearch）、系统管理（数据字典）、统一认证（Spring Security OAuth2 + JWT）、图形验证码、学习中心（选课），并通过 Spring Cloud Gateway 统一入口。

服务注册与配置**全部托管在 Nacos**，仓库里只有 `bootstrap.yml`，真正的数据库、Redis、MinIO、Elasticsearch 参数需要自己在 Nacos 里配置。

## 功能模块

### 内容管理服务 `content`（`xuecheng-plus-content`）

| 能力 | 接口 | 位置 |
| --- | --- | --- |
| 课程分页查询、新增、详情、修改、删除 | `/course/list`、`/course` | `CourseBaseInfoController` |
| 课程分类树 | `/course-category/tree-nodes` | `CourseCategoryController` |
| 课程计划（章/节）树查询、增删改、上下移动、与媒资绑定/解绑 | `/teachplan/**` | `TeachplanController` |
| 课程教师查询、添加修改、删除 | `/courseTeacher/**` | `CourseTeacherController` |
| 课程预览（FreeMarker 渲染静态页）、提交审核、发布、查询发布信息 | `/coursepreview/{courseId}`、`/courseaudit/commit/{courseId}`、`/coursepublish/{courseId}`、`/r/coursepublish/{courseId}` | `CoursePublishController` |
| 公开接口：获取课程完整信息（供学习中心 Feign 调用） | `/open/course/whole/{courseId}` | `CourseOpenController` |
| FreeMarker 模板测试 | `/testfreemarker` | `FreemarkerController` |

发布流程：`publish()` 用 `course_template.ftl` 生成课程详情 HTML → 通过 Feign 调 `media-api` 上传静态页 → 更新发布表 → 向 `mq_message` 表写入一条 `course_publish` 消息，由 xxl-job 调度的 `MessageProcessAbstract` 异步推送到 `search` 建索引。

### 媒资管理服务 `media`（`xuecheng-plus-media`）

| 能力 | 接口 | 位置 |
| --- | --- | --- |
| 文件上传前的秒传检查 | `POST /upload/checkfile` | `BigFilesController` |
| 分块上传前检查、上传分块、合并分块 | `POST /upload/checkchunk`、`/upload/uploadchunk`、`/upload/mergechunks` | `BigFilesController` |
| 媒资分页查询、上传图片 | `POST /files`、`POST /upload/coursefile` | `MediaFilesController` |
| 公开预览（MinIO 预签名/代理下载） | `GET /open/preview/{mediaId}` | `MediaOpenController` |

存储走 MinIO（`MinioConfig` 读取 `minio.endpoint/accessKey/secretKey`），大视频按分块上传后合并。

### 搜索服务 `search`（`xuecheng-plus-search`）

| 能力 | 接口 |
| --- | --- |
| 添加课程索引 | `POST /index/course` |
| 课程搜索列表（分页 + 关键字 + 分类 + 难度等） | `GET /course/list` |

基于 Elasticsearch 7.12.1 的 `RestHighLevelClient`。

### 系统管理服务 `system`（`xuecheng-plus-system`）

| 能力 | 接口 |
| --- | --- |
| 查询全部数据字典 | `GET /dictionary/all` |
| 按 code 查询字典项 | `GET /dictionary/code/{code}` |

### 认证服务 `auth`（`xuecheng-plus-auth`）

| 能力 | 接口 |
| --- | --- |
| 登录成功回调，返回用户身份与权限 | `GET /login-success` |
| 按 id 查询用户 | `GET /user/{id}` |
| 权限测试资源 | `GET /r/r1`、`GET /r/r2` |
| 微信扫码登录 | `GET /wxLogin` |

基于 `spring-cloud-starter-oauth2`，登录时通过 Feign 调 `checkcode` 校验图形验证码（`CheckCodeClient`）。

### 验证码服务 `checkcode`（`xuecheng-plus-checkcode`）

| 能力 | 接口 |
| --- | --- |
| 生成图形验证码（kaptcha，key 存 Redis） | `POST /pic` |
| 校验验证码 | `POST /verify` |

### 网关 `gateway`（`xuecheng-plus-gateway`）

- `GatewayAuthFilter`（`GlobalFilter`）：读 `security-whitelist.properties` 白名单（`/auth/**`、`/content/open/**`、`/media/open/**`），其余请求校验 JWT，非法 token 返回 401。
- `TokenConfig`：`JwtTokenStore` + `JwtAccessTokenConverter`。
- `SecurityConfig`：WebFlux Security，`/**` permitAll，实际鉴权由上面的过滤器完成。

### 消息与调度

- `xuecheng-plus-message-sdk`：**本地消息表**实现（不是 MQ 中间件），`mq_message` / `mq_message_history` 两张表 + `MessageProcessAbstract` 抽象类，业务方继承后实现 `process()`，由定时任务扫描待发送消息。
- `xxl-job-2.3.1`：调度中心源码，用于扫描消息表、清理任务等。

### 代码生成器 `xuecheng-plus-generator`

MyBatis-Plus Generator 的使用示例（`BankCodeGenerator`），按库表生成 model/mapper/service/controller。

## 技术栈

全部依据 `pom.xml` 与 `bootstrap.yml`：

| 类别 | 选型 | 版本 |
| --- | --- | --- |
| 语言 / 构建 | Java、Maven 多模块（无根聚合 POM） | 1.8 |
| 基础框架 | Spring Boot | 2.3.7.RELEASE |
| 微服务 | Spring Cloud / Spring Cloud Alibaba | Hoxton.SR9 / 2.2.6.RELEASE |
| 注册 + 配置中心 | Nacos（discovery + config，`bootstrap.yml`） | 随 SCA |
| 网关 | Spring Cloud Gateway | 随 SC |
| 安全 | Spring Security + OAuth2（JWT TokenStore） | 随 SC |
| 服务调用 | OpenFeign（含 `feign-httpclient`、`feign-form`、降级 FallbackFactory） | 随 SC |
| 持久层 | MyBatis-Plus（+ `mybatis-plus-generator`） | 3.4.1 |
| 数据库 | MySQL（`mysql-connector-java`） | 8.0.30 |
| 连接池 | Druid | 1.2.8 |
| 缓存 | Redis（验证码、令牌等） | 随 Boot |
| 对象存储 | MinIO Java SDK | 8.4.3 |
| 搜索 | Elasticsearch（`elasticsearch-rest-high-level-client`） | 7.12.1 |
| 模板引擎 | FreeMarker（课程详情静态化，`course_template.ftl`） | 随 Boot |
| 任务调度 | xxl-job-core（并内置 xxl-job 2.3.1 源码） | 2.3.1 |
| 验证码 | kaptcha | 随 SCA/独立 |
| 接口文档 | spring4all swagger-spring-boot-starter + swagger-annotations | 1.9.0.RELEASE / 1.5.20 |
| 工具 | Lombok 1.18.8、MapStruct 1.3.1.Final、fastjson 1.2.83、commons-lang(3)、guava 25.0、okhttp 4.8.1、simplemagic（文件类型识别） | — |
| 日志 | Log4j2（各服务 `log4j2-dev.xml`） | 随 Boot |

## 模块说明

仓库**没有根 `pom.xml`**，各服务独立构建；公共父 POM 是 `xuecheng-plus-parent`（需先 `install`）。

```
xuechengplus/
├── xuecheng-plus-parent/            父 POM：统一依赖版本、编译插件、资源过滤
├── xuecheng-plus-base/              公共基础包：RestResponse/PageResult、异常体系、工具类
│                                    （JsonUtil、RSAUtil、MD5Util、HttpUtil、VideoUtil、Mp4VideoUtil…）
├── xuecheng-plus-gateway/           网关服务（bootstrap.yml + security-whitelist.properties）
├── xuecheng-plus-auth/              OAuth2 认证服务（登录、wxLogin、权限测试）
├── xuecheng-plus-checkcode/         图形验证码服务（kaptcha + Redis）
├── xuecheng-plus-content/           内容管理聚合 POM
│   ├── xuecheng-plus-content-model/    实体/DTO（依赖 base）
│   ├── xuecheng-plus-content-service/  业务实现、Mapper、Feign 客户端（依赖 model + message-sdk + xxl-job）
│   └── xuecheng-plus-content-api/      Controller 层（依赖 service）
├── xuecheng-plus-media/             媒资聚合 POM
│   ├── xuecheng-plus-media-model/      实体/DTO（依赖 base）
│   ├── xuecheng-plus-media-service/    MinIO 上传、分块合并、视频处理（依赖 model）
│   └── xuecheng-plus-media-api/        Controller 层（依赖 model + service）
├── xuecheng-plus-system/            系统管理聚合 POM
│   ├── xuecheng-plus-system-model/
│   ├── xuecheng-plus-system-service/
│   └── xuecheng-plus-system-api/       字典接口
├── xuecheng-plus-search/            搜索服务（依赖 base，ES 客户端）
├── xuecheng-plus-learning/          学习中心（**不完整**：只有 learning-service/src，无 pom）
├── xuecheng-plus-message-sdk/       本地消息表 SDK（mq_message / mq_message_history）
├── xuecheng-plus-generator/         MyBatis-Plus 代码生成器示例
├── xxl-job-2.3.1/                   第三方调度平台源码（原样入库）
└── api-test/                        IntelliJ HTTP Client 接口测试脚本 + 各服务地址
```

依赖关系（依据各 `pom.xml`）：

| 模块 | 依赖的内部模块 |
| --- | --- |
| `xuecheng-plus-base` | parent |
| `xuecheng-plus-checkcode` | base |
| `xuecheng-plus-search` | base |
| `xuecheng-plus-content-model` | base |
| `xuecheng-plus-content-service` | content-model、message-sdk |
| `xuecheng-plus-content-api` | content-service |
| `xuecheng-plus-media-model` | base |
| `xuecheng-plus-media-service` | media-model |
| `xuecheng-plus-media-api` | media-model、media-service |
| `xuecheng-plus-system-model` | base |
| `xuecheng-plus-system-service` | system-model |
| `xuecheng-plus-system-api` | system-model、system-service |
| `xuecheng-plus-gateway` / `xuecheng-plus-auth` / `xuecheng-plus-message-sdk` | 仅 parent |

服务名（Nacos 中的 `spring.application.name`）：`gateway`、`auth-service`、`checkcode`、`content-api`、`content-service`、`media-api`、`media-service`、`search`、`system-api`。

## 数据库与配置

### 需要的中间件

| 组件 | 说明 |
| --- | --- |
| MySQL 8 | 各服务独立库（从 bootstrap 注释可看到 `xc402_content`、`xc402_system`、`xcplus_content` 等库名） |
| Redis | 验证码、令牌相关缓存 |
| Nacos | 注册中心 + 配置中心，默认 `192.168.101.65:8848` |
| MinIO | 媒资存储（media-service 的 `minio.endpoint/accessKey/secretKey`） |
| Elasticsearch 7.12.x | 课程搜索 |
| xxl-job | 消息表扫描任务调度 |

### 配置全部在 Nacos

仓库内只有 `bootstrap.yml`，其中固定了：

- `spring.cloud.nacos.server-addr: 192.168.101.65:8848`
- `namespace: de4da233-33dc-4bbe-9657-8983d7b69686`
- `group: xuecheng-plus-project`（公共配置在 `xuecheng-plus-common` 组）
- `file-extension: yaml`、`profiles.active: dev`

需要自己在 Nacos 里准备的数据源配置（dataId 依据各服务的 `extension-configs` / `shared-configs`）：

| dataId | group | 内容 |
| --- | --- | --- |
| `content-service-dev.yaml` | `xuecheng-plus-project` | content 服务数据源等 |
| `media-service-dev.yaml` | `xuecheng-plus-project` | MinIO + 数据源 |
| `system-service-dev.yaml` | `xuecheng-plus-project` | system 服务数据源 |
| `swagger-dev.yaml` | `xuecheng-plus-common` | 各服务 Swagger 配置 |
| `logging-dev.yaml` | `xuecheng-plus-common` | 日志配置 |
| `feign-dev.yaml` | `xuecheng-plus-common` | Feign 超时/日志级别 |
| `redis-dev.yaml` | `xuecheng-plus-common` | Redis 连接（checkcode 使用） |
| `freemarker-config-dev.yaml` | `xuecheng-plus-common` | FreeMarker 配置（content-api 使用） |

此外 `auth-service`、`checkcode`、`search`、`gateway`、`*_learning` 服务自身的 `xxx-dev.yaml` 也需按需创建（网关路由、OAuth2 客户端、JWT 等）。

### 建表脚本

**仓库未包含业务 SQL 脚本**（唯一的 `.sql` 是 `xxl-job-2.3.1/doc/db/tables_xxl_job.sql`，属于调度平台自身）。业务库（课程、媒资、字典、选课、`mq_message`、`mq_message_history` 等表）需按课程资料自行准备。

### 服务端口（来自 `api-test/http-client.env.json`）

| 服务 | 端口 |
| --- | --- |
| gateway | 63010 |
| content | 63040 |
| media | 63050 |
| auth | 63070 |
| checkcode | 63075 |
| search | 63080 |
| system | 63110 |
| learning | 63020 |

## 快速开始

```bash
# 0. 环境：JDK 8、Maven 3.6+、MySQL 8、Redis、Nacos 2.x、MinIO、Elasticsearch 7.12、xxl-job

# 1. 先安装公共父 POM 与 base（没有根 pom，必须逐个构建）
cd xuecheng-plus-parent && mvn clean install -DskipTests
cd ../xuecheng-plus-base && mvn clean install -DskipTests
cd ../xuecheng-plus-message-sdk && mvn clean install -DskipTests

# 2. 构建各业务服务（以内容服务为例）
cd ../xuecheng-plus-content && mvn clean install -DskipTests

# 3. 在 Nacos 中创建命名空间 de4da233-33dc-4bbe-9657-8983d7b69686，
#    并在 xuecheng-plus-project / xuecheng-plus-common 两个组下补齐上表列出的配置项，
#    同时按需修改各 bootstrap.yml 中的 server-addr（默认指向 192.168.101.65）

# 4. 启动（每个服务单独一个进程）
mvn -pl xuecheng-plus-content/xuecheng-plus-content-api spring-boot:run
# 网关、auth、checkcode、media-api、search、system-api 同理

# 5. 用 api-test 目录下的 .http 脚本验证（IntelliJ HTTP Client）
#    先把 http-client.env.json 里的 host 改成本机，并用 /auth 登录后的 access_token 填上
```

启动顺序建议：Nacos → MySQL/Redis/MinIO/ES → xxl-job → checkcode / auth / system / media / content / search → gateway。

## 接口文档

经网关（63010）访问，也可直连各服务端口：

| 方法 | 路径 | 用途 |
| --- | --- | --- |
| POST | `/content/course/list?pageNo=&pageSize=` | 课程分页查询（条件：课程名称、审核状态、发布状态） |
| POST | `/content/course` | 新增课程 |
| GET | `/content/course/{courseId}` | 课程详情 |
| GET | `/content/course-category/tree-nodes` | 课程分类树 |
| GET | `/content/teachplan/{courseId}/tree-nodes` | 课程计划树 |
| POST | `/content/teachplan/association/media` | 课程计划绑定媒资 |
| POST | `/content/courseaudit/commit/{courseId}` | 提交课程审核 |
| POST | `/content/coursepublish/{courseId}` | 发布课程（生成静态页 + 发消息） |
| GET | `/content/open/course/whole/{courseId}` | 公开：获取课程完整信息 |
| POST | `/media/upload/checkfile` | 上传前检查（秒传） |
| POST | `/media/upload/uploadchunk` | 上传分块 |
| POST | `/media/upload/mergechunks` | 合并分块 |
| GET | `/media/open/preview/{mediaId}` | 公开：预览媒资 |
| GET | `/search/course/list` | 课程搜索 |
| POST | `/checkcode/pic` | 生成图形验证码 |
| POST | `/checkcode/verify` | 校验验证码 |
| GET | `/system/dictionary/all` | 查询数据字典 |
| POST | `/auth/oauth/token` | OAuth2 获取令牌（`password` / `authorization_code` 模式，客户端 `XcWebApp`） |
| GET | `/auth/login-success` | 认证成功后返回用户信息 |
| GET | `/auth/wxLogin` | 微信扫码登录 |

## 已知问题

1. **没有根聚合 POM**：仓库根目录没有 `pom.xml`，20 个模块（另有一个无 POM 的 learning-service）彼此独立，必须按 parent → base → message-sdk → 业务模块的顺序手动 `install`，无法一条命令构建全量。
2. **`xuecheng-plus-learning` 模块残缺**：只有 `xuecheng-plus-learning-service/src` 下两个 Java 文件（`MyCourseTablesServiceImpl`、`MyCourseTablesService`），**没有 `pom.xml`**，且代码引用的 `ContentServiceClient`、`XcChooseCourseMapper`、`XcCourseTables`、`learning.model.*` 等类都不在仓库里，整个模块无法编译。
3. **`content-service` 没有生产启动类**：`ContentApplication` 只存在于 `src/test` 下，service 模块本身不能独立启动（设计上靠 api 模块装配，但目录结构容易误导）。
4. **硬编码 JWT 签名密钥**：`gateway`、`xuecheng-plus-auth`、`xuecheng-plus-content-api` 三处的 `TokenConfig` 里 `SIGNING_KEY` 都是同一个写死的短字符串常量；网关 `SecurityConfig` 对 `/**` 全部 `permitAll`，真正的鉴权只靠 `GatewayAuthFilter`，密钥泄漏即可伪造任意用户令牌。
5. **配置强绑定内网环境**：所有 `bootstrap.yml` 写死 `192.168.101.65:8848` 与固定 namespace `de4da233-...`，换机器必须逐个改文件；`xuecheng-plus-generator` 里还有 `jdbc:mysql://localhost:3306/...` 硬编码。
6. **测试资源里残留明文数据库口令**：`xuecheng-plus-message-sdk/src/test/resources/bootstrap.yml` 与 `content-service` 的测试配置里出现 MySQL `root` + 明文密码、内网 IP。
7. **第三方源码整体入库**：`xxl-job-2.3.1`（含 admin、core、samples、doc）被完整提交进仓库，压缩包式携带第三方代码，应改为 Maven 依赖或 submodule。
8. **无业务 SQL 脚本**：只有 xxl-job 自带的建表 SQL，业务库表结构、初始化数据（数据字典、课程分类）全部缺失，新环境无法启动使用。
9. **`api-test` 里残留令牌与账号口令**：`http-client.env.json` 的 `access_token` 是空字符串（需手工登录后填入），但 `xc-auth-api.http` / `xc-gateway-api.http` 中直接粘贴了真实签发的 JWT（虽已过期）与测试账号明文口令；`.http` 里还有 `"price": -10` 这类随手测试数据。
10. **配置分散且重复**：`shared-configs` 中 `feign-${spring.profiles.active}.yaml` 在 `content-api` 里被重复声明了两次；`swagger` 配置在 `bootstrap.yml` 里既被注释掉又出现在 `system-api` 中，风格不统一。
11. **原 README 仅一行**：原 `README.md` 内容为「学成在线项目实战」，没有任何构建、配置或启动说明。
