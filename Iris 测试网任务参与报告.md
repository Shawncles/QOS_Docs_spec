# 参与 Iris 测试网络

附录参考文档：

[参与 fuxi-8000 测试网络官方文档](http://123)

[fuxi-8000 浏览器在这里](https://testnet.irisplorer.io/#/home)

[iris 水龙头在这里]()

<br>

# 参与的项目及流程：

## 1、参与到测试网Genesis文件生成中

#### 1.1、拉取最新版本节点，编译，安装

>[iris安装教程](https://github.com/irisnet/irishub/blob/master/docs/get-started/Install-the-Software.md)

#### 1.2、创建账户（key），如创建一个名为 AA 的私钥

`iriscli keys add {account_name}`

#### 1.3、初始化节点

`iris init --home=/root/ --chain-id=fuxi-8000 --moniker=AA`

#### 1.4、执行gentx交易，并使用刚才创建的验证人账户对交易进行签名

`iris gentx --name=AA --home=/root --ip=1.2.3.4:26625`

文件位于 /root/config/gentx 下，*.json结尾

#### 1.5、fork 一份 iris-testnet 代码到自己的 github，将上述json文件 复制到 `fuxi/fuxi-8000/config/gentx` 目录下，push并提交 PR 到iris-testnet上

*注意：上述用于生成验证人的密钥默认存在 /root/config/gentx 下，要求覆盖 与/root/.iris 下的配置文件保持一致*

<br>

## 2、验证人委托及收回收益

#### 2.1、执行委托操作

`iriscli stake delegate --address-validator fva184499gxhcr8a0l80jwntry6faz7t4qs5mdvclk --amount 2000iris --fee 5iris --memo QOS --from rock --chain-id fuxi-8000`

上述命令执行了一个委托操作，将2000 iris 委托给 fva184499gxhcr8a0l80jwntry6faz7t4qs5mdvclk 地址的验证人

`tx hash: 6F177643E3E3D7584CCCD31DF6747D8B5143C4A96D891BEC3D73EDAB18E8E35D`

#### 2.2、执行委托收益赎回操作

`iriscli distribution withdraw-rewards --chain-id fuxi-8000 --fee 4iris --from rock --memo QOS`

上述命令执行收益赎回操作

`tx hash: 3469326BB40D49EE71FF110062782579A50D071296682A281D20BAF82C5B2004`

## 3、链上治理

#### 3.1、参与到链上治理，通过提案

`iriscli gov vote --chain-id=fuxi-8000 --proposal-id=1 --option=Yes --from rock --fee=0.01iris --memo QOS`

上述命令使得节点提交通过了 proposal-id=1 的提案

`tx hash: 63B731999EBA60356F029BE443332C60C542AAA427F4CE168C39DA36F04518CF`


#### 3.2、升级软件

```
wget https://github.com/irisnet/irishub/releases/download/v0.10.2/irishub_0.10.2_linux_amd64.zip
unzip -d /usr/local/bin/ ./irishub_0.10.2_linux_amd64.zip
```

#### 3.3、委托，并取回委托收益
委托: 
`tx hash: 47593FC8A1AC9B27A30712FC1676755AC4105EE7B350D4AE5D88EE186AFDC151`

取回：
`tx hash: C592C15300C01A476AEB9BB7101F584B7E3E693FF03DBDF195C89C362610D74D`

-----




## 4、测试网第四阶段

#### 4.1、投票通过共识停止systemhalt提案
`iriscli gov vote --chain-id=fuxi-8000 --from rock --fee 4iris --proposal-id 1 --option=Yes --memo=QOS`

投票结果：
`tx hash: AF8E52D6B93FF611C2233CA3B7B53F37FE72B3C893E03EE009090CA1CC7F1CEB`

#### 4.2、irishub软件升级到v0.10.3

插曲：测试链在同步上遇到了问题，未采取 unsafe-reset-all ，等待 iris 开发修复，修复完成后发现自己的节点被 jail 了，于是 unjail 自己的节点

`iriscli stake unjail --chain-id=fuxi-8000 --from rock --fee 4iris --memo=QOS`


#### 4.4、委托，并取回委托收益
委托: 
`tx hash: 81929D86425DBD8E9AB1945F234194FF74A0A682690DD6D0BE9E52934CC04064 `

取回：
`tx hash: 31B084DE526405C4E0A43F00277EDE7D9B559CF8439FFF62811D3F13D022D6D3`
