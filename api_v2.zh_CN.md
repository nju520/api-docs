## OCX开发者接口 (API version 2) 

  接口URI前缀: /api/v2 

  返回结果格式: JSON

### Public/Private API

OCX开发者接口包含两类API: Public API是不需要任何验证就可以使用的接口，而Private API是需要进行签名验证的接口。下表列出了两者的主要区别:

<table class="table">
  <thead>
    <tr>
      <th>Public API</th>
      <th>Private API</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>无需验证</td>
      <td>需要验证</td>
    </tr>
    <tr>
      <td>无限制</td>
      <td>对于每个用户, 最多6000个请求每5分钟(平均20个请求/秒); 如果有更高需求可以联系OCX管理员</td>
    </tr>
    <tr>
      <td>无需准备立即可用</td>
      <td>先要向OCX管理员申请access/secret key</td>
    </tr>
  </tbody>
</table>

### 如何签名 (验证)

在给一个Private API请求签名之前, 你必须准备好你的access/secret key. 在注册并认证通过后之后，只需访问API密钥页面就可以得到您的密钥。 所有的Private API都需要这3个用于身份验证的参数:

<table class="table">
  <tr>
    <td>access_key</td>
    <td>你的access key</td>
  </tr>
  <tr>
    <td>tonce</td>
    <td>tonce是一个用正整数表示的时间戳，代表了从<a href='http://en.wikipedia.org/wiki/Unix_epoch'>Unix epoch</a>到当前时间所经过的毫秒(ms)数。tonce与服务器时间不得超过正负30秒。一个tonce只能使用一次。</td>
  </tr>
  <tr>
    <td>signature</td><td>使用你的secret key生成的签名</td>
  </tr>
</table>

签名的生成很简单，先把请求表示为一个字符串, 然后对这个字符串做hash: 
<pre>
  hash = HMAC-SHA256(payload, secret\_key).to\_hex 
</pre>

Payload就是代表这个请求的字符串, 通过组合HTTP方法, 请求地址和请求参数得到: 
<pre><code>
  # canonical\_verb是HTTP方法，例如GET 
  # canonical\_uri是请求地址， 例如/api/v2/markets 
  # canonical\_query是请求参数通过&连接而成的字符串，参数包括access\_key和tonce, 参数必须按照字母序排列，例如access\_key=xxx&foo=bar&tonce=123456789 
  # 最后再把这三个字符串通过'|'字符连接起来，看起来就像这样: 
  # GET|/api/v2/markets|access\_key=xxx&foo=bar&tonce=123456789 
  def payload 
    "#{canonical\_verb}|#{canonical\_uri}|#{canonical\_query}" 
  end
</code></pre>

假设我的secret key是"abc", 那么使用SHA256算法对上面例子中的payload计算HMAC的结果是(以hex表示)： 
<pre>
  hash = HMAC-SHA256('GET|/api/v2/markets|access\_key=xxx&foo=bar&tonce=123456789', 'abc').to\_hex = 'e324059be4491ed8e528aa7b8735af1e96547fbec96db962d51feb7bf1b64dee' 
</pre>

现在我们就可以这样来使用这个签名请求(以curl为例): 
<pre>
  curl -X GET 'https://api.ocx.com/api/v2/markets?access\_key=xxx&foo=bar&tonce=123456789&signature=e324059be4491ed8e528aa7b8735af1e96547fbec96db962d51feb7bf1b64dee'
</pre>

### 返回结果

如果API调用失败，返回的请求会使用对应的HTTP status code, 同时返回包含了详细错误信息的JSON数据, 比如: 
<pre><code>
  {"error":{"code":1001,"message":"market does not have a valid value"}} 
</code></pre>
所有错误都遵循上面例子的格式，只是code和message不同。code是OCX自定义的一个错误代码, 表明此错误的类别, message是具体的出错信息.

对于成功的API请求, OCX则会返回200作为HTTP status code, 同时返回请求的JSON数据.

<table class="table result">
  <thead>
    <tr>
      <th>数据类型</th><th>数据结构/示例</th><th>备注</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>Market</td>
      <td>
        <pre>
          <code>
          {
            "code": "ethbtc", 
            "name": "ETH/BTC", 
            "base_unit": "eth", 
            "quote_unit": "btc"
          }
          </code>
        </pre>
      </td>
      <td>
        <p>Market包含了某一个市场(例如ethbtc)的基本信息。</p>
      </td>
    </tr>
    <tr>
      <td>Account</td>
      <td>
        <pre>
          <code>
          {
            "currency":"btc",
            "balance":"1.30",
            "locked":"0.0"
          }
          </code>
        </pre>
      </td>
      <td>
        <p>Account包含了用户某一个币种账户的信息:</p>
        <p>currency: 账户的币种, 如btc</p>
        <p>balance: 账户余额, 不包括冻结资金</p>
        <p>locked: 冻结资金</p>
      </td>
    </tr>
    <tr>
      <td>Order</td>
      <td>
        <pre>
          <code>
            {
              "id":7,
              "side":"sell",
              "price":"40100.0",
              "avg_price":"40100",
              "state":"wait",
              "market":"btccny",
              "created_at":"2018-06-18T02:02:33Z",
              "volume":"100.0",
              "remaining_volume":"89.8",
              "executed_volume":"10.2",
            }
          </code>
        </pre>
      </td>
      <td>
        <p>Order包含了某一个订单的所有信息:</p>
        <p>id: 唯一的Order ID</p>
        <p>side: Buy/Sell, 代表买单/卖单.</p>
        <p>price: 出价</p>
        <p>avg_price: 平均成交价</p>
        <p>state: 订单的当前状态, wait, done或者cancel.  wait表明订单正在市场上挂单, 是一个active order, 此时订单可能部分成交或者尚未成交; done代表订单已经完全成交; cancel代表订单已经被撤销.</p>
        <p>market: 订单参与的交易市场</p>
        <p>created_at: 下单时间, ISO8601格式</p>
        <p>volume: 购买/卖出数量</p>
        <p>remaining_volume: 还未成交的数量. remaining_volume总是小于等于volume, 在订单完全成交时变成0.</p>
        <p>executed_volume: 已成交的数量. volume = remaining_volume + executed_volume</p>
      </td>
    </tr>
    <tr>
      <td>OrderBook</td>
      <td>
        <pre>
          <code>{"asks": [...],"bids": [...]}</code>
        </pre>
      </td>
      <td><p>OrderBook包含了当前市场的挂单信息:</p><p>asks: 卖单列表</p><p>bids: 买单列表</p></td>
    </tr>
    <tr>
      <td>Ticker</td>
      <td>
        <pre>
          <code>
            {
              "market_code":"ethcny",
              "low":"3000.0",
              "high":"3000.0",
              "last":"3000.0",
              "volume":"0.11",
              "open":"3000.0",
              "timestamp":1398410899
            }
          </code>
        </pre>
      </td>
      <td>
        <p>最新成交价</p>
      </td>
    </tr>
  </tbody>
</table>

### 一些例子

以40000CNY的价格买入1BTC: 
<pre>
  <code>
  curl -X POST 'https://api.ocx.com/api/v2/orders' -d 'access\_key=your\_access\_key&tonce=1234567&signature=computed\_signature&market=btccny&price=40000&side=buy&volume=1' 
  </code>
</pre>  

同时创建多个委托: 
<pre>
  <code>
  curl -X POST 'https://api.ocx.com/api/v2/orders/multi' -d 'access\_key=your\_access\_key&tonce=123456789&signature=computed\_signature&market=btccny&orders\[\]\[price\]=40000&orders\[\]\[side\]=sell&orders\[\]\[volume\]=0.5&orders\[\]\[price\]=39999&orders\[\]\[side\]=sell&orders\[\]\[volume\]=0.99'
  </code>
</pre>  


### 注意事项

<table class="table">
  <thead>
    <tr>
      <th>API</th><th>Detail</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td>POST /api/v2/order/cancel</td>
      <td>取消挂单. 取消挂单是一个异步操作,api成功返回仅代表取消请求已经成功提交,服务器正在处理,不代表订单已经取消. 当你的挂单有尚未处理的成交(trade)事务,或者取消请求队列繁忙时,该订单会延迟取消. api返回被取消的订单,返回结果中的订单不一定处于取消状态,你的代码不应该依赖api返回结果,而应该通过/api/v2/order来得到该订单的最新状态.</td>
    </tr>
    <tr>
      <td>POST /api/v2/orders/clear</td>
      <td>取消你所有的挂单. 取消挂单是一个异步操作, api成功返回代表取消请求已经提交,服务器正在处理. api返回的结果是你当前挂单的集合,结果中的订单不一定处于取消状态.</td>
    </tr>
  </tbody>
</table>

### API列表

以下是详细的API列表，所有需要access_key/tonce/signature的都是Private API, 其他的则是Public API。