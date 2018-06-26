## 介绍

通过了解以下信息，您可以方便的使用 FCoin 提供的 API 来接入 FCoin 交易平台。

## 认证

FCoin 使用 API key 和 API secret 进行验证，请访问 设置中心，并注册成为开发者，获取 API key 和 API secret。

FCoin 的 API 请求，除公开的 API 外都需要携带 API key 以及签名

## 访问限制

目前访问频率为每个用户 100次 / 10秒，未来会按照业务区分访问频率限制。

## API 签名

签名前准备的数据如下：

HTTP_METHOD + HTTP_REQUEST_URI + TIMESTAMP + POST_BODY

连接完成后，进行 Base64 编码，对编码后的数据进行 HMAC-SHA1 签名，并对签名进行二次 Base64 编码，各部分解释如下：

***请注意需要进行两次 `Base64` 编码！***

### HTTP_METHOD

GET, POST, DELETE, PUT 需要大写

### HTTP_REQUEST_URI

https://api.fcoin.com/v2/ 为 v2 API 的请求前缀

后面再加上真正要访问的资源路径，如 orders?param1=value1，最终即 https://api.fcoin.com/v2/orders?param1=value1

对于请求的 URI 中的参数，需要按照按照字母表排序！

即如果请求的 URI 为 https://api.fcoin.com/v2/orders?c=value1&b=value2&a=value3，则进行签名时，应先将请求参数按照字母表排序，最终进行签名的 URI 为 https://api.fcoin.com/v2/orders?a=value3&b=value2&c=value1， 请注意，原请求 URI 中的三个参数顺序为 c, b, a，排序后为 a, b, c。

### TIMESTAMP

访问 API 时的 UNIX EPOCH 时间戳，需要和服务器之间的时间差少于 30 秒

### POST_BODY

如果是 POST 请求，POST 请求数据也需要被签名，签名规则如下：

所有请求的 key 按照字母顺序排序，然后进行 url 参数化，并使用 & 连接。

***请注意 POST_BODY 的键值需要按照字母表排序！ ***

### 完整示例

####　参数名称

    FC-ACCESS-KEY
    FC-ACCESS-SIGNATURE
    FC-ACCESS-TIMESTAMP

#### 说明

可以使用[开发者工具](https://developer.fcoin.com/zh.html#)（暂未开放）进行在线联调测试。

## 公开接口

### 查询服务器时间

```python
import fcoin

api = fcoin.authorize('key', 'secret', timestamp)
server_time = api.server_time()
```

    响应结果如下：

        {
          "status": 0,
          "data": 1523430502977
        }

此 API 用于获取服务器时间。

#### HTTP Request

GET https://api.fcoin.com/v2/public/server-time

### 查询可用币种

此 API 用于获取可用币种。

```python
import fcoin

api = fcoin.authorize('key', 'secret', timestamp)
currencies = api.currencies()
```

    响应结果如下：

      {
        "status": 0,
        "data": [
          "btc",
          "eth"
        ]
      }

#### HTTP Request

GET https://api.fcoin.com/v2/public/currencies

### 查询可用交易对

此 API 用于获取可用交易对。

```python
import fcoin

api = fcoin.authorize('key', 'secret', timestamp)
symbols = api.symbols()
```

    响应结果如下：

      {
        "status": 0,
        "data": [
          {
            "name": "btcusdt",
            "base_currency": "btc",
            "quote_currency": "usdt",
            "price_decimal": 2,
            "amount_decimal": 4
          },
          {
            "name": "ethusdt",
            "base_currency": "eth",
            "quote_currency": "usdt",
            "price_decimal": 2,
            "amount_decimal": 4
          }
        ]
      }

#### HTTP Request

GET https://api.fcoin.com/v2/public/symbols

## 行情

### 行情概述

行情是一个全公开的 API, 当前同时提供了 HTTP 和 WebSocket 的 API. 为确保可以更及时的获得行情, 推荐使用 WebSocket 进行接入. 为尽可能行情的实时性能, 当前公开部分只能获取最近一段时间的行情, 如果有需要获取全量或者历史行情, 请咨询 support@fcoin.com

所有 HTTP 请求的 URL base 为: https://api.fcoin.com/v2/market

所有 WebSocket 请求的 URL 为: wss://api.fcoin.com/v2/ws

下文会统一术语:

    topic 表示订阅的主题
    symbol 表示对应交易币种. 所有币种区分的 topic 都在 topic 末尾.
    ticker 行情 tick 信息, 包含最新成交价, 最新成交量, 买一卖一, 近 24 小时成交量.
    depth 表示行情深度, 买卖盘, 盘口.
    level 表示行情深度类型. 如 L20, L100.
    trade 表示最新成交, 最新交易.
    candle 表示蜡烛图, 蜡烛棒, K 线.
    resolution 表示蜡烛图的种类. 如 M1, M15.
    base volume 表示基准货币成交量, 如 btcusdt 中 btc 的量.
    quote volume 表示计价货币成交量, 如 btcusdt 中 usdt 的量
    ts 表示推送服务器的时间. 是毫秒为单位的数字型字段, unix epoch in millisecond.

#### WebSocket 首次建立链接

服务器会发送一个欢迎信息

    ts: 推送服务器当前的时间.

### 获取推送服务器时间

    gap: 推送服务器处理此语句的时间和客户端传输的时间差.
    ts: 推送服务器当前的时间.

### 获取 ticker 数据

为了使得 ticker 信息组足够小和快, 我们强制使用了列表格式.

    ticker 列表对应字段含义说明:

[
  "最新成交价",
  "最近一笔成交的成交量",
  "最大买一价",
  "最大买一量",
  "最小卖一价",
  "最小卖一量",
  "24小时前成交价",
  "24小时内最高价",
  "24小时内最低价",
  "24小时内基准货币成交量, 如 btcusdt 中 btc 的量",
  "24小时内计价货币成交量, 如 btcusdt 中 usdt 的量"
]

import fcoin

api = fcoin.authorize('key', 'secret', timestamp)
api.market.get_ticker("ethbtc")

{
  "status": 0,
  "data": {
    "type": "ticker.btcusdt",
    "seq": 680035,
    "ticker": [
      7140.890000000000000000,
      1.000000000000000000,
      7131.330000000,
      233.524600000,
      7140.890000000,
      225.495049866,
      7140.890000000,
      7140.890000000,
      7140.890000000,
      1.000000000,
      7140.890000000000000000
    ]
  }
}

HTTP 请求

GET https://api.fcoin.com/v2/market/ticker/$symbol


### 获取最新的深度明细

HTTP Request

GET https://api.fcoin.com/v2/market/depth/$level/$symbol

$level 包含的种类:
类型 	说明
L20 	20 档行情深度.
L100 	100 档行情深度.
full 	全量的行情深度, 不做时间保证和推送保证.

其中 L20 的推送时间会略早于 L100, 推送频次会略多于 L100, 看具体的压力和情况. 此处请按需使用.
WebSocket 订阅

订阅 topic: depth.$level.$symbol

import fcoin

fcoin_ws = fcoin.init_ws()
topics = ["depth.L20.ethbtc", "depth.L100.btcusdt"]
fcoin_ws.handle(print)
fcoin_ws.sub(topics)

    订阅成功的响应结果如下：

{
  "type": "topics",
  "topics": ["depth.L20.ethbtc", "depth.L100.btcusdt"]
}

    常规的推送结果

bids 和 asks 对应的数组一定是偶数条目, 买(卖)1价, 买(卖)1量, 依次往后排列.

### 获取最新的深度明细

#### HTTP Request

GET https://api.fcoin.com/v2/market/depth/$level/$symbol

$level 包含的种类:
类型 	说明
L20 	20 档行情深度.
L100 	100 档行情深度.
full 	全量的行情深度, 不做时间保证和推送保证.

其中 L20 的推送时间会略早于 L100, 推送频次会略多于 L100, 看具体的压力和情况. 此处请按需使用.
WebSocket 订阅

订阅 topic: depth.$level.$symbol

import fcoin

fcoin_ws = fcoin.init_ws()
topics = ["depth.L20.ethbtc", "depth.L100.btcusdt"]
fcoin_ws.handle(print)
fcoin_ws.sub(topics)

    订阅成功的响应结果如下：

{
  "type": "topics",
  "topics": ["depth.L20.ethbtc", "depth.L100.btcusdt"]
}

    常规的推送结果

bids 和 asks 对应的数组一定是偶数条目, 买(卖)1价, 买(卖)1量, 依次往后排列.

### 获取最新的成交明细

通过对比其中的成交 id 大小才能决定是否是更新的成交.{trade id} 需要注意, 常规由于 trade 到 transaction 过程的存在, 公开行情的成交 id 并不实际对应清算系统中的成交 id. 即使成交是一条记录, 也无法保证最新成交在重新获取时候 id 永远保持一致.

PS: 历史行情中, 是可以保证成交 id 保持恒定. {transaction id} 此处只作为行情更新通知, 不应依赖归档使用.
HTTP Request

GET https://api.fcoin.com/v2/market/trades/$symbol
查询参数(HTTP Query)
参数 	默认值 	描述
before 		查询某个 id 之前的 Trade
limit 		默认为 20 条
WebSocket 获取最近的成交

topic: trade.$symbol limit: 最近的成交条数 args: [topic, limit]

import fcoin

fcoin_ws = fcoin.init_ws()
topic = "trade.ethbtc"
limit = 3
args = [topic, limit]
fcoin_ws.req(args, rep_handler)

    请求成功的响应结果如下：

{"id":null,
 "ts":1523693400329,
 "data":[
   {
     "amount":1.000000000,
     "ts":1523419946174,
     "id":76000,
     "side":"sell",
     "price":4.000000000
   },
   {
     "amount":1.000000000,
     "ts":1523419114272,
     "id":74000,
     "side":"sell",
     "price":4.000000000
   },
   {
     "amount":1.000000000,
     "ts":1523415182356,
     "id":71000,
     "side":"sell",
     "price":3.000000000
   }
 ]
}

WebSocket 订阅

### 获取 Candle 信息

HTTP Request

GET https://api.fcoin.com/v2/market/candles/$resolution/$symbol
查询参数(HTTP Query)
参数 	默认值 	描述
before 		查询某个 id 之前的 Candle
limit 		默认为 20 条

$resolution 包含的种类
类型 	说明
M1 	1 分钟
M3 	3 分钟
M5 	5 分钟
M15 	15 分钟
M30 	30 分钟
H1 	1 小时
H4 	4 小时
H6 	6 小时
D1 	1 日
W1 	1 周
MN 	1 月
Weboskcet 订阅 Candle 数据

topic: candle.$resolution.$symbol

## 账户与资产

### 查询账户资产

import fcoin

api = fcoin.authorize('key', 'secret', timestamp)
api.accounts_balance

    响应结果如下：

{
  "status": 0,
  "data": [
    {
      "currency": "btc",
      "available": "50.0",
      "frozen": "50.0",
      "balance": "100.0"
    }
  ]
}

此 API 用于查询用户的资产列表。
HTTP Request

GET https://api.fcoin.com/v2/accounts/balance

## 订单

### 订单模型说明

订单模型由以下属性构成：
属性 	类型 	含义解释
id 	String 	订单 ID
symbol 	String 	交易对
side 	String 	交易方向（buy, sell）
type 	String 	订单类型（limit，market）
price 	String 	下单价格
amount 	String 	下单数量
state 	String 	订单状态
executed_value 	String 	已成交
filled_amount 	String 	成交量
fill_fees 	String 	手续费
created_at 	Long 	创建时间
source 	String 	来源

订单状态说明：
属性 	含义解释
submitted 	已提交
partial_filled 	部分成交
partial_canceled 	部分成交已撤销
filled 	完全成交
canceled 	已撤销
pending_cancel 	撤销已提交

### 创建新的订单

import fcoin

api = fcoin.authorize('key', 'secret', timestamp)
order_create_param = fcoin.order_create_param('btcusdt', 'buy', 'limit', '8000.0', '1.0')
api.orders.create(order_create_param)

    响应结果如下：

{
  "status": 0,
  "data": "9d17a03b852e48c0b3920c7412867623"
}

此 API 用于创建新的订单。
HTTP Request

POST https://api.fcoin.com/v2/orders
请求参数
参数 	默认值 	描述
symbol 	无 	交易对
side 	无 	交易方向
type 	无 	订单类型
price 	无 	价格
amount 	无 	下单量

### 查询订单列表

import fcoin

api = fcoin.authorize('key', 'secret', timestamp)
api.orders.get()

    响应结果如下：

{
  "status": 0,
  "data": [
    {
      "id": "string",
      "symbol": "string",
      "type": "limit",
      "side": "buy",
      "price": "string",
      "amount": "string",
      "state": "submitted",
      "executed_value": "string",
      "fill_fees": "string",
      "filled_amount": "string",
      "created_at": 0,
      "source": "web"
    }
  ]
}

此 API 用于查询订单列表。
HTTP Request

GET https://api.fcoin.com/v2/orders
查询参数
参数 	默认值 	描述
symbol 		交易对
states 		订单状态
before 		查询某个页码之前的订单
after 		查询某个页码之后的订单
limit 		每页的订单数量，默认为 20 条

### 获取指定订单

import fcoin

api = fcoin.authorize('key', 'secret', timestamp)
api.orders.get('9d17a03b852e48c0b3920c7412867623')

    响应结果如下：

{
  "status": 0,
  "data": {
    "id": "9d17a03b852e48c0b3920c7412867623",
    "symbol": "string",
    "type": "limit",
    "side": "buy",
    "price": "string",
    "amount": "string",
    "state": "submitted",
    "executed_value": "string",
    "fill_fees": "string",
    "filled_amount": "string",
    "created_at": 0,
    "source": "web"
  }
}

此 API 用于返回指定的订单详情。
HTTP Request

GET https://api.fcoin.com/v2/orders/{order_id}
URL 参数
参数 	描述
order_id 	订单 ID

### 申请撤销订单

import fcoin

api = fcoin.authorize('key', 'secret', timestamp)
api.orders.submit_cancel(2)

    响应结果如下：

{
  "status": 0,
  "msg": "string",
  "data": true
}

此 API 用于撤销指定订单，订单撤销过程是异步的，即此 API 的调用成功代表着订单已经进入撤销申请的过程，需要等待撮合的进一步处理，才能进行订单的撤销确认。
HTTP Request

POST https://api.fcoin.com/v2/orders/{order_id}/submit-cancel
URL 参数
参数 	解释
order_id 	订单 ID

### 查询指定订单的成交记录

import fcoin

api = fcoin.authorize('key', 'secret', timestamp)
api.orders.get('9d17a03b852e48c0b3920c7412867623').match_results()

    响应结果如下：

{
  "status": 0,
  "data": [
    {
      "price": "string",
      "fill_fees": "string",
      "filled_amount": "string",
      "side": "buy",
      "type": "limit",
      "created_at": 0
    }
  ]
}

此 API 用于获取指定订单的成交记录
HTTP Request

GET https://api.fcoin.com/v2/orders/{order_id}/match-results
URL 参数
参数 	解释
order_id 	订单 ID

### 订单错误代码

错误代码 	含义解释
2000 	账户错误

## 错误代码

错误代码 	含义解释
400 	Bad Request -- 错误的请求
401 	Unauthorized -- API key 或者签名，时间戳有误
403 	Forbidden -- 禁止访问
404 	Not Found -- 未找到请求的资源
405 	Method Not Allowed -- 使用的 HTTP 方法不适用于请求的资源
406 	Not Acceptable -- 请求的内容格式不是 JSON
429 	Too Many Requests -- 请求受限，请降低请求频率
500 	Internal Server Error -- 服务内部错误，请稍后再进行尝试
503 	Service Unavailable -- 服务不可用，请稍后再进行尝试


