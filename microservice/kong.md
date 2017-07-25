# kong
开源的 API 管理及微服务管理，[官网][kong]。



## 名词解释

| **英文**                 | **中文**  | **说明** |
| ---------------------- | ------- | ------ |
| Server Name Indication | 服务器名称指示 |        |





## 特性

1.   RESTful Interface：基于 Nginx 构建，通过一个简单且易用的 RESTful API 完成所有操作。

2. Plugin Oriented：将自有的服务置于 Kong 的后端，通过 Kong 插件，添加强大的功能支持。

3. Platform Agnostic：可以在任何平台运行，云、内部部署或单个，混合甚至多数据中心设置。

4. Kong Architecure：

    ![2CC5041E-8BF7-4364-BAE7-82947FBB0703](images/2CC5041E-8BF7-4364-BAE7-82947FBB0703.png)

## 安装

- 基于 Docker

  ```shell
  # 1. 安装数据库（采用 Cassandra）
  $ docker run -d --name kong-database -p 9042:9042 cassandra:2.2

  # 2. 安装 Kong
  $ docker run -d --name kong \
  --link kong-database:kong-database \
  -e "KONG_DATABASE=cassandra" \
  -e "KONG_CASSANDRA_CONTACT_POINTS=kong-database" \
  -e "KONG_PG_HOST=kong-database" \
  -p 8000:8000 \
  -p 8443:8443 \
  -p 8001:8001 \
  -p 7946:7946 \
  -p 7946:7946/udp \
  kong
  ```

## 文档

### Admin API

Kong 自带**内部** Restful Admin API 用于管理。到 Admin API 的请求可以被发送到集群中得任何节点，Kong 会保证所有节点的配置文件保持一致。

- `8001` 默认的 Admin API 监听端口；
- `8444` 默认的 Admin API HTTPS 监听端口。


#### 支持的内容类型

- **x-www-form-urlencoded**：对于基本的请求体来说足够简单，多数时候都用这种格式。注意，发送嵌套值时，Kong 使用 '.<keys>' 形式表示嵌套对象。

  ```
  config.limit=10&config.period=seconds
  ```

- **application/json**：处理复杂内容（如：复杂的插件配置），发送与上述内容类型一样的内容，可以用下面的 JSON 表述数据：

  ```json
  {
    "config": {
      "limit": 10,
      "period": "seconds"
    }
  }
  ```


#### 状态信息

##### 获取节点信息

获取节点的通用详情。

**请求**

`GET /`

**响应**

```http
HTTP 200 OK
```

```json
{
  "timers": {
    // ...
  },
  "configuration": {
    // ...
  },
  "version": "0.10.0",
  "prng_seeds": {
    // ...
  },
  "lua_version": "LuaJIT 2.1.0-beta2",
  "tagline": "Welcome to kong",
  "hostname": "519aab2bf97a",
  "plugins": {
    "available_on_server": {
      // ...
    },
    "enabled_in_cluster": {
      // ...
    }
  }
}
```

| **名称**                        | **类型**   | **说明**                                   |
| ----------------------------- | -------- | ---------------------------------------- |
| `plugins`                     | *object* | 插件信息                                     |
| `plugins.available_on_server` | *object* | 节点上安装的插件信息，以 key-value 形式：`{"syslog": true, "ldap-auth": true}` 保存。 |
| `plugins.enabled_in_cluster`  | *object* | 集群中启用/配置了的插件。即，当前数据存储中被 Kong 节点 共享的插件配置。 |



##### 获取节点状态

获取节点的使用情况，包括一些关于被 nginx 进程处理的链接，及数据存储集合中得实体数（包括插件的集合）。

如果希望监控 Kong 进程，因为 Kong 构建于 nginx 之上，所以任何现存的 nginx 监控工具或者代理都能实现。



**请求**

`GET /status`

**响应**

```http
HTTP 200 OK
```

```json
{
  "server": {
    "connections_handled": 5,
    "connections_reading": 0,
    "connections_active": 1,
    "total_requests": 10,
    "connections_accepted": 5,
    "connections_writing": 1,
    "connections_waiting": 0
  },
  "database": {
    "oauth2_credentials": 0,
    "jwt_secrets": 0,
    "plugins": 0,
    "keyauth_credentials": 0,
    "oauth2_authorization_codes": 0,
    "ssl_servers_names": 0,
    "acls": 0,
    "basicauth_credentials": 0,
    "apis": 0,
    "hmacauth_credentials": 0,
    "consumers": 0,
    "oauth2_tokens": 0,
    "targets": 0,
    "nodes": 1,
    "upstreams": 0,
    "ssl_certificates": 0
  }
}
```

- `server`：HTTP/S 服务器的指标信息。
  - `total_request`：客户端请求的总数，每次请求 +1。
  - `connections_active`：当前活动的客户端连接数，包括等待中得连接。
  - `connections_accepted`：已接受的客户端连接数，只有重新连接的才算。
  - `connections_handled`：已处理的连接总数。通常，该值与 `connections_accepted` 一致，除非有限资源达到上限。
  - `connections_reading`：Kong 当前读取连接请求头的总数。
  - `connections_writing`：Kong 当前写入响应回客户端的连接总数。
  - `connections_wating`：当前闲置等待请求的客户端连接数。
- `database`：数据库集合的指标。
  - `...`：每个数据库集合，显式保存了的集合数量。


#### 集群操作（略）



#### API 对象

API 对象描述了 Kong 暴露的 API。Kong 需要了解当消费者从代理端口调用 API 时如何检索。每个 API 对象必须指定 `hosts`，`uris` 及 `methods` 的组合。Kong 会代理所有请求到 API 指定的上游 URL。一个 API 对象类似如下：

```json
{
  "created_at": 1488830759000,
  "hosts": [
    "example.org"
  ],
  "http_if_terminated": true,
  "https_only": false,
  "id": "6378122c-a0a1-438d-a5c6-efabae9fb969",
  "name": "example-api",
  "preserve_host": false,
  "retries": 5,
  "strip_uri": true,
  "upstream_connect_timeout": 60000,
  "upstream_read_timeout": 60000,
  "upstream_send_timeout": 60000,
  "upstream_url": "http://httpbin.org"
}
```



##### 添加 API

**请求**

`POST /apis/`

**参数**

|                     **属性** | **约束**        | **描述**                                   |
| -------------------------: | :------------ | ---------------------------------------- |
|                     `name` |               | API 的名称。                                 |
|                    `hosts` | 半可选(1/3)      | API 绑定的域名列表，以逗号分隔。比如：`example.com`。      |
|                     `uris` | 半可选(2/3)      | API 绑定的 URIs 前缀列表，以逗号分隔。比如：`/my-path`。   |
|                  `methods` | 半可选(3/3)      | API 接受的 HTTP 请求方式，以逗号分隔。比如：`GET,POST`。   |
|             `upstream_url` |               | API 绑定的基础目标地址，用于代理请求。比如：`https://example.com`。 |
|                `strip_uri` | 可选，默认：`true`  | 当通过 `uris` 前缀匹配到 API，请求上游地址时，省略匹配的前缀。    |
|            `preserve_host` | 可选，默认：`false` | 当通过 `hosts` 域名匹配到 API，确保请求的 `Host` 头被传递到上游服务。未启用时，上游 `Host` 头来自配置的 `upstream_url`。 |
|                  `retries` | 可选，默认：`5`     | 代理失败后的重试次数。                              |
| `upstream_connect_timeout` | 可选，默认：`60000` | 连接上游服务的超时时间，单位：毫秒。                       |
|    `upstream_send_timeout` | 可选，默认：`60000` | 用于向上游服务发送请求的两个连续写入操作之间的超时时间。单位：毫秒。       |
|    `upstream_read_timeout` | 可选，默认：`60000` | 用于向上游服务发送请求的两个连续读取操作之间的超时时间。单位：毫秒。       |
|               `https_only` | 可选，默认：`false` | 是否仅允许 HTTPS 访问 API（默认端口：`8443`）。         |
|       `http_if_terminated` | 可选，默认：`true`  | 考虑 `X-Forwarded-Proto` 头当强制 HTTPS 访问时。   |



**响应**

```http
HTTP 201 Created
```

```
{
  "http_if_terminated": true,
  "strip_uri": true,
  "retries": 5,
  "preserve_host": false,
  "created_at": 1491966352536,
  "upstream_connect_timeout": 60000,
  "upstream_url": "http://172.16.0.237",
  "uris": [
    "/psq-test-api"
  ],
  "https_only": false,
  "upstream_send_timeout": 60000,
  "upstream_read_timeout": 60000,
  "name": "psq-test-api",
  "id": "06f5edf2-5373-485b-bfb3-fac0293ca534"
}
```



##### 获取 API 详情

`GET /apis/{name | id}`

**请求**

| **属性**      | **约束** | **描述**               |
| ----------- | ------ | -------------------- |
| `name | id` | 必填     | 用于获取 API 的唯一 id 或名称。 |



**响应**

```http
HTTP 200 OK
```

```json
{
  "http_if_terminated": true,
  "strip_uri": true,
  "retries": 5,
  "preserve_host": false,
  "created_at": 1491966352536,
  "upstream_connect_timeout": 60000,
  "upstream_url": "http://172.16.0.237",
  "uris": [
    "/psq-test-api"
  ],
  "https_only": false,
  "upstream_send_timeout": 60000,
  "upstream_read_timeout": 60000,
  "name": "psq-test-api",
  "id": "06f5edf2-5373-485b-bfb3-fac0293ca534"
}
```



##### 获取 APIs 列表

**请求**

`GET /apis/`

**参数**

|         **属性** | **约束**      | **描述**                 |
| -------------: | ----------- | ---------------------- |
|           `id` | 可选          | 以 `id` 字段过滤。           |
|         `name` | 可选          | 以 `name` 字段过滤。         |
| `upstream_url` | 可选          | 以 `upstream_url` 字段过滤。 |
|      `retries` | 可选          | 以 `retries` 字段过滤。      |
|         `size` | 可选，默认：`100` | 返回对象数限制。               |
|       `offset` | 可选          | 用于分页的游标。定义了列表中位置的对象。   |

**响应**

```http
HTTP 200 OK
```

- 未指定`size`：

  ```json
  {
    "data": [
      {
        "http_if_terminated": true,
        "strip_uri": true,
        "retries": 5,
        "preserve_host": false,
        "created_at": 1491966352536,
        "upstream_connect_timeout": 60000,
        "upstream_url": "http://172.16.0.237",
        "uris": [
          "/psq-test-api"
        ],
        "https_only": false,
        "upstream_send_timeout": 60000,
        "upstream_read_timeout": 60000,
        "name": "psq-test-api",
        "id": "06f5edf2-5373-485b-bfb3-fac0293ca534"
      }
    ],
    "total": 1
  }
  ```

  ​


- 指定 `size`：

  ```json
  {
    "next": "http://172.16.0.172:8001/apis?size=1&offset=ABAG9e3yU3NIW7%2Bz%2BsApPKU0AAcABHVyaXMAf%2F%2F%2F%2Fg%3D%3D",
    "offset": "ABAG9e3yU3NIW7+z+sApPKU0AAcABHVyaXMAf////g==",
    "data": [
      {
        "http_if_terminated": true,
        "strip_uri": true,
        "retries": 5,
        "preserve_host": false,
        "created_at": 1491966352536,
        "upstream_connect_timeout": 60000,
        "upstream_url": "http://172.16.0.237",
        "uris": [
          "/psq-test-api"
        ],
        "https_only": false,
        "upstream_send_timeout": 60000,
        "upstream_read_timeout": 60000,
        "name": "psq-test-api",
        "id": "06f5edf2-5373-485b-bfb3-fac0293ca534"
      }
    ],
    "total": 1
  }
  ```


##### 更新 API

**请求**

`PATCH /apis/{name | id}`

**参数**

|                     **属性** | **约束**        | **描述**                                   |
| -------------------------: | :------------ | ---------------------------------------- |
|                     `name` |               | API 的名称。                                 |
|                    `hosts` | 半可选(1/3)      | API 绑定的域名列表，以逗号分隔。比如：`example.com`。      |
|                     `uris` | 半可选(2/3)      | API 绑定的 URIs 前缀列表，以逗号分隔。比如：`/my-path`。   |
|                  `methods` | 半可选(3/3)      | API 接受的 HTTP 请求方式，以逗号分隔。比如：`GET,POST`。   |
|             `upstream_url` |               | API 绑定的基础目标地址，用于代理请求。比如：`https://example.com`。 |
|                `strip_uri` | 可选，默认：`true`  | 当通过 `uris` 前缀匹配到 API，请求上游地址时，省略匹配的前缀。    |
|            `preserve_host` | 可选，默认：`false` | 当通过 `hosts` 域名匹配到 API，确保请求的 `Host` 头被传递到上游服务。未启用时，上游 `Host` 头来自配置的 `upstream_url`。 |
|                  `retries` | 可选，默认：`5`     | 代理失败后的重试次数。                              |
| `upstream_connect_timeout` | 可选，默认：`60000` | 连接上游服务的超时时间，单位：毫秒。                       |
|    `upstream_send_timeout` | 可选，默认：`60000` | 用于向上游服务发送请求的两个连续写入操作之间的超时时间。单位：毫秒。       |
|    `upstream_read_timeout` | 可选，默认：`60000` | 用于向上游服务发送请求的两个连续读取操作之间的超时时间。单位：毫秒。       |
|               `https_only` | 可选，默认：`false` | 是否仅允许 HTTPS 访问 API（默认端口：`8443`）。         |
|       `http_if_terminated` | 可选，默认：`true`  | 考虑 `X-Forwarded-Proto` 头当强制 HTTPS 访问时。   |

> 支持部分属性的更新。

**响应**

```http
HTTP 200 OK
```

```json
{
  "http_if_terminated": true,
  "strip_uri": true,
  "retries": 5,
  "preserve_host": false,
  "created_at": 1491966352536,
  "upstream_connect_timeout": 60000,
  "upstream_url": "http://172.16.0.237",
  "uris": [
    "/psq-test-api",
    "/psq-api-test"
  ],
  "https_only": false,
  "upstream_send_timeout": 60000,
  "upstream_read_timeout": 60000,
  "name": "psq-test-api",
  "id": "06f5edf2-5373-485b-bfb3-fac0293ca534"
}
```



##### 更新或创建 API

**请求**

`PUT /apis/`

**参数**
|                     **属性** | **约束**        | **描述**                                   |
| -------------------------: | :------------ | ---------------------------------------- |
|                     `name` |               | API 的名称。                                 |
|                    `hosts` | 半可选(1/3)      | API 绑定的域名列表，以逗号分隔。比如：`example.com`。      |
|                     `uris` | 半可选(2/3)      | API 绑定的 URIs 前缀列表，以逗号分隔。比如：`/my-path`。   |
|                  `methods` | 半可选(3/3)      | API 接受的 HTTP 请求方式，以逗号分隔。比如：`GET,POST`。   |
|             `upstream_url` |               | API 绑定的基础目标地址，用于代理请求。比如：`https://example.com`。 |
|                `strip_uri` | 可选，默认：`true`  | 当通过 `uris` 前缀匹配到 API，请求上游地址时，省略匹配的前缀。    |
|            `preserve_host` | 可选，默认：`false` | 当通过 `hosts` 域名匹配到 API，确保请求的 `Host` 头被传递到上游服务。未启用时，上游 `Host` 头来自配置的 `upstream_url`。 |
|                  `retries` | 可选，默认：`5`     | 代理失败后的重试次数。                              |
| `upstream_connect_timeout` | 可选，默认：`60000` | 连接上游服务的超时时间，单位：毫秒。                       |
|    `upstream_send_timeout` | 可选，默认：`60000` | 用于向上游服务发送请求的两个连续写入操作之间的超时时间。单位：毫秒。       |
|    `upstream_read_timeout` | 可选，默认：`60000` | 用于向上游服务发送请求的两个连续读取操作之间的超时时间。单位：毫秒。       |
|               `https_only` | 可选，默认：`false` | 是否仅允许 HTTPS 访问 API（默认端口：`8443`）。         |
|       `http_if_terminated` | 可选，默认：`true`  | 考虑 `X-Forwarded-Proto` 头当强制 HTTPS 访问时。   |

如果更新一个已经存在的实体，需要增加 `id` 参数。

**响应**

- 新建

  ```http
  HTTP 201 Created
  ```


- 更新

  ```http
  HTTP 200 OK
  ```


##### 删除 API

**请求**

`DELETE /apis/{name | id}`

**参数**

|      **属性** | **约束** | **描述**        |
| ----------: | ------ | ------------- |
| `name | id` | 必填     | API 唯一识别或者名称。 |

**响应**

```http
HTTP 204 No Content
```



#### Consumer 对象

Consumer 对象表示一个消费者 - 或一个用户 - 或一个 API。可以将 Kong 作为主数据库，也可以将消费者列表与数据库进行映射，以保持 Kong 与现有主数据库存储之间的一致性。

```json
{
  "custom_id": "abc123"
}
```



##### 创建消费者

**请求**

`GET /consumers/`

**参数**

|      **属性** | **约束**   | **描述**                                   |
| ----------: | -------- | ---------------------------------------- |
|  `username` | 半可选(1/2) | 消费者的唯一用户名。                               |
| `custom_id` | 半可选(2/2) | 保存一个已存在唯一ID的消费者 —— 当映射 Kong 用户到已存在数据库中用户时有效。 |

> `username` 和 `custom_id` 至少提供一个。

**响应**

- 使用 `username`：

  ```http
  HTTP 201 Created
  ```

  ```json
  {
    "username": "qinghai",
    "created_at": 1491972052297,
    "id": "c8a3a84c-c854-4599-b17b-1c2ce5a991e7"
  }
  ```

- 使用 `custom_id`：

  ```http
  HTTP 201 Created
  ```

  ```json
  {
    "custom_id": "11",
    "created_at": 1491972123811,
    "id": "df111fc8-1c6b-49b1-adc2-b71f85fbeb33"
  }
  ```

  ​

##### 获取消费者详情

**请求**

`GET /consumers/{username | id}`

**参数**

|          **属性** | **约束** | **描述**       |
| --------------: | ------ | ------------ |
| `username | id` | 必填     | 消费者的用户名或 id。 |



**响应**

- 使用 `id`：

 ```http
 HTTP 200 OK
 ```

   ```json
   {
     "custom_id": "11",
     "created_at": 1491972123811,
     "id": "df111fc8-1c6b-49b1-adc2-b71f85fbeb33"
   }
   ```

- 使用 `username`：


  ```http
HTTP 200 OK
  ```

  ```json
  {
    "username": "qinghai",
    "created_at": 1491972052297,
    "id": "c8a3a84c-c854-4599-b17b-1c2ce5a991e7"
  }
  ```

  

##### 获取消费者列表

**请求**

`GET /consumers/`

**参数**

|      **属性** | **约束**      | **描述**                  |
| ----------: | ----------- | ----------------------- |
|        `id` | 可选          | 根据消费者 `id` 字段过滤。        |
| `custom_id` | 可选          | 根据消费者 `custom_id` 字段过滤。 |
|  `username` | 可选          | 根据消费者 `username` 字段过滤。  |
|      `size` | 可选，默认：`100` | 每次返回的条数限制。              |
|    `offset` | 可选          | 分页游标。对象。                |



**响应**

```http
HTTP 200 OK
```

```json
{
  "data": [
    {
      "custom_id": "11",
      "created_at": 1491972123811,
      "id": "df111fc8-1c6b-49b1-adc2-b71f85fbeb33"
    },
    {
      "username": "qinghai",
      "created_at": 1491972052297,
      "id": "c8a3a84c-c854-4599-b17b-1c2ce5a991e7"
    }
  ],
  "total": 2
}
```



##### 更新消费者

**请求**

`PATCH /consumers/{username | id}`

|          **属性** | **约束** | **描述**         |
| --------------: | ------ | -------------- |
| `username | id` |        | 需要更新的消费者识别或id。 |

**参数**

|      **属性** | **约束**   | **描述**             |
| ----------: | -------- | ------------------ |
|  `username` | 半可选(1/2) | 更新后消费者的用户名。        |
| `custom_id` | 半可选(2/2) | 更新后消费者的 custom_id。 |

> 如果原来没有此项的会追加，否则就是修改。

**响应**

```http
HTTP 200 OK
```

```json
{
  "custom_id": "11",
  "username": "luobo",
  "created_at": 1491972123811,
  "id": "df111fc8-1c6b-49b1-adc2-b71f85fbeb33"
}
```

- 已存在重复

  ```http
  HTTP 409 Conflict
  ```

  ```json
  {
    "username": "already exists with value 'qinghai'"
  }
  ```

  ​

##### 创建或修改消费者

**请求**

`PUT /consumers/`

**参数**

|      **属性** | **约束** | **描述** |
| ----------: | ------ | ------ |
|  `username` |        |        |
| `custom_id` |        |        |

> 需要有 id 属性来触发修改，否则为创建。

**响应**

- **创建**

  ```http
  HTTP 201 Created
  ```

- **修改**

  ```http
  HTTP 200 OK
  ```


##### 删除消费者

**请求**

`DELETE /consumers/{username | id}`

**参数**

| **属性** | **约束** | **描述** |
| -----: | ------ | ------ |
|        |        |        |



**响应**

```http
HTTP 204 No Content
```



#### Plugin 对象

插件实体表示在 HTTP 请求/响应期间会执行的插件配置，是为在 Kong 后运行的 APIs 添加功能的方式，如身份认证或者频率限制。安装及每个插件的配置参见：[插件集合](https://getkong.org/plugins)。

添加到 API 顶部，客户端的每个请求都会被插件根据配置进行评估。有些时候，需要针对不同的消费者有不同的值，那么在配置中，指定 `consumer_id` 值。

```json
{
    "id": "4d924084-1adb-40a5-c042-63b19db421d1",
    "api_id": "5fd1z584-1adb-40a5-c042-63b19db49x21",
    "consumer_id": "a3dX2dh2-1adb-40a5-c042-63b19dbx83hF4",
    "name": "rate-limiting",
    "config": {
        "minute": 20,
        "hour": 500
    },
    "enabled": true,
    "created_at": 1422386534
}
```



##### 添加插件

**请求**

`POST /apis/{name | id}/plugins/`

|    **属性** | **约束** | **描述**                                   |
| --------: | ------ | ---------------------------------------- |
| `name|id` | 必填     | 需要添加插件 API 的名称或id。如果需要为所有 API 添加插件，使用 `/plugins`。 |



**参数**

|              **属性** | **约束** | **描述**                                   |
| ------------------: | ------ | ---------------------------------------- |
|              `name` |        | 需添加的插件名称。当前必须保证所有 Kong 的示例都已安装此插件。       |
|       `consumer_id` | 可选     | 消费者 id。                                  |
| `config.{property}` |        | 插件的配置属性。见：[插件列表](https://getkong.org/plugins)。 |



**响应**

```http
HTTP 201 Created
```

```json
{
  "api_id": "06f5edf2-5373-485b-bfb3-fac0293ca534",
  "id": "e0f59d86-91da-4329-8a43-29c39f9ee573",
  "created_at": 1492045036807,
  "enabled": true,
  "name": "file-log",
  "config": {
    "path": "/tmp/psq-test-api.log"
  }
}
```



##### 获取插件详情

**请求**

`GET /plugins/{id}`

| **属性** | **约束** | **描述**  |
| -----: | ------ | ------- |
|   `id` | 必填     | 插件的 id。 |



**响应**

```http
HTTP 200 OK
```

```json
{
  "api_id": "06f5edf2-5373-485b-bfb3-fac0293ca534",
  "id": "e0f59d86-91da-4329-8a43-29c39f9ee573",
  "created_at": 1492045036807,
  "enabled": true,
  "name": "file-log",
  "config": {
    "path": "/tmp/psq-test-api.log"
  }
}
```



##### 获取插件列表

**请求**

`GET /plugins/`

**参数**

|        **属性** | **约束** | **描述**        |
| ------------: | ------ | ------------- |
|          `id` |        | 根据 `id` 字段过滤。 |
|        `name` |        |               |
|      `api_id` |        |               |
| `consumer_id` |        |               |
|        `size` |        |               |
|      `offset` |        |               |



**响应**

```http
HTTP 200 OK
```

```json
{
  "data": [
    {
      "api_id": "06f5edf2-5373-485b-bfb3-fac0293ca534",
      "id": "e0f59d86-91da-4329-8a43-29c39f9ee573",
      "created_at": 1492045036807,
      "enabled": true,
      "name": "file-log",
      "config": {
        "path": "/tmp/psq-test-api.log"
      }
    }
  ],
  "total": 1
}
```



##### 获取单个 API 中得插件

**请求**

`GET /apis/{api name|id}/plugins/`

**参数**

|        **属性** | **约束** | **描述**        |
| ------------: | ------ | ------------- |
|          `id` |        | 根据 `id` 字段过滤。 |
|        `name` |        |               |
|      `api_id` |        |               |
| `consumer_id` |        |               |
|        `size` |        |               |
|      `offset` |        |               |



**响应**

```http
HTTP 200 OK
```

```json
{
  "data": [
    {
      "api_id": "06f5edf2-5373-485b-bfb3-fac0293ca534",
      "id": "e0f59d86-91da-4329-8a43-29c39f9ee573",
      "created_at": 1492045036807,
      "enabled": true,
      "name": "file-log",
      "config": {
        "path": "/tmp/psq-test-api.log"
      }
    }
  ],
  "total": 1
}
```



##### 更新插件

**请求**

`PATCH /apis/{api name|api id}/plugins/{id}`

|            **属性** | **约束** | **描述** |
| ----------------: | ------ | ------ |
| `api name|api id` | 必填     |        |
|              `id` | 必填     |        |

**参数**

|              **属性** | **约束** | **描述** |
| ------------------: | ------ | ------ |
|              `name` |        |        |
|       `consumer_id` | 可选     |        |
| `config.{property}` |        |        |



**响应**

```http
HTTP 200 OK
```

```json
{
  "api_id": "06f5edf2-5373-485b-bfb3-fac0293ca534",
  "id": "e0f59d86-91da-4329-8a43-29c39f9ee573",
  "created_at": 1492045036807,
  "enabled": true,
  "name": "file-log",
  "config": {
    "path": "/tmp/psq-test-api.info.log"
  }
}
```



##### 修改或添加插件

**请求**

`PUT /apis/{api name|api id}/plugins/`

|            **属性** | **约束** | **描述** |
| ----------------: | ------ | ------ |
| `api name|api id` |        |        |

**参数**

|              **属性** | **约束** | **描述** |
| ------------------: | ------ | ------ |
|              `name` |        |        |
|       `consumer_id` | 可选     |        |
| `config.{property}` |        |        |

**响应**

- *创建*

  ```http
  HTTP 201 Created
  ```

  ​

- *修改*

  ```http
  HTTP 200 OK
  ```


##### 删除插件

**请求**

`DELETE /plugins/enabled`

**响应**

```http
HTTP 204 No Content
```



##### 获取已启动插件列表

**请求**

`GET /plugins/enabled`



**响应**

```http
HTTP 200 OK
```

```json
{
  "enabled_plugins": [
    "syslog",
    "ldap-auth",
    "rate-limiting",
    "correlation-id",
    "jwt",
    "runscope",
    "request-transformer",
    "http-log",
    "loggly",
    "response-transformer",
    "basic-auth",
    "tcp-log",
    "key-auth",
    "oauth2",
    "acl",
    "galileo",
    "udp-log",
    "cors",
    "file-log",
    "ip-restriction",
    "datadog",
    "request-size-limiting",
    "bot-detection",
    "aws-lambda",
    "statsd",
    "response-ratelimiting",
    "hmac-auth"
  ]
}
```



##### 获取插件格式

获取插件的配置格式。用于了解插件需要哪些插件，以及用于构建第三方集成的插件。

**请求**

`GET /plugins/schema/{plugin name}`

|        **属性** | **约束** | **描述** |
| ------------: | ------ | ------ |
| `plugin name` |        |        |



**响应**

```http
HTTP 200 OK
```

```json
{
  "fields": {
    "path": {
      "type": "string",
      "required": true,
      "func": "function"
    }
  }
}
```



#### Certificate 对象

保存 keys 和证书；



#### SNI 对象

关联了一个注册证书与服务名标志。



#### Upstream 对象

上游对象表示一个虚拟的主机名，用于对收到的请求在多个服务（targets）上负载。比如，`service.v1.xyz` 上游对象，及上游地址为 `upstream_url=https://service.v1.xyz/some/path` 的 API 对象。请求到此 API 会被代理到上游定义的目标中。

```json
{
    "name": "service.v1.xyz",
    "orderlist": [
        1,
        2,
        7,
        9,
        6,
        4,
        5,
        10,
        3,
        8
    ],
    "slots": 10
}
```



##### 添加上游

**请求**

`POST /upstreams/`

**参数**

|      **属性** | **约束**                      | **描述**                                   |
| ----------: | --------------------------- | ---------------------------------------- |
|      `name` |                             | 与主机名相似的名称，可以被 API 的 upstream_url 引用。     |
|     `slots` | 可选，（`10`-`65535`)。默认：`1000` | 负载均衡算法中得插槽数。                             |
| `orderlist` | 可选                          | 顺序列表，但是随即排序。决定均衡中插槽分布的整数。如果省略则自动生成。如果指定，必须严格等于 `slots` 数。 |



**响应**

```http
HTTP 201 Created
```

```json
{
  "orderlist": [
    4,
    7,
    5,
    10,
    9,
    2,
    6,
    1,
    8,
    3
  ],
  "slots": 10,
  "id": "21054908-7faf-4581-ac2c-70fb5f99473d",
  "name": "api.psq.test",
  "created_at": 1492047641904
}
```



##### 模板

**请求**

``

**参数**

| **属性** | **约束** | **描述** |
| -----: | ------ | ------ |
|        |        |        |



**响应**

```http
HTTP 200 OK
```

```json

```





#### Target 对象

目标是一个 ip 地址/主机名包含端口，表示一个后端服务的实例。每个上游可以包含多个目标，且目标可以动态添加或移除。改变的很快生效。

因为上游维护了目标变更的历史，所以目标是无法删除或者修改的。如果需要禁用一个目标，post 新的 `weight=0`。或者，使用 `DELETE` 方法效果是一样的。

```json
{
    "target": "1.2.3.4:80",
    "weight": 15,
    "upstream_id": "ee3310c1-6789-40ac-9386-f79c0cb58432"
}
```



##### 添加对象

**请求**

`PSOT /upstreams/{name|id}/targets`

**参数**

| **属性** | **约束**                  | **描述** |
| -----: | ----------------------- | ------ |
| target |                         |        |
| weight | 可选，`0`-`1000`。默认：`100`。 |        |



**响应**

```http
HTTP 200 OK
```

```json

```



##### 获取指定上游对象列表





##### 获取指定上游已激活对象列表





##### 删除对象





##### 模板

**请求**

``

**参数**

| **属性** | **约束** | **描述** |
| -----: | ------ | ------ |
|        |        |        |



**响应**

```http
HTTP 200 OK
```

```json

```





### Proxy Reference

Kong 默认监听的 4 个端口

- `8000` 接收来自客户端的请求，并前行到上游服务；
- `8443` 与 *8000* 端口功能类似，但是 HTTPS 请求，可以通过配置文件禁用；
- `8001` Admin API 用于配制 Kong 的端口；
- `8444` Admin API 的 HTTPS 端口。




##### 术语

- `API`：表示 Kong 的 API 实体。通过 Admin API 配置 APIs，指向自己的上游服务；
- `Plugin`：表示 Kong 的 "Plugins"，将业务逻辑嵌入到代理执行的生命周期中。 插件可以通过 Admin API 配置，可以是全局的，或某个特定 API 的；
- `Client`：表示下游客户端，向 Kong 代理发起了请求；
- `Upstream service`：表示你自己的 API/服务，是客户端请求被代理到的位置。



##### 概述

Kong 可以高度概括为：在配置的端口上监听 HTTP 请求（默认是 8000），识别哪个上游服务被请求，对这个 API 执行配置的插件，然后将 HTTP 请求移动到上游的 API 或 服务。

当客户端向代理端口发起一个请求，Kong 会决定路由（或 前移）到哪个上游服务或 API，依据 Kong 中有 通过 Admin API 设置的 API 配置。可以用变量属性配置 APIs，但只有三个影响路由：*hosts*，*uris* 和 *methods*。

如果 Kong 无法确定路由到哪个上游API，会响应：

```
HTTP/1.1 404 Not Found
Content-Type: application/json
Server: kong/<x.x.x>

{
  "message": "no API found with those values"
}
```



##### 提醒： 如何向 Kong 添加 API

往 Kong 的 Admin API 端口（默认8001）发送 HTTP 请求，如下：

```shell
$ curl -i -X POST http://localhost:8081/apis/ \
   -d 'name=my-api' \
   -d 'upstream_uri=http://my-api.com' \
   -d 'hosts=example.com' \
   -d 'uris=/my-api' \
   -d 'methods=GET,HEAD'

HTTP/1.1 201 Created
...
```

这个请求指示 Kong 注册一个名为 “my-api” 的API，可在 "http://exmaple.com" 访问。同时指定了路由变量属性，虽然 *hosts*，*uri* 和 *methods* 仅只有一个是必须的。

添加上述 API 意味着配置 Kong 代理所有到 `http://example.com` 的请求要符合 hosts、uris 及 methods。Kong 是透明代理，会将请求不做任何改动直接前移到上游服务，除了附加的一些请求头，如 *Connection*。



##### 路由能力

*hosts*、*uris* 及 *methods* 都是可选的，但至少要指定一个。对于客户端请求匹配匹配 API过程：

- 请求**必须**包含**所有**的配置字段；
- 请求中字段的值**必须**至少匹配一个配置的值（如果配置的字段接受一个或多个值，一个请求只要包含其中一个就算匹配）。

假设一个 API 配置如下：

```json
{
  "name": "my-api",
  "upstream_url": "http://my-api.com",
  "hosts": ["exmaple.com", "service.com"],
  "uris": ["/foo", "/bar"],
  "methods": ["GET"]
}
```

与此 API 匹配的请求可能是：

```http
GET /foo HTTP/1.1
Host: example.com
```

```http
GET /bar HTTP/1.1
Host: service.com
```

```http
GET /foo/hello/world HTTP/1.1
Host: example.com
```

而这些是不匹配的：

```http
GET / HTTP/1.1
Host: example.com
```

```http
POST /foo HTTP/1.1
Host: example.com
```

```http
GET /foo HTTP/1.1
Host: foo.com
```



**请求 Host 头**

基于请求的 Host 头进行路由时最简单的代理方式，并且与 HTTP Host 头的预期效果一致。Kong 通过 API 的 *hosts* 字段简化了操作。

*hosts* 允许多个值，需用逗号分隔。

```shell
$ curl -i -X POST http://localhost:8001/apis/ \
    -d 'name=my-api' \
    -d 'upstream_url=http://my-api.com' \
    -d 'hosts=my-api.com,example.com,service.com'
    
HTTP/1.1 201 Created
...
```

为了满足API 的 *hosts* 条件，从客户端发起的任何必须包括：

```
Host: my-api.com
```

或

```
Host: example.com
```

或

```
Host: service.com
```

**使用泛域名**

Kong 支持通配符的 `hosts`，以提供更高的灵活性。通配符主机头允许匹配任何满足条件的主机头，匹配到 API。

通配符主机名**必须**包含**唯一**的星号在域名的最左边或最右边。如：

- `*.example.com`：允许的主机头可以是 *a.example.com* 和 *x.y.example.com* 来匹配；
- `example.*`：允许主机头可以是 *example.com* 和 *example.org*来匹配。

一个完整的例子：

```json
{
  "name": "my-api",
  "upstream_url": "http://my-api.com",
  "hosts": ["*.example.com", "service.com"]
}
```

将匹配如下请求：

```http
GET / HTTP/1.1
Host: an.example.com
```

```http
GET / HTTP/1.1
Host: service.com
```

**`preserve_host` 属性**

Kong 在代理过程中，默认将 upstream 的主机头设置为 API 中的 *upstream_url* 属性的主机名。*preserve_host* 字段接受一个布尔值指示 Kong 不要那么做。

举例来说，如果 *preserve_host* 未改变并且 API 的配置如下：

```json
{
  "name": "my-api",
  "upstream_url": "http://my-api.com",
  "hosts": ["service.com"]
}
```

客户端发起的请求可能是：

```http
GET / HTTP/1.1
Host: service.com
```

Kong 会从 API 中 *upstream_url* 字段中获取主机名，并向上游服务发送如下请求：

```http
GET / HTTP/1.1
Host: my-api.com
```

但，如果显式配置 API：`preserve_url=true`：

```json
{
  "name": "my-api",
  "upstream_url": "http://my-api.com",
  "hosts": ["service.com"],
  "preserve_host": true
}
```

假设客户端请求不变：

```http
GET / HTTP/1.1
Host: service.com
```

Kong 会保留客户端请求的主机头并发送如下请求到上游服务：

```http
GET / HTTP/1.1
Host: service.com
```



**请求地址**

另一个路由请求的参数是 *uris* 属性。

```json
{
  "name": "my-api",
  "upstream_url": "http://my-api.com",
  "uris": ["/service", "/hello/world"]
}
```

下列请求可以匹配：

```http
GET /service HTTP/1.1
Host: my-api.com
```

```http
GET /service/resource?param=value HTTP/1.1
Host: my-api.com
```

```http
GET /hello/world/resource HTTP/1.1
Host: anything.com
```

对上述请求，Kong 检测到 URI 是以 API 中 *uris* 包含的某个值为前缀的。Kong 默认将请求不做任何更改前移到，URI 不变。

如果 URIs 前缀代理时，**最长的 URIs 优先**。意味着可以定义两个 APIs： */service* 和 */service/resource* ，且确保前者不影响后者。

**`strip_uri` 属性**

有些情况下，API 中用于制定匹配的前缀，并应该包含在上游的请求中。使用 *strip_uri* 布尔尚需经来配置API即可：

```json
{
  "name": "my-api",
  "upstream_url": "http://my-api.com",
  "uris": ["/service"],
  "strip_uri": true
}
```

启用该配置项指示 Kong 在代理此 API 时，不包含匹配的 URI 前缀到上游请求地址。如下客户端请求：

```HTTP
GET /service/path/to/resource HTTP/1.1
Host: 
```

Kong 会发送如下请求到上游服务：

```http
GET /path/to/resource HTTP/1.1
Host: my-api.com
```



**请求的方法**

从 Kong 0.10 开始，客户端请求路由还会依赖 HTTP 方法，通过指定 `methods` 字段。默认，Kong 会路由所有请求到一个 API 请求，忽略 HTTP 方法。一旦设置了此字段，仅符合指定 HTTP 方法的请求会被匹配。

该字段允许多个值。下例允许 `GET` 和 `HEAD` 的 HTTP 方法：

```json
{
  "name": "api-1",
  "upstream_url": "http://my-api.com",
  "methods": ["GET", "HEAD"]
}
```

允许如下请求：

```http
GET / HTTP/1.1
Host: 
```

```http
HEAD / HTTP/1.1
Host: 
```

但不会匹配 `POST` 和 `DELETE` 的请求。使得配置 APIs 和 插件支持更多的粒度。摄像，有2个 APIs 指向同一个上游服务，一个允许无限制未认证的 GET 请求，另一个只允许认证过的有限的 POSt 请求（通过应用 authentication 和 rate limiting 插件）。



##### 路由优先级

一个 API 可能定义匹配规则在 `hosts`，`uris` 和 `methods` 多个字段。对 Kong 在匹配一个 API 的请求时，所有存在的字段必须满足。但是，Kong 非常灵活，允许定义两个或以上的 APIs 配置包含相同值得字段，当这种情况出现时，Kong 应用优先级规则。

规则如下：**当评估一个请求，Kong 会先尝试 APIs 规则最多的**。

示例如下，假设两个 APIs 有如下配置：

```json

{
  "name": "api-1",
  "upstream_url": "http://my-api.com",
  "hosts": ["example.com"]
},
{
  "name": "api-2",
  "upstream_url": "http://my-api-2.com",
  "hosts": ["example.com"],
  "methods": ["POST"]
}
```

因为 api-2 有一个 `hosts` 字段和 `methods` 字段，所以会先被 Kong 评估。这样做，避免 api-1 对 api-2 影响。

因此，此请求会匹配 api-1：

```http
GET / HTTP/1.1
Host: example.com
```

而，这个请求会匹配 api-2：

```http
POST / HTTP/1.1
Host: example.com
```

顺着这个逻辑，如果有第三个 API 配置了 `hosts` 字段、`methods` 字段，以及 `uris` 字段，那么背最先被 Kong 评估。



##### 代理行为

代理规则说明了 Kong 如何前移请求到上游服务的详细过程。下面解释，Kong 在识别一个 HTTP 请求到目标服务这段时间内部实际发生了什么，和最终被前移到服务的请求。

1.  负载均衡

   从 Kong 0.10 开始，实现了负载均衡功能，分发前移请求到多个上游服务实例。

   在 Kong 0.10 之前，Kong 通常将被代理请求发送到 `upstream_url`，并且需要一个外部的负载均衡实现多个上游服务的负载均衡。

   更多参见：[负载均衡参考](https://getkong.org/docs/0.10.x/admin-api#load-balancing-guide)。

2. 插件执行

   Kong 通过 “插件” 实现扩展，通过代理请求的 request/response 生命周期中的钩子，插件可以在环境下执行各种操作且/或将代理请求进行各种转换。

   插件可以被配置在全局（应用到所有代理流量）或单个 API 通过 Admin API 以[插件配置](https://getkong.org/docs/0.10.x/admin-api#plugin-configuration-object) 的方式。

   如果一个 API 已配置了一个插件，并且这个 API 被传入的请求匹配，Kong 会执行配置的插件（们），在请求被代理到上游服务前。其中包括插件的访问阶段，可以在[插件开发指南中](https://getkong.org/docs/0.10.x/admin-api#plugin-development-guide)获取更多信息。

3. 代理 & 上游 超时

   一旦 Kong 已经执行完所有需要的逻辑（包含插件的），就应经准备前移请求到上游服务。这步通过 Nginx 的 [ngx_http_proxy_module](http://nginx.org/en/docs/http/ngx_http_proxy_module.html) 完成。从 Kong 0.10 开始，Kong 和 上游服务的链接超时周期可以通过以下 3 个 API 对象的属性配置：

   - `upstream_connect_timeout`: 定义与上游服务建立链接的超时时间，单位：毫秒。默认：`6000`；
   - `upstream_send_timeout`：定义将请求发送到上游服务器的两个连续写入操作之间的超时。单位：毫秒，默认：`6000`；
   - `upstream_read_timeout`：定义从上游服务接收请求的两个连续读操作之间的超时。单位：毫秒，默认：`6000`。

   Kong 通过 HTTP/1.1 发送请求，并设置如下请求头：

   - `Host: <your_upstream_host>`，如文档前面所述；
   - `Connection: keep-alive`，允许重用上游链接；
   - `X-Real-IP: <$proxy_add_x_forwarded_for>`, `$proxy_add_x_forwarded_for` 就是  [ngx_http_proxy_module](http://nginx.org/en/docs/http/ngx_http_proxy_module.html) 中得同名变量的值；
   - `X-Forwarded-Proto: <protocal>`，`<protocal>` 是客户端使用的协议。

   其他所有的请求头，保持不变。

   有一个例外是使用 WebSocket 协议。Kong 会设置如下头，来实客户端和上游服务间现协议升级。

   - `Connection: Upgrade`
   - `Upgrade: websocket`

4. 响应

   Kong 从上游服务接收响应并通过留形式发送回下游客户端。此时，Kong 将执行后续插件增加了此 API  `header_filter` 阶段的钩子。

   一旦所有插件的 `header_filter` 阶段执行完成，Kong 会添加以下的头并将完整的头集合发送到客户端：

   - `Via: kong/x.x.x`，`x.x.x` 表示 Kong 的版本；
   - `Kong-Proxy-Latency: <latency>`, `latency` 表示 Kong 从客户端接收请求后发送到上游服务的毫秒数；
   - `Kong-Upstream-Latency: <latency>`，`latency` 表示 Kong 从收到上游服务响应的第一个字节等待的毫秒数。

   一旦头被发送到客户端，Kong 将开始执行已经注册插件中实现了 `body_filter` 的钩子。这个钩子可能会被多次调用，取决于 Nginx 自身的流。每个上游响应的快都会被 `body_filter` 钩子成功处理并发回到客户端。更多参见[插件开发指南](https://getkong.org/docs/0.10.x/admin-api#plugin-development-guide)。


##### 配置 API 回退

实现“回退 API”，避免 Kong 响应 HTTP `404`，“API Not Found”，通过捕获这些请求，并代理到一个特殊的上游服务，或者应用插件（那些可以将请求中断并响应不同状态码或者不代理请求直接响应）。

示例：

```json
{
  "name": "root-fallback",
  "upstream_url": "http://dummy.com",
  "uris": ["/"]
}
```

所有 HTTP 请求都会匹配这个 API，因为所有的 URIs 都是以 `/` 前缀。从前面对请求URI的介绍，最长匹配的 URIs 会被优先评估，因此 `/` URI 会被最后评估，并有效提供了一个回退 API，仅在最后匹配。



##### 配置 SSL

Kong 提供了一种动态服务 SSL 证书给予每个请求。从 0.10 开始，SSL 插件被移除，SSL 证书由核心直接处理，并通过 Admin API 配置。HTTP 客户端库必须支持 [Server Name Indication](https://en.wikipedia.org/wiki/Server_Name_Indication) 扩展来支持此功能。

SSL 证书由 Kong Admin API 的两个资源处理：

- `/certificates`，保存了 keys 和 证书；
- `/snis`，关联一个支持 Server Name Indication 的注册证书。

完整文档参见：[Admin API 参考](https://getkong.org/docs/0.10.x/admin-api)

通过 Admin API 配置 SSL 证书:

1.  通过 Admin API 上传 SSL 证书 和 key：

   ```shell
   $ curl -i -X POST http://localhost:8001/certificates \
       -d "cert=@/path/to/cert.pem" \
       -d "key=@/path/to/cert.key" \
       -d "snis=ssl-example.com"
       

   HTTP/1.1 201 Created
   ```

   `snis` 直接插入一个 SNI 并将其与已上传的证书关联。

2. 注册新的如下 API，通过主机头路由：

   ```shell
   $ curl -i -X POST http://localhost:8001/apis \
       -d "name=ssl-api" \
       -d "upstream_url=http://my-api.com" \
       -d "hosts=ssl-example.com,other-ssl-example.com"
   ```

3. 通过 Https 方式请求

   ```shell
   $ curl -i https://localhost:8443/ \
       -H "Host: ssl-example.com"

   HTTP/1.1 200 OK
   ```



**`https_only` 属性**

仅允许 API 通过 HTTPS 访问，则启用该选项。

```shell
$ curl -i -X POST http://localhost:8001/apis \
    -d "name=ssl-only-api" \
    -d "upstream_url=http://example.com" \
    -d "hosts=my-api.com" \
    -d "https_only=true"
    
HTTP/1.1 201 Created   
```

通过配置 API， Kong 将拒绝所有非 HTTPS 的请求。明文的 HTTP 请求会指示客户端升级到 HTTPS

```shell
$ curl -i http://localhost:8000 \
    -H "Host: example.com"
    
HTTP/1.1 426
Content-Type: application/json; charset=utf-8
Transfer-Encoding: chunked
Connection: upgrade
Upgrade: TLS/1.2, HTTP/1.1
Server: kong/x.x.x

{"message": "Please use HTTPS protcal"}
```

**`http_if_terminated`属性**

如果希望考虑请求头中得 `X-Forwarded-Proto` 属性来强制 HTTPS 访问，启用 `http_if_terminated` 属性。

```shell
$ curl -i -X PATCH http://localhost:8001/apis/ssl-only-api \
    -d "http_if_terminated=true"

HTTP/1.1 200 OK
```

模拟客户端发起携带 `X-Forwarded-Proto` 头的请求（假设来自授信的客户端）：

```shell
$ curl -i http://localhost:8000 \
    -H "Host: example.com" \
    -H "X-Forwarded-Proto: https"
    
HTTP/1.1 200 OK
```

现在 Kong 会代理这个请求，因为其认为 SSL 终止是被架构中得上个环节完成的。



##### 代理 WebSocket 访问

因为 Nginx 本身支持 WebSocket 使得 Kong 也获得支持。如果需要监听通过 Kong 的客户端与上游服务间的 WebSocket，需要监听一个 WebSocket 握手。通过 HTTP Upgrade 机制。

此时客户端的请求类似：

```http
GET / HTTP/1.1
Connection: Upgrade
Host: my-websocket-api.com
Upgrade: WebSocket
```

使 Kong 前移 `Connection` 和 `Upgrade` 头到上游服务，而不是驳回他们，因为标准 HTTP 代理的逐跳特性。



### 插件

#### Basic Authentication

添加基本认证到你的 APIs，通过用户名和密码保护。插件会依次检测 `Proxy-Authorization` 及 `Authorization` 头是否是有效证书。

---

##### 配置

|             **属性** | **约束**          | **说明**                                   |
| -----------------: | --------------- | ---------------------------------------- |
| `hide_credentials` | 可以选，默认：`false`。 | 对上游服务隐藏凭据。在被代理请求前就被 kong 移除。             |
|        `anonymous` | 可选，默认：`''`。     | 如果认证失败，则使用此消费者id，作为匿名消费者。如果空，则返回认证失败：`4xx`。 |



##### 用法

首先，需要创建一个消费者关联一个或多个凭据。消费者表示最终使用服务/API的开发者。

1.  创建消费者：每个消费者可以有多个凭证。

2. 创建凭证：

   ```shell
   $ curl -X POST http://kong:8001/consumers/{consumer}/basic-auth \
     --data "username=Aladdin" \
     --data "password=OpenSeasme"
   ```

   ```http
   HTTP 201 Created
   ```

   ```json
   {
     "password": "3b9d6ace004714603ad5ecce67fe8615cec8aad0",
     "consumer_id": "c8a3a84c-c854-4599-b17b-1c2ce5a991e7",
     "id": "fd9dd75c-b24f-4392-973d-38afab8e31fd",
     "username": "dev_qinghai",
     "created_at": 1492066102312
   }
   ```

3. 使用凭证：

4. 自动添加上游头信息：

    当客户端一旦被认证，插件会在请求被代理到上游 API/微服务前追加额外的头，因此可以用此来识别消费者；

- `X-Consumer-ID`：Kong 消费者的 ID；
   - `X-Consumer-Custom-ID`：消费者的 `custom_id`，如果有设置；
   - `X-Consumer-Username`：消费者的 `ussername`，如果有设置；
   - `X-Credential-Username`：凭证的 `username` （仅当非匿名消费者时）；
   - `X-Anonymous-Consumer`：当认证失败，使用匿名消费者时，该值为 `true`。

> 使用以上信息来实现附加逻辑，可以用 `X-Consumer-ID` 值来查询 Kong Admin API 并获取更多关于此消费者的信息。



完整示例：

```json
{
    "started_at": 1492066577591,
    "response": {
        // ...
    },
    "authenticated_entity": {
        "consumer_id": "c8a3a84c-c854-4599-b17b-1c2ce5a991e7",
        "id": "ea89227d-9162-4050-a188-ca381fe96a37"
    },
    "request": {
        "method": "GET",
        "uri": "\/psq-test-api\/goods",
        "size": "434",
        "request_uri": "http:\/\/172.16.0.172:8000\/psq-test-api\/goods",
        "querystring": {},
        "headers": {
            "x-consumer-id": "c8a3a84c-c854-4599-b17b-1c2ce5a991e7",
            "accept-language": "zh-CN,zh;q=0.8,en;q=0.6",
            "connection": "keep-alive",
            "accept": "*\/*",
            "cache-control": "no-cache",
            "host": "172.16.0.172:8000",
            "x-credential-username": "dqinghai",
            "x-consumer-username": "qinghai",
            "accept-encoding": "gzip, deflate, sdch",
            "postman-token": "9aaede64-b9b1-89b9-e657-a1b203e4e6e5",
            "user-agent": "Mozilla\/5.0 (Macintosh; Intel Mac OS X 10_12_4) AppleWebKit\/537.36 (KHTML, like Gecko) Chrome\/57.0.2987.133 Safari\/537.36",
            "authorization": "Basic ZHFpbmdoYWk6MTIzNDU2"
        }
    },
    "client_ip": "192.168.11.152",
    "api": {
        // ...
    },
    "latencies": {
       // ...
    }
}
```



#### Key Authentication

也成为 API key。消费者在查询参数或者头中添加他们的 key 来认证请求。

> 身份头信息，优先级低于 Basic Auth（先添加 Baisc Auth的情况下，另外未测试）。

##### 术语

- `api`：放在 Kong 后面的上游服务，代理请求到这里；
- `plugin`：
- `consumer`：使用 api 的开发者或服务；
- `credential`：。



##### 配置

|             **属性** | **约束**                      | **描述**                            |
| -----------------: | --------------------------- | --------------------------------- |
|        `key_names` | 可选，默认：`apiKey`。[a-zA-Z0-9-] | 插件尝试读取凭证的键名，多个用逗号分隔。不限请求头还是查询字符串。 |
| `hide_credentials` | 可选，默认：`false`。              | 是否对上游隐藏凭证。                        |
|        `anonymous` | 可选，默认：`''`。                 | 匿名消费者id。未指定则表示不支持匿名。              |



##### 使用

1.  创建消费者；
2.  创建 API key：
3.  调用，**apikey 必须使用小写**。



#### OAuth2.0 Authentication

添加 OAuth2.0 验证层，支持下列流程：

-  [Authorization Code Grant](https://tools.ietf.org/html/rfc6749#section-4.1)；

- [Client Credentials](https://tools.ietf.org/html/rfc6749#section-4.4)：客户端证书授权，适用于运行在服务端的客户端，如

  - 客户端以自己的名字，而不是用户的名义，向“服务提供商”进行认证（Client 自己就是 Resource Owner，客户端取用的是拥有的 Protected Resources）；
  - 严格不属于 OAuth 框架，这个模式中，用户直接向客户端注册，客户端以自己的名字要求“服务提供商”提供服务，不存在授权。

  ```
       +---------+                                  +---------------+
       |         |                                  |               |
       |         |>--(A)- Client Authentication --->| Authorization |
       | Client  |                                  |     Server    |
       |         |<--(B)---- Access Token ---------<|               |
       |         |                                  |               |
       +---------+                                  +---------------+
  ```

  ```http

  ```

  ​

- [Implicit Grant](https://tools.ietf.org/html/rfc6749#section-4.2)；

- [Resource Owner Password Credentials Grant](https://tools.ietf.org/html/rfc6749#section-4.3)。



##### 配置

|                              **属性** | **约束**                | **描述**                                   |
| ----------------------------------: | --------------------- | ---------------------------------------- |
|                            `scopes` |                       | 终端用户可用的范围名称，用逗号分隔。                       |
|                   `mandatory_scope` | 可选，默认：`false`。        | 必须包含至少一个范围。                              |
|                  `token_expiration` | 可选，默认：`7200`。`0`表示禁用。 | Token 有效期持续多久，单位：秒。                      |
|         `enable_authorization_code` | 可选，默认：`false`。        | 启用 threed-legs 验证码流程([RFC 6742 Section 4.1](https://tools.ietf.org/html/rfc6749#section-4.1)) |
|         `enable_client_credentials` | 可选，默认：`false`。        | 启用 客户端凭证授权流程 ([RFC 6742 Section 4.4](https://tools.ietf.org/html/rfc6749#section-4.4))。 |
|             `enable_implicit_grant` | 可选，默认：`false`。        | 启用 implicit_grant ([RFC 6742 Section 4.2](https://tools.ietf.org/html/rfc6749#section-4.2))。 |
|             `enable_password_grant` | 可选，默认：`false`。        | 启用 password grant ([RFC 6742 Section 4.3](https://tools.ietf.org/html/rfc6749#section-4.3))。 |
|                  `hide_credentials` | 可选，默认：`false`。        |                                          |
| `accept_http_if_already_terminated` | 可选，默认：`false`。        |                                          |
|                         `anonymous` | 可选，默认：`''`。           |                                          |



##### 使用

1.  创建消费者：
2.  创建应用（第一次）：




#### HMAC Authentication

- [HTTP Signatures](https://tools.ietf.org/html/draft-cavage-http-signatures-00): 签名ietf。



##### 签名认证

```
credentials := "hmac" params
params := keyId "," algorithm ", " headers ", " signature
keyId := "username" "=" plain-string
algorithm := "algorithm" "=" DQUOTE (hmac-sha1) DQUOTE
headers := "headers" "=" plain-string
signature := "signature" "=" plain-string
plain-string   = DQUOTE *( %x20-21 / %x23-5B / %x5D-7E ) DQUOTEs
```



##### 签名参数

|      **参数** | **描述**  |
| ----------: | ------- |
|  `username` | 证书的用户名。 |
| `algorithm` |         |

```javascript
// Authorization: hmac username="mac_bob", algorithm="hmac-sha1", headers="date", signature="daVFG4POE0K723PTigjQTEsqFcA="

const signingString = 'x-date: ' + new Date().toUTCString();
const signature = HMAC(signingString);
```




### 问题

#### 提示 migrations mismatch 无法启动 kong container

尝试了删除 container 后重新创建新的 container，重复几次好了…… 具体原因未知。

[kong]: https://getkong.org	"KONG"
[docker_docs]: https://docs.docker.com/	"docker documentation"



## Joi

对象模型校验。

- [Joi on GitHub](https://github.com/hapijs/joi);

### API



# Redis

## 数据类型

### 字符串（String）

- string 类型是二进制安全的，可以包含任何数据，如 jpg 图片或者序列化对象；
- 是 Redis 最基本的数据类型；
- 一个键最多存储 512 MB。

#### `GET <name>`

获取键为 `<name>` 的值。

```shell
# 已经存在的 Key
$ GET N
$ n's value
# 不存在的 Key
$ GET M
$ (nil)
```



#### `SET <name> <value>`

设置键 `<name>` 的值为 `<value>`。

```
$ SET N "n's value"
  OK
```



### 哈希（Hash）

- 一个键值对集合；
- 一个 string 类型的 field 和 value 的映射表；
- 适合存储对象；
- 每个 hash 可以存储 2^32-1 键值对（40 多亿）。



#### `HMSET key field value [field value …]`

#### `HGETALL key`



### 列表（List）

- 简单的字符串列表，按照插入顺序排序；
- 支持头部（左边）或者尾部（右边）添加元素。



#### ``



### 集合（Set）

- String 类型的无序集合；
- 通过哈希表实现，添加、删除、查找的复杂度是 O(1)；
- 集合中最大的成员数为 2^32-1（40多亿）。

> 存储结构近似于：
>
> ```json
> {
>   "member1": true,
>   "member2": true,
>   "member3": true,
>   // ...
>   "member2^23-1": true
> }
> ```



#### `SADD key member1 [member2]`

向集合添加一个或多个成员。



#### `SMEMBERS key`

返回集合中得所有成员。



#### `



### 有序集合（zset）

- String 类型的无序集合、且不允许重复的成员；
- 每个元素都会关联一个 double 类型的分数，从小到大排序；
- 成员唯一，但是分数可以重复。