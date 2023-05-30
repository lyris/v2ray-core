## 主要改进

增加了一种针对plain http1.1的转发方式，而不是通过CONNECT代理请求使用HTTP代理服务器，使之能通过有端口限制的http服务器。

在分布式系统中，一些服务程序运行时的提供的简易的web dashboard供开发人员内部查看，其端口是往往随机的[1-65535]（一台机器上会部署大量服务），并不一定是80和443，更不可能是固定端口。

为了保密起见，这些运行时的web仪表盘往往需要通过跳板HTTP代理服务器才能访问或加速。

如果直接走V2ray的http代理，因为内部实现是走的正统CONNECT代理请求，会被跳板的HTTP代理服务器block掉。

虽然在Firefox上可以通过Foxyproxy(Chrome上的SwithyOmega同理)，定义一条白名单规则（通常是第一条）如让这些内部www服务强制走代理，剩下的走v2ray来变相满足需求:

```foxyproxy
   ^(?:[^:@/]+(?::[^@/]+)?@)?(?:192\.168\.\d+\.\d+|10\.\d+\.\d+\.\d+|172\.(?:1[6789]|2[0-9]|3[01])\.\d+\.\d+)(?::\d+)?(?:/.*)?$
```

应用本补丁后可以直接通过v2ray来使用http代理服务器了，一个v2ray搞定一切。

这应该属于小众需求，相关进一步的背景可以查看如下[issue](https://github.com/v2fly/v2ray-core/issues/1306)

## 配置


在outbound protocol为"http"的settings部分增加plainHttp(true|false)选项。

## 举例
当plainHttp设置为true时，
浏览器只需要使用socket代理(127.0.0.1:1080),就可以通过http代理服务器192.168.100.22:3128访问如 http://10.200.2.238:41115/xxx/monitor/flight/index.do 这样的服务了：
<!-- 此处 yaml 仅用作语法高亮，实际内容为 json -->
```yaml
    {
    "log": {
        "loglevel": "info"
    },
    "inbounds": [
        {
            "port": 1080,
            "listen": "127.0.0.1",
            "protocol": "socks",
            "sniffing": {
                "enabled": true,
                "destOverride": [
                    "http",
                    "tls"
                ]
            },
            "settings": {
                "auth": "noauth",
                "udp": true,
                "ip": "127.0.0.1"
            }
        }
    ],
    "outbounds": [
        {
            "protocol": "http",
            "tag": "proxy-test",
            "settings": {
                "servers": [
                    {
                        "address": "192.168.100.22",
                        "port": 3128,
                        "users": [
                            {
                              "user": "my-username",
                              "pass": "my-password",
                              "plainHttp": true  // newly introduced config
                            }
                          ]
                    }
                ],
                // or put it here "plainHttp": true
            }
        },
        {
            "protocol": "freedom",
            "tag": "direct",
            "settings": {
                "domainStrategy": "UseIP"
            }
        }
    ],
    "routing": {
        "domainStrategy": "IPOnDemand",
        "rules": [
            {
                "type": "field",
                "ip": [
                  "10.0.0.0/8"
                ],
                "network": "tcp",
                "outboundTag": "proxy-test"
            }, 
            {
                "type": "field",
                "outboundTag": "direct",
                "network": "udp,tcp"
            }
        ]
    }
}
```

## 注意
只针对目标网页http1.1有效，对http2不适用，对加密web服务https也不适用，仍然走CONNECT方式。

## 最后

祝你玩的愉快！
