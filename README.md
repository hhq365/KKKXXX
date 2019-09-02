## 接口一般约定
* **所有时间戳都精确到毫秒，用13位的bigint格式**
* **http请求的时候公共参数(每次请求http接口的时都需带上的参数)：lang,packtype,version,usertoken** 
* **每次http请求返回的结果内容都是json格式的字符串**
* **如果以post方式请求数据，post的数据尽量采用json格式，并把json字符串加密之后放在post form的param字段**


**Parameters:**

参数 | HTTP方式 | 是否加密 | 是否必须 | 类型 | 含义
------------ | ------------ | ------------ | ------------ | ------------ | ------------
usertoken | GET | 否 | 是 | string | 当前用户token
packtype | GET | 否 | 是 | int | PcWeb=1 H5=2 IOS=3 Android=4
version | GET | 否 | 是 | string | 
lang | GET | 否 | 是 | string | en表示英文 cn表示中文
operatingsystem | POST json格式的字符串需要加密 | 是 | 否 | string | 设备操作系统
deviceid | POST json格式的字符串需要加密 | 是 | 否 | string | 设备标识
**apikey** | POST json格式的字符串需要加密 | 是 | 是 | string | 当请求中有apikey且长度大于零的时候 服务端将校验apikey 并且不校验usertoken


**lang**

code | 参数
------------ | ------------
cn | 简体中文
de | 德文
en | 英文
es | 西班牙
fr | 法文
ko | 韩文
jp | 日文
ru | 俄文
tw | 繁體中文


请求数据的例子
```
curl -X GET 'https://market.happyex.net/v1/klines.ashx?lang=cn&packtype=1&version=9.9.9&usertoken=aaaaaaaa&symbol=YOYO_BTC&interval=1m'
```
```
curl -X POST 'https://broker.happyex.net/queryorder.ashx?lang=cn&packtype=1&version=9.9.9&usertoken=aaaaaaaaa' -d 'param=DA5D47C0F8762663CB24E9252D4B033428FFF5407AFB87C2ADD86624E3CF1FD094EEF19BC165A6DDA385066F43D860CC56DA5B7959982B4332E91C9E98B1A1EF6647AF4299BB79C00CEBAC46CAC20F2F57AB3C96CADA50237C67F3728A5753E4'
```

* 其中param参数后面的字符串是加密后的json参数
* 每次http请求返回的结果内容都是json格式的字符串：status, message, data，在data字段里面放业务层的内容

**status 状态：**
* 1  表示成功
* -1 失败，一般就显示下message对应的错误信息
* -2 需要登录
* -3 需要交易登录
* -4 余额不足需要弹充值框
* -5 密码重试太多，需要弹找回密码框
message	错误信息	当status==-1的时候，用来显示的错误信息




## Post加密方式

* 加密方法：3des
* 加密模式：**CBC** 
* 加密填充：**pkcs7padding**  
* 输出： **hex**

desKey与desIV 请在happyex网站apikey设置页面获取（暂未开放，需联系技术人员提供）

**C# demo**
```C#
public static string EncodeParam(string param, string desKey, string desIV)
{
    byte[] data = Encoding.UTF8.GetBytes(param);
    using (TripleDESCryptoServiceProvider tdes = new TripleDESCryptoServiceProvider())
    {
        tdes.Mode = CipherMode.CBC;
        tdes.Padding = PaddingMode.PKCS7;
        tdes.Key = Encoding.UTF8.GetBytes(desKey);
        tdes.IV = Encoding.UTF8.GetBytes(desIV);
        ICryptoTransform cTransform = tdes.CreateEncryptor();
        byte[] resultArray = cTransform.TransformFinalBlock(data, 0, data.Length);
        SoapHexBinary shb = new SoapHexBinary(resultArray);
        return shb.ToString();
    }
}

public static string DecodeParam(string hexString, string desKey, string desIV)
{
    SoapHexBinary shb = SoapHexBinary.Parse(hexString);
    byte[] data = shb.Value;
    using (TripleDESCryptoServiceProvider tdes = new TripleDESCryptoServiceProvider())
    {
        tdes.Key = Encoding.UTF8.GetBytes(desKey);
        tdes.IV = Encoding.UTF8.GetBytes(desIV);
        ICryptoTransform cTransform = tdes.CreateDecryptor();
        byte[] resultArray = cTransform.TransformFinalBlock(data, 0, data.Length);
        string ret = UTF8Encoding.UTF8.GetString(resultArray);
        return ret;
    }
}

```

**PHP demo**
```PHP
<?php
class CryptDes {
     var $key;
     var $iv;
     function CryptDes($key, $iv){
        $this->key = $key;
        $this->iv = $iv;
     }
                  
     function encrypt($input){
         $size = mcrypt_get_block_size(MCRYPT_3DES,MCRYPT_MODE_CBC);
         $input = $this->PaddingPKCS7($input);
         $td = mcrypt_module_open(MCRYPT_3DES,'',MCRYPT_MODE_CBC, '');        
         @mcrypt_generic_init($td, $this->key, $this->iv);
         $data = mcrypt_generic($td, $input);
         mcrypt_generic_deinit($td);
         mcrypt_module_close($td);
         return strtoupper(bin2hex($data));
     }
              
     function decrypt($encrypted){        
         $encrypted = $this->hex2bin($encrypted);       
         $td = mcrypt_module_open(MCRYPT_3DES,'',MCRYPT_MODE_CBC,'');         
         $ks = mcrypt_enc_get_key_size($td);
         @mcrypt_generic_init($td,  $this->key, $this->iv);
         $decrypted = mdecrypt_generic($td, $encrypted);
         mcrypt_generic_deinit($td);
         mcrypt_module_close($td);
         $y=$this->UnPaddingPKCS7($decrypted);
         return $y;
     }
              
     function hex2bin($hexData)
     {
        $binData = "";
        for($i = 0; $i < strlen ($hexData); $i += 2)
        {
           $binData .= chr(hexdec(substr($hexData, $i, 2)));
        }        
        return $binData;
    }
              
     function PaddingPKCS7($data) {
         $block_size = mcrypt_get_block_size(MCRYPT_DES, MCRYPT_MODE_CBC);
         $padding_char = $block_size - (strlen($data) % $block_size);
         $data .= str_repeat(chr($padding_char),$padding_char);
         return $data;
     }

     private function UnPaddingPKCS7 ($text)
     {
        $pad = ord($text{strlen($text) - 1});
        if ($pad > strlen($text)) {
            return false;
        }
        if (strspn($text, chr($pad), strlen($text) - $pad) != $pad) {
            return false;
        }
        return substr($text, 0, - 1 * $pad);
     }
}
              
$des = new CryptDes("key","iv");//（秘钥向量，混淆向量）
echo $ret = $des->decrypt("密文");//加密字符串
?>

```

**Java demo**
```Java
import javax.crypto.Cipher;
import javax.crypto.SecretKeyFactory;
import javax.crypto.spec.DESedeKeySpec;
import javax.crypto.spec.IvParameterSpec;
import java.security.Key;

public class Des3Utils {
    // 加解密统一使用的编码方式
    private final static String encoding = "UTF-8";
    /**
     * 3DES加密
     * @param plainText 明文
     */
    public static String encode(String plainText,String secretKey,String iv){
        try{
            Key deskey = null;
            DESedeKeySpec spec = new DESedeKeySpec(secretKey.getBytes());
            SecretKeyFactory keyfactory = SecretKeyFactory.getInstance("desede");
            deskey = keyfactory.generateSecret(spec);

            Cipher cipher = Cipher.getInstance("desede/CBC/PKCS5Padding");
            IvParameterSpec ips = new IvParameterSpec(iv.getBytes());
            cipher.init(Cipher.ENCRYPT_MODE, deskey, ips);
            byte[] encryptData = cipher.doFinal(plainText.getBytes(encoding));
            return toHexString(encryptData);
        }catch(Exception e){
            e.printStackTrace();
        }
        return plainText;
    }

    /**
     * 3DES解密
     * @param encryptText 加密文本
     */
    public static String decode(String encryptText,String secretKey,String iv){
        try{
            Key deskey = null;
            DESedeKeySpec spec = new DESedeKeySpec(secretKey.getBytes());
            SecretKeyFactory keyfactory = SecretKeyFactory.getInstance("desede");
            deskey = keyfactory.generateSecret(spec);
            Cipher cipher = Cipher.getInstance("desede/CBC/PKCS5Padding");
            IvParameterSpec ips = new IvParameterSpec(iv.getBytes());
            cipher.init(Cipher.DECRYPT_MODE, deskey, ips);

            byte[] decryptData = cipher.doFinal(fromHexString(encryptText));
            return new String(decryptData, encoding);
        }catch(Exception e){
            e.printStackTrace();
        }
        return encryptText;
    }

    public static byte[] toHex(byte[] digestByte) {
        byte[] rtChar = new byte[digestByte.length * 2];
        for (int i = 0; i < digestByte.length; i++) {
            byte b1 = (byte) (digestByte[i] >> 4 & 0x0f);
            byte b2 = (byte) (digestByte[i] & 0x0f);
            rtChar[i * 2] = (byte) (b1 < 10 ? b1 + 48 : b1 + 55);
            rtChar[i * 2 + 1] = (byte) (b2 < 10 ? b2 + 48 : b2 + 55);
        }
        return rtChar;
    }

    public static String toHexString(byte[] digestByte) {
        return new String(toHex(digestByte));
    }

    public static byte[] fromHex(byte[] sc) {
        byte[] res = new byte[sc.length / 2];
        for (int i = 0; i < sc.length; i++) {
            byte c1 = (byte) (sc[i] - 48 < 17 ? sc[i] - 48 : sc[i] - 55);
            i++;
            byte c2 = (byte) (sc[i] - 48 < 17 ? sc[i] - 48 : sc[i] - 55);
            res[i / 2] = (byte) (c1 * 16 + c2);
        }
        return res;
    }

    public static byte[] fromHexString(String hex) {
        return fromHex(hex.getBytes());
    }
}


```


# 行情接口

## 获取品种列表

```
GET https://market.happyex.net/v1/product.ashx?curMarket={curMarket}&assetCode={assetCode}&lang={lang}&version={version}
```

* curMarket: 市场分类，假如不传此参数，返回所有的市场数据
* assetCode: 资产货币，假如不传此参数，返回所有的资产数据
* lang: 语言
* version: 版本


**Payload:**
```javascript
{	
    "status": "1",
	"message": "success",
	"data": [{
		"symbol": "ETH_BTC", 
		"status": true, //开盘状态
		"statusText": "", //开盘说明
		"name": "ETH/BTC", //合约名字
		"assetCode": "ETH", //资金代码
		"curMarket": "BTC", //市场代码
		"tickSize": "0.000000", //最小价格波动
		"decimalplace": "6", //小数位
		"lots": "6", //最小交易数量
		"minOrderAmount": "6", //最小交易金额
		"open": "0.000000", //24小时开盘价
		"high": "0.000000", //24小时最高价
		"low": "0.000000", //24小时最低价
		"close": "0.000000", //最新价格
		"volume": "0.000000" //交易量
		"upDown": "0.000000" //涨跌额
		"upDownRate": "1%" //涨跌额
		"tradedMoney": "12345.90" //交易额
		"tickerid": 111 //ticker  id
		"legalAmount": “$100” //法币金额
	}]
}
```

## 获取trades数据

```
GET https://market.happyex.net/v1/aggtrades.ashx?symbol={symbol}&limit={limit}&lang={lang}&version={version}
```

* symbol:合约代码
* limit:trades 条数
* lang:语言
* version:版本

**Payload:**
```javascript
{
    "status": "1",
    "message": "success",
    "data": [{
        "a": "26129", // Aggregate tradeId
        "p": "0.01633102", // Price
        "q": "4.70443515", // Quantity
        "f": "27781", // First tradeId
        "l": "27781", // Last tradeId
        "T": "1498793709153", // Timestamp
        "m": true, // Was the buyer the maker?
        "M": true // Was the trade the best price match?
    }]
}
```

## 获取orderbook
```
GET https://market.happyex.net/v1/depth.ashx?symbol={symbol}&limit={limit}&lang={lang}&version={version}
```

* symbol:合约代码
* limit:深度
* lang:语言
* version:版本

**Payload:**
```javascript
{
  "status": "1",
  "message": "success",
  "data": {
		"symbol": "ETH_BTC", //合约代码
		"name": "ETH/BTC",  //合约显示名字
		"assetCode": "ETH",  //资金代码
		"curMarket": "BTC",  //市场代码
		"lastupdateid": "1",  //最新更新orderbookid
		"risefall": "0",  //价格颜色  0 白 1 红  2 绿
		"lastPrice": "8",  //最新价
		"decimalplace": "8",  //小数位
		"bidsFilter": "1",  //深度图规则bids 最大值
		"asksFilter": "1",  //深度图规则asks 最小值
		"asksBgRule": "1",  //背景颜色 ASKs
		"bidsBgRule": "1",  //背景颜色 BIDs
		"bids": [{price: "0.092000", quantity: "1.000", amount: "0.09200000"},{price: "0.092000", quantity: "1.000", amount: "0.09200000"}],    //bid
		"asks": [{price: "0.092000", quantity: "1.000", amount: "0.09200000"}]       //ask
  }
}
```


## 获取klines接口
```
GET https://market.happyex.net/v1/klines.ashx?symbol={symbol}&interval={interval}&lang={lang}&version={version}
```

1. symbol:合约代码
2. interval:
			time , 1m , 3m , 5m , 15m , 30m , 1h , 2h , 4h , 6h , 8h , 12h , 1d , 1w

3. lang:语言
4. version:版本
5. starttime: long  开始时间
6. endtime: long  结束时间
7. count: 记录数，默认3000

**Payload:**
```javascript
{
	"status": "1",
	"message": "success",
	"data": {
		"symbol": "ETH_BTC", //合约代码
		"status": true, //开盘状态
		"statusText": "", //开盘说明
		"name": "ETH/BTC", //合约显示名字
		"assetCode": "ETH", //资金代码
		"curMarket": "BTC", //市场代码
		"tickSize": "0.000000", //最小价格波动
		"decimalplace": "6", //小数位
		"open": "0.000000", //24小时开盘价
		"high": "0.000000", //24小时最高价
		"low": "0.000000", //24小时最低价
		"close": "0.000000", //最新价格
		"volume": "0.000000", //交易量
		"upDown": "0.000000", //涨跌额
		"upDownRate": "1%", //涨跌幅
		"tradedMoney": "12345.90", //交易额
		"lastTime": "1516125780000", //最新成交时间
		"lastTimeDataTime": "1516125780000", //最新一条时间轴的时间，对webscoket校验使用
		"legalAmount": "￥100" //法币金额
		"timeData": [
			{
			"time": "1516125780000", //时间
			"open": "0.00000028", //开盘价
			"low": "0.00000001", //最低价
			"high": "0.000001", //最高价
			"close": "0.00000023", //收盘价
			"vol": "0.00332783" //量
			"val": "0.00332783" //额
			"prevClose": "0.00332783" //昨收
			"upDown": "0.000000" //涨跌额
			"upDownRate": "1%" //涨跌幅
			}
		            ]
	        }
}
```



# 交易接口

## buypage 接口
```
POST https://broker.happyex.net/buypage.ashx?&packtype={packtype}&version={version}&lang={lang}
```

* param是由POST参数构造成以下格式加密后得到的:

**Parameters:**

{"usertoken":"test","apikey":"test","symbol":"ETH_BTC"}


参数 | HTTP method | 是否加密 | 是否必须 | 类型 | 含义
------------ | ------------ | ------------ | ------------ | ------------ | ------------
usertoken | POST | 是 | 是 | string | 用户token,通过用户登录获取，假如使用apikey获取接口的话，usertoken传apikey的值
apikey | POST | 是 | 是 | string | apikey 平台提供
symbol | POST | 是|   |  string | symbol



**Payload:**
```javascript
{
	"status": "1",
	"message": "",
	"data": {
		"assetcode": "资产币种",
		"curmarket": "币种市场",
		"assetcamount": "assetcode余额（ 可卖）",
		"curamount": "curmarket余额（ 可买）",
		"tradefeerate": "手续费比例",
		"hasdiscount": "手续费是否有折扣",
		"tradefeediscount": "折扣后的手续费比例",
		"sellmaxvol": "最大可卖数量",
		"minpx": "委托价格的最小单位",
		"minvol": "交易数量的最小单位",
		"minval": "最小交易金额",
		"morderpxrange": "市价单委托价格在最新价基础上的浮动比例",
		"minvalplace": "交易金额的最小单位",
		"legalrate": "",
		"legalcur": "法币单位"
	}
}
```



## order 接口
```
POST https://broker.happyex.net/order.ashx?&packtype={packtype}&version={version}&lang={lang}
```

* param是由POST参数构造成以下格式加密后得到的:

**Parameters:**

{"usertoken":"aaa","apikey":"aaa","assetcode":"GAS","curmarket":"BTC",...}


参数 | HTTP method | 是否加密 | 是否必须 | 类型 | 含义
------------ | ------------ | ------------ | ------------ | ------------ | ------------
usertoken | POST | 是 | 是 | string | 用户token,通过用户登录获取，假如使用apikey获取接口的话，usertoken传apikey的值
apikey | POST | 是 | 是 | string | apikey 平台提供
symbol | POST | 是|   |  string | symbol
assetcode | POST | 是 |  |  string | 资产币种
curmarket | POST | 是 |  | string | 币种市场
ordertype | POST | 是 |  | int	 | 委托类型。市价单 1；限价单 2；止盈止损单 3 
bstype | POST | 是 |  | int	 | 买、卖。买入 1；卖出 2 
price | POST | 是 |  | decimal | 委托价格
quantity | POST | 是 |  | decimal | 委托数量
triggerdir | POST | 可选 |  | int | 触发方向大于等于触发价 1; 小于等于触发价触发 2
triggerprice | POST | 可选 |  | decimal | 触发价格
packtype | POST | 是 |  | int | 包类型
devicetype | POST | 否 |  | string | 设备型号（huawei、小米、iphone7、chrome、Safari等)
deviceid | POST | 否 |  | string | 设备唯一标识ID
timewindow | POST | 否 |  | bigint | 时间窗口。默认是5000，单位ms
timeinforce | POST | 否 |  | string | 委托过期时间限制。IOC；GTC

  

**Payload:**
```javascript
{
	"status": "1",
	"message": "下单请求已发送",
	"data": {}
}

{
	"status": "-1",
	"message": "下单请求发送失败",
	"data": {}
}

{
	"status": "4",
	"message": "余额不足",
	"data": {}
}
```



## batchorder 批量下单接口
```
POST https://broker.happyex.net/batchorder.ashx?&packtype={packtype}&version={version}&lang={lang}
```

* param是由POST参数构造成以下格式加密后得到的:

**Parameters:**

{"usertoken":"aaa","apikey":"aaa",...}


参数 | HTTP method | 是否加密 | 是否必须 | 类型 | 含义
------------ | ------------ | ------------ | ------------ | ------------ | ------------
usertoken | POST | 是 | 是 | string | 用户token,通过用户登录获取，假如使用apikey获取接口的话，usertoken传apikey的值
apikey | POST | 是 | 是 | string | apikey 平台提供
list | POST | 是 | 是 | object | 
-> ordermark |  |   | 是 |long | order在当前数组中的唯一标识
-> symbol |  |   | 是 |string | symbol
-> assetcode |  |   | 是 |  string | 资产币种
-> curmarket |  |   | 是 | string | 币种市场
-> ordertype |  |   | 是 | int	 | 委托类型。市价单 1；限价单 2；止盈止损单 3 
-> bstype |  |   | 是 | int	 | 买、卖。买入 1；卖出 2 
-> price |  |   | 是 | decimal | 委托价格
-> quantity |  |   | 是 | decimal | 委托数量
-> triggerdir |  |   | 可选 | int | 触发方向大于等于触发价 1; 小于等于触发价触发 2
-> triggerprice |  |   | 可选 | decimal | 触发价格
-> packtype |  |   | 是 | int | 包类型
-> devicetype |  |   | 否 | string | 设备型号（huawei、小米、iphone7、chrome、Safari等)
-> deviceid |  |   | 否 | string | 设备唯一标识ID
-> timewindow |  |   | 否 | bigint | 时间窗口。默认是5000，单位ms
-> timeinforce |  |   | 否 | string | 委托过期时间限制。IOC；GTC

  

**Payload:**
```javascript
{
	"status": "1",
	"message": "",
	"data": [{
		"ordermark": 1,
		"message": "",
		"status": "1",
		"orderno": "476"
	}]
}

{
	"status": "-1",
	"message": "下单请求发送失败",
	"data": {}
}

{
	"status": "4",
	"message": "余额不足",
	"data": {}
}
```



## 成交详情接口
```
POST https://broker.happyex.net/orderexedetail.ashx?&packtype={packtype}&version={version}&lang={lang}
```

* param是由POST参数构造成以下格式加密后得到的:

**Parameters:**

{"usertoken":"aaa","apikey":"aaa","orderno":"1"}

参数 | HTTP method | 是否加密 | 是否必须 | 类型 | 含义
------------ | ------------ | ------------ | ------------ | ------------ | ------------
usertoken | POST | 是 | 是 | string | 用户token,通过用户登录获取，假如使用apikey获取接口的话，usertoken传apikey的值
apikey | POST | 是 | 是 | string | apikey 平台提供
orderno | POST | 是|  是 |  bigint | 订单号

**Payload:**
```javascript
{
	"status": "1",
	"message": "",
	"data": {
		"totalfee": "总手续费",
		"totalamount": "总成交额",
		"list": [{
			"exetime": "成交时间",
			"filledprice": "成交价",
			"filledquantity": "成交数量",
			"tradefee": "手续费",
			"filledamount": "成交金额"
		}]
	}
}
```




## 查询成交历史接口
```
POST https://broker.happyex.net/queryexecution.ashx?&packtype={packtype}&version={version}&lang={lang}
```

* param是由POST参数构造成以下格式加密后得到的:

**Parameters:**

{"usertoken":"aaa","apikey":"test","page":"1","pagesize":"100"}

参数 | HTTP method | 是否加密 | 是否必须 | 类型 | 含义
------------ | ------------ | ------------ | ------------ | ------------ | ------------
usertoken | POST | 是 | 是 | string | 用户token,通过用户登录获取，假如使用apikey获取接口的话，usertoken传apikey的值
apikey | POST | 是 | 是 | string | apikey 平台提供
startdate |  | 是|  否 |  bigint | 开始日期（Unix timestamp，毫秒）
enddate |  | 是|  否 |  bigint | 结束日志（Unix timestamp，毫秒）
assetcode |  | 是|  否 |  string | 资产货币
curmarket |  | 是|  否 |  string | 资产货币市场
bstype |  | 是|  否 |  int | 买入 1；卖出 2
page |  | 是|  否 |  int | 页码
pagesize |  | 是|  否 |  int | 每页个数

 

**Payload:**
```javascript
{
	"status": "1",
	"message": "",
	"data": {
		"pagecount": 页数,
		"datacount": 条目总数,
		"list": [{
			"exetime": 成交时间（ 时间戳）,
			"symbol": symbol,
			"name": name,
			"ordertype": 委托类型,
			"ordertypename": ordertype的文字形式,
			"bstype": 买卖方向,
			"bstypename": bstype的文字形式,
			"filledprice": 成交价,
			"filledquantity": 成交数量,
			"tradefee": 手续费,
			"filledamount": 成交金额,
			"exeid": 成交ID,
			"orderno": 委托ID
		}]
	}
}
```




## cancelorder 撤单接口
```
POST https://broker.happyex.net/cancel.ashx?&packtype={packtype}&version={version}&lang={lang}
```

* param是由POST参数构造成以下格式加密后得到的:

**Parameters:**

{"usertoken":"aaa","apikey":"aaa","orderno":1,...}


参数 | HTTP method | 是否加密 | 是否必须 | 类型 | 含义
------------ | ------------ | ------------ | ------------ | ------------ | ------------
usertoken | POST | 是 | 是 | string | 用户token,通过用户登录获取，假如使用apikey获取接口的话，usertoken传apikey的值
apikey | POST | 是 | 是 | string | apikey 平台提供
orderno | POST | 是|   |  bigint | 订单号
packtype | POST | 是 |  | int | 包类型
devicetype | POST | 否 |  | string | 设备型号（huawei、小米、iphone7、chrome、Safari等)
deviceid | POST | 否 |  | string | 设备唯一标识ID

  

**Payload:**
```javascript
{
	"status": "1",
	"message": "撤单请求已发送",
	"data": {}
}

{
	"status": "-1",
	"message": "撤单请求发送失败",
	"data": {}
}

```



## batchcancelorder 批量撤单接口
```
POST https://broker.happyex.net/batchcancel.ashx?&packtype={packtype}&version={version}&lang={lang}
```

* param是由POST参数构造成以下格式加密后得到的:

**Parameters:**

{"usertoken":"aaa","apikey":"aaa","ordernos":1,...}


参数 | HTTP method | 是否加密 | 是否必须 | 类型 | 含义
------------ | ------------ | ------------ | ------------ | ------------ | ------------
usertoken | POST | 是 | 是 | string | 用户token,通过用户登录获取，假如使用apikey获取接口的话，usertoken传apikey的值
apikey | POST | 是 | 是 | string | apikey 平台提供
ordernos | POST | 是|  可选 |  string | 订单号，多个订单用逗号分隔，如 {"ordernos":"1,2,3"} ordernos、all 二选一，如果该字段有值，优先取该值，all=1会无效
all | POST | 是|  可选 |  int | 1 全部撤单；否则，传0或者不传
symbol | POST | 是|  可选 |  string | all=1时，symbol != ''，撤销symbol的所有挂单
ordertype | POST | 是|  可选 |  int | all=1时，该字段才有效。2 限价单；3 止盈止损单。 all=1 && ordertype=2，表示限价单全部撤单  all=1 && ordertype=3，表示止盈止损单全部撤单
devicetype | POST | 否 |  | string | 设备型号（huawei、小米、iphone7、chrome、Safari等)
deviceid | POST | 否 |  | string | 设备唯一标识ID

  

**Payload:**
```javascript
{
	"status": "1",
	"message": "撤单请求已发送",
	"data": {}
}

{
	"status": "-1",
	"message": "撤单请求部分失败",
	"data": {}
}

```


## 查询当前委托接口
```
POST https://broker.happyex.net/queryorder.ashx?&packtype={packtype}&version={version}&lang={lang}
```
* param是由POST参数构造成以下格式加密后得到的:

**Parameters:**

{"usertoken":"test","apikey":"test","symbol":"ETH_BTC"}

参数 | HTTP method | 是否加密 | 是否必须 | 类型 | 含义
------------ | ------------ | ------------ | ------------ | ------------ | ------------
usertoken | POST | 是 | 是 | string | 用户token,通过用户登录获取，假如使用apikey获取接口的话，usertoken传apikey的值
apikey | POST | 是 | 是 | string | apikey 平台提供
symbol |  | 是|  可选 |  string | 

**Payload:**
```javascript
{
	"status": "1",
	"message": "",
	"data": {
		"list": [{
			"orderno": "委托ID（ 撤单时使用）",
			"ordertime": " 委托时间",
			"symbol": "symbol",
			"name": "name",
			"ordertype": "委托类型",
			"ordertypename": "ordertype的文字形式",
			"orderstatus": "委托状态",
			"bstype": "买卖",
			"bstypename": "bstype的文字形式",
			"orderprice": "委托价格",
			"orderquantity": "委托数量",
			"filledquantity": "成交数量",
			"filledrate": "成交率",
			"amount": "金额",
			"triggercon": "触发条件",
			"bsandotype": "委托类型及买卖拼接字段"
		}]
	}
}
```



## 批量查询委托
```
POST https://broker.happyex.net/querybatchorders.ashx?&packtype={packtype}&version={version}&lang={lang}
```

* param是由POST参数构造成以下格式加密后得到的:

**Parameters:**

{"usertoken":"test","apikey":"test"}

参数 | HTTP method | 是否加密 | 是否必须 | 类型 | 含义
------------ | ------------ | ------------ | ------------ | ------------ | ------------
usertoken | POST | 是 | 是 | string | 用户token,通过用户登录获取，假如使用apikey获取接口的话，usertoken传apikey的值
apikey | POST | 是 | 是 | string | apikey 平台提供
ordernos | POST | | 是 |  string | 多个orderno，用逗号分隔，如 "ordernos":"1,2,3"

 

**Payload:**
```javascript
{
	"status": "1",
	"message": "",
	"data": {
		"list": [{
			"orderno": 委托ID（ 撤单时使用）,
			"ordertime": 委托时间（ 时间戳）,
			"symbol": symbol,
			"name": name,
			"ordertype": 委托类型,
			"ordertypename": ordertype的文字形式,
			"orderstatus": 委托状态,
			"bstype": 买卖,
			"bstypename": bstype的文字形式,
			"orderprice": 委托价格,
			"orderquantity": 委托数量,
                                                "filledprice": 成交均价,
			"filledquantity": 成交数量,
			"filledrate": 成交率,
			"amount": 金额,
			"triggercon": 触发条件,
			"bsandotype": 委托类型及买卖拼接字段
		}]
	}
}
```




## 查询历史委托接口
```
POST https://broker.happyex.net/queryhisorder.ashx?&packtype={packtype}&version={version}&lang={lang}
```


**Parameters:**

{"usertoken":"test","apikey":"test"}

参数 | HTTP method | 是否加密 | 是否必须 | 类型 | 含义
------------ | ------------ | ------------ | ------------ | ------------ | ------------
usertoken | POST | 是 | 是 | string | 用户token,通过用户登录获取，假如使用apikey获取接口的话，usertoken传apikey的值
apikey | POST | 是 | 是 | string | apikey 平台提供
startdate | POST | 是|  否 |  bigint | 开始日期（时间戳，毫秒）
pagesize | POST | 是|  否 |  int | 每页个数
page | POST | 是|  否 |  int | 页码
enddate | POST | 是|  否 |  bigint | 结束日志（时间戳，毫秒）
curmarket | POST | 是|  否 |  string | 资产货币市场
bstype | POST | 是|  否 |  int | 买入 1；卖出 2
hidecancel | POST | 是|  否 |  int | 是否隐藏已撤单。不隐藏 0；隐藏 1。默认是0。
assetcode | POST | 是|  否 |  string | 资产货币

 


**Payload:**
```javascript
{
	"status": "1",
	"message": "",
	"data": {
		"pagecount": 页数,
		"datacount": 条目总数,
		"list": [{
			"orderno": 委托ID,
			"ordertime": 委托时间（ 时间戳）,
			"symbol": symbol,
			"name": name,
			"ordertype": 委托类型,
			"ordertypename": ordertype的文字形式,
			"bstype": 买卖,
			"bstypename": bstype的文字形式,
			"orderprice": 委托价格,
			"orderquantity": 委托数量,
			"filledprice": 成交价格,
			"filledquantity": 成交数量,
			"orderstatus": 委托状态,
			"orderstatusname": orderstatus的文字形式,
			"amount": 金额,
			"triggercon": 触发条件
		}]
	}
}
```




## 用户资金列表接口
```
POST https://broker.happyex.net/userasset.ashx?&packtype={packtype}&version={version}&lang={lang}
```
* param是由POST参数构造成以下格式加密后得到的:

**Parameters:**

{"usertoken":"test","apikey":"test","sortfield":"amount","sortdir":"1"}

参数 | HTTP method | 是否加密 | 是否必须 | 类型 | 含义
------------ | ------------ | ------------ | ------------ | ------------ | ------------
usertoken | POST | 是 | 是 | string | 用户token,通过用户登录获取，假如使用apikey获取接口的话，usertoken传apikey的值
apikey | POST | 是 | 是 | string | apikey 平台提供
sortfield | POST | | 否 |  string | 排序字段，默认amount
sortdir | POST | | 否 |  string | 0：降序，1：升序；默认0
 

**Payload:**
```javascript
{
	"status": "1",
	"message": "",
	"data": {
		"totalval": "当前总估值",
		"btctotalval": "btc估值",
		"legaltotalval": "法币估值",
		"withdrawlimit": "24 小时提现额度",
		"limitused": "已用额度",
		"pettyval": "小额资产的定义值, 多少算小额资产",
		"list": [{
			"assetcode": "资产代码",
			"assetfullname": "资产全称",
			"amount": "总额",
			"canusedamount": "可用余额",
			"frozenamount": "下单冻结金额",
			"frozenwithdraw": "提现冻结",
			"btcval": "btc估值",
			"legalval": "法币估值",
			"depositstatus": "充值按钮状态",
			"withdrawstaus": "提现按钮状态",
			"tradestatus": "交易按钮状态",
			"hastag": "",
			"logourl": "图片url地址"
		}]
	}
}
```
