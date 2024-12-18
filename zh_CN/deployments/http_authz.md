# HTTP 授权

EMQX 支持基于 HTTP 应用进行授权。此时，用户需在外部自行搭建一个 HTTP 应用作为数据源，EMQX 将向 HTTP 服务发起请求并根据 HTTP API 返回的数据判定授权结果，从而实现复杂的授权逻辑。

## HTTP 授权原理

授权过程类似一个 HTTP API 调用，EMQX 作为请求客户端需要按照 "API" 要求的格式构造并向 HTTP 服务发起请求，而 HTTP 服务需要按照 "客户端" 的要求返回结果：

- 响应编码格式 `content-type` 必须是 `application/json`。
- 授权结果通过 body 中的 `result` 标示，可选 `allow`、`deny`、`ignore`。
- 如果返回的 HTTP 状态码为 `204`，认证结果标示为允许发布或订阅。
- 除了 `200` 和 `204` 以外的其他 HTTP 状态码均标示为 ignore，比如 HTTP 服务不可用。

响应示例：
```json
HTTP/1.1 200 OK
Headers: Content-Type: application/json
...
Body:
{
    "result": "allow" | "deny" | "ignore" // Default `"ignore"`
}
```


## 配置 HTTP 授权

在部署中点击 **访问控制** -> **授权** -> **扩展授权**，选择 **HTTP 授权**，点击**配置授权**。


进行身份授权时，EMQX Platform 将使用当前客户端信息填充并发起用户配置的授权查询请求，查询出该客户端在 HTTP 服务器端的授权数据。

您可根据如下说明完成相关配置：


- **请求方式**：选择 HTTP 请求方式，可选值： `POST` 或 `GET`。
  ::: tip
  推荐使用 `POST` 方法。 使用 `GET` 方法时，一些敏感信息（如纯文本密码）可能通过 HTTP 服务器日志记录暴露。此外，对于不受信任的环境，请使用 HTTPS。
  :::

- **URL**：输入 HTTP 服务的 URL 地址。

  - URL 地址必须以 `http://` 或 `https://` 开头。

  - 避免在域名中使用占位符。

  - 您可以在 URL 路径中使用以下占位符：

    - `${clientid}`

    - `${username}`

    - `${password}`

    - `${peerhost}`

    - `${cert_subject}`

    - `${cert_common_name}`

- **请求头**（可选）：HTTP 请求头配置。可以添加多个请求头。
  连接配置：在此部分进行并发连接、连接超时等待时间、最大 HTTP 请求数以及请求超时时间。

- **启用 TLS**：配置是否启用 TLS。

- **连接池大小**（可选）：整数，指定从 EMQX 节点到外部 HTTP Server 的并发连接数；默认值：`8`。

- **连接超时**（可选）：填入连接超时等待时长，可选单位：小时、分钟、秒、毫秒。

- **HTTP 管道**（可选）：正整数，指定无需等待响应可发出的最大 HTTP 请求数；默认值：`100`。

- **请求超时**（可选）：填入连接超时等待时长，可选单位：小时、分钟、秒、毫秒。

- **请求体**：请求模板，对于 `POST` 请求，它以 JSON 形式在请求体中发送。对于 `GET` 请求，它被编码为 URL 中的查询参数（Query String）。映射键和值可以使用占位符。


::: tip
* 如果当前部署为专有版，需创建 [VPC 对等连接](../deployments/vpc_peering.md)，服务器地址填写内网地址。
* 如果当前部署为 BYOC 版，需在您的公有云控制台中创建 VPC 对等连接，具体请参考 [创建 BYOC 部署 - VPC 对等连接配置](../create/byoc.md#vpc-对等连接配置) 章节。服务器地址填写内网地址。
* 若提示 Init resource failure! 请检查服务器地址是否无误、安全组是否开启。
:::


### 请求格式与返回结果
当客户端发起订阅、发布操作时，HTTP 授权器会根据配置的请求模板构造并发送请求到外部 Web 服务（授权服务）。用户需要在请求模版中实现授权检查逻辑并确保授权检查结果能按要求返回，EMQX 根据响应结果判断是否具备权限。

#### 请求格式

根据授权服务要求而定，可以使用 JSON 格式，支持在 URL 与请求体中使用以下占位符：

- `${clientid}`: 客户端的 ID
- `${username}`: 客户端登录时用的用户名
- `${peerhost}`: 客户端的源 IP 地址
- `${proto_name}`: 客户端使用的协议名称。例如 `MQTT`，`CoAP` 等
- `${mountpoint}`: 网关监听器的挂载点（主题前缀）
- `${action}`: 当前执行的动作请求，例如 `publish`，`subscribe`
- `${topic}`: 当前请求想要发布或订阅的主题（或主题过滤器）
- `${qos}`: 当前请求想要发布或订阅的消息 QoS
- `${retain}`: 当前请求想要发布的消息是否为保留消息

#### 响应格式
授权服务完成检查后，返回符合以下要求的响应：
- 响应编码格式 `content-type` 必须是 `application/json`。
- 如果返回的 HTTP 状态码为 `200`，授权结果通过 Body 中的 `result` 标示，可选值为：
    - allow：允许发布或订阅
    - deny：禁止发布或订阅
    - ignore： 忽略请求，移交下一个授权器以继续执行授权链
- 如果返回的 HTTP 状态码为 204，授权结果标示为允许发布或订阅。
- 除了 200 和 204 以外的其他 HTTP 状态码均标示为 ignore，比如 HTTP 服务不可用。
响应示例：

```
HTTP/1.1 200 OK
Headers: Content-Type: application/json
...
Body:
{
    "result": "allow" | "deny" | "ignore" // Default `"ignore"`
}
```

::: tip
推荐使用 `POST` 方法。 使用 `GET` 方法时，一些敏感信息可能通过 HTTP 服务器日志记录暴露。 对于不受信任的环境，应使用 HTTPS。
:::