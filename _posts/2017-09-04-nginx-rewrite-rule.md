---
layout: post
title: Nginx 设置rewrite规则
category : 编程
tags : [Nginx]
---

# Nginx 设置rewrite规则

* 规则1：重定向指定的url到其他url
```
        location ~ .*/mengalong/shuangyi_merchant.apk$ {
                rewrite ^(.*)$ shuangyi_ios_user.php permanent;
        }
```

* 规则2： 重定向指定url到其他域名
```
        location ~ .*/mengalong/shuangyi_user.apk$ {
                rewrite ^(.*)$ http://www.baidu.com permanent;
        }
```

* 规则3：根据正则表达式获取url中的指定参数
```
        location ~ .*/mengalong/t/d/shuangyi_user.apk$ {
                rewrite ^/mengalong/(t.*)/(shuangyi.*)$ http://www.baidu.com/s?wd=$1_$2 permanent;
        }
```
