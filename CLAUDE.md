# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

佳明运动数据同步工具 (Garmin Daily Sync) — 在佳明中国区 (garmin.cn) 和佳明国际区 (garmin.com) 之间双向同步/迁移运动数据。额外支持 RQ (RunningQuotient) 跑力数据采集和 Google Sheets 记录。

## Commands

```shell
# 安装依赖
yarn

# 同步：国际区 → 中国区
yarn sync_global

# 同步：中国区 → 国际区
yarn sync_cn

# 迁移历史数据：国际区 → 中国区
yarn migrate_garmin_global_to_cn

# 迁移历史数据：中国区 → 国际区
yarn migrate_garmin_cn_to_global

# RQ 跑力数据采集
yarn rq

# 测试
yarn test

# Docker 打包运行
docker-compose up
```

没有 lint 或格式化工具。没有测试框架（`yarn test` 只运行 `src/test.ts`）。

## Architecture

### Entry Points (`src/`)

每个入口文件对应一个独立功能，都是 try/catch 包裹 + Bark 推送失败通知的模式：

| 文件 | 功能 |
|------|------|
| `sync_garmin_cn_to_global.ts` | 增量同步 CN → Global |
| `sync_garmin_global_to_cn.ts` | 增量同步 Global → CN |
| `migrate_garmin_cn_to_global.ts` | 批量迁移 CN → Global |
| `migrate_garmin_global_to_cn.ts` | 批量迁移 Global → CN |
| `rq.ts` | RQ 跑力数据采集 → Google Sheets |

### Core Modules (`src/utils/`)

- **garmin_cn.ts / garmin_global.ts** — 分别封装中国区/国际区的 GarminConnect 客户端初始化（含 session 缓存和重登录）、同步逻辑、迁移逻辑
- **garmin_common.ts** — 通用操作：`downloadGarminActivity`（下载 .fit 并解压）、`uploadGarminActivity`（上传 .fit）、`getGarminStatistics`（提取跑步数据）
- **sqlite.ts** — SQLite session 持久化，AES 加密存储 Garmin OAuth token，避免每次重新登录
- **type.ts** — `GarminClientType` 类型定义
- **number_tricks.ts** — 数字转中文数字，用于绕过 GitHub Actions 日志中 Secrets 被 `***` 屏蔽的问题
- **runningquotient.ts** — RQ 网页数据抓取（HTML 解析），获取跑力/疲劳/训练负荷
- **google_sheets.ts** — Google Sheets API 读写（JWT 认证）
- **strava.ts** — Strava 集成（当前已废弃，通过 Garmin Global 关联 Strava 实现）

### Sync vs Migration

- **Sync（增量同步）**：比对两侧最新活动的 `startTimeLocal`，只同步新活动（每次最多检查 `GARMIN_SYNC_NUM=10` 条）
- **Migration（批量迁移）**：按 `GARMIN_MIGRATE_START` 和 `GARMIN_MIGRATE_NUM` 配置批量迁移历史活动

### Data Flow

```
Garmin CN API ←→ @gooin/garmin-connect ←→ 本地 .fit 文件 ←→ @gooin/garmin-connect ←→ Garmin Global API
                                              ↓
                                        SQLite (session cache, AES encrypted)
```

## Configuration

所有配置通过环境变量注入（GitHub Actions Secrets 或 `.env` 文件），回退到 `src/constant.ts` 中的默认值。

关键环境变量：
- `GARMIN_USERNAME` / `GARMIN_PASSWORD` — 中国区账号
- `GARMIN_GLOBAL_USERNAME` / `GARMIN_GLOBAL_PASSWORD` — 国际区账号
- `GARMIN_MIGRATE_NUM` — 每次迁移数量（默认 100）
- `GARMIN_MIGRATE_START` — 从第几条开始迁移
- `BARK_KEY` — iOS Bark 推送通知 key
- `RQ_*` — RunningQuotient 相关配置
- `GOOGLE_*` / `STRAVA_*` — 第三方集成

## Key Dependencies

- `@gooin/garmin-connect` — Garmin Connect API 封装（fork 版本，支持中国区）
- `sqlite` / `sqlite3` — session 持久化
- `crypto-js` — session AES 加密
- `@actions/core` — GitHub Actions 集成
- `googleapis` / `google-auth-library` — Google Sheets
- `axios` / `lodash` — HTTP 请求和工具函数

## Runtime Requirements

- Node.js >= 18
- 需要能访问国际互联网（访问 sso.garmin.com）
- GitHub Actions 定时运行（每 6 小时），或本地 cron / Docker
