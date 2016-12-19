---
layout: post
featured: true
---

Hyperledger Fabirc拜占庭 共识算法 spbft(simplified Practical Byzantine Fault Tolerance)

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
// 客户端向共识节点发出Request消息，
// 主节点收到的Request消息达到一定数量或者一定的时间间隔后会为其分配View序列号，并向其它共识节点发起Preprepare消息
// 当共识节点收到合法的Preprepare消息后将发起Prepare消息
// 当共识节点收到n-f-1个Prepare消息验证合法后将发起Commit消息

// 当共识节点过了request_timeout_nsec仍然未收到Preprepare消息则发起ViewChange消息重选主节点
// 当共识节点收到2f+1个ViewChange消息后则发起NewView消息
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
