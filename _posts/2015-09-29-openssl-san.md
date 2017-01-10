---
layout:     post
title:      OpenSSL SAN 证书
subtitle:   使用 OpenSSL 制作一个包含 SAN（Subject Alternative Name）的证书
date:       2015-09-29 21:00:00
author:     Liao
catalog:    true
header-img: img/post-bg-2015.jpg
permalink:  /openssl-san/
tags:
    - OpenSSL
---

## 什么是 SAN

SAN(Subject Alternative Name) 是 SSL 标准 x509 中定义的一个扩展。使用了 SAN 字段的 SSL 证书，可以扩展此证书支持的域名，使得一个证书可以支持多个不同域名的解析。

先来看一看 Google 是怎样使用 SAN 证书的，下面是 Youtube 网站的证书信息：

![](/img/in-post/openssl-san/youtube-ssl.png)

这里可以看到这张证书的 Common Name 字段是 *.google.com，那么为什么这张证书却能够被 www.youtube.com 这个域名所使用呢。原因就是这是一张带有 SAN 扩展的证书，下面是这张证书的 SAN 扩展信息：

![](/img/in-post/openssl-san/youtube-san-1.png)
![](/img/in-post/openssl-san/youtube-san-2.png)

这里可以看到，这张证书的 Subject Alternative Name 段中列了一大串的域名，因此这张证书能够被多个域名所使用。对于 Google 这种域名数量较多的公司来说，使用这种类型的证书能够极大的简化网站证书的管理。

## 使用 openssl 生成带有 SAN 扩展的证书请求文件（CSR）

首先我们将 openssl 的配置文件复制一份作临时使用，CentOS6 中 openssl 的配置文件在 `/etc/pki/tls/openssl.cnf`，将这个文件复制到 `/tmp` 下。

此文件的格式是类似 `ini` 的配置文件格式，找到 **[ req ]** 段落，加上下面的配置：

```
req_extetions = v3_req
```

这段配置表示在生成 CSR 文件时读取名叫 `v3_req` 的段落的配置信息，因此我们再在此配置文件中加入一段名为 `v3_req` 的配置：

```
[ v3_req ]
# Extensions to add to a certificate request

basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature, keyEncipherment
subjectAltName = @alt_names
```

这段配置中最重要的是在最后导入名为 `alt_names` 的配置段，因此我们还需要添加一个名为 `[ alt_names ]` 的配置段：

```
[ alt_names ]
DNS.1 = www.ustack.in
DNS.2 = www.test.ustack.com
```

这里填入需要加入到 Subject Alternative Names 段落中的域名名称，可以写入多个。

接着使用这个临时配置生成证书：

```
$ openssl req -new -nodes -keyout ustack.key -out ustack.csr -config /tmp/openssl.cnf
```

查看证书请求文件的内容：

```
$ openssl req -text -noout -in ustack.csr
```

可以看到此证书请求文件中会包含 Subject Alternative Names 字段，并包含之前在配置文件中填写的域名。

## 使用 openssl 签署带有 SAN 扩展的证书请求

假设使用本机作为子签署 CA 对此证书请求进行签署，签署的方式为：

```
$ openssl ca -policy policy_anything -out ustack.crt -config /tmp/openssl.cnf -extensions v3_req -infiles ustack.csr
```

签署后，查看证书的内容：

```
$ openssl x509 -text -noout -in ustack.crt
```

## 使用单条命令实现
觉得上面的方式太麻烦了？使用命令一步生成带 SAN 扩展的证书请求文件：


```
$ openssl req -new -sha256 \
    -key ustack.key \
    -subj "/C=CN/ST=Beijing/L=Beijing/O=UnitedStack/OU=Devops/CN=www.ustack.com" \
    -reqexts SAN \
    -config <(cat /etc/pki/tls/openssl.cnf \
        <(printf "[SAN]\nsubjectAltName=DNS:www.ustack.in,DNS:www.test.ustack.com")) \
    -out ustack.csr
```

上面生成证书请求时的几个字段的意义：

```
C  => Country
ST => State
L  => City
O  => Organization
OU => Organization Unit
CN => Common Name (证书所请求的域名)
emailAddress => main administrative point of contact for the certificate
```

签署上面生成的证书：

```
$ openssl ca -in ustack.csr \
	-extensions SAN \
	-config <(cat /etc/pki/tls/openssl.cnf \
        <(printf "[SAN]\nsubjectAltName=DNS:www.ustack.in,DNS:www.test.ustack.com")) \ 
	-out ustack.crt
```

查看证书内容：

```
$openssl x509 -text -noout -in ustack.crt
```

<br>

参考：

- [Creating and signing an SSL cert with alternative names](http://blog.zencoffee.org/2013/04/creating-and-signing-an-ssl-cert-with-alternative-names/)
- [Provide subjectAltName to openssl directly on command line](http://security.stackexchange.com/questions/74345/provide-subjectaltname-to-openssl-directly-on-command-line)











