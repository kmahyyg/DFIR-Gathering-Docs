---
sidebar_position: 2
---

# 开发文档 - 开发环境部署

- Go 1.18.1, IDE 使用 Goland
- Git 版本管理
- 使用 Go Modules

使用加速器拉取依赖，配置下列环境变量：

```shell
export GOPROXY=https://goproxy.cn,direct
export GOPATH=${HOME}/go
export PATH=${GOPATH}/bin:${PATH}
```

- Protobuf 最新版

```shell
go install google.golang.org/protobuf/cmd/protoc-gen-go@latest
go get google.golang.org/protobuf
```