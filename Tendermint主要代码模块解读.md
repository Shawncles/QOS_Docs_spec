# Tendermint 架构

#### Tendermint共识过程架构图

![Tendermint共识过程](https://cdn-images-1.medium.com/max/1600/1*7yVAe1NvayAd6WDhXoqKSg.png 'Tendermint 共识过程')

#### 共识过程：
>1、用户提交交易
>
>2、对app发送检查交易请求，通过检查后加入mempool
>
>3、选出一个validator节点作为proposer，打包区块后广播给众校验节点
>
>4、校验节点校验新的块 prevote
>
>5、校验通过后，签名并广播回proposer节点 procommit
>
>6、proposer收到大于 2/3 校验节点的确认，对app请求 BeginBlock deliverTx EndBlock Commit 确认了新的块，将其上链
>
>7、回到第 3 步

备注：proposer打包区块可以挑选自己想要的交易，一般以最大手续费为准

<br>

# Tendermint主要代码模块解读：

主要模块：

* 用户控制
* P2P网络
* RPC请求
* 数据存储



## 1、用户控制

### 1.1 tendermint init

	initFilesWithConfig(config *cfg.Config)

> 在 *～/.tendermint/* 下生成以下文件：

> *genesis.json、node_key.json、priv_validator.json*

//config.toml

### 1.2 tendermint node
> tendermint node 检查并启动了以下服务
	
	|-- n, err := nodeProvider(config, logger)(*Node, error) //建立新的node对象
	|
	|-- n.Start() ---->
					  |-- n.eventBus.Start() //启动订阅服务，eventBus说明：
					  |		//EventBus is a common bus for all events going through the system. All calls are proxied to underlying pubsub server. All events must be published using EventBus to ensure correct data types.
					  |
					  |-- n.startRPC() 
					  |		//启动 http 服务，注册并监听 /websocket 下的流量，同时启动一个 grpc 服务
					  |
					  |-- n.startPrometheusServer(addr string)
					  | 	// starts a Prometheus HTTP server, listening for metrics collectors on addr.
					  |
					  |-- n.transport.Listen(*addr)
					  |		// 启动 p2p transport 服务
					  |		// MultiplexTransport accepts and dials tcp connections and upgrades them to multiplexed peers.
					  |
					  |-- n.sw.Start() // 启动p2p服务关口，并且主动连接到主节点
					  | 	// Switch handles peer connections and exposes an API to receive incoming messages on `Reactors`. Each `Reactor` is responsible for handling incoming messages of one or more `Channels`. So while sending outgoing messages is typically performed on the peer, incoming messages are received on the reactor.
					  |
					  |-- n.indexerService.Start() 
					  		//IndexerService connects event bus and transaction indexer together in order to index transactions coming from event bus.
						
	

### 1.3 gen\_node_key
> 检查节点私钥是否存在，否则利用 ed25519.GenPrivKey() 生成私钥


### 1.4 gen_validator
> 同样使用 ed25519 生成私钥后，将结构体 marshal 成字符串并打印出来

除了上述之外还有诸多显示节点信息相关的命令行命令。

<br>

## 2、P2P 传输
p2p 主要包含以下几个部分：

### 2.1 Reactor 
	base_reactor.go
一个 reactor 可以实现以下方法：

	SetSwitch 		//allows setting a switch
 	GetChannels 	//returns the list of channel descriptors
 	AddPeer 		//is called by the switch when a new peer is added
 	RemovePeer 		//is called by the switch when the peer is stopped
 	还有最重要的功能： Receive(chID byte, peer Peer, msgBytes []byte) // Receive is called when msgBytes is received from peer

### 2.2 MConnConfig
MConnConfig 结构体保存了 P2PConfig 的内容，包含了诸如：发送接收频率，消息包的大小，刷新频率，连接频率和timeout时长等配置

### 2.3 Metrics
一个 Metrics 结构体包含以下参数：

	Peers 					// number of peers
	PeerReceiveBytesTotal 	// Number of bytes received from a given peer.
	PeerSendBytesTotal 		// Number of bytes sent to a given peer.
	PeerPendingSendBytes 	// Pending bytes to be sent to a given peer.
	NumTxs 					// Number of transactions submitted by each peer

### 2.4 Peer
一个 Peer 可以实现包含以下方法：

	Send(byte, []byte) //通过通道发送数据
	TrySend(byte, []byte)
	Set(string, interface{})
	Get(string) interface{}
	
以及与连接有关的函数：

	IsOutbound()
	IsPersistent()
	OriginalAddr()
	RemoteIP()

2.2、2.3，2.4 三部分是构成 peer 结构体的主要内容。

2.1 是对peer的各种方法的抽象，对应了对peer能够执行的操作。

### 2.5 Switch
本质上 P2P 网络中的所有节点都是 Peer，但当节点传输消息时，将其抽象为一个 Switch，具有连接的功能，负责接收与广播消息，并可以实现 2.1 中的所有方法，

### 2.5 其他
除了上述构成 P2P 网络的主体结构、之外，还有维护 P2P 网络高效性的一些附属结构：
	
	AddrBook //用于存储 peer addresses.
	MultiplexTransport 握手和通信的主体
	
因此，简单的流程就是 peer 作为 P2P 网络的主体结构，在通信时抽象成 Switch 结构广播和接收消息，并通过 MultiplexTransport 进行通信

<br>

## 3、RPC：Utility and Query
除了 用户控制功能，P2P网络通信之外，作为区块链共识 tendermint 还准备了查询功能。

RPC理解起来比较简单，就是启动http服务，在路由上注册路径，请求路径上的资源，返回结果给查询端。

tm的RPC最常用的应该就是 提交交易，查询，这两部分。

tendermint的提交交易，可以通过三个函数来完成：

	1、BroadcastTxCommit
	2、BroadcastTxAsync
	3、BroadcastTxSync

从源码上看，BroadcastTxAsync 和 BroadcastTxSync 做了一样的事情（v0.27.4）, BroadcastTxCommit 和另外两者的差别在于 Commit 等待 deliverTx 并在返回结果中包含 deliverTx 的信息，Sync 提交Tx之后直接返回，只包含 CheckTx 的结果

深入 BroadcastTxCommit 能够看到函数内部调用逻辑如下：

	|-- 调用 rpc.Call，传入 method，parameters，和 result 作为返回参数
				|
				|-- 把 method, params marshal 成字符串，
				|
				|-- Post(address, "text/json", requestBuf) //使用 post 方法将请求 post 到某个具体的地址，比如 localhost:26657/broadcast_tx_commit
				|
				|-- 服务端根据 handlers 中注册的的地址对应的服务，返回相应的逻辑结果，比如 BroadcastTxCommit 则是在 endpoint 上注册了 /broadcast_tx_commit 的地址，在请求该资源时调用 mempool.go 下的 BroadcastTxCommit 函数
				|
				|-- 将返回结果 marshall 成字符串，返回到请求端
				
所有的 RPC 请求处理逻辑与上述一致，具体的业务会有差异。可以对应某项业务查看源代码了解其工作内容

<br>
				
## 4、存储
存储涉及到交易缓存、链状态更新，包括的主要内容有：mempool 的更新、创建区块、新区块的确认、新 state 的更新等

### 4.1、mempool
service.go
mempool的接口中包含以下方法：

	Lock()
	Unlock()

	Size() int
	CheckTx(types.Tx, func(*abci.Response)) error
	ReapMaxBytesMaxGas(maxBytes, maxGas int64) types.Txs
	Update(int64, types.Txs, mempool.PreCheckFunc, mempool.PostCheckFunc) error
	Flush()
	FlushAppConn() error

	TxsAvailable() <-chan struct{}
	EnableTxsAvailable()

能够看到，mempool 所执行的功能主要有：

* 锁定、解锁
* 向 app 发起 checkTx 的请求
* 更新 mempool，更新的需求来源可能包括：1）用户提交了新的合法的交易；2）出了新的块
* 刷新

相关函数：
	
	proxyApp.CommitSync //Commit block, get hash back
		|-- BeginBlockSync
		|-- DeliverTxAsync
		|-- EndBlockSync 
		|-- Commit
		
	mempool.Update(block.Height,
			block.Txs,
			TxPreCheck(state),	//to filter transactions before processing.
			TxPostCheck(state)) //filter transactions after processing.

### 4.2 State

> State 结构体中的主要内容
> 

	Version: state.Version,
	ChainID: state.ChainID,
	ckHeight=0 at genesis (ie. block(H=0) does not exist)
	LastBlockHeight  int64
	LastBlockTotalTx int64
	LastBlockID      types.BlockID
	LastBlockTime    time.Time

	// LastValidators is used to validate block.LastCommit.
	// Validators are persisted to the database separately every time they change,
	// so we can query for historical validator sets.
	// Note that if s.LastBlockHeight causes a valset change,
	// we set s.LastHeightValidatorsChanged = s.LastBlockHeight + 1 + 1
	// Extra +1 due to nextValSet delay.
	NextValidators              *types.ValidatorSet
	Validators                  *types.ValidatorSet
	LastValidators              *types.ValidatorSet
	LastHeightValidatorsChanged int64

	// Consensus parameters used for validating blocks.
	// Changes returned by EndBlock and updated after Commit.
	ConsensusParams                  types.ConsensusParams
	LastHeightConsensusParamsChanged int64

	// Merkle root of the results from executing prev block
	LastResultsHash []byte


### 4.3 BlockExecutor

	db dbm.DB

	// execute the app against this
	proxyApp proxy.AppConnConsensus

	// events
	eventBus types.BlockEventPublisher

	// update these with block results after commit
	mempool Mempool
	evpool  EvidencePool

	logger log.Logger

	metrics *Metrics
	
BlcokExcutor 主要是对 Block 进行验证和操作

主要涉及到的函数：ApplyBlock

	ApplyBlock(state State, blockID types.BlockID, block *types.Block)
		|
		|-- ValidateBlock(state, block) // 单纯验证区块信息，不更新State
		|			|-- basic info、Version、ChainID、Height
		|			|-- LastBlockID
		|			|-- TotalTxs
		|			|-- AppHash、ConsensusHash、LastResultsHash、ValidatorsHash、NextValidatorsHash
		|			|-- LastCommit
		|			|-- Time
		|			|-- ProposerAddress
		|			
		|-- execBlockOnProxyApp
		|				|-- SetResponseCallback //Execute transactions and get hash.
		|				|-- getBeginBlockValidatorInfo
		|				|-- BeginBlockSync
		|				|-- DeliverTxAsync
		|				|-- EndBlockSync 
		|				
		|-- saveABCIResponses //Save the results before we commit.
		|
		|-- validateValidatorUpdates //validate the validator, updates and convert to tendermint types
		|
		|-- updateState
		|		|-- NextValidators.Copy()
		|		|-- LastHeightValidatorsChanged
		|		|-- LastHeightConsensusParamsChanged
		|		|-- 返回更新后的 State
		|
		|-- blockExec.Commit
		|		|-- proxyApp.CommitSync //Commit block, get hash back
		|		|					|-- BeginBlockSync
		|		|					|-- DeliverTxAsync
		|		|					|-- EndBlockSync 
		|		|					|-- Commit
		|		|
		|		|-- mempool.Update(block.Height,
		|							block.Txs,
		|							TxPreCheck(state),	//to filter transactions before processing.
		|							TxPostCheck(state)) //filter transactions after processing.
		|
		|-- evpool.Update(block, state)
		|
		|-- SaveState(blockExec.db, state) //Update the app hash and save the state.
		|
		|-- fireEvents
		|
		|-- 返回最新的state



### 4.4 TxIndexer

TxIndexer 对交易内容做索引，索引标签包含，tags, height, 和hash

	// AddBatch analyzes, indexes and stores a batch of transactions.
	AddBatch(b *Batch) error
		|-- storeBatch.Set(k, v)
	
	// Index analyzes, indexes and stores a single transaction.
	Index(result *types.TxResult) error

	// Get returns the transaction specified by hash or nil if the transaction is not indexed or stored. // 根据hash，返回 tx_result
	Get(hash []byte) (*types.TxResult, error)

	// Search allows you to query for transactions.
	Search(q *query.Query) ([]*types.TxResult, error)
		|-- query hash, return tx for spec hash immediately
		|
		|-- query range, like: tx.height>5, 使用lookForRanges 将数学语言翻译成数组，根据key（tag），匹配出交易的 hash, 调用Get 获得交易内容

用户提交交易
更新mempool
创造新的块
更新State（上链）
更新索引

---

## 5 共识模块

#### 5.1、ConsensusState // 处理 vote、proposal、达成共识、确认出块，触发事件 

其中： RS // 内部状态机，应对peer收发信息、确认状态

	回执函数：
	readReplayMessage //把 msg 的信息logger出来
	catchupReplay // 确认 csHeight 对应的同步状态，把信息写到 cs.Log 中
	
	查询函数：GetState GetRoundState GetValidators // 针对性的返回cs中指定的信息
	
	写入消息队列：
	SetProposal // 把 proposal 写入 msg queue 中
	AddProposalBlockPart // 
	SetProposalAndBlock
	
	updateToState // 更新 ConsensusState 增加 state 高度，重置cs中的部分信息，开启下一个step
	
	receiveRoutine // 从各种 msgQueue 中读取数据执行相应的处理
	调用了比如： 
		|--	handleMsg // 根据 msg 的类型，调用函数更新 cs
		|-- handleTimeout // timeoutInfo中的Step，选择再次进入共识的某个阶段
				|-- enterNewRound
				|-- enterPropose
				|-- enterPrevote
				|-- enterPrecommit
	
	共识
	SignVote // vote 结构体中包含 Height、Round、Timestamp、BlockID、ValidatorAddress、Address、ValidatorIndex、Signature 这些字段
		|-- PV file，// 核对 PVfile 中和当前vote中包含的信息的一致性，比如高度，round 之类的。PrivValidator // 用来防止双签
		|-- vote.SignBytes(chainID) 获得sign data
		|-- 使用 ed25519 sign 签名
				
	一轮共识结束共识出块
	createProposalBlock
		|-- 读取 ConsensusParams 中各个参数
		|-- cs.mempool.ReapMaxBytesMaxGas //从mempool 中取出tx
		|-- 获得当前验证节点的信息
		|-- MakeBlock //创建 block
	
	确认出块的过程中：
	enterCommit // +2/3 precommits for block
		|-- Votes.Precommits(commitRound).TwoThirdsMajority()
		|-- cs.LockedBlock 写入 cs.ProposalBlock
		
	finalizeCommit
		|-- blockStore.SaveBlock(block, blockParts, seenCommit)
		|-- ApplyBlock //Execute and commit the block, update and save the state, and update the mempool
		
#### 5.2、PeerState

	更新信息：
	SetHasProposal // 在 PRS 中写入 proposal 信息
	SetHasProposalBlockPart // 
	SetHasVote
	
	PickSendVote // 执行发送vote 的主体 CS 
	
	更新 state 状态，把结构体数据写入 queue 或者重制结构体
	ApplyNewRoundStepMessage /
	ApplyNewValidBlockMessage
	ApplyProposalPOLMessage
	ApplyHasVoteMessage
	ApplyVoteSetBitsMessage

其他的数据结构：

* RS // defines the internal consensus state. // 保存自身维护的共识信息

* PRS // contains the known state of a peer. // 对 peer 的状态抽象，暂时的，复用的数据结构，保存了自身维护的peer在共识各个阶段的信息




#### 5.3、ConsensusReactor

* rs, prs, ps, peer 的主要关系

* rs： consensus state 包含 高度、round、block、vote、proposal等等需要共识的内容

* prs： peer round state 包含 高度、round、proposal、provote、procommit、commit相关的信息

* ps： 包含上述两项的数据结构

* peer：p2p的一个节点，有节点id之类的信息

```

	* gossip
		gossipDataRoutine
				|-- DataChannel
						|-- BlockPartMessage H R P / H R S
				ProposalBlockPartsHeader？
				|-- GetXXState //获得round state / peer round state
				|-- SetHasProposalBlockPart // 把 prs 的信息更新到 ps 中
				
				|-- gossipDataForCatchup // 检查自己的同步情况，有落后就跟上
					
				prs.Height < rs.Height？// 检查 prs 和 rc 的高度
				|-- gossipDataForCatchup
							|-- blockMeta := conS.blockStore.LoadBlockMeta(prs.Height) 根据 prs 的高度，从cs 中逐个读取
							|—- peer.Send(DataChannel, msg) // 发送 block 数据
							|-- ps.SetHasProposalBlockPart(prs.Height, prs.Round, index) //更新本地的 peer state
							
		gossipVotesRoutine //同步 vote 信息
				|-- gossipVotesForHeight // 这个函数中会根据 prs 中各变量的状态，选择发送某种 vote set
							|-- PickSendVote // 执行发送vote 的主体
										|-- ps.peer.Send(VoteChannel, msg) // cdc编码后把msg发送到router的VoteChannel通道上
										|-- ps.SetHasVote(vote) //更新 ps 数据
	
	* broadcast
		broadcastNewRoundStepMessage
				|-- makeRoundStepMessage //返回 round step 数据结构
				|-- Broadcast(StateChannel, nrsMsg) //编码后发送到 statechannel
				
		broadcastNewValidBlockMessage
		broadcastHasVoteMessage
	
	* receive //receive 函数是上述操作的数据入口，也是各 peer 间通信入口，输入输出和涉及到的主要模块结构如下：
	Receive(chID byte, src p2p.Peer, msgBytes []byte)
		|
		|-- switch chID
				|
				|-- StateChannel：NewRound、NewValidBlock、Vote、23Vote
				|		|-- ApplyNewRoundStepMessage // 对 PRS 进行高度，round等参数的判断，如果是新的高度，那就更新PRS
				|		|-- ApplyNewValidBlockMessage // 把msg中包含的block信息写入PRS
				|		|-- ApplyHasVoteMessage // 检查 PRS 在指定 高度，轮次，msg类型下是否存在该 Vote 信息
				|		|-- SetPeerMaj23 //获取指定轮次和签名类型的 HeightVoteSet，检查peerID 和 BlockID，更新 blockKey中 BlockVotes
				|		|-- src.TrySend	// 根据 msg 类型，将其 vote_msg 发送给其他参与者
				|
				|-- DataChannel: Proposal
				|			|-- SetHasProposal // 将 proposal 的信息更新到 PRS 中
				|			|-- ApplyProposalPOLMessage // 将 proposalPOL 的信息更新到PRS中
				|			|-- SetHasProposalBlockPart // 将 ProposalBlock 的信息更新到 PRS 中，把 msg 写到 cS.peerMsgQueue 中
				|			
			 	|-- VoteChannel：检查 consensus state 和 peer state 的高度，更新numValidator 参数，在新的状态上写入 msg.Vote 中新增的信息，把msg 追加到cs.peerMsgQueue 中
				|
				|-- VoteSetBitsChannel //检查msg 和consensus state的高度，相同则根据 msg 中指定的 Round 和 blockID 获得 vote 内容，更新到PS（peer state）中
```

#### 5.4、和 App 的交互
##### 5.4.1 Handshaker // 和APP确认同步

	Handshake	
		|-- proxyApp.Query().InfoSync // 获得app的基本信息，比如块高
		|-- saveState(h.stateDB, h.initialState) //保存 handshaker 中的信息
		|-- ReplayBlocks //对 proxyApp 同步 state 信息

##### 5.4.2、mockProxyApp

返回ABCIResponses

	DeliverTx(tx []byte) // txCount ++
		
	EndBlock // abciResponses.EndBlock
	
	Commit // ResponseCommit{Data: mock.appHash}





uml
