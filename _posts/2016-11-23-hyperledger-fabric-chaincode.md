---
layout: post
title: "Hyperledger Fabric chaincode"
description: ""
category: 
tags: []
---

## Hyperledger Fabric chaincode

上文通过源码分析简单了解Hyberledger faric chaincode的运行环境 VM，本文着重简单分析fabric中最为重要的元素chaincode。

chaincode作为运行在fabric区块链上的程序，其承载了其所有的商业逻辑，它类似于ethereum上的DAPP, fabric的chaincode分为两种，一种是系统chaincode,另一种则是用户chaincode，系统chaincode用来初始化区块链的参数，以及用户chaincode需要共同遵守的规则。 
<!--more-->
用户chaincode通过自定义逻辑访问/修改账本的数据并将结果返回给用户。fabric定义了系统chaincode和用户chaincode需要实现的接口

####/core/shim/interface.go

```go
type Chaincode interface {
    //当VM启动后，init函数会被调用，用于将chaincode“安装“到区块链上，  
    //可初始化只用于此chaincode的数据
	Init(stub ChaincodeStubInterface) ([]byte, error)

	// Invoke函数用于调用/修改此chaincode的数据并跟新到全局账本中
	Invoke(stub ChaincodeStubInterface) ([]byte, error)

	// Query函数则用于且只用于读取此中chaincode的账本数据
	Query(stub ChaincodeStubInterface) ([]byte, error)
}
```

fabric要求系统的chaincode和用户chaincode都必须实现以上的接口函数

以下的`ChaincodeStubInterface`作为参数传入chaincode,为chaincode提供账本的操作函数，

```go
type ChaincodeStubInterface interface {

    ...
	// chaincode只能更新自己的数据，如有权限，也通过调用其它chaincode操作其它chaincode的数据
	InvokeChaincode(chaincodeName string, args [][]byte) ([]byte, error)

	// 获取账本上的k-v数据
	GetState(key string) ([]byte, error)
    
	// 写入账本数据
	PutState(key string, value []byte) error

	// 删除账本数据
	DelState(key string) error
    ...
}
```


