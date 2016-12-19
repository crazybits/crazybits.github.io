---
layout: post
featured: true
---

Hyperledger Fabirc拜占庭 共识算法 spbft(simplified Practical Byzantine Fault Tolerance)的基本流程如下：  
客户端向共识节点发出`Request`消息，    
主节点收到的`Request`消息所占内存容量大小累计达到一定`batch_size_bytes `或者一定的时间间隔`batch_duration_nsec`后会为其分配View序列号，并向其它共识节点发起`Preprepare`消息  
当共识节点的非主节点收到合法的`Preprepare`消息后将发起`Prepare`消息  
当共识节点收到n-f-1个`Prepare`消息验证合法后将发起`Commit`消息  
当共识节点收到n-f个`Commit`消息后则将信息写入账本并发出`Checkpoint`消息  

当共识节点过了`request_timeout_nsec`仍然未收到`Preprepare`消息则发起`ViewChange`消息重选主节点  
当新选出的共识节点主节点收到f-n个`ViewChange`消息后则发起`NewView`消息  
<!--more-->
```go
基本的配置
message Config {
        // 参与共识的节点数n=3f+1
        uint64 n = 1;
        // 允许最多的坏节点
        uint64 f = 2;
        // 当时间间隔为batch_duration_nsec或者
        // 收到的消息大小为batch_size_bytes时则发起PrePrepare消息
        uint64 batch_duration_nsec = 3;
        uint64 batch_size_bytes = 4;
        // 当共识节点超过request_timeout_nsec未收到PrePrepare消息发起ViewChange消息
        uint64 request_timeout_nsec = 5;
};
message Msg {
        oneof type {
                Request request = 1;
                Preprepare preprepare = 2;
                Subject prepare = 3;
                Subject commit = 4;
                Signed view_change = 5;
                NewView new_view = 6;
                Checkpoint checkpoint = 7;
                Hello hello = 8;
        };
};

message Request {
        bytes payload = 1;
};

message SeqView {
        uint64 view = 1;
        uint64 seq = 2;
};

message BatchHeader {
        uint64 seq = 1;
        bytes prev_hash = 2;
        bytes data_hash = 3;
};

message Batch {
        bytes header = 1;
        repeated bytes payloads = 2;
        map<uint64, bytes> signatures = 3;
};

message Preprepare {
        SeqView seq = 1;
        Batch batch = 2;
};

message Subject {
        SeqView seq = 1;
        bytes digest = 2;
};
// pset Prepare消息
// qset Commit 消息
message ViewChange {
        uint64 view = 1;
        repeated Subject pset = 2;
        repeated Subject qset = 3;
        Batch checkpoint = 4;
};

message Signed {
        bytes data = 1;
        bytes signature = 2;
};

message NewView {
        uint64 view = 1;
        map<uint64, Signed> vset = 2;
        Subject xset = 3;
        Batch batch = 4;
};

message Checkpoint {
        uint64 seq = 1;
        bytes digest = 2;
        bytes signature = 3;
};

message Hello {
        Batch batch = 1;
        NewView new_view = 2;
};
```
