# Raha-Xray 软件

Raha-Xray 软件是一个API服务提供者，简化了 [xray-core](https://github.com/XTLS/Xray-core) 的各种使用场景所需的功能。

# 工作原理

要创建一个能够连接到 xray-core 的用户，必须按照以下步骤操作：
1. 添加一个配置。
2. 添加一个入站（inbound），并设置 config.id 以使用已添加的配置。
3. 添加一个客户端（client），并设置 client_inbound 以使用已添加的入站。

## 独特功能

1. 创建多种服务以与 xray-core 协同工作。
2. 通过创建用户，将其同时连接到多个服务。
3. 用户流量自动管理，可应用不同的流量/时间限制模型。
4. 作为可选功能，可获取入站、出站和客户端流量数据，格式适合用于图表设计。
5. 可获取全面的服务器分析报告。
6. 对于小型服务器或低使用量场景，可使用 SQLite 数据库；对于专业应用，可使用 MySQL。

# 安装方法

请参阅 [安装指南](https://github.com/Raha-Project/Raha/blob/main/README-FA.md#installation-methods)。

# 配置

该软件的初始配置包括两个文件，如果文件不存在，将以默认数据创建：
1. `raha-xray.json` 文件：软件设置在此部分输入。
2. `xrayDefault.json` 文件：xray-core 的基础设置，软件用其应用更改。

## 重要说明

* 这些文件将在软件主目录（/usr/local/raha-xray/）中创建。
* 在安装或软件更新期间，系统会询问有关 `raha-xray.conf` 文件更改的问题。您也可以通过在文本环境中使用 `raha-xray` 命令并选择选项 4 来配置此文件。
* 这些文件也可以手动修改，但建议使用自动化方法。
* 这些设置也可以通过 API 进行修改。

# 使用 API

要使用此 API，可以使用专用工具或您自己的软件。该软件通过设置中指定的端口和地址，通过 Web 服务访问。
* 建议在设置中提供 SSL 以确保连接安全。

## 安全与认证

要向此 API 发送请求，必须通过在 Linux 文本环境中使用 `raha-xray` 命令并选择选项 5-8 来获取令牌。
> 安装时始终会创建一个默认令牌，您可以使用该令牌。

所有请求的 HTTP 头部中必须包含此令牌，名称为 `X-Token`。

## 方法描述

本节首先解释主要路径及其用途：

Base path: /api
> 基础路径可在 `raha-xray.conf` 配置文件中修改。

| Path          | Purpose                                           | xray-core API |
|---------------|--------------------------------------------------|---------------|
| `/configs`    | Different service models for use in inbounds      | ✅            |
| `/inbounds`   | Inbounds                                         | ✅            |
| `/clients`    | Users                                            | ✅            |
| `/outbounds`  | Outbounds                                        | ✅            |
| `/rules`      | Routing rules                                    |               |
| `/server`     | Server information                               |               |
| `/settings`   | Settings                                         |               |

* 对通过 API 与 xray-core 交互的路径应用更改时，无需重启 xray-core，因此不会断开所有用户的连接。

### Configs 路径方法指南

<details>
  <summary>点击查看描述</summary>

定义的模型：
```go
type Config struct {
    Id             uint     `json:"id" form:"id" gorm:"primaryKey;autoIncrement"`
    Protocol       Protocol `json:"protocol" form:"protocol"`
    Settings       string   `json:"settings" form:"settings"`
    StreamSettings string   `json:"streamSettings" form:"streamSettings"`
    Sniffing       string   `json:"sniffing" form:"sniffing"`
    ClientSettings string   `json:"clientSettings" form:"clientSettings"`
}
```
**API 方法：**
Base: /api/configs

| Method | Path                            | Action                                    | Request Body |
|--------|---------------------------------|-------------------------------------------|--------------|
| `GET`  | `"/"`                           | Get all configs                           | -            |
| `GET`  | `"/get/:id"`                    | Get a config with config.id               | -            |
| `POST` | `"/save"`                       | Add/Edit a config                         | [JSON](#description-apiconfigssave)     |
| `POST` | `"/del/:id"`                    | Delete a config                           | -            |

##### /api/configs/save 的样本 JSON：
```json
{
    "id": 1,
    "protocol": "vless",
    "settings": "{\"decryption\":\"none\",\"fallbacks\": []}",
    "streamSettings": "{\"network\": \"tcp\",\"security\": \"none\"}",
    "sniffing": "{\"destOverride\": [\"http\",\"tls\",\"quic\"],\"enabled\": true}",
    "clientSettings": ""
}
```
##### /api/configs/save 描述
| Parameter         | Type   | Required | Description                                      |
|-------------------|--------|----------|--------------------------------------------------|
| `id`              | uint   | No       | If not provided, a new record is created; if provided, the specified record is edited |
| `protocol`        | string | Yes      | Inbound protocol                                 |
| `settings`        | string | Yes      | Protocol settings, excluding the users section   |
| `streamSettings`  | string | Yes      | [Stream settings](https://xtls.github.io/en/config/transport.html#streamsettingsobject) |
| `clientSettings`  | string | No       | Settings required for user links                 |

</details>

### Inbounds 路径方法指南
<details>
  <summary>点击查看描述</summary>

定义的模型：
```go
type Inbound struct {
    Id     uint   `json:"id" form:"id" gorm:"primaryKey;autoIncrement"`
    Name   string `json:"name" form:"name"`
    Enable bool   `json:"enable" form:"enable" gorm:"default:true"`

    // config part
    Listen   string `json:"listen" form:"listen"`
    Port     uint   `json:"port" form:"port"`
    ConfigId uint   `gorm:"not null" json:"configId" form:"configId"`
    Config   Config `gorm:"foreignKey:ConfigId;references:Id" json:"config"`
    Tag      string `gorm:"unique" json:"tag" form:"tag"`

    // clients part
    ClientInbounds []ClientInbound `gorm:"foreignKey:InboundId;references:Id" json:"clients"`
}
```
**API 方法：**
Base: /api/inbounds

| Method | Path                            | Action                                    | Request Body |
|--------|---------------------------------|-------------------------------------------|--------------|
| `GET`  | `"/"`                           | Get all inbounds                          | -            |
| `GET`  | `"/get/:id"`                    | Get an inbound with inbound.id            | -            |
| `POST` | `"/save"`                       | Add/Edit an inbound                       | [JSON](#description-apiinboundssave)     |
| `POST` | `"/del/:id"`                    | Delete an inbound                         | -            |
| `GET`  | `"/traffics/:tag"`              | Get an inbound's traffics (if enabled)    | -            |

##### /api/inbounds/save 的样本 JSON：
```json
{
    "id": 2,
    "name": "inbound-2",
    "enable": true,
    "listen": "",
    "port": 443,
    "configId": 1,
    "tag": "in-2"
}
```
##### /api/inbounds/save 描述
| Parameter  | Type   | Required | Description                                      |
|------------|--------|----------|--------------------------------------------------|
| `id`       | uint   | No       | If not provided, a new record is created; if provided, the specified record is edited |
| `name`     | string | No       | Inbound name                                     |
| `enable`   | bool   | Yes      | Enable/disable status                            |
| `listen`   | string | No       | IP address the inbound listens to               |
| `port`     | uint   | Yes      | Port the inbound listens to                     |
| `configId` | uint   | Yes      | ID of the config used                           |
| `tag`      | string | Yes      | Inbound tag (must be unique)                    |

* 检索入站时，关联的客户端也将列出。[ClientInbound 模型](#client_inbounds-model)。配置信息也可见。

</details>

### Clients 路径方法指南
<details>
  <summary>点击查看描述</summary>

定义的模型：
```go
type Client struct {
    Id     uint   `json:"id" form:"id" gorm:"primaryKey;autoIncrement"`
    Name   string `json:"name" form:"name" gorm:"unique"`
    Enable bool   `json:"enable" form:"enable" gorm:"default:true"`
    Quota  uint64 `json:"quota" form:"quota" gorm:"default:0"`
    Expiry uint64 `json:"expiry" form:"expiry" gorm:"default:0"`
    Reset  uint   `json:"reset" from:"reset" gorm:"default:0"`
    Once   uint   `json:"once" from:"once" gorm:"default:0"`
    Up     uint64 `json:"up" form:"up" gorm:"default:0"`
    Down   uint64 `json:"down" form:"down" gorm:"default:0"`
    Remark string `json:"remark" form:"remark"`

    // inbounds part
    ClientInbounds []ClientInbound `gorm:"foreignKey:ClientId;references:Id" json:"inbounds"`
}
```
**API 方法：**
Base: /api/clients

| Method | Path                            | Action                                    | Request Body |
|--------|---------------------------------|-------------------------------------------|--------------|
| `GET`  | `"/"`                           | Get all clients                           | -            |
| `GET`  | `"/get/:id"`                    | Get a client with client.id               | -            |
| `POST` | `"/add"`                        | Add client(s)                             | [JSON](#description-apiclientsadd) |
| `POST` | `"/update"`                     | Edit a client                             | [JSON](#description-apiclientsupdate) |
| `POST` | `"/inbounds/:id"`               | Edit inbounds of a client with client.id  | [JSON](#description-apiclientsinbounds) |
| `POST` | `"/del/:id"`                    | Delete a client                           | -            |
| `POST` | `"/onlines"`                    | Get the list of last online users         | -            |
| `GET`  | `"/traffics/:tag"`              | Get traffics (if enabled)                | -            |

##### /api/clients/add 的样本 JSON：
```json
[
    {
        "name": "client1",
        "enable": true,
        "quota": 0,
        "expiry": 0,
        "reset": 0,
        "once": 0,
        "up": 0,
        "down": 0,
        "remark": "exampleRemark1",
        "inbounds": [
            {
                "inboundId": 1,
                "config": "{\n  \"id\": \"62f65b16-b6d3-48af-9d13-8c200b002505\"\n}"
            },
            {
                "inboundId": 2,
                "config": "{\n  \"password\": \"fc8f3f8651bc\"\n}"
            }
        ]
    },
    {
        "name": "client2",
        "enable": true,
        "quota": 102400,
        "expiry": 0,
        "reset": 0,
        "once": 0,
        "up": 0,
        "down": 0,
        "remark": "Remark2",
        "inbounds": [
            {
                "inboundId": 2,
                "config": "{\n  \"password\": \"8c200b002505\"\n}"
            }
        ]
    }
]
```
##### /api/clients/add 描述
| Parameter  | Type   | Required | Description                                      |
|------------|--------|----------|--------------------------------------------------|
| `name`     | string | Yes      | User name (must be unique)                       |
| `enable`   | bool   | Yes      | Enable/disable status                            |
| `quota`    | uint64 | No       | User's allowed data volume in bytes per period   |
| `expiry`   | uint64 | No       | User expiration time in milliseconds (Unix epoch) |
| `reset`    | uint   | No       | Days for resetting the quota per period         |
| `once`     | uint   | No       | Days for the first period after initial connection |
| `up`       | uint64 | No       | User's upload data in bytes                      |
| `down`     | uint64 | No       | User's download data in bytes                    |
| `remark`   | string | No       | Alias used in links                             |
| `inbounds` | client_inbounds | No | Usable inbounds for the user                     |

##### /api/clients/update 的样本 JSON：

更新用户时，无需发送所有字段（版本 0.0.2 及以上）。

* 包含所有字段的样本：
```json
{
    "id": 1,
    "name": "Test1",
    "enable": true,
    "quota": 0,
    "expiry": 0,
    "reset": 0,
    "once": 0,
    "up": 0,
    "down": 0,
    "remark": "theFirstTest"
}
```
* 重置用户流量的样本：
```json
{
    "id": 1,
    "up": 0,
    "down": 0
}
```
* 禁用用户的样本：
```json
{
    "id": 1,
    "enable": false
}
```

##### /api/clients/update 描述
| Parameter  | Type   | Required | Description                                      |
|------------|--------|----------|--------------------------------------------------|
| `id`       | uint   | Yes      | Unique ID of the user to be edited               |
| `name`     | string | No       | User name (must be unique)                       |
| `enable`   | bool   | No       | Enable/disable status                            |
| `quota`    | uint64 | No       | User's allowed data volume in bytes per period   |
| `expiry`   | uint64 | No       | User expiration time in milliseconds (Unix epoch) |
| `reset`    | uint   | No       | Days for resetting the quota per period         |
| `once`     | uint   | No       | Days for the first period after initial connection |
| `up`       | uint64 | No       | User's upload data in bytes                      |
| `down`     | uint64 | No       | User's download data in bytes                    |
| `remark`   | string | No       | Alias used in links                             |

##### /api/clients/inbounds 的样本 JSON：
```json
[
    {
        "id": 38,
        "inboundId": 1,
        "config": "{\"id\": \"fbcad46d-c9ab-40a3-a3cc-5d1ccf9269b7\"\n}"
    }
]
```

##### /api/clients/inbounds 描述
| Parameter   | Type   | Required | Description                                      |
|-------------|--------|----------|--------------------------------------------------|
| `id`        | uint   | No       | If not provided, a new record is created; if provided, the specified record is edited |
| `inboundId` | uint   | Yes      | Inbound ID (inbound_id)                          |
| `config`    | string | Yes      | User settings for this inbound                   |

</details>

### ClientInbounds 模型
<details>
  <summary>点击查看描述</summary>

```go
type ClientInbound struct {
    Id        uint   `json:"id" form:"id" gorm:"primaryKey;autoIncrement"`
    InboundId uint   `json:"inboundId" form:"inboundId"`
    ClientId  uint   `json:"clientId" form:"clientId"`
    Config    string `json:"config" form:"config"`
}
```
##### inbounds 和 clients 接收的样本 JSON：
```json
[
    {
        "id": 38,
        "inboundId": 1,
        "clientId": 1,
        "config": "{\"id\": \"fbcad46d-c9ab-40a3-a3cc-5d1ccf9269b7\"\n}"
    }
]
```
</details>

### Outbounds 路径方法指南
<details>
  <summary>点击查看描述</summary>

定义的模型：
```go
type Outbound struct {
    Id             uint   `json:"id" form:"id" gorm:"primaryKey;autoIncrement"`
    SendThrough    string `json:"sendThrough" form:"sendThrough"`
    Protocol       string `json:"protocol" form:"protocol"`
    Settings       string `json:"settings" form:"settings"`
    Tag            string `gorm:"unique" json:"tag" form:"tag"`
    StreamSettings string `json:"streamSettings" form:"streamSettings"`
    ProxySettings  string `json:"proxySettings" form:"proxySettings"`
    Mux            string `json:"mux" form:"mux"`
}
```
**API 方法：**
Base: /api/outbounds

| Method | Path                            | Action                                    | Request Body |
|--------|---------------------------------|-------------------------------------------|--------------|
| `GET`  | `"/"`                           | Get all outbounds                         | -            |
| `GET`  | `"/get/:id"`                    | Get an outbound with inbound.id           | -            |
| `POST` | `"/save"`                       | Add/Edit an outbound                      | [JSON](#description-apioutboundssave)     |
| `POST` | `"/del/:id"`                    | Delete an outbound                        | -            |
| `GET`  | `"/traffics/:tag"`              | Get an outbound's traffics (if enabled)   | -            |

##### /api/outbounds/save 的样本 JSON：
```json
{
    "id": 1,
    "sendThrough": "",
    "protocol": "freedom",
    "settings": "{\"domainStrategy\": \"UseIPv6\"}",
    "tag": "direct-IPv6",
    "streamSettings": "",
    "proxySettings": "",
    "mux": ""
}
```
##### /api/outbounds/save 描述
| Parameter       | Type   | Required | Description                                      |
|-----------------|--------|----------|--------------------------------------------------|
| `id`            | uint   | No       | If not provided, a new record is created; if provided, the specified record is edited |
| `sendThrough`   | string | No       | Server IP address for outgoing traffic           |
| `protocol`      | string | Yes      | Outbound protocol                                |
| `settings`      | string | No       | Settings                                        |
| `tag`           | string | Yes      | Outbound tag (must be unique)                   |
| `streamSettings`| string | No       | Stream settings                                 |
| `proxySettings` | string | No       | Forward outbound to another outbound with tag    |
| `mux`           | string | No       | Multiplexing settings                           |

[更多详情](https://xtls.github.io/en/config/outbound.html)

</details>

### Rules 路径方法指南
<details>
  <summary>点击查看描述</summary>

定义的模型：
```go
type Rule struct {
    Id            uint   `json:"id" form:"id" gorm:"primaryKey;autoIncrement"`
    DomainMatcher string `json:"domainMatcher" form:"domainMatcher"`
    Type          string `json:"type" form:"type"`
    Domain        string `json:"domain" form:"domain"`
    Ip            string `json:"ip" form:"ip"`
    Port          string `json:"port" form:"port"`
    SourcePort    string `json:"sourcePort" form:"sourcePort"`
    Network       string `json:"network" form:"network"`
    Source        string `json:"source" form:"source"`
    User          string `json:"user" form:"user"`
    InboundTag    string `json:"inboundTag" form:"inboundTag"`
    Protocol      string `json:"protocol" form:"protocol"`
    Attrs         string `json:"attrs" form:"attrs"`
    OutboundTag   string `json:"outboundTag" form:"outboundTag"`
    BalancerTag   string `json:"balancerTag" form:"balancerTag"`
}
```
**API 方法：**
Base: /api/rules

| Method | Path                            | Action                                    | Request Body |
|--------|---------------------------------|-------------------------------------------|--------------|
| `GET`  | `"/"`                           | Get all rules                             | -            |
| `GET`  | `"/get/:id"`                    | Get a rule with rule.id                   | -            |
| `POST` | `"/save"`                       | Add/Edit a rule                           | [JSON](#description-apirulessave) |
| `POST` | `"/del/:id"`                    | Delete a rule                             | -            |

##### /api/rules/save 的样本 JSON：
```json
{
    "id": 1,
    "domainMatcher": "hybrid",
    "type": "field",
    "domain": "[\"baidu.com\", \"qq.com\", \"geosite:cn\"]",
    "ip": "[\"::/0\"]",
    "port": "53,443,1000-2000",
    "sourcePort": "53,443,1000-2000",
    "network": "tcp",
    "source": "[\"10.0.0.1\"]",
    "user": "[\"love@xray.com\"]",
    "inboundTag": "[\"tag-vmess\"]",
    "protocol": "[\"http\", \"tls\", \"bittorrent\"]",
    "attrs": "{ \":method\": \"GET\" }",
    "outboundTag": "direct",
    "balancerTag": "balancer"
}
```
##### /api/rules/save 描述
| Parameter       | Type   | Required | Description                                      |
|-----------------|--------|----------|--------------------------------------------------|
| `id`            | uint   | No       | If not provided, a new record is created; if provided, the specified record is edited |
| `domainMatcher` | string | No       | Domain matching algorithm                       |
| `domain`        | string | No       | Domains                                         |
| `ip`            | string | No       | Destination IP addresses                        |
| `port`          | string | No       | Destination ports                               |
| `sourcePort`    | string | No       | Source port                                     |
| `network`       | string | No       | Network used (tcp/udp)                          |
| `source`        | string | No       | Source IP addresses                             |
| `user`          | string | No       | Users                                           |
| `inboundTag`    | string | No       | Inbound tags                                    |
| `protocol`      | string | No       | Communication protocols                         |
| `attrs`         | string | No       | Request header attributes                       |
| `outboundTag`   | string | Yes      | Outbound tag                                    |
| `balancerTag`   | string | No       | LoadBalancer tag                                |

[更多详情](https://xtls.github.io/en/config/routing.html#ruleobject)

</details>

### Server 路径方法指南
<details>
  <summary>点击查看描述</summary>

此方法用于检索和发送服务器信息。

**API 方法：**
Base: /api/server

| Method | Path                            | Action                                    | Request Body |
|--------|---------------------------------|-------------------------------------------|--------------|
| `POST` | `"/status"`                     | Get all statistics of server              | -            |
| `POST` | `"/getXrayVersion"`             | Get xray-core version                     | -            |
| `POST` | `"/setXrayVersion/:version"`    | Change the xray-core version              | -            |
| `POST` | `"/stopXrayService"`            | Stop xray-core service                    | -            |
| `POST` | `"/restartXrayService"`         | Restart xray-core service                 | -            |
| `POST` | `"/getConfigJson"`              | Download the configuration of xray-core   | -            |
| `POST` | `"/logs/:app/:count"`           | Get logs of services                      | -            |
| `POST` | `"/getNewX25519Cert"`           | Get reality keys                          | -            |

##### /status 接收的样本 JSON：
```json
{
    "cpu": 2.676659528908924,
    "cpuCount": 4,
    "mem": {
        "current": 691990528,
        "total": 4123820032
    },
    "swap": {
        "current": 0,
        "total": 1073737728
    },
    "disk": {
        "current": 30789402624,
        "total": 62671097856
    },
    "xray": {
        "state": "running",
        "version": "1.8.4"
    },
    "uptime": 42755,
    "loads": [
        0.08,
        0.03,
        0
    ],
    "tcpCount": 7,
    "udpCount": 3,
    "netIO": {
        "up": 0,
        "down": 0
    },
    "netTraffic": {
        "sent": 21401,
        "recv": 51696
    },
    "appStats": {
        "threads": 10,
        "mem": 15045896
    },
    "xrayStats": {
        "NumGoroutine": 29,
        "NumGC": 572,
        "Alloc": 4047584,
        "TotalAlloc": 998091656,
        "Sys": 36017416,
        "Mallocs": 2650983,
        "Frees": 2637414,
        "LiveObjects": 13569,
        "PauseTotalNs": 1274427636,
        "Uptime": 42426
    },
    "hostInfo": {
        "hostname": "1681e5650ba4",
        "ipv4": "172.18.0.5/16 ",
        "ipv6": ""
    }
}
```

</details>

### Settings 路径方法指南
<details>
  <summary>点击查看描述</summary>

此方法用于检索和发送软件设置。

**API 方法：**
Base: /api/settings

| Method | Path                            | Action                                    | Request Body |
|--------|---------------------------------|-------------------------------------------|--------------|
| `POST` | `"/getXrayDefault"`             | Get xray-core base configuration          | -            |
| `POST` | `"/setXrayDefault"`             | Change xray-core base configuration       | JSON         |
| `POST` | `"/getSettings"`                | Get App Configuration                     | -            |
| `POST` | `"/setSettings"`                | Update App Configuration                  | [JSON](#description-apisettingssetsettings) |
| `POST` | `"/restartApp"`                 | Restart Raha-Xray                         | -            |

##### /setXrayDefault 的样本 JSON：
```json
{
  "log": {
    "loglevel": "warning",
    "access": "/dev/null"
  },
  "api": {
    "tag": "api",
    "services": ["HandlerService", "LoggerService", "StatsService"]
  },
  "inbounds": [
    {
      "tag": "api",
      "listen": "127.0.0.1",
      "port": 62789,
      "protocol": "dokodemo-door",
      "settings": {
        "address": "127.0.0.1"
      }
    }
  ],
  "outbounds": [
    {
      "tag": "direct",
      "protocol": "freedom",
      "settings": {}
    },
    {
      "tag": "blocked",
      "protocol": "blackhole",
      "settings": {}
    }
  ],
  "policy": {
    "levels": {
      "0": {
        "statsUserDownlink": true,
        "statsUserUplink": true
      }
    },
    "system": {
      "statsInboundDownlink": true,
      "statsInboundUplink": true,
      "statsOutboundUplink": true,
      "statsOutboundDownlink": true
    }
  },
  "routing": {
    "domainStrategy": "AsIs",
    "rules": [
      {
        "type": "field",
        "inboundTag": ["api"],
        "outboundTag": "api"
      },
      {
        "type": "field",
        "outboundTag": "blocked",
        "ip": ["geoip:private"]
      },
      {
        "type": "field",
        "outboundTag": "blocked",
        "protocol": ["bittorrent"]
      }
    ]
  },
  "stats": {}
}
```

##### /setSettings 的样本 JSON：
```json
{
    "listen": "",
    "domain": "",
    "port": 8080,
    "certFile": "",
    "keyFile": "",
    "basePath": "/api",
    "timeLocation": "Asia/Tehran",
    "dbType": "mysql",
    "dbAddr": "root:rahaXray@tcp(localhost:3306)",
    "trafficDays": 30
}
```
##### /api/settings/setSettings 描述
| Parameter       | Type   | Required | Description                                      |
|-----------------|--------|----------|--------------------------------------------------|
| `listen`        | string | Yes      | IP address for the API service                   |
| `domain`        | string | Yes      | API domain                                      |
| `port`          | string | Yes      | API port                                        |
| `certFile`      | string | Yes      | Path to the digital certificate file            |
| `keyFile`       | string | Yes      | Path to the certificate key file                |
| `basePath`      | string | Yes      | Default path (default: /api)                    |
| `timeLocation`  | string | Yes      | Time zone                                       |
| `dbType`        | string | Yes      | Database type (SQLite/MySQL)                    |
| `dbAddr`        | string | Yes      | Database address                                |
| `trafficDays`   | string | Yes      | Days to store usage data                        |

</details>
