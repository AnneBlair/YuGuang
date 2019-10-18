---
title: 'Swift 部署智能合约'
subtitle: '开发Dapp应用必须通过移动端直接跟区块链进行交互，那么如何部署智能合约？看看这篇文章吧' 
summary: 开发Dapp应用必须通过移动端直接跟区块链进行交互，那么如何部署智能合约？看看这篇文章吧
authors:
- admin
tags:
- Academic
categories:
- 
date: "2018-07-12T00:00:00Z"
lastmod: "2018-07-12T00:00:00Z"
featured: false
draft: false

# Featured image
# To use, add an image named `featured.jpg/png` to your page's folder.
# Placement options: 1 = Full column width, 2 = Out-set, 3 = Screen-width
# Focal point options: Smart, Center, TopLeft, Top, TopRight, Left, Right, BottomLeft, Bottom, BottomRight
image:
  placement: 2
  caption: 'Image credit: [**Unsplash**](https://unsplash.com/photos/CpkOjOcXdUY)'
  focal_point: ""
  preview_only: false

# Projects (optional).
#   Associate this post with one or more of your projects.
#   Simply enter your project's folder or file name without extension.
#   E.g. `projects = ["internal-project"]` references `content/project/deep-learning/index.md`.
#   Otherwise, set `projects = []`.
projects: []
---

[博客原文链接](https://juejin.im/post/5b46f3b1f265da0f6d72c279)

**因为项目业务原因，需要在新的项目中部署以太坊的智能合约，关于这方面安卓方面貌似走在了前面。 [web3j](https://github.com/web3j/web3j) 的存在实在是喜人,而 iOS版本也是有的,功能貌似没有那么强大，对付平常的区块链交互是足够的了。但是iOS这个他没有仔细的说明，我研究了一下其实部署起来挺简单. 年纪大了搞过的东西很容易忘记，于是在这里记录一下.一来自己以后也可以回顾，二来为有需要的人略尽绵薄之力**

## 使用 [web3swift](https://github.com/BANKEX/web3swift) 部署以太坊智能合约

### 1.配置服务
#### a. IP地址
配置以太坊的服务器地址：
我这里是本地IP地址

```
struct Api {

    static let host = "http://192.168.6.66:6666"
    
    static func map(path: String) -> String {
        
        return host + path
    }
    
    static var chainUrl: String {
        return map(path: "")
    }
}
```
#### b. abiString
部署合约会需要

```
public let abiString = "[{\"constant\":true,\"inputs\":[],\"name\":\"getFlagData\",\"outputs\":[{\"name\":\"data\",\"type\":\"string\"}],\"payable\":false,\"stateMutability\":\"view\",\"type\":\"function\"},{\"constant\":false,\"inputs\":[{\"name\":\"data\",\"type\":\"string\"}],\"name\":\"setFlagData\",\"outputs\":[],\"payable\":false,\"stateMutability\":\"nonpayable\",\"type\":\"function\"}]"
```

#### c. byteCode
这个是一串很长的东西，合约生产的。没有的记得找写合约的要.

我这里的例子：

```
public let BINARY = "606060409081526003805460a060020a61ffff021916905560006004558051908101604052600981527f4c696e67546f6b656e0000000000000000000000000000000000000000000000602082015260059080516100619291602001906100cf565b5060408051908101604052600481527f4c494e4700000000000000000000000000000000000000000000000000000000602082015260069080516100a99291602001906100cf565b50601260075560038054600160a060020a03191633600160a060020a031617905561016a565b82805460018160011615610100020316600290049若干省略号
```

#### d. 集成 pod


```
pod 'web3swift', '~> 0.8.0'
```


### 2.部署前奏

#### a. 创建账户，**web3swift** 的示例中有，同学可以仔细研究研究，我这里就简单过。
上代码：

```
class YYGKey {
    
    var key_password = "BANKEXFOUNDATION"
    
    var storeManager: KeystoreManager? {
        return keyStoreManager()
    }
    
    var keyAddress: EthereumKeystoreV3? {
        return addressManage()
    }
    
    convenience init(password: String) {
        self.init()
        self.key_password = password
    }

    
    func addressManage() -> EthereumKeystoreV3? {
        guard let userDir = NSSearchPathForDirectoriesInDomains(.documentDirectory, .userDomainMask, true).first else { return nil }
        let storeManager = KeystoreManager.managerForPath(userDir + "/keystore")
        if let manage = storeManager, let address = manage.addresses {
            return address.isEmpty ? creatAddress(dir: userDir) : getAddress(manage: manage)
        }
        return nil
    }
    
    func creatAddress(dir: String) -> EthereumKeystoreV3? {
        let ks = try! EthereumKeystoreV3(password: "BANKEXFOUNDATION")
        if let ks_ = ks {
            let keydata = try! JSONEncoder().encode(ks_.keystoreParams)
            FileManager.default.createFile(atPath: dir + "/keystore"+"/key.json", contents: keydata, attributes: nil)
        }
        return ks
    }
    
    func getAddress(manage: KeystoreManager) -> EthereumKeystoreV3? {
        if let address = manage.addresses, let first = address.first {
            return manage.walletForAddress(first) as? EthereumKeystoreV3
        }
        return nil
    }
    
    func keyStoreManager() -> KeystoreManager? {
        guard let userDir = NSSearchPathForDirectoriesInDomains(.documentDirectory, .userDomainMask, true).first else { return nil }
        return KeystoreManager.managerForPath(userDir + "/keystore")
    }
}
```

b. 连接到部署的服务器

```
class YYG {
    
    var chain: web3? {
        guard let url = URL(string: Api.chainUrl), let web = Web3.new(url) else { return nil }
        return web
    }
    
    static let yyg = YYG()
    
    private init() { }
}
```

### 3.开始部署

**需要的数据**

* EthereumKeystoreV3
* EthereumAddress
* web3
* byteCode
* web3contract
* Web3Options
* TransactionIntermediate
* KeystoreManager

上代码：

```
struct YYGOperate {
    
    typealias result = Result<[String : String], Web3Error>
    typealias res = [String : String]
    
    func contractsResult(message: @escaping (Bool,res?) -> Void) {
        if let resul = smartContracts() {
            switch resul {
            case .success(let res):
                DispatchQueue.main.async {
                    message(true, res)
                }
            case .failure(let error):
                message(false, nil)
                print(error)
            }
        } else {
            message(false, nil)
        }
    }
    
    private func smartContracts() -> result? {
        guard let manage = YYGKey().addressManage(),let addresss = manage.addresses,let address = addresss.first, let chain = YYG.yyg.chain else { return nil }
        chain.provider.network = nil
        chain.addKeystoreManager(YYGKey().keyStoreManager())
        
        guard let contract = chain.contract(abiString, at: nil, abiVersion: 2), let byteCode = Data.fromHex(BINARY) else { return nil }
        var options = Web3Options.defaultOptions()
        options.from = address
        options.gasLimit = BigUInt(3000000)
        let inter = contract.deploy(bytecode: byteCode, options: options)
        guard let intermediate = inter else { return nil }
        return intermediate.send(password: "BANKEXFOUNDATION", options: options)
    }
}
```

### 3.执行

```
YYGOperate().contractsResult { state, res in
    if state { printLogDebug(res) }
}
```

结果：

```
Transaction
Nonce: 10
Gas price: 5000000000
Gas limit: 1111340
To: 0x
Value: 0
Data: 0x606060409081526003805460a060020a61ffff02191690556090810183818151815260200191508051906020019080838360005b8381101561017657808201518382015260200161015e565b50、5260200160405180910390a35050505050565b6000828201610acc82fd76404b5c55184a2e555312d3353bcb43e75666dd36d0af562b0c220029a165627a7a72305820f3efa245dad6ee48cc6d771a54067fb6e3b17b0caba5ecd4eed13aca2b1c00960029
v: 28
r: 79879464740714101996923170089344479265591202785750775462035438111705924396692
s: 39143622105442894879914151031821436277241889279036648521816051027427579390397
Intrinsic chainID: nil
Infered chainID: nil
sender: Optional("0x9E40Fb081777c232aBafc256165772572c057F95")
hash: Optional("0xcc16c3a245e4fd27b85e53b8cb6d0ac64028b60e61ca3de938e896e95202bde3")

```

### 授人以鱼不如授人以渔 
**学习资料**

* [以太坊智能合约](https://github.com/nebulasio/wiki/blob/master/tutorials/%5B中文%5D%20Nebulas%20101%20-%2003%20编写智能合约.md)
* [web3swift](https://github.com/BANKEX/web3swift)
* [以太坊](https://github.com/ethereum/go-ethereum)
* [以太坊官方文档](https://github.com/ethereum/go-ethereum/wiki/Management-APIs)
* [安卓web3j](web3j)
* [学习区块链全部资料](https://github.com/chaozh/awesome-blockchain-cn)