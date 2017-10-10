---
layout: post
title:  "通过shell实现中文urlencode和urldecode"
date:   2017-10-09 14:10:22
categories: linux
tags: linux
author: "sxzhou"
---

## 编码
`echo '上海' | tr -d '\n' | xxd -plain | sed 's/\(..\)/%\1/g'`
%e4%b8%8a%e6%b5%b7

## 解码
`url="http://www.baidu.com?param=%e4%b8%8a%e6%b5%b7"`    

`printf $(echo -n $url | sed 's/\\/\\\\/g;s/\(%\)\([0-9a-fA-F][0-9a-fA-F]\)/\\x\2/g')"\n"`

http://www.baidu.com?param=上海
