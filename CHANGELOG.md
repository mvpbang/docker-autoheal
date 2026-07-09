# 变更记录

## Unreleased

- 将 `WEBHOOK_URL` 默认通知 payload 改为飞书交互式消息卡片。
- 新增 `WEBHOOK_TYPE` 环境变量，默认值为 `feishu_card`。
- 保留 `WEBHOOK_TYPE=text` 兼容旧版普通 JSON 文本 webhook，并继续支持 `WEBHOOK_JSON_KEY`。
- 新增 `AUTOHEAL_HOSTNAME`，告警默认带上主机名；未配置时优先从 Docker API `/info.Name` 获取，失败时回退到 `hostname`。
- Dockerfile 新增 `APK_MIRROR` 构建参数，默认使用阿里云 Alpine 源加速 `apk` 安装。
- 使用 `jq` 生成 webhook JSON，避免容器名或消息内容中的特殊字符破坏 payload。
- README 改为中文说明，并补充项目结构、飞书通知配置和环境变量说明。
