## reality配置

### 安装xray

```
apt install xray
```

### 生成uuid和密钥对

```
xray uuid
6adcc59e-ac9e-46f3-b28e-57882c308b61


xray  x25519
Private key: oDM7rQwU-IEXaQkkvpPw3qa_EX0zNfkwHmn1xupklk0
Public key: Jz0MYxPV8Xa3bD4Xke9UXSilA1Eq37smrOZtTwGbizs
```



### 配置

路径：` /usr/local/etc/xray/config.json`

```javascript
{
  "log": {
          "access": "/var/log/xray/access.log",
          "error": "/var/log/xray/error.log",
          "loglevel": "info"
  },
  "inbounds": [
     {
            "tag": "dokodemo-in",
            "port": 443,
            "protocol": "dokodemo-door",
            "settings": {
                "address": "127.0.0.1",
                "port": 4431,  // 指向内网中的 reality 端口，示例是这个端口，如果要自己修改了记得这里和下面的 reality 入站都要修改
                "network": "tcp"
            },
            "sniffing": { // 这里的 sniffing 不是多余的，别乱动
                "enabled": true,
                "destOverride": [
                    "tls"
                ],
                "routeOnly": true
            }
    },
    {
      "tag": "reality",
      "listen": "0.0.0.0",
      "port": 443,
      "protocol": "vless",
      "settings": {
        "clients": [
          {
            "id": "6adcc59e-ac9e-46f3-b28e-57882c308b61",//这里用生成的uuid       
            "flow": "xtls-rprx-vision"
          }
        ],
        "decryption": "none"
      },
      "streamSettings": {
        "network": "tcp",
        "security": "reality",
        "realitySettings": {
          "show": false,
          "dest": "www.vultr.com:443", // 必填，这里是你伪装的网站
          "serverNames": [ // 必填，客户端可用的 serverName 列表，用你伪装的就好
            "www.vultr.com",
            "vultr.com"
          ],
          "privateKey": "oDM7rQwU-IEXaQkkvpPw3qa_EX0zNfkwHmn1xupklk", //这里用上面生成的私钥
          "shortIds": [ // 必填，客户端可用的 shortId 列表，可用于区分不同的客户端
            "1111", //如果是空串表示所有id都行
            "aabb" // 0 到 f，长度为 2 的倍数，长度上限为 16
          ]
        }
      }
    }
  ],
  "outbounds": [
    {
      "protocol": "freedom" 
    }
  ]
}
```

### 启动

```
systemctl restart xray
```

### 客户端配置

```yml
proxies:
  - name: "pr_ccp"
    server: 10.11.41.11 #vps ip
    port: 443
    reality-opts:
      public-key: oDM7rQwU-IEXaQkkvpPw3qa_EX0zNfkwHmn1xupklk0 #上面生成的私钥
      short-id: "aabb" #服务端配置文件里的shortid之一
    client-fingerprint: chrome #必须
    type: vless
    uuid: 6adcc59e-ac9e-46f3-b28e-57882c308b61 #前面生成的uuid
    tls: true
    tfo: false
    flow: xtls-rprx-vision
    skip-cert-verify: false
    servername: www.vultr.com #服务端配置里允许的servername，用伪装的网站的即可
    network: tcp
```

URL格式

```
vless://{uuid}@{server}:{port}?encryption=none&security=reality&sni={servername}&type=tcp&flow=xtls-rprx-vision&pbk={public-key}&sid={short-id}&fp=chrome#name


{}的内容用客户端配置对应字段替换
```

### 扩展

其实上面就可以用了，但是xray占用了443端口，而且有爬虫或者其他的服务来扫服务器的端口，如果伪装的网站是cloudflare这种，就可能被偷跑流量（有类似cdn或者github page这种功能的网站），所以前面需要加一层过滤。

路径：` /usr/local/etc/xray/config.json`

```javascript
{
  "log": {
          "access": "/var/log/xray/access.log",
          "error": "/var/log/xray/error.log",
          "loglevel": "info"
  },
  "inbounds": [
    {
      "tag": "dokodemo-in",
      "port": 443,
      "protocol": "dokodemo-door",
      "settings": {
          "address": "127.0.0.1",
          "port": 4431,  // 指向内网中的 reality 端口，示例是这个端口，如果要自己修改了记得这里和下面的 reality 入站都要修改
          "network": "tcp"
      },
      "sniffing": { // 这里的 sniffing 不是多余的，别乱动
          "enabled": true,
          "destOverride": [
              "tls"
          ],
          "routeOnly": true
      }
    },
    {
      "tag": "reality",
      "listen": "127.0.0.1",
      "port": 4431,
      "protocol": "vless",
      "settings": {
        "clients": [
          {
            "id": "6adcc59e-ac9e-46f3-b28e-57882c308b61",//这里用生成的uuid       
            "flow": "xtls-rprx-vision"
          }
        ],
        "decryption": "none"
      },
      "streamSettings": {
        "network": "tcp",
        "security": "reality",
        "realitySettings": {
          "show": false,
          "dest": "www.vultr.com:443", // 必填，这里是你伪装的网站
          "serverNames": [ // 必填，客户端可用的 serverName 列表，用你伪装的就好
            "www.vultr.com",
            "vultr.com"
          ],
          "privateKey": "oDM7rQwU-IEXaQkkvpPw3qa_EX0zNfkwHmn1xupklk", //这里用上面生成的私钥
          "shortIds": [ // 必填，客户端可用的 shortId 列表，可用于区分不同的客户端
            "1111", 
            "aabb" // 0 到 f，长度为 2 的倍数，长度上限为 16
          ]
        }
      }
    }
  ],
   "outbounds": [
        {
            "protocol": "freedom",
            "tag": "direct"
        },
        {
            "protocol": "blackhole",
            //如果需要443端口，使用下面的配置，自己的服务用4430端口
            // "protocol": "freedom",
            // "redirect": "127.0.0.1:4430",
            "tag": "block"
        }
    ],
    "routing": {
        "rules": [
            {
                "inboundTag": [
                    "dokodemo-in"
                ],
                // 重要，这个域名列表需要和 realitySettings 的 serverNames 保持一致
                "domain": [
                    "www.vultr.com"
                ],
                "outboundTag": "direct"
            },
            {
                "inboundTag": [
                    "dokodemo-in"
                ],
                "outboundTag": "block"
            }
        ]
    }
}
```




