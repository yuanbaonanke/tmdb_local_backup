# TMDB 本地镜像

**无需代理 · 国内直连 · 告别限流被墙**  
自建 TMDB 数据镜像服务。预爬影视元数据到本地 SQLite，提供与 TMDB v3 完全兼容的 API + 图片代理。

- 刮削器直接改 API 地址即可用，零代码改动
- 全量预爬 movie 122 万 + tv 22 万，中文优先英文回退
- 图片实时代理，支持全部 TMDB 尺寸
- Docker 单容器，开箱即用

## 快速开始

```bash
# 1. 导入镜像
docker load -i tmdb-backup.tar
# 2. 修改 config.yaml 中 auth.password
# 3. 启动
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

## 数据量与热度档位

`min_popularity` 控制抓取范围，值越大数据越精（热门），越小越全。TMDB popularity 呈长尾分布，绝大多数条目集中在低分区域。

| min_popularity | movie | tv | 总计 | 约DB大小 | 说明 |
|---|---|---|---|---|---|
| 0 | 122万 | 22.6万 | ~145万 | ~15GB | 全量 |
| 1 | ~40万 | ~8万 | ~48万 | ~5GB | 略去极端冷门 |
| 3 | ~20万 | ~5万 | ~25万 | ~2.5GB | 常规冷门以上 |
| 5 | ~12万 | ~3万 | ~15万 | ~1.5GB | **推荐**，主流影视均覆盖 |
| 10 | ~5万 | ~1.5万 | ~6.5万 | ~700MB | 仅热门 |
| 20 | ~1.5万 | ~5000 | ~2万 | ~250MB | 仅最热 |

> 以上为典型值，实际根据 TMDB 数据日更会有浮动。0 档包含所有条目包括无海报/无简介的冷门数据。

## 预爬数据

```bash
# 后台全量预爬 movie+tv
docker compose exec tmdb-backup /app/tmdb-backup -mode crawl -kinds movie,tv
```

支持随时中断重启续爬。

### 抓取速率

速率取决于网络到 TMDB API 的延迟和 `concurrency`/`min_interval_ms` 配置。

| 并发 | 间隔 | 速率 | 全量(145万) | pop≥5(15万) |
|---|---|---|---|---|
| 8 | 60ms | ~5/s | ~3天 | ~8小时 |
| 16 | 25ms | ~8/s | ~2天 | ~5小时 |
| 24 | 15ms | ~12/s | ~1.5天 | ~3小时 |

> 以上为 `api.tmdb.org` 直连实测。TMDB 官方限流约 50 req/s，保持 `min_interval_ms≥15`（~66/s 上限）即安全。

```bash
# 查看实时进度
curl -c /tmp/c.txt -d "username=admin&password=xxx" http://localhost:18898/login -o /dev/null
curl -b /tmp/c.txt http://localhost:18898/api/stats
```

## 配置

`config.yaml`，主要配置项：

| 配置项 | 说明 |
|---|---|
| `api_base` | TMDB API 源 |
| `auth.username` / `auth.password` | Web 管理员登录 |
| `crawler.concurrency` | 爬虫并发 |
| `crawler.min_popularity` | 最低热度(0=全量) |

## 版本

| 版本 | 功能 | 获取方式 |
|---|---|---|
| 个人版 | 管理员账号、API 调用、海报墙、数据浏览，**永久免费无限使用** | 开箱即用 |
| 旗舰版 | + 创建用户 + 卡密注册 + 订阅管理 + 流量配额 | 捐赠获取许可证 |

### 旗舰版功能详情


**多用户体系**
- 管理员可创建多个用户，各自独立 API 密钥
- 用户凭密钥调用 API，密钥可随时重置/删除

**卡密自助注册**
- 管理员批量生成卡密（可设自定义天数或流量额度）
- 用户访问 `/register` 输入卡密即完成注册，自动激活订阅并获取 API 密钥

**订阅管理（时间/流量双模式）**

| 套餐 | 说明 |
|---|---|
| 月卡 / 季卡 / 半年卡 / 年卡 / 五年卡 | 固定天数，到期自动失效 |
| 自定义天数 | 管理员填任意天数，灵活设置 |
| 流量套餐 | 设置 API 调用次数上限，每次请求消耗 1 次，耗尽即停 |
| 续期叠加 | 同一用户可多次创建订阅，自动生效最新的有效订阅 |

**流量控制**
- 流量模式：设置 `quota`（如 10000 次），每次 API 请求消耗 1 次配额
- 配额耗尽后返回 401，用户需续购
- 支持同时拥有时间+流量订阅，流量优先消耗
- 管理后台实时查看每个用户的配额使用进度（如 3,281 / 10,000）

**管理后台**
- 用户列表、订阅创建/取消、密钥管理
- 卡密生成记录（含状态/使用者/使用时间）
- 实时配额消耗监控

> **捐赠获取旗舰版许可证**：请通过以下方式联系，提供服务器机器码即可签发专属许可证密钥。
> - 邮箱：925181769@qq.com
> - 微信：无
> - 将 `license_key` 填入 `config.yaml` 后重启即生效
