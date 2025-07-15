Raha-Xray Software
This software is an API service provider that simplifies the necessary features for various uses of xray-core.
How It Works
To create a user capable of connecting to xray-core, the following steps must be followed:

Add a config.
Add an inbound with the config.id setting to use the added config.
Add a client with the client_inbound setting to use the added inbound.

Special Features

Create various services to work with xray-core.
Connect users to multiple services simultaneously by creating users.
User traffic is automatically managed, and different traffic/time restriction models can be applied.
As an optional feature, inbound, outbound, and client traffic data can be obtained in a format suitable for chart design.
Comprehensive server analysis reports can be obtained.
For use on small servers with low usage, SQLite database can be used, while MySQL can be used for professional applications.

Installation Method
Refer to the installation guide.
Configuration
The initial configuration of this software includes two files, which will be created with default data if they do not exist:

The raha-xray.json file: Software settings are entered in this section.
The xrayDefault.json file: Base xray-core settings that the software uses to apply changes.

Important Notes

These files will be created in the software's main directory (/usr/local/raha-xray/).
During installation or software updates, you will be prompted regarding changes to the raha-xray.conf file. You can also configure this file by using the raha-xray command in a text-based environment and selecting option 4.
These files can also be manually modified, but it is recommended to use automated methods.
These settings are also accessible for modification via the API.

Working with the API
To use this API, you can utilize specialized tools or your own software. The software is accessible via a web service with the port and address specified in the settings.

It is recommended to ensure connection security by providing SSL in the settings.

Security and Authentication
To send requests to this API, you must obtain a token using the raha-xray command in a Linux text-based environment with options 5–8.

A default token is always created during installation, which you can use.

This token must be included in the HTTP Header of all your requests under the name X-Token.
Method Descriptions
This section first explains the main paths and their purposes:
Base path: /api

The base path can be modified in the raha-xray.conf configuration file.




Path
Purpose
xray-core API



/configs
Different service models for use in inbounds
✅


/inbounds
Inbounds
✅


/clients
Users
✅


/outbounds
Outbounds
✅


/rules
Routing rules



/server
Server information



/settings
Settings




Changes applied to paths that interact with xray-core via the API will take effect without restarting xray-core, thus avoiding disconnection of all users.

Configs Path Methods Guide

  Click to view description

Defined model:
type Config struct {
    Id             uint     `json:"id" form:"id" gorm:"primaryKey;autoIncrement"`
    Protocol       Protocol `json:"protocol" form:"protocol"`
    Settings       string   `json:"settings" form:"settings"`
    StreamSettings string   `json:"streamSettings" form:"streamSettings"`
    Sniffing       string   `json:"sniffing" form:"sniffing"`
    ClientSettings string   `json:"clientSettings" form:"clientSettings"`
}

API methods:Base: /api/configs



Method
Path
Action
Request Body



GET
"/"
Get all configs
-


GET
"/get/:id"
Get a config with config.id
-


POST
"/save"
Add/Edit a config
JSON


POST
"/del/:id"
Delete a config
-


Sample JSON for sending to /save:
{
    "id": 1,
    "protocol": "vless",
    "settings": "{\"decryption\":\"none\",\"fallbacks\": []}",
    "streamSettings": "{\"network\": \"tcp\",\"security\": \"none\"}",
    "sniffing": "{\"destOverride\": [\"http\",\"tls\",\"quic\"],\"enabled\": true}",
    "clientSettings": ""
}

Description of /api/configs/save



Parameter
Type
Required
Description



id
uint
No
If not provided, a new record is created; if provided, the specified record is edited


protocol
string
Yes
Inbound protocol


settings
string
Yes
Protocol settings, excluding the users section


streamSettings
string
Yes
Stream settings


clientSettings
string
No
Settings required for user links




Inbounds Path Methods Guide

  Click to view description

Defined model:
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

API methods:Base: /api/inbounds



Method
Path
Action
Request Body



GET
"/"
Get all inbounds
-


GET
"/get/:id"
Get an inbound with inbound.id
-


POST
"/save"
Add/Edit an inbound
JSON


POST
"/del/:id"
Delete an inbound
-


GET
"/traffics/:tag"
Get an inbound's traffics (if enabled)
-


Sample JSON for sending to /save:
{
    "id": 2,
    "name": "inbound-2",
    "enable": true,
    "listen": "",
    "port": 443,
    "configId": 1,
    "tag": "in-2"
}

Description of /api/inbounds/save



Parameter
Type
Required
Description



id
uint
No
If not provided, a new record is created; if provided, the specified record is edited


name
string
No
Inbound name


enable
bool
Yes
Enable/disable status


listen
string
No
IP address the inbound listens to


port
uint
Yes
Port the inbound listens to


configId
uint
Yes
ID of the config used


tag
string
Yes
Inbound tag (must be unique)



When retrieving an inbound, the associated clients will also be listed. ClientInbound model. Config information is also visible.



Clients Path Methods Guide

  Click to view description

Defined model:
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

API methods:Base: /api/clients



Method
Path
Action
Request Body



GET
"/"
Get all clients
-


GET
"/get/:id"
Get a client with client.id
-


POST
"/add"
Add client(s)
JSON


POST
"/update"
Edit a client
JSON


POST
"/inbounds/:id"
Edit inbounds of a client with client.id
JSON


POST
"/del/:id"
Delete a client
-


POST
"/onlines"
Get the list of last online users
-


GET
"/traffics/:tag"
Get traffics (if enabled)
-


Sample JSON for sending to /add:
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

Description of /api/clients/add



Parameter
Type
Required
Description



name
string
Yes
User name (must be unique)


enable
bool
Yes
Enable/disable status


quota
uint64
No
User's allowed data volume in bytes per period


expiry
uint64
No
User expiration time in milliseconds (Unix epoch)


reset
uint
No
Days for resetting the quota per period


once
uint
No
Days for the first period after initial connection


up
uint64
No
User's upload data in bytes


down
uint64
No
User's download data in bytes


remark
string
No
Alias used in links


inbounds
client_inbounds
No
Usable inbounds for the user


Sample JSON for sending to /update:
For updating a user, not all fields need to be sent (version 0.0.2 and above).

Sample with all fields:

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


Sample for resetting user traffic:

{
    "id": 1,
    "up": 0,
    "down": 0
}


Sample for disabling a user:

{
    "id": 1,
    "enable": false
}

Description of /api/clients/update



Parameter
Type
Required
Description



id
uint
Yes
Unique ID of the user to be edited


name
string
No
User name (must be unique)


enable
bool
No
Enable/disable status


quota
uint64
No
User's allowed data volume in bytes per period


expiry
uint64
No
User expiration time in milliseconds (Unix epoch)


reset
uint
No
Days for resetting the quota per period


once
uint
No
Days for the first period after initial connection


up
uint64
No
User's upload data in bytes


down
uint64
No
User's download data in bytes


remark
string
No
Alias used in links


Sample JSON for sending to /inbounds:
[
    {
        "id": 38,
        "inboundId": 1,
        "config": "{\"id\": \"fbcad46d-c9ab-40a3-a3cc-5d1ccf9269b7\"\n}"
    }
]

Description of /api/clients/inbounds



Parameter
Type
Required
Description



id
uint
No
If not provided, a new record is created; if provided, the specified record is edited


inboundId
uint
Yes
Inbound ID (inbound_id)


config
string
Yes
User settings for this inbound




ClientInbounds Model

  Click to view description

type ClientInbound struct {
    Id        uint   `json:"id" form:"id" gorm:"primaryKey;autoIncrement"`
    InboundId uint   `json:"inboundId" form:"inboundId"`
    ClientId  uint   `json:"clientId" form:"clientId"`
    Config    string `json:"config" form:"config"`
}

Sample JSON received in inbounds and clients:
[
    {
        "id": 38,
        "inboundId": 1,
        "clientId": 1,
        "config": "{\"id\": \"fbcad46d-c9ab-40a3-a3cc-5d1ccf9269b7\"\n}"
    }
]



Outbounds Path Methods Guide

  Click to view description

Defined model:
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

API methods:Base: /api/outbounds



Method
Path
Action
Request Body



GET
"/"
Get all outbounds
-


GET
"/get/:id"
Get an outbound with inbound.id
-


POST
"/save"
Add/Edit an outbound
JSON


POST
"/del/:id"
Delete an outbound
-


GET
"/traffics/:tag"
Get an outbound's traffics (if enabled)
-


Sample JSON for sending to /save:
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

Description of /api/outbounds/save



Parameter
Type
Required
Description



id
uint
No
If not provided, a new record is created; if provided, the specified record is edited


sendThrough
string
No
Server IP address for outgoing traffic


protocol
string
Yes
Outbound protocol


settings
string
No
Settings


tag
string
Yes
Outbound tag (must be unique)


streamSettings
string
No
Stream settings


proxySettings
string
No
Forward outbound to another outbound with tag


mux
string
No
Multiplexing settings


More details


Rules Path Methods Guide

  Click to view description

Defined model:
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

API methods:Base: /api/rules



Method
Path
Action
Request Body



GET
"/"
Get all rules
-


GET
"/get/:id"
Get a rule with rule.id
-


POST
"/save"
Add/Edit a rule
JSON


POST
"/del/:id"
Delete a rule
-


Sample JSON for sending to /save:
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

Description of /api/rules/save



Parameter
Type
Required
Description



id
uint
No
If not provided, a new record is created; if provided, the specified record is edited


domainMatcher
string
No
Domain matching algorithm


domain
string
No
Domains


ip
string
No
Destination IP addresses


port
string
No
Destination ports


sourcePort
string
No
Source port


network
string
No
Network used (tcp/udp)


source
string
No
Source IP addresses


user
string
No
Users


inboundTag
string
No
Inbound tags


protocol
string
No
Communication protocols


attrs
string
No
Request header attributes


outboundTag
string
Yes
Outbound tag


balancerTag
string
No
LoadBalancer tag


More details


Server Path Methods Guide

  Click to view description

This method is used to retrieve and send server information.
API methods:Base: /api/server



Method
Path
Action
Request Body



POST
"/status"
Get all statistics of server
-


POST
"/getXrayVersion"
Get xray-core version
-


POST
"/setXrayVersion/:version"
Change the xray-core version
-


POST
"/stopXrayService"
Stop xray-core service
-


POST
"/restartXrayService"
Restart xray-core service
-


POST
"/getConfigJson"
Download the configuration of xray-core
-


POST
"/logs/:app/:count"
Get logs of services
-


POST
"/getNewX25519Cert"
Get reality keys
-


Sample JSON received from /status:
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



Settings Path Methods Guide

  Click to view description

This method is used to retrieve and send software settings.
API methods:Base: /api/settings



Method
Path
Action
Request Body



POST
"/getXrayDefault"
Get xray-core base configuration
-


POST
"/setXrayDefault"
Change xray-core base configuration
JSON


POST
"/getSettings"
Get App Configuration
-


POST
"/setSettings"
Update App Configuration
JSON


POST
"/restartApp"
Restart Raha-Xray
-


Sample JSON for sending to /setXrayDefault:
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

Sample JSON for sending to /setSettings:
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

Description of /api/settings/setSettings



Parameter
Type
Required
Description



listen
string
Yes
IP address for the API service


domain
string
Yes
API domain


port
string
Yes
API port


certFile
string
Yes
Path to the digital certificate file


keyFile
string
Yes
Path to the certificate key file


basePath
string
Yes
Default path (default: /api)


timeLocation
string
Yes
Time zone


dbType
string
Yes
Database type (SQLite/MySQL)


dbAddr
string
Yes
Database address


trafficDays
string
Yes
Days to store usage data


