# TMDB 本地镜像

自建 TMDB 数据镜像服务。预爬影视元数据到本地 SQLite，提供与 TMDB v3 完全兼容的 API + 图片代理，彻底解决 TMDB 限流/被墙问题。

- 刮削器直接改 API 地址即可用，零代码改动
- 全量预爬 movie 122 万 + tv 22 万，中文优先英文回退
- 图片实时代理，支持全部 TMDB 尺寸
- Docker 单容器，开箱即用

## 快速开始

```bash
# 1. 修改 config.yaml 中 auth.password
# 2. 启动
docker compose up -d
```

访问 http://localhost:18898/ 登录（admin / 你设的密码）

## 刮削器接入

把刮削器的 TMDB API 地址指向本服务：

```
API Base:  http://<你的IP>:18898/3
Image URL: http://<你的IP>:18898/t/p
API Key:   登录管理后台查看你的密钥
```

## API 端点

| 端点 | 说明 |
|---|---|
| `GET /3/movie/{id}` | 电影详情(含 credits/images/videos) |
| `GET /3/tv/{id}` | 剧集详情(含 seasons/credits) |
| `GET /3/tv/{id}/season/{n}` | 分季逐集(标题/简介/剧照) |
| `GET /3/person/{id}` | 人物详情(含参演作品) |
| `GET /3/search/{movie\|tv\|person}` | 搜索 |
| `GET /3/find/{imdb_id}` | IMDb 反查 |
| `GET /t/p/{size}{path}` | 图片代理 |

## 预爬数据

```bash
# 后台全量预爬 movie+tv
docker compose exec tmdb-backup /app/tmdb-backup -mode crawl -kinds movie,tv
```

数据量参考：movie ~122 万 + tv ~22 万，全量约 15GB，数天完成。支持随时中断重启续爬。

## 配置

`config.yaml`，主要配置项：

| 配置项 | 说明 |
|---|---|
| `api_base` | TMDB API 源 |
| `auth.username` / `auth.password` | Web 管理员登录 |
| `crawler.concurrency` | 爬虫并发 |
| `crawler.min_popularity` | 最低热度(0=全量) |

## 技术栈

Go + SQLite(FTS5) + bcrypt + HMAC，单二进制，Alpine 镜像约 8MB。
