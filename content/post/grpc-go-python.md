---
title: "gRPC + Golang + python 初窥门径"
date: 2020-01-08T15:32:38+08:00
draft: true
tags:
    - "gRPC"
    - "Golang"
    - "Python"
    - "Protocol Buffer"
categories:
    - "入门"
---

## 环境准备

1. 配置 gRPC for Golang
    - 安装 *grpc*
        > go get -u google.golang.org/grpc
    - 安装 *protoc-gen-go*
        > go get -u github.com/golang/protobuf/protoc-gen-go
2. 配置 gRPC for python
    - 安装 *grpcio*
        > python -m pip install grpcio
    - 安装 *grpcio-tools*
        > python -m pip install grpcio-tools

## 编辑 ***protocol buffer*** 协议

```go
    syntax = "proto3";

    package myhello;

    // my test service
    service MyTest {
        // my test foo
        rpc MyFoo (MyRequest) returns (MyResponse) {}
    }

    // the test req
    message MyRequest {
        string name = 1;
    }

    // the test reply
    message MyResponse {
        string reply = 1;
    }
```

## 生成 ***protocol*** 对应代码

1. 生成 Golang gRPC 代码
    - 执行 *protoc* 命令
        > protoc -I myhello\ --go_out=plugins=grpc:myhello\ myhello\myhello.proto
    - 得到 *myhello.pb.go* 库文件
2. 生成 Python gRPC 代码
    - 调用 *grpc_tools* 模块
        > python -m grpc_tools.protoc -I myhello\ --python_out=myhello\ --grpc_python_out=myhello\ myhello\myhello.proto
    - 得到 *myhello_pb2.py*， *myhello_pb2_grpc.py* 两个库文件

## gRPC 服务端 Golang

```go
    package main

    import (
        "context"
        "log"
        "net"
        pb "test/grpc_test/myhello"

        "google.golang.org/grpc"
    )

    const (
        port = ":10002"
    )

    type server struct {
        pb.UnimplementedMyTestServer
    }

    func (s *server) MyFoo(ctx context.Context, in *pb.MyRequest) (*pb.MyResponse, error) {
        log.Printf("Received from : %v", in.GetName())
        return &pb.MyResponse{Reply: "Reply for " + in.GetName()}, nil
    }

    func main() {
        lis, err := net.Listen("tcp", port)
        if err != nil {
            log.Fatalf("failed to listen: %v", err)
        }

        log.Printf("start to serve on localhost" + port)
        s := grpc.NewServer()
        pb.RegisterMyTestServer(s, &server{})
        if err := s.Serve(lis); err != nil {
            log.Fatalf("failed to serve: %v", err)
        }
    }
```

## gRPC 客户端 Python

```python
    import grpc
    import myhello_pb2
    import myhello_pb2_grpc

    if __name__ == '__main__':
        channel = grpc.insecure_channel("localhost:10002")
        stub = myhello_pb2_grpc.MyTestStub(channel)
        rep = stub.MyFoo(myhello_pb2.MyRequest(name="My Python Client"))
        print(rep.reply)
```

## 测试

1. *golang* 服务端
    > PS C:\Users\admin\go\src\test\grpc_test> go run .\server.go  
    > 2020/01/08 14:54:34 start to serve on localhost:10002  
    > 2020/01/08 15:18:38 Received from : My Python Client
2. *python* 客户端
     > PS C:\Users\admin\go\src\test\grpc_test> & C:/python37/python.exe .\client.py  
     > Reply for My Python Client

## 小结

1. ***gRPC*** 使用 ***protocol buffer*** 定义接口和交互数据，客户端可直接远程调用服务端接口。
2. ***gRPC*** 的客户端和服务端可以运行在不同的环境上，也可以使用不同的语言实现， 对分布式的微服务十分友好。
