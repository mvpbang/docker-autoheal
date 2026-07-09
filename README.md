# Docker Autoheal

监控 Docker 容器健康状态，并在容器进入 `unhealthy` 后自动重启。该镜像通过 Docker API 查询容器健康状态，不需要在业务容器内安装额外组件。

## 项目结构

```text
.
├── Dockerfile                 # autoheal 镜像构建文件
├── docker-entrypoint          # 主入口脚本，负责监控、重启与通知
├── README.md                  # 项目说明
├── CHANGELOG.md               # 变更记录
├── LICENSE                    # 开源许可证
└── tests/                     # 本地测试与示例编排文件
    ├── docker-compose.yml
    ├── docker-compose.autoheal.yml
    ├── README.md
    ├── tests.sh
    └── watch-autoheal/
```

## 使用方式

### Docker CLI

通过 Unix Socket 监控本机 Docker：

```bash
docker run -d \
  --name autoheal \
  --restart=always \
  -e AUTOHEAL_CONTAINER_LABEL=all \
  -v /var/run/docker.sock:/var/run/docker.sock \
  willfarrell/autoheal
```

通过 TCP Socket 监控远端 Docker：

```bash
docker run -d \
  --name autoheal \
  --restart=always \
  -e AUTOHEAL_CONTAINER_LABEL=all \
  -e DOCKER_SOCK=tcp://$HOST:$PORT \
  -v /path/to/certs/:/certs/:ro \
  willfarrell/autoheal
```

通过 TCP mTLS 监控远端 Docker：

```bash
docker run -d \
  --name autoheal \
  --restart=always \
  --tlscacert=/certs/ca.pem \
  --tlscert=/certs/client-cert.pem \
  --tlskey=/certs/client-key.pem \
  -e AUTOHEAL_CONTAINER_LABEL=all \
  -e DOCKER_HOST=tcp://$HOST:2376 \
  -e DOCKER_SOCK=tcps://$HOST:2376 \
  -e DOCKER_TLS_VERIFY=1 \
  -v /path/to/certs/:/certs/:ro \
  willfarrell/autoheal
```

证书文件需要挂载到容器内 `/certs`，文件名固定为：

- `ca.pem`
- `client-cert.pem`
- `client-key.pem`

### 业务容器标签

业务镜像需要先配置 `HEALTHCHECK`。autoheal 支持三种选择器：

- 给需要监控的容器添加 `autoheal=true` 标签。
- 设置 `AUTOHEAL_CONTAINER_LABEL=all` 监控全部容器。
- 设置 `AUTOHEAL_CONTAINER_LABEL` 为自定义标签名，且业务容器标签值为 `true`。

Docker Compose 示例：

```yaml
services:
  app:
    image: your-app:latest
    labels:
      autoheal-app: true

  autoheal:
    image: willfarrell/autoheal:latest
    environment:
      AUTOHEAL_CONTAINER_LABEL: autoheal-app
    network_mode: none
    restart: always
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - /var/run/docker.sock:/var/run/docker.sock
```

### 飞书消息卡片通知

配置 `WEBHOOK_URL` 后，容器重启成功或失败时会默认发送飞书交互卡片消息：

```bash
docker run -d \
  --name autoheal \
  --restart=always \
  -e AUTOHEAL_CONTAINER_LABEL=all \
  -e WEBHOOK_URL="https://open.feishu.cn/open-apis/bot/v2/hook/xxxx" \
  -v /var/run/docker.sock:/var/run/docker.sock \
  willfarrell/autoheal
```

默认 payload 类型为 `WEBHOOK_TYPE=feishu_card`，发送内容包含宿主主机名、容器名称、短 ID、处理结果、发生时间和重启结果说明。失败通知使用红色卡片头，成功通知使用绿色卡片头。

`AUTOHEAL_HOSTNAME` 未配置时会优先从 Docker API `/info.Name` 获取宿主主机名；获取失败时回退到容器内 `hostname` 命令结果。如需固定展示名称，可在部署时显式传入：

```bash
-e AUTOHEAL_HOSTNAME="$(hostname)"
```

如需兼容旧版普通 JSON 文本 webhook，可设置：

```bash
-e WEBHOOK_TYPE=text
-e WEBHOOK_JSON_KEY=content
```

此时 payload 格式为：

```json
{
  "content": "Host prod-node-01: Container app (abc123def456) found to be unhealthy. Successfully restarted the container!"
}
```

## 可选容器标签

| 标签 | 说明 |
| --- | --- |
| `autoheal.stop.timeout=20` | 单容器重启超时时间，单位秒 |

## 环境变量

| 变量 | 默认值 | 说明 |
| --- | --- | --- |
| `AUTOHEAL_CONTAINER_LABEL` | `autoheal` | 监控标签名，标签值需要为 `true`；设置为 `all` 时监控全部容器 |
| `AUTOHEAL_INTERVAL` | `5` | 健康状态检查间隔，单位秒 |
| `AUTOHEAL_START_PERIOD` | `0` | 首次检查前等待时间，单位秒 |
| `AUTOHEAL_DEFAULT_STOP_TIMEOUT` | `10` | Docker 重启容器时等待停止的默认超时时间 |
| `AUTOHEAL_ONLY_MONITOR_RUNNING` | `false` | 设置为 `true` 时仅监控运行中的容器 |
| `AUTOHEAL_HOSTNAME` | 自动取 Docker 宿主机名 | 告警中展示的宿主主机名；未配置时优先取 Docker API `/info.Name` |
| `DOCKER_SOCK` | `/var/run/docker.sock` | Docker API Socket 地址 |
| `CURL_TIMEOUT` | `30` | Docker API 请求超时时间，单位秒 |
| `WEBHOOK_URL` | 空 | 重启成功或失败时发送 webhook 通知 |
| `WEBHOOK_TYPE` | `feishu_card` | webhook payload 类型；`feishu_card` 为飞书消息卡片，`text` 为普通 JSON 文本 |
| `WEBHOOK_JSON_KEY` | `content` | `WEBHOOK_TYPE=text` 时的 JSON 字段名 |
| `APPRISE_URL` | 空 | Apprise 通知地址 |
| `POST_RESTART_SCRIPT` | 空 | 容器重启后异步执行的脚本 |

## 本地构建测试

```bash
docker buildx build -t autoheal .

docker run -d \
  -e AUTOHEAL_CONTAINER_LABEL=all \
  -v /var/run/docker.sock:/var/run/docker.sock \
  autoheal
```

构建镜像时默认使用阿里云 Alpine 源加速 `apk` 安装。如需切换镜像源，可传入：

```bash
docker buildx build \
  --build-arg APK_MIRROR=https://mirrors.tuna.tsinghua.edu.cn/alpine \
  -t autoheal .
```
