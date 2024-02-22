---
layout: post
title: 如何部署国密openresty
categories: 国密 openresty
---

## 起因
OpenResty是一个基于Nginx与Lua的高性能Web平台，由于其可以使用Lua进行扩展，比起apached httpd + python更为灵活，一直以来，我们项目中使用openresty作为反向代理。近期，产品需要进行国密认证，需要支持国密算法。实现思路有两种，一是在openresty前再放一个nginx做为前置代理，该方案是可以直接使用国密网站上已经编译好的国密nginx，但缺点时要损失一部分性能，套娃也不利于维护；二是直接编译openresty使其支持国密算法，openresty官网没有提供过国密版本的程序，需要自己编译，但openresty是基于nginx的，集成国密算法应该问题不大，所以就采用方案二，自己来踩踩坑。

## 编译Openresty

1. 去github下载openresty源码，我这里以1.15.8.3为例，解压出来：
```sh
tar xvf openresty-1.15.8.3.tar.gz 
```

2. 下载gmssl_openssl，将其解压在/usr/local目录下：
```sh
wget https://www.gmssl.cn/gmssl/down/gmssl_openssl_1.1_b2024_x64_1.tar.gz
tar xvf gmssl_openssl_1.1_b2024_x64_1.tar.gz -C /usr/local
```

3. 参考国密网编译nginx是对openssl的修改，将`bundle/nginx-1.15.8/auto/lib/openssl/conf`中的所有`$OPENSSL/.openssl/`修改为`$OPENSSL`。

4. 编译并安装
```
./configure \
--without-http_gzip_module \
--with-http_ssl_module \
--with-http_stub_status_module \
--with-http_v2_module \
--with-file-aio \
--with-openssl="/usr/local/gmssl" \
--with-cc-opt="-I/usr/local/gmssl/include" \
--with-ld-opt="-lm"

make
make install
```

## 配置Openresty以启用国密套件

为了测试，我们先去[国密证书实验室](https://www.gmcrt.cn/gmcrt/index.jsp)申请测试证书。修改nginx配置以使用该证书，示例如下：
```
worker_processes  1;
error_log /var/log/openresty.log warn;
events {
    worker_connections 1024;
}

http {
    #配置共享会话缓存大小
    ssl_session_cache   shared:SSL:10m;

    #配置会话超时时间
    ssl_session_timeout 30m;

    server {
        listen 80 default_server;
        server_tokens off;
        return 301 https://$host$request_uri;
    }

    server {
        listen 443 ssl;
        server_tokens off;
        more_clear_headers    "Server";

        ssl_password_file        /usr/local/etc/password.txt;

        ssl_certificate          /usr/local/etc/sm2/sm2.WEB.sig.crt.pem;
        ssl_certificate_key      /usr/local/etc/sm2/sm2.WEB.sig.key.pem;
        ssl_certificate          /usr/local/etc/sm2/sm2.WEB.enc.crt.pem;
        ssl_certificate_key      /usr/local/etc/sm2/sm2.WEB.enc.key.pem;

        ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
        ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:AES128-SHA:DES-CBC3-SHA:ECC-SM4-CBC-SM3:ECC-SM4-GCM-SM3;
        ssl_verify_client off;


        #SAMEORIGIN：frame页面的地址只能为同源域名下的页面
        add_header X-Frame-Options SAMEORIGIN;

        # 为了put请求能获取到参数，ngx.req.get_post_args()
        lua_need_request_body on;

        #HSTS）强制执行到服务器的安全（HTTP over SSL / TLS）连接
        add_header Strict-Transport-Security "max-age=16070400";
    }
}
```

这时，使用普通的chrome等浏览器访问服务器会报`ERR_SSL_VERSION_OR_CIPHER_MISMATCH`这个错误, 是正常的，因为他们并不支持国密套间，使用国密浏览器访问服务器来来确认服务器是否正常工作。