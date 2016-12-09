---
layout: post
featured: true
---

Hyperledger Fabirc 1.0 对底层的数据结构做了很大的改动，本文通过proto bufferd的定义文件来了解Fabric基本的数据结构。

区块连是由基本的块组成的，下面是块的数据结构：
```go
// 区块
message Block {
    // 区块头
    BlockHeader Header = 1;    
    // 区块体，由事务列表序列化而成       
    BlockData Data = 2;  
    // 区块元数据                
    BlockMetadata Metadata = 3;    
}

// 区块头
message BlockHeader {
    // 区块号
    uint64 Number = 1;     
    // 前一个区块头的哈希              
    bytes PreviousHash = 2;
    // BlockData的哈希    
    bytes DataHash = 3;                
}
// 区块体
message BlockData {
    repeated bytes Data = 1;         //事务列的序列数组
}
// 区块元数据
message BlockMetadata {
    repeated bytes Metadata = 1;     //区块元数据字节数组
}
```

以上的`BlockData`是事务列表的序列化字节数组，而一个已签名的事务的整体数据结构如下
```go
SignedTransaction
|\_ Signature                                    (事务的发起者对交易的签名，发起者的信息在事务头中)
 \_ Transaction
     \_ TransactionAction (1...n)
        |\_ Header (1)                           (事务头部，等同于提议头部)
         \_ ChaincodeActionPayload (1)
            |\_ ChaincodeProposalPayload (1)     (payload of the proposal that requested this action)
             \_ ChaincodeEndorsedAction (1)
                |\_ Endorsement (1...n)          (背书者对背书结果的签名endorsers signatures over the whole response payload)
                 \_ ProposalResponsePayload
                     \_ ChaincodeAction          (提议中的操作)
```

服务层接收到的事务请求，可能包含多个事务动作。每个事务动又对应一个提议的多个操作，一个客户端在一个事务里可以发起多个独立的提议。   

未签名事务的数据结构如下
```go
message Transaction {

	// 协议版本号
	int32 version = 1;

	// 事务发起的时间
	google.protobuf.Timestamp timestamp = 2;

	// 事务由一系列的事务动作组成
	repeated TransactionAction actions = 3;
}
```

已签名事务的数据结构
```go
message SignedTransaction {

	//事务的签名，签名者的公钥在事务动作（TransactionAction）的头部，
	//一个事务中可能有多个事务动作，所以会有多个事务动作的头部
	//但是事务动作的发起者应该都是同一个人
	bytes signature = 2;
	// 事务动作列表的序列化数组
	bytes transactionBytes = 1;
}
```
事务动作等同于提议中的操作，事务动作的类型在事务动作的头部。  
```go
message TransactionAction {

	// 事务动作头部的字节数组格式
	bytes header = 1;

	// ChaincodeActionPayload的字节数组格式，它的链码类型信息在以上的头部里。  
	bytes payload = 2;
}
```
```go
message ChaincodeAction {

	bytes results = 1;
	bytes events = 2;
}
```

事务消息是由提议和提议回复消息装配而成，

```go
message ChaincodeActionPayload {

	bytes chaincodeProposalPayload = 1;
	ChaincodeEndorsedAction action = 2;
}
```
对提议的回复消息
```go
message ChaincodeEndorsedAction {

	bytes proposalResponsePayload = 1;
	repeated Endorsement endorsements = 2;
}
```
 提议是用来向背书者发起的操作建议，提议由提议头和提议体两部分组成，提议头也相当于交易的头
```go
message Proposal {

	// 以下Header类型的序列化字节数组
	bytes header = 1;

	// Header中的提议类型的序列化数组
	bytes payload = 2;

	// 非必要扩展字段，当提议的类型为链码时，
	// 以上payload则为ChaincodeProposalPaylo类型
	bytes extension = 3;
}
```

```go
message Header {
    ChainHeader chainHeader = 1;
    SignatureHeader signatureHeader = 2;
}
```
Header包含以下两个头不信息，ChainHeader和SignatureHeader
```go
message ChainHeader {
    int32 type = 1; // Header types 0-10000 are reserved and defined by HeaderType

    // 版本号
    int32 version = 2;

    // 链初始化时间
    google.protobuf.Timestamp timestamp = 3;

    // 链ID
    bytes chainID = 4;

    // 唯一的事务ID
    string txID = 5;

    // 链头产生的时间，可用于防止重返攻击。
    uint64 epoch = 6;

    // 扩展字段
    bytes extension = 7;
}

message SignatureHeader {

    // 消息的发起者
    bytes creator = 1;

    // 可用于防止重返攻击。
    bytes nonce = 2;
}
```
当在提议头中的提议类型是链码时，它包含调用链码的输入参数  
```go
message ChaincodeProposalPayload {

	//以下ChaincodeInvocationSpec类型的序列化字节数组
	bytes Input  = 1;

	bytes Transient = 2;
}
```

链码相关的数据结构:  

链码ID，包括源码路径和ID名
```go
message ChaincodeID {
    //源码路径
    string path = 1;

    //源码名，其实是一个哈希值，与运行此链码的docke容器同名。
    string name = 2;
}
```

链码输入，包括链码的函数名和函数参数
```go
message ChaincodeInput {
    repeated bytes args  = 1;
}
```
链码定义消息
```go
message ChaincodeSpec {

    // 链码编写语言，目前支持Go,NodeJs,Car,Java。
    enum Type {
        UNDEFINED = 0;
        GOLANG = 1;
        NODE = 2;
        CAR = 3;
        JAVA = 4;
    }
    // 链码编写语言
    Type type = 1;
    // 链码ID
    ChaincodeID chaincodeID = 2;
    // 链码输入	
    ChaincodeInput ctorMsg = 3;
    int32 timeout = 4;
    string secureContext = 5;
    ConfidentialityLevel confidentialityLevel = 6;
    bytes metadata = 7;
    repeated string attributes = 8;
}
```
链码发布消息，主要包括链码定义消息和运行环境参数
```go
message ChaincodeDeploymentSpec {
    // 链码运行环境，Docker或者系统环境
    enum ExecutionEnvironment {
        DOCKER = 0;
        SYSTEM = 1;
    }
    // 链码定义
    ChaincodeSpec chaincodeSpec = 1;
    // 链码可被执行的时间
    google.protobuf.Timestamp effectiveDate = 2;
    // 未实现，未来可能不需要以上的chaincodeSpec
    // 直接将链码压缩成字节包
    bytes codePackage = 3;
    ExecutionEnvironment execEnv=  4；
}
```

链码调用消息，主要包括链码定义消息和产生ID的算法
```go
message ChaincodeInvocationSpec {

    ChaincodeSpec chaincodeSpec = 1;
    string idGenerationAlg = 2;
}
```
