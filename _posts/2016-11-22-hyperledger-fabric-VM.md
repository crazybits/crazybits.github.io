---
layout: post
---

Hyperledger最核心的东西是chaincode,而用来运行chaincode的环境则是图灵完备的VM,本文试图通过源码分析来理解fabric的VM设计

fabric从设计上是支持vm的多种实现的，目前可选择的有inproc和docker，但是现在测试和实际运行环境基本都选择用docker。严格来说，目前fabric本身并不实现VM,而是选择现成的VM,比如docker。
具体的做法通过docker client发送相关的操作到宿主主机的docker server进程上，而docker client则选择（github.com/fsouza/go-dockerclient）

<!--more-->
下面简单分析相关代码的用途

### core/container/vm.go
```go
type VM struct{  
    Client *docker.Client //VM本质上是一个docker client  
}
```

列出现有的image，相当于在宿主发送docker命令`docker images`  
```go
func (vm *VM) ListImages(context context.Context) error {
    ...
    imgs, err := vm.Client.ListImages(docker.ListImagesOptions{All: false})
    ...
}
```

通过chaincode描述结构`ChaincodeSpec`建立image,相当于在宿主主机发送docker命令`docker　build`  
```go
func (vm *VM) BuildChaincodeContainer(spec *pb.ChaincodeSpec) ([]byte, error){
    ...
    err := vm.Client.BuildImage(opts)
}
```

### core/container/control.go  
则声明的vm的共同函数接口，不同的vm需要实现这些方法
```go
type vm interface {
	Deploy(...) error              //创建image
	Start(...) error               //启动VM
	Stop(...) error                //停止VM
	Destroy(...) error             //销毁VM
	GetVMName(...) (string, error) //获取VM名
}

根据接口`VMCReqIntf`不同实现的do方法调用具体VM的Deploy,Start,Stop,Destory方法
func VMCProcess(ctxt context.Context, vmtype string, req VMCReqIntf) (interface{}, error) {
    ...
    resp = req.do(ctxt, v)
    ...
}
type VMCReqIntf interface {
	do(ctxt context.Context, v vm) VMCResp
	getCCID() ccintf.CCID
}

```
目前VM的具体的实现有以下两种:  
`core/container/dockercontroller/dockercontroller.go`  
`core/container/inproccontroller/inproccontroller.go`


