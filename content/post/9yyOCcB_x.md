---
title: NAS自签名证书的一种实践
date: '2023-02-07T00:00:00+08:00'
url: /post/9yyOCcB_x/
tags:
- nas
- ssl
feature: https://i1.wp.com/www.mojoengine.com/wp-content/uploads/2020/08/featured-ssl.png?fit=1200%2C628&ssl=1
---

这篇文章针对个人或家庭用途的 NAS 等私有服务器场景，阐述基于没有公共域名的背景下，虚构 CA 并自签名证书的一种实践过程、问题及解决办法，实现纯 IP 地址的 https 访问。涉及的主题包括：利用 openssl 签发证书、在域名与 IP 地址混合访问场景（也包括纯 IP 访问场景）启用 https、iOS 添加自签名证书等。

通常消费级 NAS 提供 Web 管理界面，用户通常会在 NAS 上发布 Web 应用页面或接口，例如 Jellyfin、owncloud、aria2 jsonrpc 等。如果在酒店、咖啡厅、公司等环境通过公共网络以 http、ws 等非加密方式访问这些服务，用户名和口令是很容易被截取，进而形成安全问题。因此很有必要对需要在外部网络访问的服务进行 SSL 加密。

![使用多种 IP 地址访问均能确保安全](https://s2.loli.net/2023/02/08/KDtEepqziNmyrY5.png)

## 1 需求

1. 要求采用先“创建 CA 、为其他服务器签发证书”的模式。CA 是指证书管理机构（Certificate Autority），在公共 PKI 体系下，CA 是具有公共信任的机构，而在自签名场景中，CA 是虚构的信任中心。采用该模式的目的是为了便于管理，终端设备仅需信任一个根证书后就可以自动信任由其签发的其他证书。
2. 在终端设备信任根证书后，由根证书签发的其他服务器证书在终端设备上不会弹出各种类型的证书错误页面。
3. 以 QNAP NAS 的 QTS 系统为例，自签名证书应能应用到所有可用的 HTTP 访问方式上。

- 威联通免费域名 `DN.1` `exampleserver.myqnapcloud.cn`：威联通（中国）DDNS 服务提供的域名。由于我没有 IPv4 公网地址，此处是解析 IPv6 公网地址。
- 家庭局域网 `IP.1` `192.168.50.200`。
- 设备的 ZeroTier 私有地址 `IP.2` `192.168.192.200`：这是 NAS 在 Zerotier 私网中的托管 IPv4 地址。
- 反向代理 frps 所在服务器的公有地址 `IP.3` `101.101.101.101`：通过在腾讯云轻量应用服务器搭建的 frps，反向代理 NAS Web 服务。

## 2 基本过程和方法

## 2.1 创建 CA 私钥

你可以将 CA 私钥看做虚构证书管理机构的钥匙。CA 私钥主要用于创建 CA 证书、为服务器证书签名。

这一步的目标是生成 `ca.key`。主要工具是 `openssl genrsa`。由于 QTS 系统目前不支持 ECDSA 加密，本文均采用 RSA 加密。

```
openssl genrsa -out ca.key 2048
```

## 2.2 创建 CA 证书申请文件（CSR）

CSR 是证书签名请求，它以文件为载体，记载了证书用途等基本信息，由证书需求方向 CA 秘钥持有人、CA 证书持有人发起申请。

这一步的目标是生成 `ca.csr`。主要工具是 `openssl req`。有的文章会告诉你通过一条命令生成私钥、证书，本文为了便于尝试、验证，先创建配置文件 `ca.cnf`，再由配置文件创建 CSR。

```
[req] 
days = 730 
prompt = no 
default_md = sha256 
default_bits = 2048 
req_extensions = v3_req 
distinguished_name = dn 
utf8 = yes 

[dn] 
C = CN 
O = myOname 
OU = caOUname 
CN = caCN 

[v3_req] 
basicConstraints = CA:true
```

注：一般教程中，`[dn]` 一节中还有其他诸多字段，由于我没需求，所以做了简化。

```
openssl req -sha256 -config ca.cnf -key ca.key -new -out ca.csr
```

## 2.3 签发 CA 证书

CA 证书是你所有终端设备的信任中心点，是服务器站点信任链的起源。CA 证书用于为服务器证书签名。因此这一阶段签发证书不需要上级证书。

这一步的目标是生成 `ca.crt`，根据需要转换为 `ca.pem`。主要工具是 `openssl x509`。

```
openssl x509 -req -sha256 -days 730 -in ca.csr -signkey ca.key -out ca.crt -extfile erx.cnf -extensions v3_req openssl x509 -in ca.crt -out ca.pem
```

## 2.4 创建服务器私钥

不同的服务器、WEB应用可以用不同的私钥来标识身份，生成对应的证书来声明不同的用途。例如 NAS 的 WEB 管理页面，与托管在 NAS 上的 Plex 服务，IP 地址不同，你既可以用一个证书包含所有可能的 IP，也可以为它们分别生成证书。

这一步的目的是生成 `server.key`。

```
openssl genrsa -out server.key 2048
```

## 2.5 创建服务器 CSR

由服务器作为主体，用 CSR 作为请求，向 CA 申请签名、形成证书。

这一步的目的是生成 `server.cnf`、`server.csr`。

```
[req]
days = 365
prompt = no
default_md = sha256
default_bits = 2048
distinguished_name = dn
req_extensions = v3_req
utf8 = yes

[dn]
C = CN
O = myOname
OU = serverOUname
CN = serverCN

[v3_req]
basicConstraints = CA:false
subjectAltName = @alt_names

[alt_names]
DNS.1 = exampleserver.myqnapcloud.cn
IP.1 = 192.168.50.200
IP.2 = 192.168.192.200
IP.3 = 101.101.101.101
```

```
openssl req -sha256 -config server.cnf -key server.key -new -out server.csr
```

## 2.6 签发服务器证书

通过 CA 私钥和 CA 证书，对服务器的申请签名、生成服务器证书。

这一步的目的是生成 `server.crt`，根据需要转换为 `server.pem`。

```
openssl x509 -req -sha256 -days 365 -in server.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out server.crt -extfile server.cnf -extensions v3_req openssl x509 -in server.crt -out server.pem
```

## 2.7 在终端设备信任 CA 证书

在 Windows、macOS、iOS、iPadOS 等终端设备中，将 `ca.pem` 添加至受信任的根证书存储区。需要注意的是，Firefox 使用软件内置的证书管理模块，而不是直接采信操作系统的信任证书。

## 2.8 在服务器中安装服务器证书

以威联通 NAS 为例（QTS系统）。通过 http 方式访问，转到控制台 > 系统 > 安全 > 证书和私钥，转到服务器证书，点击取代当前证书，选择导入证书，分别上传 `server.pem`、`server.key`，点击"应用"。退出登录后再通过 https 方式访问，并确认浏览器没有报任何警告、错误。

## 3 问题和解决

## 3.1 证书错误

常见证书错误之一，是证书是由未知（未受信任）的主体签发的。在公共 CA 体系下，用户通常向具有公信力的 CA 机构申请证书，这些 CA 机构的证书已经内置在主流操作系统、浏览器的证书信任链中，由其签发的各类证书很容易被验证是由可信机构签发的。在自签名体系下，这个问题显而易见是固有的。如果直接创建服务器私钥、自签名证书，随着服务器数量增加、服务器证书增加，你会更经常碰到这个问题。因此这篇文章采取先创建 CA 的方式来签发其他证书。

常见证书错误之二，是证书的通用名无效。证书的通用名 `commonName` 是证书的主要属性之一，一般填写站点的域名或泛域名，不包含 http 或 https 等协议名，也不包含端口。上述错误主要是因为用户当前访问站点所使用的链接与证书所认证的链接不一致，例如用户正访问子域名 `sub.example.com`，而证书只认证了 `www.example.com`；例如通过家用局域网 IP 访问 NAS，而证书中没有包含局域网 IP 地址。那么有没有可能在一个证书中认证多个域名或 IP 地址呢？答案是可以的，x.509 证书扩展选项的 `subject alternative names(SAN)` 可以解决这个问题。见 **2.5**。

![可以在一个证书中设置多个域名/泛域名和 IP](https://s2.loli.net/2023/02/08/XU82CL7mY95tFMu.png)

## 3.2 x509 命令在签发服务器证书时没有拷贝请求中的扩展字段

早期尝试过程中，签发证书的命令如下。虽然在 CSR 中指定了 SAN，但签发出来的证书却缺少该扩展字段。如果导入服务器，会提示通用名无效的错误。

```
openssl x509 -req -sha256 -days 365 -in server.csr -CA ca.crt -CAkey ca.key -CAcreateserial -out server.crt
```

查找资料发现，这一设计并非 x509 命令的 bug，而是基于安全性考虑。一般推荐 `openssl ca` 命令来实现 copy\_extensions 的目的，换句话说，copy\_extensions 是为 `ca` 而非 `x509` 命令设计的。但使用 `openssl ca` 的前期配置相对麻烦，需要建立相对完整的 CA 配置结构。

后来我找到一种相对折中的办法，可以在使用 x509 命令的前提下拷贝申请中的扩展字段，即 **2.3** 与 **2.6** 命令中的 `-extfile erx.cnf -extensions v3_req`。

## 3.3 x509 命令在签发服务器证书时没有包含请求中的有效期限

早期尝试过程中，由于 CSR 中已经设置了 `days`，我便将签发服务器证书命令中的 `-days xxx` 移除了。实际上应该在 `x509` 命令中指定 `days`。

## 3.4 在 QTS 中上传服务器私钥提示无效

QTS 不支持 ECDSA 加密，建议使用 RSA 加密，且长度不超过 2048 位。见这篇[帖子](https://forum.qnap.com/viewtopic.php?t=164839) 。

## 3.5 iOS / iPadOS 中如何信任自有 CA 证书

1. 通过打开 PEM 证书添加描述文件并安装  
   建议通过邮件附件形式，将 `ca.pem` 发送到 iOS / iPadOS 设备，在邮件 App 中打开该附件，会提示已添加描述文件、到"设置"中进行进一步操作。打开"设置"，会在"Apple ID"与"飞行模式"之间发现新增一栏选项，点击进入，并安装描述文件。
2. 启用对根证书的完全信任  
   安装描述文件后，转到"设置"-"通用"-"关于本机"-"证书信任设置"，将出现新的一栏，即与描述文件匹配的根证书信任选项。启用即可。

## 结语

目前我已经为自己托管的这些服务使用上了 SSL

- 威联通 NAS Web 页面 `https`
- EdgeRouter-X 路由器 Web 页面 `https`
- Portainer Web 页面 `https`
- Jellyfin Web 页面 `https`
- FRPS 服务端控制面板页面 `https`
- aria2 jsonrpc 接口 `https` `wss`

## 后记

上述实践的背景是，当时我缺少域名，需要保障纯 IP 访问的安全性。后来我发现，为每个 IP 签发证书实际上也给管理带来很大压力。后来我申请了域名，为 aria2, transmission, jellyfin 等每个服务以二级域名的方式做了 frp 穿透和反向代理，同时为 IPv6 地址的域名访问单独做了反向代理，统一签发使用一个自签名的泛域名证书，才真正实现了“易于管理”的目标。