---
layout: post
sharing: true
featured: true
---
本文主要通过源码分析来了解Hyperledger Fabric的密码学模块。

Fabric对密码学相关操作进行了模块化设计，并完成了软件的实现方式，目前的软件实现中，默认对消息的签名验证用ECDSA算法，哈希用SHA256算法,对称加密解密用AES算法。
可快速替换其它解决方案，未来IBM应该也会引入自己的硬件加密方案（如HSM）。  

以下是主要接口
```GO
// BCCSP是区块链密码服务提供商的简称，此接口提供了Fabric里密码学相关操作的函数
type BCCSP interface {

	// 根据参数产生密钥
	KeyGen(opts KeyGenOpts) (k Key, err error)

	// 根据参数派生秘钥
	KeyDeriv(k Key, opts KeyDerivOpts) (dk Key, err error)

	//根据参数导入密钥
	KeyImport(raw interface{}, opts KeyImportOpts) (k Key, err error)

	//根据密钥ID返回密钥
	GetKey(ski []byte) (k Key, err error)

	// 根据参数对消息作哈希运算，返回字节数组
	Hash(msg []byte, opts HashOpts) (hash []byte, err error)

	// 以hash.hash格式返回哈希结果
	GetHash(opts HashOpts) (h hash.Hash, err error)

	// 根据参数用密钥对消息的摘要进行签名
	Sign(k Key, digest []byte, opts SignerOpts) (signature []byte, err error)

	// 根据参数用密钥和消息的摘要对摘要的签名进行验证
	Verify(k Key, signature, digest []byte, opts SignerOpts) (valid bool, err error)

	// 根据参数用密钥对明文进行加密
	Encrypt(k Key, plaintext []byte, opts EncrypterOpts) (ciphertext []byte, err error)

	// 根据参数用密钥对密文进行解密
	Decrypt(k Key, ciphertext []byte, opts DecrypterOpts) (plaintext []byte, err error)
}
```
以下接口是以上接口需要用到的接口
```go
// 密钥接口
type Key interface {

	// 密钥字节数组
	Bytes() ([]byte, error)

	// SKI 可简单理解为密钥的ID.
	SKI() []byte

	// 是否是对称加密的密钥
	Symmetric() bool

	//是否是私钥
	Private() bool

	// 返回公钥
	PublicKey() (Key, error)
}

// 产生密钥的算法接口
type KeyGenOpts interface {

	// 产生密钥的算法
	Algorithm() string

	// 是否临时的密钥
	Ephemeral() bool
}

// 密钥的派生选项接口
type KeyDerivOpts interface {

	// 派生密钥的算法
	Algorithm() string

	// 是否临时的密钥
	Ephemeral() bool
}

// 导入密钥选项接口
type KeyImportOpts interface {
	// 导入密钥
	Algorithm() string

	// 是否临时的密钥
	Ephemeral() bool
}

// 哈希选项接口
type HashOpts interface {
	// 哈希算法
	Algorithm() string
}

// 签名选项接口.
type SignerOpts interface {
	crypto.SignerOpts
}

// 加密选项接口
type EncrypterOpts interface{}

// 解密选项接口
type DecrypterOpts interface{}
```

