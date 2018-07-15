# 依赖

MOSN采用dep进行依赖管理。

`Gopkg.toml` 文件中定义了MOSN使用到的依赖。

- github.com/deckarep/golang-set

  Go语言缺少set集合，在Go内置前使用它。

- github.com/envoyproxy/go-control-plane

  envoy的API定义和go实现。

- github.com/gogo/protobuf

  protocol buffer 3支持

- github.com/hashicorp/go-syslog

  安全导入sys log

- github.com/julienschmidt/httprouter

  A lightweight high performance HTTP request router

- github.com/magiconair/properties

  Java properties scanner for Go

- github.com/nu7hatch/gouuid

  Go binding for libuuid。Pure Go UUID implementation。

- github.com/orcaman/concurrent-map

  A thread-safe concurrent map for go

- github.com/rcrowley/go-metrics

  Go port of Coda Hale's Metrics library

- github.com/urfave/cli

  A simple, fast, and fun package for building command line apps in Go

- google.golang.org/grpc

  implements an RPC system called gRPC

- gopkg.in/natefinch/lumberjack.v2

  Package lumberjack provides a rolling logger

- github.com/fatih/structs

  Utilities for Go structs

- github.com/valyala/fasthttp

  Fast HTTP package for Go. Tuned for high performance. Zero memory allocations in hot paths. Up to 10x faster than net/http

- sdf