# Qbase & QOS代码解读报告

## Qbase 的几大调用模块
#### 1、账户
	
	数据结构：BaseAccount 			和 AccountMapper // 对BaseAccount存储操作进行包装的结构，可进行序列化
				|-- AccountAddress			|-- GetAccount(addr types.Address)
				|-- Publickey						|-- AddressStoreKey(addr) //// 从存储中获得账户
				|-- Nonce										|--DecodeObject				
		
	
#### 2、BaseApp

	BeginBlock
		|-- setDeliverState // Initialize the DeliverTx state
		|-- ResetBlockTxIndex // 重置block tx index
		|-- beginBlocker //返回 abci.ResponseBeginBlock
		
	CheckTx // FIXME：checkTx 过于简单，to 是一个 “a” 也可以是一个合法的交易（销毁token的地址？）
		|-- ctx.WithTxBytes(txBytes) // 初始化 context 相关数据
		|			|-- c.withValue(contextKeyTxBytes, txBytes) 
		|
		|-- checkTxStd(ctx, implTx) //校验的主体
		|			|-- ValidateBasicData //校验基础信息
		|			|-- validateTxStdUserSignatureAndNonce //校验签名
		|
		|-- toResponseCheckTx //通过 receive 的多路复用，更新信息
	
	DeliverTx //和 CheckTx 的过程有些类似
		|-- lastBlockTxIndex // deliverTx处理tx时，设置tx index
		|-- deliverTxStd、toResponseDeliverTx
				|-- 基础数据、签名
				|-- saveCrossChainResult // crossTxQcp结果判断是否保存跨链结果
	EndBlock
	 	|-- abci.ResponseEndBlock
	 	
	Commit
		|-- cacheKVStore.Write() //Writes operations to underlying KVStore cacheKVStore
		|			|-- Set([]byte(key), cacheValue.value) //执行存储操作
		|-- commitmultistore.Commit() //返回 commit hash
		|					|-- commitStores(version, stores) 返回 commitInfo
		|					|-- 写入数据库
		|					|-- 返回 commitID
		|
		|-- 返回 abci.ResponseCommit{Data: commitID.Hash}
		
#### 3、keys
	
	BIP44Params
		

	DerivePrivateKeyForPath(privKeyBytes, chainCode, path)
				|-- derivePrivateKey(data, chainCode, uint32(idx), harden) // idx harden 都是根据 path 中每一部分生成
				|			|-- PrivKeyFromBytes // priv -> pub
				|			|-- SerializeCompressed_pubkey
				|			|-- i64 //sha3 512 返回两个 32 位值
				|-- 返回 derivedKey
				
	// &BIP44Params{44, 118, 52, true, 41}, "44'/118'/52'/1/41"}
	//   			purpose coinType account change addressIdx
	
	Keybase 
	 	|-- CreateEnMnemonic // 创建 mnemonic 和对应的 私钥
	 	|-- Derive
	 			|-- persistDerivedKey(seed, passwd, name, fullHdPath) // 传入上述参数，生成 priKey 并存储到本地
	 			|-- DerivePrivateKeyForPath
	 			
	 Sign
	 	|-- info 的多路复用，根据 import、offline、local 不同类型，解密出私钥，用私钥进行签名
	 
	 Export
	 	|-- db.Get(infoKey(name)) // 根据 name，导出私钥
	 
	 Import
	 	|-- Import(name string, armor string) // 导入 name 和，ASCII 编码的私钥，在kv数据库中存储
	 
	 Update 		   oldpass 定位kvstore func
	 	|-- writeLocalKey(priv, name, newpass)
	 // 更新数据库的私钥的解锁密码
	 
	 Delete //删除私钥
		|-- Delete(name, passphrase）
		
4、txs

	SignTx 
		|-- BuildSignatureBytes // 一些序列化的操作 （由应用层抽象？）
		|-- ed25519 签名

---

# QOS

#### 1、account

#### 2、app

	QOSApp
		|-- NewApp 
				|-- initChainer
				|-- SetBeginBlocker // handerler 定义 qbase 库中的abci.Responce
				|-- RegisterMapper // 为 app 注册预定义的数据结构{cdc key store}

#### 3、cmd

#### 4、txs

##### 4.1、approve 
	
	ValidateData
	
	Exec
	
	结构体（操作类型）：
	TxCreateApprove
		BuildApproveKey
		BaseMapper.Set(key, approve)
		
	TxIncreaseApprove
	
	TxDecreaseApprove
	
	TxUseApprove
	
	TxCancelApprove
	
##### 4.2、qcp

##### 4.3、qsc

##### 4.4、staking

	* ABCI：// 对validator表，进行CRUD操作
	BeginBlocker
		|-- handleValidatorValidatorVoteInfo
						|-- MissedBlocksCounter > maxMissedCounter？
						|-- blockValidator(ctx, valAddr)
									|-- MakeValidatorInactive
									
	EndBlocker //将所有Inactive到一定期限的validator删除，统计新的validator
	
	GetUpdatedValidators
	
	* Tx*Validator // validator 的操作交易		
	TxCreateValidator：
		|-- NewCreateValidatorTx
		|-- Get*
		|-- 
	TxRevokeValidator：
		|-- NewRevokeValidatorTx
		
	TxActiveValidator：
		|-- NewActiveValidatorTx
			
	* BuildValidator： //根据传参创建validator					
						
##### 4.5、transfer
	
	* 数据校验
	ValidateData
	
	* 转账
	Exec

	CalcGas // size？1）relay fee 共识的基础费用，bestblockfee 打包效率
	GetGasPayer //from amount？
	GetSignData //序列化

功能注释
				
				
				
