# 1. 开发环境配置
sdk技术问题沟通QQ群：609994083</br>

**注：SDK在首次获取token过程中，用户手机必须在打开数据网络情况下才能成功**

## 1.1. 总体使用流程

1. 调用SDK方法来获得`token`，步骤如下： 

    a. 构造SDK中认证工具类`AuthnHelper`的对象；</br>

    b. 使用`AuthnHelper`中的`getTokenImp`方法，获得token。</br>

2. 在业务服务端调用`获取用户信息接口`获取相关用户信息</br>

## 1.2. 导入SDK的jar文件

1. 将`quick_login_android_**.jar`,拷贝到应用工程的libs目录下，如没有该目录，可新建；
2. 将sdk所需要的证书文件`clientCert.crt`、`serverPublicKey.pem`拷贝到项目`assets`目录下。

## 1.3. 配置AndroidManifest

注意：为避免出错，请直接从Demo中复制带<!-- required -->标签的代码

**1. 配置权限**

```java
<uses-permission android:name="android.permission.INTERNET" />
<uses-permission android:name="android.permission.READ_PHONE_STATE" />
<uses-permission android:name="android.permission.ACCESS_WIFI_STATE" />
<uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />
<uses-permission android:name="android.permission.SEND_SMS" />
<uses-permission android:name="android.permission.CHANGE_NETWORK_STATE" />
<uses-permission android:name="android.permission.WRITE_SETTINGS" />
<uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE" />
<uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE"/>
```


通过以上步骤，工程就已经配置完成了。接下来就可以在代码里使用统一认证的SDK进行开发了

</br>

## 1.4. SDK使用步骤

**1. 创建一个AuthnHelper实例** 

`AuthnHelper`是SDK的功能入口，所有的接口调用都得通过AuthnHelper进行调用。因此，调用SDK，首先需要创建一个AuthnHelper实例，其代码如下：

```java
public void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    mContext = this;    
    ……
    mAuthnHelper = AuthnHelper.getInstance(MainActivity.this);
  	mAuthnHelper.init(Constant.APP_ID, Constant.APP_KEY);
    }
```

**2. 实现回调**

所有的SDK接口调用，都会传入一个回调，用以接收SDK返回的调用结果。结果以`JsonObjent`的形式传递，`TokenListener`的实现示例代码如下：

```java
mListener = new TokenListener() {
    @Override
    public void onGetTokenComplete(JSONObject jObj) {
        if (jObj != null) {
            mResultString = jObj.toString();
            mHandler.sendEmptyMessage(RESULT);
            if (jObj.has("token")) {
                mtoken = jObj.optString("token");
            }
        }
    }
};
```

**3. 接口调用**

```java
mAuthnHelper.getTokenImp(mLoginType, AuthnHelper.AUTH_TYPE_SMS, mListener);
```

# 2. SDK方法说明

## 2.1. 初始化接口

### 2.1.1. 方法描述

初始化接口

</br>

**原型**

```java
public void init(String appId, 
                 String appKey, 
                 boolean whetherToClearKs)
```

</br>

**中间件是什么？**
```
为了减少应用在登录或做单点登录时，频繁做网关取号操作，减少取号失败概率，应用首次登录时，SDK会将用户加密的登录信息保存在本地，称为中间件；
中间件会在用户每天首次登录/单点登录时会去访问服务端判断是否失效；
服务端中间件有效期30天；
服务端中间件失效后，用户登录或使用单点登录时，将需要重新走网关取号流程
```

### 2.1.2. 参数说明

| 参数               | 类型      | 说明            |
| ---------------- | ------- | ------------- |
| appId            | String  | 应用ID          |
| appKey           | String  | 应用Key         |
| whetherToClearKs | boolean | 更换SIM卡是否清除中间件 |

</br>

## 2.2. 获取sim卡状态

### 2.2.1. 方法描述

**功能**

获取sim卡数量以及每张卡的状态

</br>

**原型**

```java
public void getSimInfo(final TokenListener listener)
```

</br>

### 2.2.2. 参数说明

**请求参数**

| 参数       | 类型            | 说明                                       |
| :------- | :------------ | :--------------------------------------- |
| listener | TokenListener | TokenListener为回调监听器，是一个java接口，需要调用者自己实现；TokenListener是接口中的认证登录token回调接口，OnGetTokenComplete是该接口中唯一的抽象方法，即void OnGetTokenComplete(JSONObject  jsonobj) |

</br>

**响应参数**

OnGetTokenComplete的参数JSONObject，含义如下：

| 字段               | 类型     | 含义                           |
| ---------------- | ------ | ---------------------------- |
| resultCode       | Int    | 接口返回码，“103000”为成功。           |
| resultDesc       | String | 返回码描述                        |
| isDualSim        | String | "true":是双卡;</br>"false":不是双卡 |
| sim1Operator     | String | 卡槽1运营商名称                     |
| sim2Operator     | String | 卡槽2运营商名称                     |
| defaultDataSimId | String | 默认数据卡卡槽                      |
| loginMethod      | String | 当前使用方法名                      |

</br>

### 2.2.3. 示例

**请求示例代码**

```java
mAuthnHelper.getSimInfo(mListener);
```

**响应示例代码**

```
{
	"resultCode": "103000",
	"resultDesc": "获取SIM卡信息成功",
	"isDualSim": "false",
	"sim1Operator": "移动",
	"sim2Operator": "未知",
	"defaultDataSimId": "1",
	"loginMethod": "getSimInfo"
}
```

## 2.3. 隐式登录

### 2.3.1. 方法描述

**功能**

本方法用于实现**获取用户信息**功能。使用本方法获取到的token，可通过`获取用户信息接口`交换用户信息。如果业务A需要做单点登录时，也可使用该方法获得token后，再将token传递到业务B，由业务B带着token去换取业务A使用者的用户手机号码，完成单点登录流程

</br>

**原型**

```java
public void getTokenImp(int loginType, 
                        String authType, 
                        final TokenListener listener) 
```

</br>

### 2.3.2. 参数说明

**请求参数**

| 参数        | 类型            | 说明                                       |
| :-------- | :------------ | :--------------------------------------- |
| loginType | Int           | 登录方式：</br>LOGIN_TYPE_DEFAULT = 0:默认登录方式，登录优先级为：中间件 -> 上网卡；</br>LOGIN_TYPE_SIM1 = 1:使用SIM1登录；</br>LOGIN_TYPE_SIM2 = 2:使用SIM2登录；</br>LOGIN_TYPE_CMCC_FIRST = 3:使用移动卡优先登录；</br>LOGIN_TYPE_WAP = 4:使用上网卡登录； |
| authType  | String        | 认证方式：</br>AUTH_TYPE_WAP = "3":网关鉴权；|
| listener  | TokenListener | TokenListener为回调监听器，是一个java接口，需要调用者自己实现；TokenListener是接口中的认证登录token回调接口，OnGetTokenComplete是该接口中唯一的抽象方法，即void OnGetTokenComplete(JSONObject  jsonobj) |

</br>

**响应参数**

OnGetTokenComplete的参数JSONObject，含义如下：

| 字段            | 类型     | 含义                                       |
| ------------- | ------ | ---------------------------------------- |
| resultCode    | String | 接口返回码，“103000”为成功。                       |
| authType      | String | 认证类型：0:其他;</br>1:WiFi下网关鉴权;</br>2:网关鉴权;</br>4: WIFI下网关鉴权复用中间件登录;</br>5: 网关鉴权复用中间件登录; |
| authTypeDec   | String | 认证类型描述，对应authType                        |
| selectSim     | String | 当前选中的卡槽                                  |
| securityphone | String | 手机号码掩码，如“138XXXX0000”                    |
| token         | String | 成功时返回：临时凭证，token有效期2min，一次有效；同一用户（手机号）10分钟内获取token且未使用的数量不超过30个 |
| openId        | String | 成功时返回：用户身份唯一标识                           |

</br>

### 2.3.3. 示例

**请求示例代码**

```java
/*
  public static final String AUTH_TYPE_WAP = "3";//网关鉴权
  */
mAuthnHelper.getTokenImp(AuthnHelper.LOGIN_TYPE_DEFAULT , AuthnHelper.AUTH_TYPE_WAP, mListener);

```

**响应示例代码**

```
{
	"resultCode": "103000",
	"authType": "1",
	"authTypeDes": "WIFI下网关鉴权",
	"selectSim": "1",
	"securityphone": "188****7241",
	"openId": "BcLpJvyI1GZSQffq1AHXsL0bqmIfNs6_XALVMNsdvozIl3XufPo4",
	"token": "84840100013602003A4D7A6330517A6846515556434E30453052454E464E45457A40687474703A2F2F3132302E3139372E3233352E32373A383038302F72732F40303103000405A0ED3D040012383030313230313731313135313034373036050010AC042E4B80D8447490DC06746F77522F06000131070003323030FF0020AE3E9EC069A700B06087E608AC273721076A50336F4D001EF2C99EE6AF2CC9EA"
}

```

</br>

## 2.4. 清除中间件缓存

### 2.4.1. 方法描述

**功能**

该方法用于清除中间件缓存信息，当用户不想使用中间件信息进行登录或者用户换了另外的sim卡的时候，可以使用该方法将中间件信息清除，然后再进行登录操作。中间件清除后，用户再次登录时，将走网关/短信逻辑重新取号。

</br>

**原型**

```java
 public void clearChache()
```

#3. 平台接口说明

## 3.1. 获取用户信息接口

业务平台或服务端携带用户授权成功后的token来调用统一认证服务端获取用户手机号码等信息。**注：本接口仅适用于5.4.0及以上版本SDK**

### 3.1.1. 业务流程

SDK在获取token过程中，用户手机必须在打开数据网络情况下才能获取成功，纯wifi环境下会走短信上行获取token（如果有配置）或者返回错误码



### 3.1.2. 接口说明

**生产环境请求地址：**

https://wap.cmpassport.com:8443/uniapi/uniTokenValidate 

http://wap.cmpassport.com:8080/uniapi/uniTokenValidate 

**联调环境请求地址：**http://218.205.115.220:10085/test/api/uniTokenValidate

**协议：** HTTPS+ application/json

**请求方法：** POST+json

</br>

### 3.1.3. 参数说明

**请求参数**

| 参数名称                | 约束   | 层级   | 参数类型   | 说明                                       |
| ------------------- | ---- | ---- | ------ | ---------------------------------------- |
| header              | 必选   | 1    |        |                                          |
| version             | 必选   | 2    | string | 填1.0                                     |
| strictcheck         | 必选   | 2    | string | 填0                                     |
| msgid               | 必选   | 2    | string | 标识请求的随机数即可(1-36位)                        |
| systemtime          | 必选   | 2    | string | 请求消息发送的系统时间，精确到毫秒，共17位，格式：20121227180001165 |
| id                  | 必选   | 2    | string | sourceId或appId。临时凭证校验时，id必须为sourceid               |
| idtype              | 必选   | 2    | string | id类型：0：sourceid 1:appid。临时凭证校验时，idtype必须为0；融合sdk token校验时，如果使用开放平台申请的appid，就应该使用appid，否则校验会失败，如果是用杭研系统申请的sourceid和sourcekey，就应该传入sourceid（目前是对内接入全部使用sourceid进行token校验）                 |
| apptype             | 必选   | 2    | string | app类型：1:BOSS</br> 2:web</br> 3:wap</br> 4:pc客户端</br> 5:手机客户端 |
| userip              | 可选   | 2    | string | 客户端用户来源ip                                |
| message             | 可选   | 2    | string | 接入方预留参数，该参数会透传给通知接口，此参数需urlencode编码      |
| expandparams        | 可选   | 2    | json   | 业务扩展参数，json格式。                           |
| sign                | 必选   | 2    | string | 当**encryptionalgorithm≠"RSA"**时，sign = MD5（appid + version + msgid + systemtime + strictcheck + token + appkey)（注：“+”号为合并意思，不包含在被加密的字符串中），输出32位大写字母；</br>当**encryptionalgorithm="RSA"**，业务端RSA私钥签名（appid+token）, 服务端使用业务端提供的公钥验证签名（公钥可以在开发者社区配置）。 |
| encryptionalgorithm | 可选   |      |        | 开发者如果需要使用非对称加密算法时，填写“RSA”。（当该值不设置为“RSA”时，执行MD5签名校验） |
| body                | 必选   | 1    |        |                                          |
| token               | 必选   | 2    | string | 需要解析的凭证值。                                |

</br>

**响应参数**

| 参数名称                | 约束   | 层级   | 参数类型   | 说明                                       |
| ------------------- | ---- | ---- | ------ | ---------------------------------------- |
| header              | 必选   | 1    |        |                                          |
| version             | 必选   | 2    | string | 1.0有升级时调整                                |
| inresponseto        | 必选   | 2    | string | 对应的请求消息中的msgid                           |
| systemtime          | 必选   | 2    | string | 响应消息发送的系统时间，精确到毫秒，共17位，格式：20121227180001165 |
| resultcode          | 必选   | 2    | string | 返回码，返回码对应说明见附录A                          |
| body                | 必选   | 1    |        |                                          |
| openid              | 可选   | 2    | string | 成功时返回：用户身份唯一标识                           |
| msisdntype          | 可选   | 2    | string | 手机号码的归属运营商：</br> 0：中国移动</br>1：中国电信</br>2：中国联通</br> 99：未知的异网手机号码 |
| usessionid          | 可选   | 2    | string | 暂忽略                                      |
| passid              | 必选   | 2    | string | 用户统一账号的系统标识                              |
| loginidtype         | 可选   | 2    | string | 登录使用的用户标识：</br>0：手机号码</br>1：邮箱           |
| authtime            | 可选   | 2    | string | 统一认证平台认证用户的时间                            |
| lastactivetime      | 可选   | 2    | string | 暂无                                       |
| authtype            | 可选   | 2    | string | 认证方式                                     |
| relateToAndPassport | 可选   | 2    | string | 是否已经关联到统一账号，暂无用处                         |
| msisdn              | 可选   | 2    | string | 手机号 如果请求参数idtype为0，用sourcekey进行加密，如果idtype为1用appkey进行加密|

</br>

### 3.1.4 示例

**请求示例**

```
{
	"body": {
		"token": "84840100013602003A4D4456474F4442464E5463784F555244517A42434D54564540687474703A2F2F3132302E3139372E3233352E32373A383038302F72732F40303103000402C1FE800400123830303132303137313031373130333137330500206233363839343039333639373465393639643461386334363233306166643532060001300700023530FF0020321FDF1384B40B343B6D7ADDC69AF3D6C87DD82DCB98869F5A764A522E86067B"
	},
	"header": {
		"msgid": "c1eeff4eeb3b4dc1897f9013673d5eb3",
		"idtype": "1",
		"id": "300011012484",
		"apptype": "5",
		"strictcheck": "0",
		"version": "1.0",
		"sign": "4368018959d0e51b8aafd26193c2e8b4",
		"systemtime": "20180227193858817"
	}
}
```

**响应示例**

```
{
	"body": {
		"msisdntype": "1",
		"usessionid": "QzJCQ0VENzhDRjNFOTY0NDNC@http://120.197.235.27:8080/rs/@01",
		"passid": "1378119606",
		"loginidtype": "0",
		"authtime": "2018-02-27 20:13:53",
		"msisdn": "13800138000",
		"lastactivetime": "",
		"authtype": "WAPGW",
		"relateToAndPassport": "1"
	},
	"header": {
		"inresponseto": "807806355a9e47678a26e1735920733d",
		"resultcode": "103000",
		"systemtime": "20180227201357620",
		"version": "1.0"
	}
}
```

# 4. 返回码说明

##4.1. SDK返回码

使用SDK时，SDK会在认证结束后将结果回调给开发者，其中结果为JSONObject对象，其中resultCode为结果响应码，103000代表成功，其他为失败。成功时在根据token字段取出身份标识。失败时根据resultCode定位失败原因。

| 错误码    | 返回错误码的描述                                 |
| ------ | ---------------------------------------- |
| 103000 | 通用 成功返回                                  |
| 102102 | 网络异常                                     |
| 102223 | 数据解析异常                                   |
| 200002 | 手机未安装SIM卡                       |
| 200005 | 用户未授权 READ_PHONE_STATE                   |
| 200006 | 用户未授权 SEND_SMS                           |
| 200010 | 不支持的认证方式                          |
| 200013 | 短信上行发送短信失败                               |
| 200014 | 手机号码格式错误                                 |
| 200015 | 短信验证码格式错误                                |
| 200016 | 更新KS失败                                   |
| 200021 | 数据解析异常                                   |
| 200022 | 无网络状态                                    |
| 200023 | 请求超时                                     |
| 200024 | 数据网络切换失败                                 |
| 200025 | 未知错误，一般出现在线程捕获异常，请配合异常打印分析                |
| 200026 | loginType参数错误                                   |
| 200028 | 网络请求出错                                   |
| 200031 | 生成token失败                                |
| 200033 | 复用中间件获取Token失败                           |
| 200035 | 协商ks失败                                   |
| 200036 | 预取号失败                                   |
| 200037 | 获取不到openid                           |
| 200042 | 获取不到btid                                 |

</br>

##4.2. 获取用户信息接口返回码

| 平台错误码 | 描述                                       |
| ---------- | ------------------------------------------ |
| 103101     | 签名错误(token信息不对)                    |
| 103012     | 包签名错误                                 |
| 103103     | 用户不存在                                 |
| 103104     | 用户不支持这种登录方式                     |
| 103105     | 密码错误                                   |
| 103106     | 用户名错误                                 |
| 103107     | 已存在相同的随机数                         |
| 103108     | 短信验证码错误                             |
| 103109     | 短信验证码超时                             |
| 103111     | wap 网关IP错误                             |
| 103112     | 错误的请求                                 |
| 103113     | Token内容错误                              |
| 103114     | token验证 KS过期                           |
| 103115     | token验证 KS不存在                         |
| 103116     | token验证 sqn错误                          |
| 103117     | mac异常                                    |
| 103118     | sourceid不存在                             |
| 103119     | appid不存在                                |
| 103120     | clientauth不存在                           |
| 103121     | passid不存在                               |
| 103122     | btid不存在                                 |
| 103123     | redisinfo不存在                            |
| 103124     | ksnaf校验不一致                            |
| 103125     | 手机号格式错误                             |
| 103127     | 证书验证：版本过期                         |
| 103128     | gba:webservice error                       |
| 103129     | 获取短信验证码的msgtype异常                |
| 103130     | 新密码不能与当前密码相同                   |
| 103131     | 密码过于简单                               |
| 103132     | 用户注册失败                               |
| 103133     | sourceid不合法                             |
| 103134     | wap方式手机号码为空                        |
| 103135     | 昵称非法                                   |
| 103136     | 邮箱非法                                   |
| 103138     | appid已存在                                |
| 103139     | sourceid已存在                             |
| 103200     | 不需要更新ks                           |
| 103202     | 缓存用户不存在或者验证短信输入失败次数过多 |
| 103203     | 缓存用户不存在                             |
| 103204     | 缓存随机数不存                             |
| 103205     | 服务器异常                                 |
| 103207     | 发送短信失败                               |
| 103210     | 修改密码失败                               |
| 103211     | 其他错误                                   |
| 103212     | 校验密码失败                               |
| 103213     | 旧密码失败                                 |
| 103214     | 访问缓存或数据库错误                       |
| 103226     | sqn过小或过大                              |
| 103265     | 用户已存在                                 |
| 103270     | 随机校验凭证过期                           |
| 103271     | 随机校验凭证错误                           |
| 103272     | 随机校验凭证不存在                         |
| 103000     | 通用SUCCESS                                |
| 103901     | 短信验证码下发次数已达上限                 |
| 103303     | sip 用户未开户（获取应用密码）             |
| 103304     | sip 用户未开户（注销用户）                 |
| 103305     | sip 开户用户名错误                         |
| 103306     | sip 用户名不能为空（获取应用密码）         |
| 103307     | sip 用户名不能为空（注销用户）             |
| 103308     | sip 手机号不合法                           |
| 103309     | sip opertype 为空                          |
| 103310     | sip sourceid 不存在                        |
| 103311     | sip sourceid 不合法                        |
| 103312     | sip btid 不存在                            |
| 103313     | sip ks 不存在                              |
| 103314     | sip密码变更失败                            |
| 103315     | sip密码推送失败                            |
| 103399     | sip sys错误                                |
| 103400     | authorization 为空                         |
| 103401     | 签名消息为空                               |
| 103402     | 无效的 authWay                             |
| 103404     | 加密失败                                   |
| 103405     | 保存数据短信手机号为空                     |
| 103406     | 保存数据短信短信内容为空                   |
| 103407     | 此sourceId, appPackage, sign已注册         |
| 103408     | 此sourceId注册已达上限  99次               |
| 103409     | query 为空                                 |
| 103412     | 无效的请求                                 |
| 103414     | 参数效验异常                               |
| 103505     | 重放攻击                                   |
| 103511     | 源IP不合法                                 |
| 103810     | 校验失败，接口token版本不一致              |
| 103811     | token为空                                  |
| 103899     | aoi token 其他错误                         |
| 103901     | 短信验证码下发次数已达上限                 |
| 103902     | 凭证校验失败                               |
| 103903     | 调用webservice错误                         |
| 103904     | 配置不存在                                 |
| 103905     | 获取手机号码错误                           |
| 103906     | 平台迁移访问错误 - （访问旧地址）          |
| 103911     | 请求过于频繁                               |
| 103920     | 没有存在的版本更新                         |
| 103921     | 下载时间戳超时                             |
| 103922     | 自动升级文件没找到                         |
| 104001     | APPID和APPKEY已存在                        |
| 104201     | 凭证已失效或不存在                         |
| 104202     | 短信验证失败过多                           |
| 103412     | 无效的请求                                 |
| 103413     | 系统异常                                   |
| 103810     | 非新版本token，不予与校验                  |
| 105001     | 联通网关取号失败                           |
| 105002     | 移动网关取号失败                           |
| 105003     | 电信网关取号失败                           |
| 105004     | 短信上行ip检测不合法                       |
| 105005     | 短信上行发送信息为空                       |
| 105006     | 手机号码为空                               |
| 105007     | 手机号码格式错误                           |
| 105008     | 短信内容为空                               |
| 105009     | 解析失败                                   |
| 105010     | phonescrip失效或者非法                     |
| 105011     | getPhonescrip参数加密的私钥失效或者非法    |
| 105012     | 不支持电信取号                             |
| 105013     | 不支持联通取号                             |
| 105014     | 校验本机号码失败                           |
| 105015     | 校验有数三要素失败                         |
| 105018     | 用户权限不足                               |
| 105019     | 应用未授权                                 |
| 105017     | 短信随机数校验非法失败                     |
| 105016     | 达到使用次数限制                           |
