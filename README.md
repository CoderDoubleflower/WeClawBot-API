# WeClawBot-API

基于 `微信ClawBot` (iLink) 实现的个人微信消息推送 API 服务。

**它体积极小(≈10MB)，内存占用极低(≈10MB)，极易部署(Docker/二进制)，极易调用(HTTP API)。**

无额外依赖，独立运行，方便供其他程序请求，发送微信通知。

![WeClawBot-API](img/main.png)

> [!TIP]
> 没错！个人微信消息推送它来了！~~再也不用~~折腾企业微信/第三方服务号了！
>
> **当前灰度中，可用性待观察*

## 功能特性

- **多账号支持**：支持同时登录多个微信号，每个账号独立推送、互不干扰
- **扫码登录**：控制台直接打印二维码，微信扫码即可完成授权
- **持久化存储**：登录凭证（Token、游标等）自动保存，重启后自动重连
- **命令行交互**：内置简易控制台，可直接在终端中收发微信消息
- **HTTP API**：提供标准 RESTful 接口，支持通过 API 发送文本消息及设置“正在输入”状态
- **Bark 兼容接口**：支持 Bark 格式的 GET / POST 调用，将通知转发到对应账号的微信

## 部署

### Docker Compose (推荐)

创建 `docker-compose.yml` 文件：

```yaml
name: weclawbot-api
services:
  weclawbot-api:
    image: cp0204/weclawbot-api:latest
    container_name: weclawbot-api
    ports:
      - "26322:26322"
    volumes:
      - ./config:/app/config
    restart: unless-stopped
```

### Docker Run

```bash
docker run -d \
  --name weclawbot-api \
  -p 26322:26322 \
  -v ./config:/app/config \
  --restart unless-stopped \
  cp0204/weclawbot-api:latest
```

### 初次扫码登录

容器启动后，进入容器终端(sh)，输入 `bot` 扫码登录，授权后的信息将保存在挂载的 `config` 目录下。

NAS 部署可通过 WebUI 进入容器终端，也可以在宿主机上通过以下命令进入：

```bash
# 宿主机进入容器终端
docker exec -it weclawbot-api bot
```

> [!TIP]
> 授权成功后，在微信发一条消息给`微信ClawBot`，否则无法激活 API 发信。

> [!WARNING]
> 请妥善保管 `config/auth.json` 文件及 `api_token`，不要泄露。

## 常用命令

- `/login` : 发起新的扫码登录流程
- `/bots`  : 列出当前所有已登录 `bot_id` 及其 `api_token`
- `/bot <序号>` : 切换当前活跃发送身份
- `/del <序号>` : 删除指定索引的 Bot 账号配置

## API 文档

> [!TIP]
> 新手省流版，发消息直接替换参数访问：
> ```
> http://192.168.8.8:26322/bots/{bot_id}/messages?token={api_token}&text=Hello
> ```

API 支持 `GET` 和 `POST` 请求，兼容以下多种提交方式：

- GET
- POST
  - `application/json`
  - `application/x-www-form-urlencoded`
  - `multipart/form-data`

### 身份验证

所有接口均需验证 `api_token`（可通过 `/bots` 命令或查看 `config/auth.json` 获取）。Bark 接口使用它作为 `key` / `device_key`，原有 `/bots/` 接口的传递方式如下。

你可以通过以下任一方式传递 Token：

- **Header**: `Authorization: Bearer <api_token>`
- **Body/Query**: `token=<api_token>`


### 发送消息

**Endpoint:** `/bots/{bot_id}/messages`

**参数:**
- `text`: 消息文本内容。

**示例:**
```bash
# GET
curl http://192.168.8.8:26322/bots/{xxx@im.bot}/messages?token={api_token}&text=Hello
```
```bash
# POST
curl -X POST http://192.168.8.8:26322/bots/{xxx@im.bot}/messages \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer {api_token}" \
  -d '{
    "text": "Hello, this is a POST request!"
  }'
```

### 发送输入状态

**Endpoint:** `/bots/{bot_id}/typing`

**参数:**
- `status`: `1`=正在输入，`2`=停止输入

**示例:**
```bash
# GET
curl http://192.168.8.8:26322/bots/{xxx@im.bot}/typing?token={api_token}&status=1
```
```bash
# POST
curl -X POST http://192.168.8.8:26322/bots/{xxx@im.bot}/typing \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer {api_token}" \
  -d '{
    "status": 1
  }'
```

### Bark 兼容调用

兼容 [Bark 官方文档](https://github.com/Finb/Bark/blob/master/docs/en-us/tutorial.md)中的常用文本推送格式。将 Bark 服务地址改为本服务地址，设备 Key 填写目标微信账号的 `api_token`，无需另传 `bot_id`。

支持以下路径（均可使用 GET / POST）：

```text
/{key}?body=正文&title=标题
/{key}/{body}
/{key}/{title}/{body}
/{key}/{title}/{subtitle}/{body}
/push
```

POST 支持 JSON、URL 编码表单和 multipart 表单；GET 支持 Query 参数。`/push` 通过 `device_key` 指定账号，其余路径使用 `{key}`。路径中的标题、副标题、正文优先于提交的同名参数，路径内容请进行 URL 编码（尤其是 `/`、`?`、`#` 和换行）。

| 参数 | 说明 |
| --- | --- |
| `device_key` | `/push` 的必填参数，值为账号的 `api_token` |
| `title` | 标题 |
| `subtitle` | 副标题 |
| `body` | 正文；标题、副标题、正文至少提供一项 |
| `url` | 可选链接，追加到文本末尾 |

非空的标题、副标题、正文和链接以换行拼接为一条微信消息。`sound`、`icon`、`group`、`badge` 等客户端通知选项会被忽略；不支持 `device_keys` 批量推送、`ciphertext` 加密推送或 Bark App 的设备注册。单次请求体上限为 1 MiB。

仍需先在微信向 ClawBot 发送一条消息以激活发信上下文。

```bash
# GET 路径格式
curl 'http://192.168.8.8:26322/{api_token}/Hello'

# GET Query 格式，自动编码参数
curl -G 'http://192.168.8.8:26322/{api_token}' \
  --data-urlencode 'title=任务完成' \
  --data-urlencode 'body=备份已完成'

# POST JSON
curl 'http://192.168.8.8:26322/push' \
  -H 'Content-Type: application/json' \
  -d '{"device_key":"{api_token}","title":"任务完成","body":"备份已完成","url":"https://example.com"}'

# POST 表单
curl 'http://192.168.8.8:26322/{api_token}' \
  --data-urlencode 'title=任务完成' \
  --data-urlencode 'body=备份已完成'
```

Bark 接口返回 `code`、`message`、`timestamp`（Unix 秒），HTTP 状态码与 `code` 一致：

```json
{"code":200,"message":"success","timestamp":1770000000}
```

参数错误或上下文未就绪返回 400，Key 无效返回 401，不支持的请求方法返回 405，微信发送失败返回 500。

### 原有接口响应格式

`/bots/` 接口返回以下 JSON 结构：

**成功响应 (200 OK):**
```json
{
  "code": 200,
  "message": "OK"
}
```

**错误响应 (4xx/5xx):**
```json
{
  "code": 401,
  "error": "Unauthorized"
}
```

## 致谢

- 微信官方插件 [@tencent-weixin/openclaw-weixin](https://www.npmjs.com/package/@tencent-weixin/openclaw-weixin)
