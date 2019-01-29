### SDK 接口使用说明
#### 1. 创建并部署 SDK 合约
```
//创建 SDK 合约账号
cleos create account eosio dappproxy EOSxxxxx EOSxxxx
//部署 ABI
cleos set abi dappproxy sdk/firewall.abi
//部署 Code
cleos set code dappproxy sdk/firewall.wasm
//设置 eosio.code 权限
cleos set account permission dappproxy active '{
  "threshold": 1,
  "keys": [
    {
      "key": "EOSxxxxx",
      "weight": 1
    }
  ],
  "accounts": [
    {
      "permission": {
        "actor": "dappproxy",
        "permission": "eosio.code"
      },
      "weight": 1
    }
  ],
  "waits": []
}'

```

#### 2. 初始化 SDK 合約
详细参考[控制台使用教程](前端功能)

#### 3. 将 sdk/dappshield.hpp 拷贝到 DApp 项目目录下，并在项目中引用
```c++
/*
 * 必须在 `include` 之前 `define` SDK 合约账号
 * 就是 1. 中创建的帐号 (此例中为 dappproxy)
 */
#define FIREWALL_CONTRACT "dappproxy"_n
#include "dappshield.hpp"
```

#### 4. SDK 支持一键熔断，能随时关停 DApp, 开发者可依需求设置
```c++
void apply( uint64_t receiver, uint64_t code, uint64_t action ) { \
   if( !checkenable() ) { eosio_exit(0); } \
   ...
}
```

当需要确认用户帐号属性时，可调用 check 接口, 下面我们以 transfer 为例
```c++
void transfer( name from, name to, asset quantity, std::string memo )
{
    require_auth( from );
    eosio_assert( from != to, "cannot transfer to self" );

    if ( from == _self || to != _self ) {
        return;
    }

    transfer_proxy(from, to, quantity, memo);
}

void transfer_proxy( name from, name to, asset quantity, std::string memo )
{
    // 若 DApp 没有被启用，可在此返回并做相关处理
    if( !checkenable() ) {
    	print("DApp isn't enabled");
    	// you can return user's money here....
    	return;
    }

    capi_checksum256 txhash = get_txhash(); // 取得 txhash

    check(_self, from, txhash); // 调用 check 接口

    // check 会将检查结果在下一个 tx 返回，可以用延迟交易来获取
    transaction tx;
    tx.delay_sec = 0;
    // txhash, tapos_block_num 是用来识别我们在哪个交易中做了检查，需要带入到延迟交易中供结果搜寻
    tx.actions.emplace_back( permission_level{ _self, "active"_n }, _self, "reveal"_n, std::make_tuple(from, to, quantity, memo, txhash, tapos_block_num()) );
    tx.send(now(), _self);

    return;
}

[[eosio::action]]
void reveal( name from, name to, asset quantity, std::string memo, capi_checksum256 txhash, int tbn )
{
    // 取得检查结果
    uint32_t result = getresult(from, txhash, tbn);
    print_f("result is %\n", result);

    // 依检查结果做出处理 (可参照 dappshield.hpp)
    if(result) {
        // you should check the result code, maybe the SDK isn't registered or it's expired,
        // perhaps your DApp isn't configured, or, you know ;)
        print_f("sorry, % is not welcome\n", from);
        // you can return user's money here....
    }
}
```
完整代码可参考 sdk/example.cpp
