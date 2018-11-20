# EOS智能合约间通讯

## 一、EOS通知概述

我们首先看一看eosio.token合约中issue的通知。跳过基本的合约和账户配置，我们直接进入eosio.token合约，首先创建一个token：

```bash
[kingnet@pdev1 nodeos1]$ cleos --wallet-url http://localhost:9800 --url http://localhost:9800 push action eosio.token create '{"issuer":"eosio","maximum_supply":"100000000000.0000 EOS","can_freeze":0,"can_recall":0,"can_whitelist":0}' -p eosio.token
executed transaction: 756ecf4050184dd06c7f27d77cca3864e41fa3c319b23c326180d7cc386b62d9  120 bytes  6597 us
#   eosio.token <= eosio.token::create          {"issuer":"eosio","maximum_supply":"100000000000.0000 EOS","can_freeze":0,"can_recall":0,"can_whitel...
warning: transaction executed locally, but may not be confirmed by the network yet
```

然后eosio向配置好的账户helloworld发行token：

```bash
[kingnet@pdev1 dice]$ cleos --wallet-url http://localhost:9800 --url http://localhost:9800 push action eosio.token issue '{"to":"helloworld", "quantity":"100.0000 EOS", "memo":"m"}' -p eosio
executed transaction: 732e883ae259644de6c4442a0b4007fdbc0d1b95157c5b000a3f7c9cbb1eecf7  120 bytes  1871 us
#   eosio.token <= eosio.token::issue           {"to":"helloworld","quantity":"100.0000 EOS","memo":"m"}
>> issueeosio balance: 100.0000 EOS
#   eosio.token <= eosio.token::transfer        {"from":"eosio","to":"helloworld","quantity":"100.0000 EOS","memo":"m"}
>> transfer from eosio to helloworld 100.0000 EOS
#         eosio <= eosio.token::transfer        {"from":"eosio","to":"helloworld","quantity":"100.0000 EOS","memo":"m"}
#    helloworld <= eosio.token::transfer        {"from":"eosio","to":"helloworld","quantity":"100.0000 EOS","memo":"m"}
warning: transaction executed locally, but may not be confirmed by the network yet
```

从以上issue合约执行的日志中可以看出账户之间存在多次的通知，可以用以下图来描述整个消息流：

![img](eos_token_issue.png)

eosio.token合约首先收到issue action的消息，在eosio.token合约执行时发起了一个新的transfer action消息给eosio.token合约，即发给自己，如下是issue action代码片段：

```c++
void token::issue( account_name to, asset quantity, string memo )
{
    print( "issue" );
    auto sym = quantity.symbol;
    eosio_assert( sym.is_valid(), "invalid symbol name" );
 
    auto sym_name = sym.name();
    stats statstable( _self, sym_name );
    auto existing = statstable.find( sym_name );
    eosio_assert( existing != statstable.end(), "token with symbol does not exist, create token before issue" );
    const auto& st = *existing;
 
    require_auth( st.issuer );
    eosio_assert( quantity.is_valid(), "invalid quantity" );
    eosio_assert( quantity.amount > 0, "must issue positive quantity" );
 
    eosio_assert( quantity.symbol == st.supply.symbol, "symbol precision mismatch" );
    eosio_assert( quantity.amount <= st.max_supply.amount - st.supply.amount, "quantity exceeds available supply");
 
    statstable.modify( st, 0, [&]( auto& s ) {
       s.supply += quantity;
    });
 
    add_balance( st.issuer, quantity, st, st.issuer );
 
    if( to != st.issuer ) {
       SEND_INLINE_ACTION( *this, transfer, {st.issuer,N(active)}, {st.issuer, to, quantity, memo} );
    }
}
```

我可以看到在第27行，发送了一个action，采用的是inline方式（这是eos两种通信方式的一种，还有一种是Deferred ），发送到自己即eosio.token的transfer action上。

以下是eosio.token的transfer action代码片段：

```c++
void token::transfer( account_name from,
                      account_name to,
                      asset        quantity,
                      string       /*memo*/ )
{
    print( "transfer from ", eosio::name{from}, " to ", eosio::name{to}, " ", quantity, "\n" );
    eosio_assert( from != to, "cannot transfer to self" );
    require_auth( from );
    eosio_assert( is_account( to ), "to account does not exist");
    auto sym = quantity.symbol.name();
    stats statstable( _self, sym );
    const auto& st = statstable.get( sym );
 
    require_recipient( from );
    require_recipient( to );
 
    eosio_assert( quantity.is_valid(), "invalid quantity" );
    eosio_assert( quantity.amount > 0, "must transfer positive quantity" );
    eosio_assert( quantity.symbol == st.supply.symbol, "symbol precision mismatch" );
    
    sub_balance( from, quantity, st );
    add_balance( to, quantity, st, from );
}
```

从第14、15行可以看到其给from和to（即eosio和helloworld）各发送一个通知消息。

那么该通知信息会被eosio和helloworld两个账户收到吗？如果账户能收到通知消息，通知消息发送给账户时的参数又是什么呢？那么账户怎么处理通知消息呢？通过以下编程来深入了解EOS的通知消息。

## 二、EOS通知编程

游戏平台合约：

```c++
/**
 * *  The apply() methods must have C calling convention so that the blockchain can lookup and
 * *  call these methods.
 * */
#include <eosiolib/eosio.hpp>
 
extern "C" {
 
	struct gameinfo {
	   uint64_t id;
	   account_name player1;
	   uint64_t x1;
	   uint64_t y1;
	   account_name player2;
	   uint64_t x2;
	   uint64_t y2;
	   uint64_t info;
 
	   uint64_t primary_key()const { return id; }
 
	   //EOSLIB_SERIALIZE( gameinfo, (id)(player)(x)(y) )
	};
 
	typedef eosio::multi_index< N(gameinfo), gameinfo> gameinfo_index;
 
	struct notifyinfo {
	   uint64_t id;
	   account_name receiver;
	   account_name code;
	   account_name action;
 
	   uint64_t primary_key()const { return id; }
 
	   //EOSLIB_SERIALIZE( notifyinfo, (id)(receiver)(code)(action) )
	};
 
	typedef eosio::multi_index< N(notifyinfo), notifyinfo> notifyinfo_index;
 
	struct dummy_action {
		account_name player1;
		account_name player2;
		uint64_t info;
	};
 
   /// The apply method implements the dispatch of events to this contract
   void apply( uint64_t receiver, uint64_t code, uint64_t action ) {
	 //
	 auto _self = receiver;
	 {
		 notifyinfo_index notifyinfos(_self, _self);
		 auto new_notifyinfo_itr = notifyinfos.emplace(_self, [&](auto& info){
			info.id           = notifyinfos.available_primary_key();
			info.receiver     = receiver;
			info.code         = code;
			info.action       = action;
		 });
	 }
	 //
	 if(receiver == code && action == N(creategame)) {
		 dummy_action dummy = eosio::unpack_action_data<dummy_action>();
		 gameinfo_index gameinfos(_self, _self);
		 auto new_gameinfos_itr = gameinfos.emplace(_self, [&](auto& info){
			info.id         = gameinfos.available_primary_key();
			info.player1    = dummy.player1;
			info.x1         = 0;
			info.y1         = 0;
			info.player2    = dummy.player2;
			info.x2         = 0;
			info.y2         = 0;
			info.info       = dummy.info;
		 });
 
		 //
		 require_recipient(dummy.player1);
		 require_recipient(dummy.player2);
	 } else if(action == N(creategame)) {
		 // do something....
		 //
	 } else if(receiver == code && action == N(reset)) {
		 gameinfo_index gameinfos(_self, _self);
		 auto cur_gameinfos_itr = gameinfos.begin();
		 while(cur_gameinfos_itr != gameinfos.end()) {
			 cur_gameinfos_itr = gameinfos.erase(cur_gameinfos_itr);
		 }
 
		 notifyinfo_index notifyinfos(_self, _self);
		 auto cur_notifyinfos_itr = notifyinfos.begin();
		 while(cur_notifyinfos_itr != notifyinfos.end()) {
			 cur_notifyinfos_itr = notifyinfos.erase(cur_notifyinfos_itr);
		 }
	 }
   }
} // extern "C"
```

用户合约代码：

```c++
/**
 * *  The apply() methods must have C calling convention so that the blockchain can lookup and
 * *  call these methods.
 * */
#include <eosiolib/eosio.hpp>
 
extern "C" {
 
	struct gameinfo {
	   uint64_t id;
	   account_name player1;
	   uint64_t x1;
	   uint64_t y1;
	   account_name player2;
	   uint64_t x2;
	   uint64_t y2;
	   uint64_t info;
 
	   uint64_t primary_key()const { return id; }
 
	   //EOSLIB_SERIALIZE( gameinfo, (id)(player)(x)(y) )
	};
 
	typedef eosio::multi_index< N(gameinfo), gameinfo> gameinfo_index;
 
	struct notifyinfo {
	   uint64_t id;
	   account_name receiver;
	   account_name code;
	   account_name action;
 
	   uint64_t primary_key()const { return id; }
 
	   //EOSLIB_SERIALIZE( notifyinfo, (id)(receiver)(code)(action) )
	};
 
	typedef eosio::multi_index< N(notifyinfo), notifyinfo> notifyinfo_index;
 
	struct dummy_action {
		account_name player1;
		account_name player2;
		uint64_t info;
	};
 
   /// The apply method implements the dispatch of events to this contract
   void apply( uint64_t receiver, uint64_t code, uint64_t action ) {
	 auto _self = receiver;
	 {
		 notifyinfo_index notifyinfos(_self, _self);
		 auto new_notifyinfo_itr = notifyinfos.emplace(_self, [&](auto& info){
			info.id           = notifyinfos.available_primary_key();
			info.receiver     = receiver;
			info.code         = code;
			info.action       = action;
		 });
	 }
	 //
	 if(action == N(creategame)) {
		 dummy_action dummy = eosio::unpack_action_data<dummy_action>();
		 gameinfo_index gameinfos(_self, _self);
		 auto new_gameinfos_itr = gameinfos.emplace(_self, [&](auto& info){
			info.id         = gameinfos.available_primary_key();
			info.player1    = dummy.player1;
			info.x1         = 0;
			info.y1         = 0;
			info.player2    = dummy.player2;
			info.x2         = 0;
			info.y2         = 0;
			info.info       = dummy.info;
		 });
 
		 //
		 require_recipient(code);
	 }
   }
} // extern "C"
```

我们将创建三个账户，分别运行游戏平台合约和两个游戏玩家合约：

```bash
[kingnet@pdev1 test_notify_s]$ cleos --wallet-url http://localhost:9800 --url http://localhost:9800 create account eosio game.s EOS6MRyAjQq8ud7hVNYcfnVPJqcVpscN5So8BhtHuGYqET5GDW5CV EOS6MRyAjQq8ud7hVNYcfnVPJqcVpscN5So8BhtHuGYqET5GDW5CV
executed transaction: c7dc538ecf56ba612eae50eb266848e1ae6a9907cbbab62ae8ade48018c38644  200 bytes  451 us
#         eosio <= eosio::newaccount            {"creator":"eosio","name":"game.s","owner":{"threshold":1,"keys":[{"key":"EOS6MRyAjQq8ud7hVNYcfnVPJq...
warning: transaction executed locally, but may not be confirmed by the network yet
[kingnet@pdev1 test_notify_s]$ cleos --wallet-url http://localhost:9800 --url http://localhost:9800 create account eosio game.c.1 EOS6MRyAjQq8ud7hVNYcfnVPJqcVpscN5So8BhtHuGYqET5GDW5CV EOS6MRyAjQq8ud7hVNYcfnVPJqcVpscN5So8BhtHuGYqET5GDW5CV
executed transaction: f44bdfa622c06fb170e5b564bca0fd52f69d4cfc068bad9546a820ad822d6ece  200 bytes  403 us
#         eosio <= eosio::newaccount            {"creator":"eosio","name":"game.c.1","owner":{"threshold":1,"keys":[{"key":"EOS6MRyAjQq8ud7hVNYcfnVP...
warning: transaction executed locally, but may not be confirmed by the network yet
[kingnet@pdev1 test_notify_s]$ cleos --wallet-url http://localhost:9800 --url http://localhost:9800 create account eosio game.c.2 EOS6MRyAjQq8ud7hVNYcfnVPJqcVpscN5So8BhtHuGYqET5GDW5CV EOS6MRyAjQq8ud7hVNYcfnVPJqcVpscN5So8BhtHuGYqET5GDW5CV
executed transaction: c52b61f22f6df8b46ac7a8623e3114404cfc2dceec81363abfee094f7e450c28  200 bytes  476 us
#         eosio <= eosio::newaccount            {"creator":"eosio","name":"game.c.2","owner":{"threshold":1,"keys":[{"key":"EOS6MRyAjQq8ud7hVNYcfnVP...
warning: transaction executed locally, but may not be confirmed by the network yet
```

在游戏平台账户上部署游戏平台合约：

```bash
[kingnet@pdev1 test_notify_s]$ cleos --wallet-url http://localhost:9800 --url http://localhost:9800 set contract game.s /home/kingnet/tangy/eos/mycontracts/test_notify_s/
Reading WAST/WASM from /home/kingnet/tangy/eos/mycontracts/test_notify_s/test_notify_s.wasm...
Using already assembled WASM...
Publishing contract...
executed transaction: 240396da31c7d4d193befc07bfe4db4956a250e0760a0156adeae8d3979a7017  3848 bytes  1273 us
#         eosio <= eosio::setcode               {"account":"game.s","vmtype":0,"vmversion":0,"code":"0061736d0100000001560f6000006000017e60027e7e006...
#         eosio <= eosio::setabi                {"account":"game.s","abi":{"types":[],"structs":[{"name":"creategame","base":"","fields":[{"name":"p...
warning: transaction executed locally, but may not be confirmed by the network yet
```

分别在两个游戏玩家账户张部署游戏客户端合约：

```bash
[kingnet@pdev1 test_notify_g1]$ cleos --wallet-url http://localhost:9800 --url http://localhost:9800 set contract game.c.1 /home/kingnet/tangy/eos/mycontracts/test_notify_g1
Reading WAST/WASM from /home/kingnet/tangy/eos/mycontracts/test_notify_g1/test_notify_g1.wasm...
Using already assembled WASM...
Publishing contract...
executed transaction: 1c1b448f52cd4bca5c251cacf56c93ffaa49abfb2a03d7e065b18068f522cb9b  3368 bytes  1201 us
#         eosio <= eosio::setcode               {"account":"game.c.1","vmtype":0,"vmversion":0,"code":"0061736d0100000001560f6000006000017e60027e7e0...
#         eosio <= eosio::setabi                {"account":"game.c.1","abi":{"types":[],"structs":[{"name":"gameinfo","base":"","fields":[{"name":"i...
warning: transaction executed locally, but may not be confirmed by the network yet
[kingnet@pdev1 test_notify_g1]$ cleos --wallet-url http://localhost:9800 --url http://localhost:9800 set contract game.c.2 /home/kingnet/tangy/eos/mycontracts/test_notify_g1
Reading WAST/WASM from /home/kingnet/tangy/eos/mycontracts/test_notify_g1/test_notify_g1.wasm...
Using already assembled WASM...
Publishing contract...
executed transaction: d3456d22a8ea450edbf1ee014833dbd00aecc6fd22883fbe500460681bae653b  3368 bytes  1000 us
#         eosio <= eosio::setcode               {"account":"game.c.2","vmtype":0,"vmversion":0,"code":"0061736d0100000001560f6000006000017e60027e7e0...
#         eosio <= eosio::setabi                {"account":"game.c.2","abi":{"types":[],"structs":[{"name":"gameinfo","base":"","fields":[{"name":"i...
warning: transaction executed locally, but may not be confirmed by the network yet
```

我们接下来执行一次游戏创建操作：

```bash
[kingnet@pdev1 test_notify_g1]$ cleos --wallet-url http://localhost:9800 --url http://localhost:9800 push action game.s creategame '{"player1":"game.c.1","player2":"game.c.2","info":1}' -p game.s
executed transaction: 5048ffd4082325ca9f72ad9ac2c6fd7f3a6befbf87766343b064b57b6fac2fd5  120 bytes  4871 us
#        game.s <= game.s::creategame           {"player1":"game.c.1","player2":"game.c.2","info":1}
#      game.c.1 <= game.s::creategame           {"player1":"game.c.1","player2":"game.c.2","info":1}
#      game.c.2 <= game.s::creategame           {"player1":"game.c.1","player2":"game.c.2","info":1}
warning: transaction executed locally, but may not be confirmed by the network yet
```

从以上执行的结果可以看到，creategame这个action首先发给了game.s，这是我们发起的，这个action消息又转发给了game.c.1和game.c.2，这是我们在creategame这个action中发送的通知消息。

我们可以继续看看每个合约账户上收到的action通知消息：

```bash
[kingnet@pdev1 test_notify_g1]$ cleos --wallet-url http://localhost:9800 --url http://localhost:9800 get actions game.s
#  seq  when                              contract::action => receiver      trx id...   args
================================================================================================================
#    0   2018-05-17T09:56:55.500            eosio::setcode => eosio         240396da... {"account":"game.s","vmtype":0,"vmversion":0,"code":"0061736...
#    1   2018-05-17T09:56:55.500             eosio::setabi => eosio         240396da... {"account":"game.s","abi":{"types":[],"structs":[{"name":"cr...
#    2   2018-05-17T10:03:41.000        game.s::creategame => game.s        5048ffd4... {"player1":"game.c.1","player2":"game.c.2","info":1}...
#    3   2018-05-17T10:03:41.000        game.s::creategame => game.c.1      5048ffd4... {"player1":"game.c.1","player2":"game.c.2","info":1}...
#    4   2018-05-17T10:03:41.000        game.s::creategame => game.c.2      5048ffd4... {"player1":"game.c.1","player2":"game.c.2","info":1}...
[kingnet@pdev1 test_notify_g1]$ cleos --wallet-url http://localhost:9800 --url http://localhost:9800 get actions game.c.1
#  seq  when                              contract::action => receiver      trx id...   args
================================================================================================================
#    0   2018-05-17T09:59:50.500            eosio::setcode => eosio         1c1b448f... {"account":"game.c.1","vmtype":0,"vmversion":0,"code":"00617...
#    1   2018-05-17T09:59:50.500             eosio::setabi => eosio         1c1b448f... {"account":"game.c.1","abi":{"types":[],"structs":[{"name":"...
#    2   2018-05-17T10:03:41.000        game.s::creategame => game.c.1      5048ffd4... {"player1":"game.c.1","player2":"game.c.2","info":1}...
[kingnet@pdev1 test_notify_g1]$ cleos --wallet-url http://localhost:9800 --url http://localhost:9800 get actions game.c.2
#  seq  when                              contract::action => receiver      trx id...   args
================================================================================================================
#    0   2018-05-17T10:00:05.000            eosio::setcode => eosio         d3456d22... {"account":"game.c.2","vmtype":0,"vmversion":0,"code":"00617...
#    1   2018-05-17T10:00:05.000             eosio::setabi => eosio         d3456d22... {"account":"game.c.2","abi":{"types":[],"structs":[{"name":"...
#    2   2018-05-17T10:03:41.000        game.s::creategame => game.c.2      5048ffd4... {"player1":"game.c.1","player2":"game.c.2","info":1}...
```

我们在入口跟踪了通知消息并写入了数据库，如果查看数据库可以得出一致的消息，同时我们在收到通知后也正确解析了请求的参数，查看数据库可以得到数据库中三个表的内容完全一致。

我们现在可以回答上面关于eos通知消息的问题了：

1. 向一个合约发送的通知会被这个合约接收并在应用层处理吗？

一个合约向另一个合约发出的通知消息会被另一个合约收到，不管有没有ABI定义接口，都会在应用层收到。

2. 参数是什么呢？

一个合约向另一个合约发送的通知消息就是本合约收到的消息，包括code、action和参数

3. 账户如何处理该通知消息呢？

和处理正常的消息一样，根据code、action和参数进行逻辑处理

## 三、拒绝接受转账交易及DoS

我们目前已经清楚了eos合约通知机制，同时也知道了给一个账户转账交易会给该账户消息通知，其实我们还有一个疑问，如果账户合约处理通知消息失败了，那么这个交易状态会回滚么？

我们现在开发一个合约，合约处理交易的transfer事件通知，在通知处理程序中，我们抛出一个异常，使得该通知处理失败。同时我们照样在入口记录所有收到的action。

我们会将该合约部署在账户test.notify上。我们转账给test.notify，那么eosio.token合约的transfer action会被执行，在eosio.token合约执行转账时，给向目标账户test.notify发送事件通知，但事件通知处理失败了，到底这个转账交易会回滚么？

拒绝接收转账合约：

```c++
/**
 * *  The apply() methods must have C calling convention so that the blockchain can lookup and
 * *  call these methods.
 * */
#include <eosiolib/eosio.hpp>
 
extern "C" {
 
	struct notifyinfo {
	   uint64_t id;
	   account_name receiver;
	   account_name code;
	   account_name action;
 
	   uint64_t primary_key()const { return id; }
 
	   //EOSLIB_SERIALIZE( notifyinfo, (id)(receiver)(code)(action) )
	};
 
	typedef eosio::multi_index< N(notifyinfo), notifyinfo> notifyinfo_index;
 
   /// The apply method implements the dispatch of events to this contract
   void apply( uint64_t receiver, uint64_t code, uint64_t action ) {
	 auto _self = receiver;
	 notifyinfo_index notifyinfos(_self, _self);
	 auto new_notifyinfo_itr = notifyinfos.emplace(_self, [&](auto& info){
		info.id           = notifyinfos.available_primary_key();
		info.receiver     = receiver;
		info.code         = code;
		info.action       = action;
	 });
 
	 if(action == N(transfer)) {
		 eosio_assert("reject transfer!");
	 }
 
	 if(receiver == code && action == N(reset)) {
		 auto cur_notify_itr = notifyinfos.begin();
		 while(cur_notify_itr != notifyinfos.end()) {
			 cur_notify_itr = notifyinfos.erase(cur_notify_itr);
		 }
	 }
   }
} // extern "C"
```

我们执行转账操作并检查结果：

```bash
[kingnet@pdev1 test_notify]$ cleos --wallet-url http://localhost:9800 --url http://localhost:9800 create account eosio test.notify EOS6MRyAjQq8ud7hVNYcfnVPJqcVpscN5So8BhtHuGYqET5GDW5CV EOS6MRyAjQq8ud7hVNYcfnVPJqcVpscN5So8BhtHuGYqET5GDW5CV
executed transaction: 4576dec103dab2abfbfff775f8208c8fff46996cc9d3717d576574544f381d72  200 bytes  476 us
#         eosio <= eosio::newaccount            {"creator":"eosio","name":"test.notify","owner":{"threshold":1,"keys":[{"key":"EOS6MRyAjQq8ud7hVNYcf...
warning: transaction executed locally, but may not be confirmed by the network yet
[kingnet@pdev1 test_notify]$ cleos --wallet-url http://localhost:9800 --url http://localhost:9800 set contract test.notify /home/kingnet/tangy/eos/mycontracts/test_notify
Reading WAST/WASM from /home/kingnet/tangy/eos/mycontracts/test_notify/test_notify.wasm...
Using already assembled WASM...
Publishing contract...
executed transaction: 7bd410a214ed2700eea182bebcdf94304ee49911e8f0c53dcfa58068150067d4  5048 bytes  1200 us
#         eosio <= eosio::setcode               {"account":"test.notify","vmtype":0,"vmversion":0,"code":"0061736d0100000001520e6000006000017e60027e...
#         eosio <= eosio::setabi                {"account":"test.notify","abi":{"types":[],"structs":[{"name":"hi","base":"","fields":[{"name":"user...
warning: transaction executed locally, but may not be confirmed by the network yet
[kingnet@pdev1 test_notify]$ cleos --wallet-url http://localhost:9800 --url http://localhost:9800 push action eosio.token issue '{"to":"test.notify","quantity":"1001.0000 EOS", "memo":"m"}' -p eosio
executed transaction: 995b01cd4206ee19894400f09d2d0bbbba1b5d9e74010438447387e4be1f226e  120 bytes  10667 us
#   eosio.token <= eosio.token::issue           {"to":"test.notify","quantity":"1001.0000 EOS","memo":"m"}
>> issueeosio balance: 1001.0000 EOS
#   eosio.token <= eosio.token::transfer        {"from":"eosio","to":"test.notify","quantity":"1001.0000 EOS","memo":"m"}
>> transfer from eosio to test.notify 1.0000 EOS
#         eosio <= eosio.token::transfer        {"from":"eosio","to":"test.notify","quantity":"1001.0000 EOS","memo":"m"}
#   test.notify <= eosio.token::transfer        {"from":"eosio","to":"test.notify","quantity":"1001.0000 EOS","memo":"m"}
warning: transaction executed locally, but may not be confirmed by the network yet
[kingnet@pdev1 test_notify]$ cleos --wallet-url http://localhost:9800 --url http://localhost:9800 get currency balance eosio.token test.notify
1001.0000 EOS
[kingnet@pdev1 test_notify]$ cleos --wallet-url http://localhost:9800 --url http://localhost:9800 push action eosio.token transfer '{"from":"alice","to":"test.notify","quantity":"1.0000 EOS","memo":"m"}' -p alice
1566956ms thread-0   main.cpp:2316                 main                 ] Failed with error: Assert Exception (10)
condition: assertion failed: reject transfer!
[kingnet@pdev1 test_notify]$ cleos --wallet-url http://localhost:9800 --url http://localhost:9800 get currency balance eosio.token test.notify
1001.0000 EOS
```

我们可以看到转账失败了，这个失败是由转账的接收方处理转账通知事件时给出错误应答引起的，这样就可以让一个账户拒绝接收转账。

这个例子演示了时事件通知处理失败会导致整个交易回滚，如果一个transaction中包含了许多的action，但只要某一个账户处理action的通知事件失败将引起整个交易回滚，这样某一个账户可以进行DoS（拒绝服务攻击）。

下一章介绍EOS合约中通信编程。

## 合约通信编程

### 一、通信模型和执行流程

EOSIO智能合约可以相互通信，例如让另一个合约执行某些与当前action相关的操作，或触发当前action范围之外的未来交易。

EOSIO支持Inline和Deferred两种基本通信模式。Inline通信可以理解为在当前action中执行操作，可视为同步通信，Deferred通信可以理解为触发未来事务操作，可视为异步通信。

 

1. Inline通信

Inline通信的形式是请求作为调用操作的一部分而执行。Inline通信使用原始交易相同的scope和权限作为执行上下文，并保证与当前action一起执行。可以被认为是transaction中的嵌套transaction。如果transaction的任何部分失败，Inline动作将和其他transaction一起回滚。无论成功或失败，Inline都不会在transaction范围外生成任何通知。

2. Deferred通信

Deferred通信在概念上等同于发送一个transaction给一个账户。这个transaction的执行是eos产快节点自主判断进行的，Deferrd通信无法保证消息一定成功或者失败。

如前所述，Deferred通信将在稍后由产快节点自行决定，从原始transaction（即创建deferred通信的transaction）的角度来看，它只能确定创建请求是成功提交还是失败（如果失败，transaction将立即失败），Deferred通信承担发送transaction的权力，也可以取消Deferred的transaction。

3. Inline通信流程

下图说明了多级嵌套的多action事务，雇主运行工资单，将支付转入员工账户，员工账户由银行管理，银行向其客户（员工）提供通知，以便客户可以利用银行提供的服务，例如支票和储蓄账户之间的自动转账。

![img](inline_action.png)

在本例中，原始transaction对employer合约执行两项操作：employer::runpayroll和employer::dootherstuff。这两项action中更有趣的是runpayroll。以下是它的功能，缩进与嵌套的Inline操作一致。

runpayroll进行从雇主账户到银行账户的eosio::token::transfer这个Inline通信，此示例选择在银行所需的信息的“memo”字段中加入了以确保合适的员工获得付款的信息。

        eosio::token::transfer  action执行转账操作，
    
        然后通知雇主，
    
        然后通知银行。
    
                银行有一份合约，收到eosio::token::transfer通知。收到后，原始transaction中的“memo”中的参数
    
                将用于某些特定的业务，以将资金转入员工的帐户。
    
                银行的行为通知员工账户存款。
    
                        员工有一个监听eosio::token::transfer通知的合约。收到后，参数用于确定转移
    
                        是否为自己，Employee的合约发送Inline操作bank::doacctpolicy。
    
                                bank::doacctpolicy 根据客户配置的政策为客户执行一些操作，例如，
    
                                在员工的银行支票和储蓄账户之间转移资金。

dootherstuff 做了一些其他的行动，说明一个交易可以有多个行动。

下图描绘了上例中transction跟踪：

![img](inline_transaction.png)

\4. Deferred通信流程

上述情况也可以使用Deferred通信完成，下图显示了Deferred通信场景。

![img](deferred_action.png)

Deferred通信和Inline通信有一些差异：

Deferred通信和Inline通信有一些差异：

1. 员工提交Deferred操作而不是Inline操作。
2. Deferred通信将导致transaction进行排队等待可能的进一步处理，而不是直接处理给出处理结果。这是Deferred交易一个非常重要的特征！ 
3. 当被调用时，动作执行的上下文与发出Deferred请求的原始transaction上下文无关。

### 二、Inline/Deferred通信编程





## 参考资源

[EOS合约开发第十七章-合约通信编程](https://blog.csdn.net/bedrock_stable/article/details/80327817)

[EOS合约开发第十八章-合约通信编程(2)](https://blog.csdn.net/bedrock_stable/article/details/80362187)

