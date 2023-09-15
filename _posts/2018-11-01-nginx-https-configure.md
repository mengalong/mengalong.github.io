---
layout: post
title: Nginx 配置https
category : 编程
tags : [https,nginx]
---

# Https简介
简单的理解，在http的基础上增加一层 SSL加密的过程，就实现了https。服务端和客户端传输信息的时候都会通过TLS进行加密。基本的原理如下：

1. 客户端与服务端建立连接，生成各自的私钥和公钥
2. 服务端返回给客户端一个公钥
3. 客户端拿着这个公钥把需要传输给服务端的数据进行加密，形成密文，连同自己的公钥一起返回给服务器
4. 服务器拿自己的私钥对密文解密之后，将响应数据用客户端的公钥加密，返回到客户端
5. 客户端拿着自己的私钥解密，将数据呈现出来

搞清楚上边的原理之后，一个http服务要实现https，最关键的环节就是生成自己的证书+私钥。证书的来源基本有如下几种途径：
1. 自己在服务器上生成。如果是自己做测试使用，推荐这种，简单方便
2. 在CA证书机构申请免费的证书(目前阿里云、百度云都有提供)。如果服务不是很重要或者主要用来测试，那么也可以用
3. 在CA证书机构申请付费证书。商用推荐方案

接下来就以第1中方式来记录一下https的服务配置

# 自己生成免费证书
我用的是nginx，因此在nginx的conf目录下创建ssl目录，之后进入ssl目录生成证书

1. 生成秘钥key
```
openssl genrsa -des3 -out server.key 2048
```
过程中需要输入密码，两次输入一致，记住即可，结束之后生成 server.key文件，后续使用该文件生成其他配置的时候都需要输入这里的密码。如果不想后续输入密码，则执行（建议执行，因为如果不做这一步，后续配置了nginx之后，nginx服务启动的时候都需要让你输入密码)：
```
openssl rsa -in server.key -out server.key
```
2. 生成服务器证书的申请文件server.csr
```
openssl req -new -key server.key -out server.csr
```
输出的内容如下：
```
Enter pass phrase for root.key:  输入前面创建的密码 
Country Name (2 letter code) [AU]:CN  国家代号，中国CN 
State or Province Name (full name) [Some-State]:shannxi 省的全名 
Locality Name (eg, city) []:xi'an 市的全名，拼音 
Organization Name (eg, company) [Internet Widgits Pty Ltd]: xxxx 公司名称 
Organizational Unit Name (eg, section) []: 可以不输入，直接回车
Common Name (eg, YOUR name) []: 可以不输入，也可以输入你的域名
Email Address []:admin@mycompany.com 邮箱地址
Please enter the following ‘extra’ attributes 
to be sent with your certificate request 
A challenge password []: 可以不输入，直接回车
An optional company name []: 可以不输入，直接回车
```

3. 生成CA证书：
```
openssl req -new -x509 -key server.key -out ca.crt -days 3650
```

4. 生成一个有效期为10年的服务器证书，server.crt
```
openssl x509 -req -days 3650 -in server.csr -CA ca.crt -CAkey server.key -CAcreateserial -out server.crt
```
完成后，在ssl目录下生成的server.crt和server.key就是我们nginx需要的证书

# 配置Nginx(1)
nginx需要支持https，依赖于nginx编译时打开了ssl编译选项。这一步继续配置之前，可以通过命令查看当前nginx是否支持ssl,执行nginx -V 可以查看--with-http_ssl_module 包含了这个选项
```
./nginx -V
nginx version: nginx/1.12.1
built by gcc 5.4.0 20160609 (Ubuntu 5.4.0-6ubuntu1~16.04.4)
built with OpenSSL 1.0.2g  1 Mar 2016
TLS SNI support enabled
configure arguments: --prefix=/home/work/lnmp/nginx --http-client-body-temp-path=/home/work/lnmp/nginx/tmp/client_body --http-proxy-temp-path=/home/work/lnmp/nginx/tmp/proxy --http-fastcgi-temp-path=/home/work/lnmp/nginx/tmp/fastcgi --http-uwsgi-temp-path=/home/work/lnmp/nginx/tmp/uwsgi --lock-path=/home/work/lnmp/nginx/var/lock/ --with-file-aio --with-http_ssl_module --with-http_realip_module --with-http_sub_module --with-http_gzip_static_module --with-http_stub_status_module --with-pcre
```

# 配置Nginx(2):
![image.png](https://upload-images.jianshu.io/upload_images/13183512-efbe051084cc0917.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

如上图红框中的内容，我当前的nginx启动的3389端口对应http请求，启动443端口对应https请求。
修改完配置文件重启nginx服务生效即可。
之后就可以通过https://your.domain.name/xxx 来访问你的服务了

# 注意：
我测试用的是阿里云上的ecs服务器，在阿里云，默认服务器是不开放443端口的，因此需要去你的阿里云对应实例的安全组上，放开 443端口
![image.png](https://upload-images.jianshu.io/upload_images/13183512-4b3ea5f6e4fdcf13.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

