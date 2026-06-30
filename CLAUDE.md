# CLAUDE.md - TripTracer 项目开发规范与指令白皮书

## 1. 项目概况与愿景
* **项目名称：** TripTracer
* **核心定位：** 家庭旅行管理平台
* **架构模式：** 彻底解耦的 **Monorepo (单体仓库)** 模式，采用**分布式混部设计**。
* **跨平台流水线：** 开发者在 Ubuntu 24.04 笔记本上进行逻辑开发，最终目标部署并测试于本地 Windows (i7-12700 + 4070Super) 宿主机的 Docker/K8s (WSL2) 环境中。

## 2. 核心业务需求
1. **协同行程规划（Pre-trip）：** 支持多人并发编辑、管理出游与预算计划。通过 Redis 分布式锁/乐观锁防冲突，并利用文本 Diff 算法实现所有修改动作的**“全留痕审计”**。
2. **动态地理建议（In-trip）：** 移动端/H5 根据当前 GPS 坐标，利用 Redis Geo/Geopy 算力检索 2 公里内攻略提及的点位，结合当前时间结合云端 LLM 给出下一步动态行动建议。
3. **多媒体智能归档（Post-trip）：** 支持批量上传照片/视频至私有对象存储。在本地利用轻量化算法进行图像哈希（pHash/aHash）或 Embedding 余弦相似度聚类，自动去重/折叠相似照片。
4. **智能记账与对账（Post-trip）：** 支持上传支付宝等账单截图，本地触发异步 Worker 调度 OCR（PaddleOCR）提取结构化数据，自动生成旅行花销报告并与预算对比。

## 3. 完整技术栈架构

### 前端 (Frontend)
* **框架：** React (使用单页面 H5 响应式布局，适配未来移动端/APP 转化)
* **组件库 & 样式：** TailwindCSS + Ant Design Mobile (或类似轻量手机端组件库)
* **托管：** Nginx (作为反向代理与静态资源服务器，开启 Gzip 与 HTTPS)

### 核心后端 (Backend)
* **框架：** Python 3.11 + FastAPI (异步多实例设计)
* **任务调度：** Celery 或 Arq (基于 Redis 队列的异步任务架构)

### 算法大脑 (Algo)
* **运行环境：** 独立 Python 容器（初期采用纯 CPU 模式抹平异构硬件差异，预留 NVIDIA Container Toolkit GPU 运行时接口）
* **基础算法：** `difflib` (协同留痕), `pHash/aHash` 或 `scikit-learn/K-Means` (照片去重聚类)
* **AI 核心：** 本地轻量化 PaddleOCR 容器 + 云端 LLM API (用于动态地理推荐)

### 基础设施与数据层 (Infra & Data)
* **业务数据库：** MySQL 8.0
* **文本和日志数据库： ** Elasticsearch
* **缓存与空间引擎：** Redis 7.0 (承担分布式锁、地理位置 Geo 计算、异步任务队列)
* **多媒体存储：** MinIO (本地私有化部署，100% 兼容 AWS S3 协议)

---

## 4. 目录结构约定 (Monorepo)
Agent 必须严格按照以下目录结构创建和修改代码，严禁跨目录乱放：
```text
TripTracer/                     # 仓库根目录
├── .gitignore               # 分布式忽略配置
├── docker-compose.yml       # 本地开发一键拉起
├── CLAUDE.md                # 本规范
├── frontend/                # React 前端
│   ├── Dockerfile
│   └── ...
├── backend/            # FastAPI 核心业务后端
│   ├── Dockerfile
│   └── app/
└── algo/              # 算法异步 Worker
    ├── Dockerfile
    └── src/
```

## 开发版本：
### v0.0.1
初版本，先创建项目框架，不需要实施具体实现，确定前端、后端、MYSQL、Elasticsearch、Redis、MinIO 的镜像，并补充到本文档。在 `docker-compose.yml` 中，后续优先拉起 `MYSQL`、`Elasticsearch`、`Redis`、`MinIO` 这些中间件镜像，并将对应的端口与访问方式固化。

#### v0.0.1 文档交付边界
* 当前轮次已完成目录骨架、占位 `Dockerfile`、`.env.example` 与 `docker-compose.yml` 基线编排。
* 默认仍不自动执行镜像下载、不自动启动容器，由开发者按需在本地手动操作。
* 前端、后端、算法容器在 `v0.0.1` 仍只保留基础镜像与占位启动命令，不实现业务代码。

#### v0.0.1 镜像选型基线

| 模块 | 推荐镜像 | 用途说明 | 备注 |
| --- | --- | --- | --- |
| Frontend 构建环境 | `node:20-alpine` | React 前端依赖安装与构建 | 用于前端镜像构建阶段 |
| Frontend 运行环境 | `nginx:1.27-alpine` | 托管前端静态文件并承担反向代理入口 | 生产运行镜像 |
| Backend | `python:3.11-slim` | FastAPI 核心业务服务 | 兼顾体积与 Python 生态兼容性 |
| Algo Worker | `python:3.11-slim` | 算法容器、OCR/聚类/异步任务执行 | 后续按需要叠加系统依赖 |
| MySQL | `mysql:8.0` | 核心业务库 | 开发环境建议固定为 8.0 系列 |
| Elasticsearch | `docker.elastic.co/elasticsearch/elasticsearch:8.13.4` | 文本检索、日志与审计索引 | 开发环境采用单节点模式 |
| Redis | `redis:7.2-alpine` | 缓存、分布式锁、Geo、队列 | 轻量、启动快 |
| MinIO | `minio/minio:RELEASE.2024-05-28T17-19-04Z` | 私有对象存储，兼容 S3 | 提供对象 API 与管理控制台 |

#### v0.0.1 端口与访问方式约定

| 模块 | 容器端口 | 规划宿主机端口 | 访问方式 | 用途说明 |
| --- | --- | --- | --- | --- |
| Frontend | `80` | `8080` | `http://127.0.0.1:8080` | 未来 H5 页面访问入口 |
| Backend | `8000` | `8000` | `http://127.0.0.1:8000` | 未来 FastAPI API 入口 |
| MySQL | `3306` | `3306` | `mysql -h 127.0.0.1 -P 3306 -u <user> -p` | 业务数据库连接 |
| Elasticsearch HTTP | `9200` | `9200` | `http://127.0.0.1:9200` | REST API / 健康检查 |
| Elasticsearch Transport | `9300` | `9300` | 集群内部通信 | 单节点开发环境通常无需手动访问 |
| Redis | `6379` | `6379` | `redis-cli -h 127.0.0.1 -p 6379` | 缓存与队列访问 |
| MinIO API | `9000` | `9000` | `http://127.0.0.1:9000` | S3 兼容对象存储入口 |
| MinIO Console | `9001` | `9001` | `http://127.0.0.1:9001` | MinIO Web 管理控制台 |

#### v0.0.1 中间件初始化约定
* `MySQL`
  * 默认数据库名建议为：`triptracer`
  * 默认用户名建议为：`triptracer`
  * 密码通过环境变量注入，例如 `TRIPTRACER_MYSQL_PASSWORD`
  * `docker-compose.yml` 已为开发环境提供默认回退值；如需自定义，优先在根目录创建 `.env`
* `Elasticsearch`
  * 开发环境建议先采用单节点模式：`discovery.type=single-node`
  * `v0.0.1` 为便于本地联调，可先关闭复杂集群配置
* `Redis`
  * `v0.0.1` 可先使用默认单实例，无哨兵、无集群
* `MinIO`
  * Root 用户名与密码通过环境变量注入，例如 `MINIO_ROOT_USER`、`MINIO_ROOT_PASSWORD`
  * 默认创建业务 Bucket 的动作可延后到下一轮自动化脚本阶段

#### v0.0.1 本地准备建议
* 可参考根目录 `.env.example` 复制生成 `.env`，按本机需求覆盖数据库、Elasticsearch、MinIO 密码
* 可直接执行 `docker compose pull` 预下载 `MySQL`、`Elasticsearch`、`Redis`、`MinIO` 基础镜像
* 预下载镜像只是在本地缓存镜像层，不会自动创建或启动容器

#### v0.0.1 目录与容器职责预留
* `frontend/`
  * 预留 React 项目目录，后续在该目录中编写前端源码与 `Dockerfile`
* `backend/`
  * 预留 FastAPI 项目目录，后续在 `backend/app/` 中编写业务服务
* `algo/`
  * 预留算法与异步任务目录，后续在 `algo/src/` 中编写 OCR、相似图片处理与推荐逻辑

#### v0.0.1 下一步落地项
* 已创建 Monorepo 目录骨架：`frontend/`、`backend/app/`、`algo/src/`
* 已新增 `docker-compose.yml`，当前优先编排 `MySQL`、`Elasticsearch`、`Redis`、`MinIO`
* 已为前端、后端、算法模块补齐占位 `Dockerfile`
* 开发者可在本地根据 `docker-compose.yml` 手动执行 `docker compose pull` 预下载基础镜像，为后续联调做准备
