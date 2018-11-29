# EOS智能合约安全编码规范——快速参考指南

随着EOS上各种DApp的蓬勃发展，吸引越来越多的用户和开发者加入其中，这篇文档旨在为EOS智能合约的开发人员提供一些基本安全理念和可快速参考的安全检查项。
同时希望能够获得社区对该文档提出建议及合并请求(Pull Request)，以完善文档，更好地促进EOS的生态发展。

## 目录

* [安全理念](#安全理念)
* [安全编码规范检查项](#安全编码规范检查项)
   * [禁止获取用户敏感权限](#禁止获取用户敏感权限)
      * [检查示例](#检查示例)
      * [异常实例](#异常实例)
   * [数值范围检查](#数值范围检查)
      * [检查示例](#检查示例)
      * [攻击实例](#攻击实例)
   * [输入验证](#输入验证)
      * [检查示例](#检查示例)
      * [攻击实例](#攻击实例)
   * [权限校验](#权限校验)
      * [检查示例](#检查示例)
      * [攻击实例](#攻击实例)
   * [调用来源校验](#调用来源校验)
      * [检查示例](#检查示例)
      * [攻击实例](#攻击实例)
   * [假转账通知](#假转账通知)
      * [检查示例](#检查示例)
      * [攻击实例](#攻击实例)
   * [避免交易回滚](#避免交易回滚)
      * [检查示例](#检查示例)
      * [攻击实例](#攻击实例)
   * [随机数问题](#随机数问题)
      * [检查示例](#检查示例)
      * [攻击实例](#攻击实例)

## 安全理念
虽然EOS目前仍处在快速的开发与版本迭代中，但已经吸引了大量的开发者为其提供各式各样的DApp，也受到了黑客们的关注。近来安全事件频发，在关注最新的漏洞修复与安全事件原因的同时，需要具备一些基本的安全理念，以应对不断发展变化的安全威胁。

理解EOS链上特性
具有异常检测与处理逻辑
完整的测试发布流程
基本的应急响应能力

## 安全编码规范检查项

### 禁止获取用户敏感权限
禁止获取用户的敏感权限，避免对用户资产的直接转移
#### 检查示例
检查用户帐号在使用DApp后，是否被增加了智能合约帐号的敏感权限，如eosio.code权限
#### 异常实例
[【捭阖命物eos】狼人游戏中需要的code权限真的安全吗，以及被埋在资金盘里的我正在思考什么](https://bihu.com/article/992656)

### 数值范围检查
在数据操作前后，检查数值范围是否符合预期
#### 检查示例
```
// eosio.token合约里transfer中部分检查代码
eosio_assert( quantity.amount > 0, "must transfer positive quantity" );
```
#### 攻击实例
[【EOS Fomo3D你千万别玩】狼人杀遭到溢出攻击, 已经凉凉](https://bihu.com/article/995093)

### 输入验证
对从外部获取的参数数据，在参数数量和内容上进行验证
#### 检查示例
```
```
#### 攻击实例
暂无

### 权限校验
对外部可调用函数进行必要的权限检查
#### 检查示例
```
// 如合约内部函数，可限制只有合约自己能够调用
require_auth( _self );
```
#### 攻击实例
暂无

### 调用来源校验
在apply中处理合约调用的分发时，应检查调用来源是否正确，如检查EOS及各种代币的发行合约
#### 检查示例
```
// 如限定onerror仅来自eosio
if (code == N(eosio) && action == N(onerror)) {}

// 如限定EOS仅来自eosio.token
if(code == N(eosio.token) && action == N(transfer)) { }
```
#### 攻击实例
[EOSBet Transfer Hack Statement](https://www.reddit.com/r/eos/comments/9fxyd4/eosbet_transfer_hack_statement/)
问题代码片段
```
// extend from EOSIO_ABI, because we need to listen to incoming eosio.token transfers
#define EOSIO_ABI_EX( TYPE, MEMBERS ) \
extern "C" { \
  void apply( uint64_t receiver, uint64_t code, uint64_t action ) { \
    auto self = receiver; \
    if( action == N(onerror)) { \
      /* onerror is only valid if it is for the "eosio" code account and authorized by "eosio"'s "active permission */ \
     eosio_assert(code == N(eosio), "onerror action's are only valid from the \"eosio\" system account"); \
    } \
    /* 问题点，直接调用合约的transfer函数，并传递相关参数即可 */ \
    if( code == self || code == N(eosio.token) || action == N(onerror) ) { \
      TYPE thiscontract( self ); \
      switch( action ) { \
        EOSIO_API( TYPE, MEMBERS ) \
      } \
    /* does not allow destructor of thiscontract to run: eosio_exit(0); */ \
  } \
} \

```

### 假转账通知
在transfer中对接收到的转账通知，应检查接收方和发送方是否正确

#### 检查示例
```
if (from == _self || to != _self){
  return;
}
```

#### 攻击实例
[“伪造转账通知”漏洞细节披露：EOSBet遭黑客攻击损失近14万EOS始末](https://mp.weixin.qq.com/s/Yo-tHB_2GHdnSt5pyPnnpA)
问题代码片段
```
void transfer(uint64_t sender, uint64_t receiver) {

    auto transfer_data = unpack_action_data<st_transfer>();
    // 问题点，没有判断 to 是否为合约自己
    if (transfer_data.from == _self || transfer_data.from == N(eosbetcasino)){
        return;
    }
    ...
}

```

### 避免交易回滚
transaction中一个action的失败，导致这个transaction中的所有action的执行状态回滚到执行之前

#### 检查示例
```
//在收到转账后发起 deferred transaction
eosio::transaction tx;
auto tx_data = make_tuple(from, quantity, game_id);
tx.actions.emplace_back(eosio::permission_level{_self, N(active)}, _self, N(nextaction), tx_data);
tx.delay_sec = 1;
tx.send(sender_id, _self);
```
#### 攻击实例
[PeckShield 安全播报: EOS竞猜类游戏nutsgambling遭黑客交易回滚攻击](https://www.bianews.com/news/flash?id=25688)

### 随机数问题
合约中的随机数需要借助合约外部提供的不可控因素来参与生成
#### 检查示例
```
//如两次延迟后的 transaction id 信息
//如两次延迟后的 block prefix 信息
tapos_block_prefix()
```
#### 攻击实例
[探究随机数漏洞背后的技术原理：EOS.WIN竞猜游戏是如何被攻破的？]()