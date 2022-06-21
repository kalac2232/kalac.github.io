---
title: http基础知识
date: 2021-05-13 15:15:24
tags: http
categories: 计算机网络
---

GET: 获取资源 没有body

POST: 增加或修改资源，有body

PUT：修改资源，有body，*应用时大多被POST取代*

DELETE：删除资源，没有body

HEAD:与GET基本相同，区别为服务器返回内容时是不会返回body，一个应用场景：下载前获取文件的大小、是否支持断点续传

<!-- more -->

### Status Code

- 1xx: 临时性消息 场景：先发个消息去服务器确认是否支持http2.0
- 2xx: 成功 *很多不规范的使用，将所有错误都归为200*
  - 201: 创建用户成功
- 3xx: 重定向
  - 304:内容没有改变
- 4xx: 客户端错误
- 5xx: 服务器错误

### Header

标识HTTP消息的具体属性，如长度、类型

- Host: 服务器的主机地址。

  因为访问时是通过DNS服务器解析出服务器的ip地址后，通过ip进行访问，当这个ip地址所对应的服务器下部署了多个域名的网站，服务器将通过Host传来的地址确定对应的具体网址。

- Content-Type/Content-Length : 内容的类型和长度
  - text/html：html文本
  - application/x-www-form-urlencoded：普通表单（纯文字表单）
  - multipart/form-data:多部分形式，包含二进制的内容
  - Application/json
  - Image/jpeg

- Transfer-Encoding: 分块传输，Content-Length不可用，传输结束时内容为0

- Location:重定向的位置

- User-Agent:让服务器识别设备类型

- Range/Accept-Range：使用断点续传/是否支持断点续传

- Accept:客户端能接受的数据类型

- Accept-Charset：客户端能接受的字符集

- Accept-Encoding:客户端能接受的压缩编码类型，如gzip

- Content-Encoding：传输内容的压缩类型