## fabric 虚拟机 

Hyperledger最核心的东西就是chaincode,而用来运行chaincode的环境则是图灵完备的VM,本文试图通过源码分析来理解fabric的VM设计

fabric从设计上是支持vm的多种实现的，目前可选择的有inproc和docker，但是现在基本的测试和实际运行环境都选择用docker。严格来说，目前fabric本身并不实现VM,而是选择现成的VM,比如docker。
具体的做法通过docker client发送相关的操作到宿主主机的docker server进程上，而docker client则选择（github.com/fsouza/go-dockerclient）

下面简单分析相关代码的用途

#### core/container/vm.go
```go
type VM struct{  
    Client *docker.Client //VM本质上一个docker client  
}
```

列出现有的image，相当于在宿主发送docker命令`docker images`  
```go
func (vm *VM) ListImages(context context.Context) error {
}
```

通过chaincode描述结构ChaincodeSpec建立image,相当于在宿主发送docker命令`docker　build`  
```go
func (vm *VM) BuildChaincodeContainer(spec *pb.ChaincodeSpec) ([]byte, error){
}
```

#### core/container/control.go  
则声明的vm的共同函数接口，不同的vm需要实现这些方法
```go
type vm interface {
	Deploy(...) error              //创建image
	Start(...) error               //启动VM
	Stop(...) error                //停止VM
	Destroy(...) error             //销毁VM
	GetVMName(...) (string, error) //获取VM名
}
```
目前具体的实现有  
`core/container/dockercontroller/dockercontroller.go` 和`core/container/inproccontroller/inproccontroller.go`


