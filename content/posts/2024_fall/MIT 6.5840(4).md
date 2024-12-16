---
title: "MIT 6.5840(4) - Lab2 & Lab4"
date: 2024-10-30 
tags : [
    "Coding",
]
type: "post"
showTableOfContents: true
---

# Lab2 Key/Value Server
lab2要求实现一个线性一致的KV数据库，可能会存在不稳定的网络（比如高延迟、丢包等情况），但是server不会crash。

难点在于client由于超时重传（类似于TCP），可能会发送一个请求多次，server接收到的这些重复请求也不一定是连续的。此时就需要server维护一个hashMap用于记录请求ID以及对应的第一次成功的返回结果。
hashSet显然需要及时清理。我采用的方案是client在确认发包成功后，继续发送携带请求ID的ResolveRPC直到成功，server通过ResovleRPC把hashMap中的请求ID删除。删除操作是幂等的，不需要保证线性一致。
例如，一个完整的PUT请求如下：

server维护done = make(map[int64]string)

● client发送PUT(x,1)，ID=1

● server判断ID是否存在于done。如果存在，返回里面的值；否则处理并把结果存在done里

● client收到之后发送Resolve(1)

● server执行delete(done, 1)

一次上述的操作是原子性的。

# Lab4 Fault-tolerant KVServer
基于RAFT协议实现一个Fault-tolerant KVServer

## Client实现
所有的读写请求都必须直接从client发送给leader server。有两种情况client不知道leader是谁：

+ 第一次发送请求
+ server易主/crash，返回报错

此时client就需要不断重新尝试另一个server，直到请求被成功接收。注意到不同的命令有相同的发送方式，可以抽象出一层来专门用于发送RPC，我这里使用了策略模式。

## Server实现
### 如何返回RPC结果
对于RPC接受的一个Op，leader需要将它通过Raft的Start接口达成共识。对于写请求（GET），client只需要知道它成功就行；对于读请求（PUT/APPEND），client还需要知道结果。

我们用一个专门的协程applier循环读取Raft commit上来的Op。如果是读请求，不做处理；如果是写请求，则更新KVmap。这样每一个Op都对应着一个独立的commandIndex，定义一个这样的map：

```go
resultChMap map[int]chan opResult
```

RPC协程与applier协程通过这个map里的管道通信，以返回RPC结果。

要注意RPC在返回结果后需要销毁对应的通道。

### 幂等性设计
#### raft层commitIndex幂等
raft有可能提交log过时的commitIndex，因此要注意在真正apply的时候添加如下判断

```go
 msg.CommandIndex > kv.lastApplied
```

但这样可能会造成过时请求得不到channel的信息而发生阻塞，所以还需要在RPC处添加timeout。

```go
ticker := time.NewTicker(time.Duration(CHANNEL_TIMEOUT) * time.Millisecond)
defer ticker.Stop()

select {
case opResult := <-ch:
    if opResult.err != "" {
        reply.Err = Err(opResult.err)
    }
case <-ticker.C:
    reply.Err = NOTALEADER
}
```

#### client请求幂等
由于网络故障等原因，server接收到的命令可能是重复的，并且client只认最新结果（这和TCP超时重传有所差异）。对于读请求，并不会使得KVmap状态发生改变；但是对于写请求，就必须保证它的幂等性。

考虑维护这样一个map，key是clientID，value是序列号（从0开始增长）

```go
seqMap      map[int64]int
```

applier每次读取到一个Op就判断一下序列号，如果Op序列号是旧序列号+1，则可以正常处理写请求（或者no-op的读请求）。

```go
func (kv *KVServer) processSeq(op Op) bool {
	seq0, ok := kv.seqMap[op.ClientID]
	if !ok {
		kv.seqMap[op.ClientID] = op.Seq
		return true
	}
	if !(seq0+1 == op.Seq) {
		return false
	}
	kv.seqMap[op.ClientID]++
	return true
}
// applier处理RAFT层传上来的Op逻辑：
if kv.processSeq(op) {
    k := op.OpKey
    switch op.OpType {
    case PUTOP:
        {
            kv.mp[k] = op.OpValue
        }
    case APPENDOP:
        {
            kv.mp[k] = kv.mp[k] + op.OpValue
        }
    }
}
```

### Snapshot
理解snapshot的数据流传递至关重要：

+ server发现Raft State太大，leader对其raft层发送snapshot指令，传递一些关于server状态的信息
+ raft层向persister保存snapshot和raft状态
+ raft层向follower发送InstallSnapshotRPC
+ follower向server层传递snapshot，server层将snapshot里的server状态应用于自身。

对于server检查raft state是否达到snapshot阈值的时机，在接收到applyMsg并更新了状态后是比较合适的。不应该在start后立即检查，因为此时还没有立刻达成共识。

server需要持久化的状态如下：

+ lastApplied
+ mp(即KVMap)
+ seqMap

## 拓展
一个上网冲浪搜集到的拓展，目前我还没实现

**Read-only Operations Optimize**

论文中第8节讲述其实现在处理读请求的时候，可以不用往log里加东西。这需要实现两点：

+ leader需要确保拥有最新的commit log。这可以在每一个leader选举成功后commit一个no-op实现。
+ leader在处理读请求的时候需要确保自身还是leader，这就需要leader在处理前与大多数节点确认自己仍是leader。

