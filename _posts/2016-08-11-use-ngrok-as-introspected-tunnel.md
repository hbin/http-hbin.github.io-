---
layout: post
title: "用 Ngrok 搭建自用内网穿透服务"
date: 2016-08-11 18:04:55 +0800
comments: true
categories: [Tool]
tag: [Ngrok]
---

最近做微信开发，怎么用开发机调试回调就成了麻烦事。一开始试用了 [localtunnel](https://github.com/localtunnel/localtunnel) 可是这货隔几分钟就会访问不了，需要重启，但是这样又来回来的改回调地址，实在太麻烦了。于是放弃。

正好在 Ruby-China 上搜到一篇环境搭建的[文章](https://ruby-china.org/topics/26443)，里面介绍了另一利器  [ngrok](https://github.com/inconshreveable/ngrok)。

安装过程很简单，照着 https://gist.github.com/lyoshenka/002b7fbd801d0fd21f2f 做就可以了，不过这里有三点需要注意！！！

### 1) 编译 Mac Client
编译客户端 ngrok 时需要注意，Mac 用户编译需要加上：GOOS 和 GOARCH 两个参数，如:

```
GOOS="linux" GOARCH="amd64" make release-server    # For Linux Server
sudo GOOS="darwin" GOARCH="amd64" make release-client   # For Mac Client, 这里提示要 sudo 权限
```

如果你需要其它操作系统的客户端，可以参见完整的 GOOS/GOARCH 列表：https://golang.org/doc/install/source#environment

### 2) 80 接口
由于微信公众号接口只支持80接口，所以在启动 Server 的时候得用 80 端口，也就是：

```
sudo bin/ngrokd -tlsKey=device.key -tlsCrt=device.crt -domain="$NGROK_DOMAIN" -httpAddr=":80" -httpsAddr=“:443"
```

### 3）子域名
如果你像我一样，喜欢用 `my.xxxx.com` or `you.xxxx.com` 子域名作为回调的话，那么编译和启动 Server 的时候 `export NGROK_DOMAIN=xxxx.com`。

### 我的完整命令：

```
############# 登陆服务器

# 编译 Ngrok 服务端和客户端
export NGROK_DOMAIN="xxxx.com"
git clone https://github.com/inconshreveable/ngrok.git
cd ngrok

openssl genrsa -out rootCA.key 2048
openssl req -x509 -new -nodes -key rootCA.key -subj "/CN=$NGROK_DOMAIN" -days 5000 -out rootCA.pem
openssl genrsa -out device.key 2048
openssl req -new -key device.key -subj "/CN=$NGROK_DOMAIN" -out device.csr
openssl x509 -req -in device.csr -CA rootCA.pem -CAkey rootCA.key -CAcreateserial -out device.crt -days 5000

cp rootCA.pem assets/client/tls/ngrokroot.crt

GOOS="linux" GOARCH="amd64" make release-server    # For Linux Server
sudo GOOS="darwin" GOARCH="amd64" make release-client   # For Mac Client, 这里提示要 sudo 权限

# 启动服务端
sudo bin/ngrokd -tlsKey=device.key -tlsCrt=device.crt -domain="$NGROK_DOMAIN" -httpAddr=":80" -httpsAddr=":443"

############# 下载 Client 到本地，在本地执行下面的命令：

# 生成客户端配置文件：
export NGROK_DOMAIN="xxxx.com"
echo -e "server_addr: $NGROK_DOMAIN:4443\ntrust_host_root_certs: false" > ngrok-config

# 启动客户端
./ngrok -config=ngrok-config -subdomain=my 3000
```

然后，我就可以使用 `http://my.xxxx.com` 访问我本地了。

Done.
