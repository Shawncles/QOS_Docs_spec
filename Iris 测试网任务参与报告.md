# 参与 Iris 测试网络

附录参考文档：

[参与 fuxi-8000 测试网络官方文档](http://123)

[fuxi-8000 浏览器在这里]()

[iris 水龙头在这里]()

<br>

##参与的项目及流程：

###1、参与到测试网Genesis文件生成中

1.1、拉取最新版本节点，编译，安装

>[iris安装教程](https://github.com/irisnet/irishub/blob/master/docs/get-started/Install-the-Software.md)

1.2、创建账户（key），如创建一个名为 AA 的私钥

`iriscli keys add {account_name}`

1.3、初始化节点

`iris init --home=/root/ --chain-id=fuxi-8000 --moniker=AA`

1.4、执行gentx交易，并使用刚才创建的验证人账户对交易进行签名

`iris gentx --name=AA --home=/root --ip=1.2.3.4:26625`

文件位于 /root/config/gentx 下，*.json结尾

1.5、fork 一份 iris-testnet 代码到自己的 github，将上述json文件 复制到 `fuxi/fuxi-8000/config/gentx` 目录下，push并提交 PR 到iris-testnet上

*注意：上述用于生成验证人的密钥默认存在 /root/config/gentx 下，要求覆盖 与/root/.iris 下的配置文件保持一致*

<br>

##2、验证人委托及收回收益

