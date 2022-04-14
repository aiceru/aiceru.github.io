---
layout: post
title: '[CentOS] console 의 해상도와 폰트 변경하기'
categories: [Development]
tags: [centos, linux, console, resolution, font]
last_modified_at: '2022-04-09 23:50:01'
redirect_from: /2017/01/16/centos-console-resolution/
---

<!-- more -->

# 해상도 resolution 변경

`etc/grub.conf` 파일 중, `kernel` 옵션의 가장 마지막에 `vga=0x347` 을 추가.

# font 변경

```bash
$ yum install terminus-fonts-console
$ vim /etc/sysconfig/i18n

------------------
LANG="us_US.UTF-8"
SYSFONT="ter-116n"
------------------
```

# 결과

![centos_console_screen](https://3.bp.blogspot.com/-X7y7may3pJo/WHxFgxV4xqI/AAAAAAAAexE/rXAzT5f6AmcCtYLB4Xyed6UIa6a7fz_hQCLcB/s640/%25EC%258A%25A4%25ED%2581%25AC%25EB%25A6%25B0%25EC%2583%25B7%252C%2B2017-01-16%2B12-46-20.png)
