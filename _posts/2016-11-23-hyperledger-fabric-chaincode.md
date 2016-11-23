---
layout: post
---

## Hyperledger Fabric chaincode

上文通过源码分析简单了解Hyberledger faric chaincode的运行环境 VM，本文着重简单分析fabric中最为重要的元素chaincode。

chaincode作为运行在fabric区块链上的程序，承载了所有的商业逻辑，它类似于ethereum上的DAPP, 一个去中心化的链上程序。fabric的chaincode分为两种，一是系统chaincode,另一种是用户chaincode，系统chaincode用来初始化区块链的参数，以及用户chaincode需要共同遵守的规则。 
<!--more-->
chaincode通过自定义逻辑访问/修改账本的数据并将结果返回给用户。以下文件定义了系统chaincode和用户chaincode都需要实现的接口

###/core/shim/interface.go

```go
type Chaincode interface {
    //当包含chaincode的VM启动后，init函数会被调用，用于将chaincode“安装“到区块链上，  
    //可初始化只用于此chaincode的数据
	Init(stub ChaincodeStubInterface) ([]byte, error)
    
	// Invoke函数用于调用/修改此chaincode的数据并跟新到全局账本中
	Invoke(stub ChaincodeStubInterface) ([]byte, error)

	// Query函数则用于且只用于读取此中chaincode的账本数据
	Query(stub ChaincodeStubInterface) ([]byte, error)
}
```

可在以下路径查看chaincode具体实现:  
系统chaincode:/core/system_chaincode  
用户chaincode:/examples  

而用于编写用户chaincode的语言，farbic目前支持三种语言，分别是golang,Java和car。  

以下的`ChaincodeStubInterface`作为参数传入chaincode,为chaincode提供账本的操作接口。  

```go
type ChaincodeStubInterface interface {  
    ...
	// chaincode只能更新自己的数据，如要操作其它chaincode的数据（有权限的情况下），  
    // 则需要通过调用目标chaincode的函数
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
以下则是`ChaincodeStubInterface`接口的具体实现  
/core/shim/mockstub.go     用于测试  
/core/shim/chaincode.go    实际环境  



chaincode运行于VM中，而账本的区块链数据并不在VM里，chaincode对账本的操作通过以下的桥接方式。

VM{chaincode <-> (shim([stub](https://github.com/hyperledger/fabric/blob/master/core/chaincode/shim/chaincode.go)<->[handler](https://github.com/hyperledger/fabric/blob/master/core/chaincode/shim/handler.go)->[FSM](https://github.com/hyperledger/fabric/blob/master/core/chaincode/shim/handler.go#L152-L181)))} <-> gRPC <-> {([handler](https://github.com/hyperledger/fabric/blob/master/core/chaincode/handler.go)<->[FSM](https://github.com/hyperledger/fabric/blob/master/core/chaincode/handler.go#L390-L450))Validator Peer([chaincode_support](https://github.com/hyperledger/fabric/blob/master/core/chaincode/chaincode_support.go))} <-> Ledger

chaincode用以下的消息类型由shim经gRPC与位于同一host的Peer进程通信，分别处于VM的shim和处于host的Peer进程都有一个有限状态机FSM(github.com/looplab/fsm)，双方都通过状态机的状态变化调用各自handler的方法来发送不同的信息进行通信。

```go
message ChaincodeMessage {
    enum Type {
        UNDEFINED = 0;
        REGISTER = 1;
        REGISTERED = 2;
        INIT = 3;
        READY = 4;
        TRANSACTION = 5; v
        COMPLETED = 6;
        ERROR = 7;
        GET_STATE = 8;
        PUT_STATE = 9;
        DEL_STATE = 10;
        INVOKE_CHAINCODE = 11;
        INVOKE_QUERY = 12;
        RESPONSE = 13;
        QUERY = 14;
        QUERY_COMPLETED = 15;
        QUERY_ERROR = 16;
        RANGE_QUERY_STATE = 17;
        RANGE_QUERY_STATE_NEXT = 18;
        RANGE_QUERY_STATE_CLOSE = 19;
        KEEPALIVE = 20;
    }
    Type type = 1;
    google.protobuf.Timestamp timestamp = 2;
    bytes payload = 3;
    string txid = 4;
    ChaincodeSecurityContext securityContext = 5;
    ChaincodeEvent chaincodeEvent = 6;
}
```
