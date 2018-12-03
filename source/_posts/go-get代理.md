---
title: go get代理
tags:
  - go
abbrlink: 7f6b281c
date: 2018-11-27 18:08:04
---
平时在用go get命令下载依赖时，经常会遇到类似
> package google.golang.org/grpc/encoding/gzip: unrecognized import path "google.golang.org/grpc/encoding/gzip" (https fetch: Get https://google.golang.org/grpc/encoding/gzip?go-get=1: dial tcp 216.239.37.1:443: i/o timeout)

的问题，其实是这些资源被墙，所幸的是有很多使用代理的方式解决了这个问题。

以*google.golang.org/grpc/encoding/gzip*为例：

## 设置git代理

```shell
$ git config [--global] http.proxy http://proxy.example.com:port
$ go get "google.golang.org/grpc/encoding/gzip"
```

## shell环境变量

```shell
# socks
http_proxy=socks5://127.0.0.1:1080 go get "google.golang.org/grpc/encoding/zip"
# http
http_proxy=http://127.0.0.1:1080 go get "google.golang.org/grpc/encoding/gzip"
```

## VPN
那就直接*go get "google.golang.org/grpc/encoding/gzip"*吧


