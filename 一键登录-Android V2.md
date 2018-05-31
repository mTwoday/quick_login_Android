# 1. 接入指南
sdk技术问题沟通QQ群：609994083</br>

**注意事项：**

1. 目前SDK支持中国移动2/3/4G、中国电信4G的取号能力，中国联通的取号能力暂未开放。
2. 由于运营商取号能力是通过数据网关实现，取号过程必须在数据流量打开的情况下才能进行（WiFi和数据流量同时打开时，SDK会强制切换到数据流量执行取号逻辑，将会消耗用户少量流量），当信号弱或者网络有干扰时，时延会高于平均值，取号成功率较低。
3. 本SDK同时提供一键登录和本机号码校验功能，开发者根据实际的需求调用对应方法和接口。

## 1.1. 接入流程

**1.申请appid和appkey**

根据《开发者接入流程文档》，前往中国移动开发者社区（dev.1008.cn)，按照文档要求创建开发者账号并申请appid和appkey。

**2.申请能力**

应用创建完成后，在能力配置页面上，勾选应用需要接入的能力类型，如一键登录，并配置应用的服务器出口IP地址。（如果在服务端需要用非对称加密方法对一些重要信息进行加密处理，请在能力配置页面填写RSA加密的公钥）

**3.添加appid白名单**

开发者在完成步骤1和步骤2后，将appid提供给移动认证产品经理，移动方将在2个工作日内将appid加入能力白名单；

**4.上线审核**

应用上线前，开发者需要将一键登录取号能力的场景所使用的授权页面（授权页面参考授权页面规范）提供给移动认证产品接口人，审核无误后可正式上线。

## 1.2. 开发流程

**第一步：下载SDK及相关文档**

请在开发者群或官网下载最新的SDK包

**第二步：搭建开发环境**

1. 在Eclipse中建立你的工程。 
2. 将`*.jar`拷贝到工程的libs目录下，如没有该目录，可新建。
3. 将sdk所需要的证书文件`clientCert.crt`、`serverPublicKey.pem`从提供的demo工程拷贝到项目`assets`目录下。

**第三步：开始使用移动认证SDK**

**[1] AndroidManifest.xml设置**

添加必要的权限支持: 

```java
<uses-permission android:name="android.permission.INTERNET" />
<uses-permission android:name="android.permission.READ_PHONE_STATE" />
<uses-permission android:name="android.permission.ACCESS_WIFI_STATE" />
<uses-permission android:name="android.permission.ACCESS_NETWORK_STATE" />
<uses-permission android:name="android.permission.CHANGE_NETWORK_STATE" />
<uses-permission android:name="android.permission.WRITE_SETTINGS"/>
<uses-permission android:name="android.permission.GET_TASKS"/>
```

注意：在调用`取号`或`授权登录`方法时，需提前申请`READ_PHONE_STATE`，否则会报错

**[2] 创建一个AuthnHelper实例。**

`AuthnHelper`是SDK的功能入口，所有的接口调用都得通过AuthnHelper进行调用。因此，调用SDK，首先需要创建一个AuthnHelper实例

**示例代码：**

```java
public void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    mContext = this;    
    ……
    mAuthnHelper = AuthnHelper.getInstance(mContext);
    }
```
**方法原型：**

```java
public static AuthnHelper getInstance(Context context)
```

**参数说明：**

| 参数    | 类型    | 说明                                               |
| ------- | ------- | -------------------------------------------------- |
| context | Context | 调用者的上下文环境，其中activity中this即可以代表。 |

</br>

**[3] 实现回调。**

所有的SDK接口调用，都会传入一个回调，用于接收SDK返回的调用结果。结果以`JsonObject`的形式传递，`TokenListener`的实现示例代码如下：

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
<div STYLE="page-break-after: always;"></div>

# 2. 一键登录功能

## 2.1. 准备工作

在接入一键登录功能之前，开发者必须先按照1.1 接入流程，在中国移动开发者社区注册开发者账号，创建一个包含移动认证能力的应用，获取响应的AppId和AppKey。并且在开发者社区中勾选一键登录能力，配置应用服务器出口ip地址。

##2.2. 流程说明 

移动认证一键登录允许开发者在用户同意授权后，在客户端侧获取接口调用凭证（token），第三方服务器携带token调用获取手机号码接口，实现获取当前授权登录用户的手机号码等信息。

整体流程为：

1. 开发者调用取号方法，取号成功后，SDK将缓存取号临时凭证scrip；
2. 开发者创建授权页activity；
3. 用户同意应用获取本机号码，调用授权登录方法，获取接口调用凭证token；
4. 携带token进行接口调用，获取用户的手机号码信息。

一键登录整体流程：

![](image/login_process.png)

##2.3. 获取临时取号凭证

开发者在需要使用一键登录的场景，调用SDK取号方法`getPhoneInfo`，认证服务端取号成功后，会返回一个临时取号凭证scrip缓存在SDK中，在授权过程中换取调用获取接口调用凭证token。

为了保证SDK取号成功率，开发者在取号前必须保证：

1. 应用必须提前获取用户手机`READ_PHONE_STATE`权限。
2. 运营商目前只支持中国移动2/3/4G和中国电信4G，开发者在调用取号方法前，可以预先判断当前用户的运营商类型和网络制式，仅针对SDK目前支持的运营商和网络制式使用一键登录功能。
3. 受运营商取号原理的限制，网关取号必须在数据流量打开的情况下进行，因此，用户如果关闭数据流量，将无法成功取号。（Android SDK 9.0+版本后，对于首次通过数据流量成功登录过的用户，二次+登录时，允许在数据流量关闭时取号）
4. 使用VPN代理，不管是路由器VPN还是手机VPN APP，都无法取号。

**请求示例代码**

```java
/***
判断和获取READ_PHONE_STATE权限逻辑
***/   

//创建AuthnHelper实例
public void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    mContext = this;    
    ……
    mAuthnHelper = AuthnHelper.getInstance(mContext);
    }

//实现取号回调
mListener = new TokenListener() {
    @Override
    public void onGetTokenComplete(JSONObject jObj) {
        …………	// 应用接收到回调后的处理逻辑
    }
};

//调用取号方法
mAuthnHelper.getPhoneInfo(APP_ID, APP_KEY, mListener);
```

**取号方法原型：**

```java
public void getPhoneInfo(final String appId, 
                         final String appKey, 
                         final TokenListener listener)
```

**参数说明：**

| 参数     | 类型          | 说明                                                         |
| :------- | :------------ | :----------------------------------------------------------- |
| appId    | String        | 应用的AppID，在开发者社区创建应用时获取                      |
| appkey   | String        | 应用密钥，在开发者社区创建应用时获取                         |
| listener | TokenListener | TokenListener为回调监听器，是一个java接口，需要调用者自己实现；TokenListener是接口中的认证登录token回调接口，OnGetTokenComplete是该接口中唯一的抽象方法，即void OnGetTokenComplete(JSONObject  jsonobj) |

**返回说明：**

OnGetTokenComplete的参数JSONObject，含义如下：

| 字段          | 类型   | 含义                                                  |
| ------------- | ------ | ----------------------------------------------------- |
| resultCode    | String | 接口返回码，“103000”为成功。具体返回码见4.1 SDK返回码 |
| desc          | String | 成功标识，true为成功。                                |
| securityPhone | String | 手机号码掩码，如“138XXXX0000”                         |

**关于临时取号凭证scrip：**

1. scrip有效期30天，scrip刷新时，有效期将重置为30天。
2. 开发者在2个场景会刷新scrip：1. 请求获取scrip时；2. 成功获取token时。
3. 首次取号或scrip失效时，需要手机终端的数据流量打开的状态下，才可以请求获取scrip。
4. 二次+登录时，scrip在有效期内，客户端即使在数据流量关闭状态下也能取号（网络要通）。
5. scrip保存在客户端私有目录下，使用keystore加密保存，在请求token时使用keystore解密scrip。
6. 部分机型不支持keystore加密，如果加密失败，scrip只保存在内存中。
7. 对于root的系统，scrip保存在内存中。
8. SDK提供删除临时凭证方法`delScrip`，提前让scrip失效。

## 2.4. 创建授权页

为了确保用户在登录并将手机号码信息提供给开发者的过程中的知情权，一键登录需要开发者提供授权页登录页面供用户授权确认。开发者在调用授权登录方法前，必须弹出授权页，明确告知用户当前操作会将用户的本机号码信息传递给应用。授权页面的设计、布局、生成、弹出和消失，由开发者自行处理，但必须遵守移动认证授权页面设计规范。

### 2.4.1. 调用逻辑

![](C:/Users/tonyl/Documents/GitHub/image/authPage.png)

### 2.4.2. 页面规范细则

1、页面必须包含登录/注册按钮，授权登录方法`loginAuth`必须绑定该按钮使用。

2、登录按钮文字描述必须包含“登录”或“注册”等文字，不得诱导用户授权。

3、页面需要提示应用获取到的是用户的本机号码，例如，可以在页面显示本机号码的掩码（securityphone），或者提示用户将使用“本机号码”作为账号登录或注册。

4、页面必须包含移动认证协议条款，其中：

​	条款名称：《中国移动认证服务条款》

​	条款页面地址：http://wap.cmpassport.com/resources/html/contract.html

5、应用在上线前需将满足上述1~4的授权页面（正式上线版的）截图提供给产品接口人审核。

6、应用后续升级时，如果授权页面有较大改动（针对1~4内容进行修改），需将改动的授权页面截图提供给产品接口人审核。

7、对于未遵照1~4设计要求，或通过技术手段故意屏蔽不弹出授权页面但获得调用接口凭证token的行为，移动有权将APP所有一键登录相关的能力下架，待整改后再重新上架服务。

</br>

## 2.5. 授权并获取接口凭证

开发者完成2.3和2.4的逻辑后，确保临时取号凭证scrip有效，并且用户点击同意授权登录后，开发者可以调用授权方法，获取接口凭证token。为了保证scrip的有效性，建议开发者在调用授权登录方法前，再次调用取号方法（如果scrip有效，再次调用取号方法的耗时非常少）。

在调用授权方法时，必须将2.4中开发者创建的授权页activity传入到本方法中，且需要保证activity正在内存运行中，否则取号失败。

**请求示例代码：**

```java
// 在调用取号方法getPhoneInfo并返回正确成功时，可以提前知道网络是否可取号并获取到手机号掩码；
// 在弹出授权页前不调用getPhoneInfo方法，无法获取手机号掩码，用户点击授权登录可能会失败


//创建AuthnHelper实例
public void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    mContext = this;    
    ……
    mAuthnHelper = AuthnHelper.getInstance(mContext);
    }

//实现回调
mListener = new TokenListener() {
    @Override
    public void onGetTokenComplete(JSONObject jObj) {
        …………	// 应用接收到回调后的处理逻辑
    }
};

/***
应用创建授权页activity并拉起页面
***/

//用户点击授权按钮时，调用loginAuth方法，传入应用定义的activity，获取token
    mAuthnHelper.loginAuth(activity, Constant.APP_ID, Constant.APP_KEY, mListener);
    ……
```

**授权方法原型：**

```java
public void loginAuth(final Activity activity, 
                      final String appId, 
                      final String appKey, 
                      final TokenListener listener)
```

**参数说明：**

| 参数     | 类型          | 说明                                                         |
| :------- | :------------ | :----------------------------------------------------------- |
| appId    | String        | 应用的AppID                                                  |
| appkey   | String        | 应用密钥                                                     |
| activity | Activity      | 授权页面，由开发者完成页面的设计布局。**当activity传值为null时，将不弹出授权页，登录方式为隐式登录** |
| listener | TokenListener | TokenListener为回调监听器，是一个java接口，需要调用者自己实现；TokenListener是接口中的认证登录token回调接口，OnGetTokenComplete是该接口中唯一的抽象方法，即void OnGetTokenComplete(JSONObject  jsonobj) |

**返回说明：**

TokenListener的参数JSONObject，含义如下：

| 字段        | 类型   | 含义                                                         |
| ----------- | ------ | ------------------------------------------------------------ |
| resultCode  | String | 接口返回码，“103000”为成功。具体响应码见4.1 SDK返回码        |
| authType    | String | 登录方式：1：WIFI下网关鉴权</br>2：网关鉴权</br>3：其他      |
| authTypeDes | String | 登录方式描述                                                 |
| token       | String | 成功时返回：临时凭证，token有效期2min，一次有效；同一用户（手机号）10分钟内获取token且未使用的数量不超过30个 |
| openId      | String | 成功时返回：用户身份唯一标识。（activity传入null时，不返回） |

## 2.6. 获取用户手机号码信息

开发者获取token后，需要将token传递到应用服务器，由应用服务器发起获取用户手机号接口的调用。

调用本接口，必须保证：

1. token在有效期内（2分钟）。
2. token还未使用过。
3. 应用服务器出口IP地址在开发者社区中配置正确。
4. 如果使用RSA加密，确保应用的公钥在开发者社区正确填写。

**接口说明：**

请求地址：https://www.cmpassport.com/unisdk/rsapi/loginTokenValidate

协议： HTTPS 

请求方法： POST+json,Content-type设置为application/json

**请求示例代码：**

```
{
    appid = 10000001;
    msgid = 34a5588136d6404784831609cdcdc633;
    sign = 2240b9213b9b8dccfe7f6257a21071cf;
    strictcheck = 0;
    systemtime = 20180529112443243;
    token = STsid00000015275642798949tUyg6KsmyEWKk005bCfuxUmCXeqeFRK;
    version = "2.0";
}

```

**参数说明**

| 参数                | 约束 | 说明                                                         |
| :------------------ | :--: | :----------------------------------------------------------- |
| version             | 必选 | 填2.0                                                        |
| msgid               | 必选 | 标识请求的随机数即可(1-36位)                                 |
| systemtime          | 必选 | 请求消息发送的系统时间，精确到毫秒，共17位，格式：20121227180001165 |
| strictcheck         | 必选 | 暂时填写"0"                                                  |
| appid               | 必选 | 业务在统一认证申请的应用id                                   |
| expandparams        | 可选 | 扩展参数                                                     |
| token               | 必选 | 需要解析的凭证值。                                           |
| sign                | 必选 | 当**encryptionalgorithm≠"RSA"**时，sign = MD5（appid + version + msgid + systemtime + strictcheck + token + appkey)（注：“+”号为合并意思，不包含在被加密的字符串中），输出32位大写字母；</br>当**encryptionalgorithm="RSA"**，业务端RSA私钥签名（appid+token）, 服务端使用业务端提供的公钥验证签名（公钥可以在开发者社区配置）。 |
| encryptionalgorithm | 可选 | 推荐使用。开发者如果需要使用非对称加密算法时，填写“RSA”。（当该值不设置为“RSA”时，执行MD5签名校验） |

</br>

**返回说明**

| 参数         | 类型   | 约束 | 说明                                                         |
| ------------ | ------ | ---- | ------------------------------------------------------------ |
| inresponseto | string | 必选 | 对应的请求消息中的msgid                                      |
| systemtime   | string | 必选 | 响应消息发送的系统时间，精确到毫秒，共17位，格式：20121227180001165 |
| resultcode   | string | 必选 | 返回码                                                       |
| msisdn       | string | 必选 | 表示手机号码，如果加密方式为RSA，应用需要用私钥进行解密      |

<div STYLE="page-break-after: always;"></div>

# 3. 本机号码校验功能

## 3.1. 准备工作

在接入本机号码校验功能之前，开发者必须先按照1.1 接入流程，在中国移动开发者社区注册开发者账号，创建一个包含移动认证能力的应用，获取响应的AppId和AppKey。并且在开发者社区中勾选本机号码校验能力，配置应用服务器出口ip地址。

## 3.2. 流程说明

移动认证本机号码校验用于校验用户当前输入的手机号码是否为本机号码。

整体流程为：

1. 开发者调用取号方法，取号成功后，SDK将缓存取号临时凭证scrip；
2. 调用授权登录方法，获取接口凭证token；
3. 开发者使用本机号码校验的交互页面；
4. 携带token和手机号码信息进行接口调用，获取手机号码校验结果。

本机号码校验整体流程：

![](image/verify_process.png)

## 3.3. 获取临时取号凭证

开发者调用SDK取号方法`getPhoneInfo`，认证服务端取号成功后，会返回一个临时取号凭证scrip缓存在SDK中，在授权过程中换取调用获取接口调用凭证token。

为了保证SDK取号成功率，开发者在取号前必须保证：

1. 应用必须提前获取用户手机`READ_PHONE_STATE`权限。
2. 运营商目前只支持中国移动2/3/4G和中国电信4G，开发者在调用取号方法前，可以预先判断当前用户的运营商类型和网络制式，仅针对SDK目前支持的运营商和网络制式使用一键登录功能。
3. 受运营商取号原理的限制，网关取号必须在数据流量打开的情况下进行，因此，用户如果关闭数据流量，将无法成功取号。（Android SDK 9.0+版本后，对于首次通过数据流量成功登录过的用户，二次+登录时，允许在数据流量关闭时取号）
4. 使用VPN代理，不管是路由器VPN还是手机VPN APP，都无法取号。

**请求示例代码**

```java
/***
判断和获取READ_PHONE_STATE权限逻辑
***/   

//创建AuthnHelper实例
public void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    mContext = this;    
    ……
    mAuthnHelper = AuthnHelper.getInstance(mContext);
    }

//实现取号回调
mListener = new TokenListener() {
    @Override
    public void onGetTokenComplete(JSONObject jObj) {
        …………	// 应用接收到回调后的处理逻辑
    }
};

//调用取号方法
mAuthnHelper.getPhoneInfo(APP_ID, APP_KEY, mListener);
```

**取号方法原型：**

```java
public void getPhoneInfo(final String appId, 
                         final String appKey, 
                         final TokenListener listener)
```

**参数说明：**

| 参数     | 类型          | 说明                                                         |
| :------- | :------------ | :----------------------------------------------------------- |
| appId    | String        | 应用的AppID，在开发者社区创建应用时获取                      |
| appkey   | String        | 应用密钥，在开发者社区创建应用时获取                         |
| listener | TokenListener | TokenListener为回调监听器，是一个java接口，需要调用者自己实现；TokenListener是接口中的认证登录token回调接口，OnGetTokenComplete是该接口中唯一的抽象方法，即void OnGetTokenComplete(JSONObject  jsonobj) |

**返回说明：**

OnGetTokenComplete的参数JSONObject，含义如下：

| 字段          | 类型   | 含义                                                  |
| ------------- | ------ | ----------------------------------------------------- |
| resultCode    | String | 接口返回码，“103000”为成功。具体返回码见4.1 SDK返回码 |
| desc          | String | 成功标识，true为成功。                                |
| securityPhone | String | 手机号码掩码，如“138XXXX0000”                         |

**关于临时取号凭证scrip：**

1. scrip有效期30天，scrip刷新时，有效期将重置为30天。
2. 开发者在2个场景会刷新scrip：1. 请求获取scrip时；2. 成功获取token时。
3. 首次取号或scrip失效时，需要手机终端的数据流量打开的状态下，才可以请求获取scrip。
4. 二次+登录时，scrip在有效期内，客户端即使在数据流量关闭状态下也能取号（网络要通）。
5. scrip保存在客户端私有目录下，使用keystore加密保存，在请求token时使用keystore解密scrip。
6. 部分机型不支持keystore加密，如果加密失败，scrip只保存在内存中。
7. 对于root的系统，scrip保存在内存中。
8. SDK提供删除临时凭证方法`delScrip`，提前让scrip失效。

## 3.4. 授权并获取接口凭证

开发者完成3.3逻辑后，确保临时取号凭证scrip有效，调用授权方法，获取接口凭证token。

在调用授权方法时，将activity对象传入值设为null

**请求示例代码：**

```java
//创建AuthnHelper实例
public void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    mContext = this;    
    ……
    mAuthnHelper = AuthnHelper.getInstance(mContext);
}

//实现回调
mListener = new TokenListener() {
    @Override
    public void onGetTokenComplete(JSONObject jObj) {
        …………		// 应用接收到回调后的处理逻辑
    }
};

    
//调用授权登录，activity传null    
    mAuthnHelper.loginAuth(null, Constant.APP_ID, Constant.APP_KEY, mListener);
    ……
```

**授权方法原型：**

```java
public void loginAuth(final Activity activity, 
                      final String appId, 
                      final String appKey, 
                      final TokenListener listener)
```

**参数说明：**

| 参数     | 类型          | 说明                                                         |
| :------- | :------------ | :----------------------------------------------------------- |
| appId    | String        | 应用的AppID                                                  |
| appkey   | String        | 应用密钥                                                     |
| activity | Activity      | 授权页面，由开发者完成页面的设计布局。**当activity传值为null时，将不弹出授权页，登录方式为隐式登录** |
| listener | TokenListener | TokenListener为回调监听器，是一个java接口，需要调用者自己实现；TokenListener是接口中的认证登录token回调接口，OnGetTokenComplete是该接口中唯一的抽象方法，即void OnGetTokenComplete(JSONObject  jsonobj) |

**返回说明：**

TokenListener的参数JSONObject，含义如下：

| 字段        | 类型   | 含义                                                         |
| ----------- | ------ | ------------------------------------------------------------ |
| resultCode  | String | 接口返回码，“103000”为成功。                                 |
| authType    | String | 登录方式：1：WIFI下网关鉴权</br>2：网关鉴权</br>3：其他      |
| authTypeDes | String | 登录方式描述                                                 |
| token       | String | 成功时返回：临时凭证，token有效期2min，一次有效；同一用户（手机号）10分钟内获取token且未使用的数量不超过30个 |
| openId      | String | 成功时返回：用户身份唯一标识。（activity传入null时，不返回） |

## 3.5. 客户端交互页面设计

开发者在本机号码校验的使用场景（页面），引导提示用户输入本机号码，用户确认输入后，将手机号码和3.4中获取的接口凭证token传回应用服务器。

## 3.6. 本机号码校验

开发者获取token后，需要将token传递到应用服务器，由应用服务器发起本机号码校验接口的调用。

调用本接口，必须保证：

1. token在有效期内（2分钟）。
2. token还未使用过。
3. 应用服务器出口IP地址在开发者社区中配置正确。

对于本机号码校验，需要注意：

1. 本产品属于收费业务，开发者未签订服务合同前，每天总调用次数有限，详情可咨询商务。
2. 签订合同后，将不在提供每天免费的测试次数。

**接口说明：**

请求地址： https://www.cmpassport.com/openapi/rs/tokenValidate

协议： HTTPS

请求方法： POST+json,Content-type设置为application/json

**请求示例代码：**

```
{
    body =     {
        openType = 1;
        phoneNum =0A2050AC434A32DE684745C829B3DE570590683FAA1C9374016EF60390E6CE76;
        requesterType = 0;
        sign = 87FCAC97BCF4B0B0D741FE1A85E4DF9603FD301CB3D7100BFB5763CCF61A1488;
        token = STsid0000001517194515125yghlPllAetv4YXx0v6vW2grV1v0votvD;
    };
    header =     {
        appId = 3000******76;
        msgId = f11585580266414fbde9f755451fb7a7;
        timestamp = 20180129105523519;
        version = "1.0";
    };
}
```

**参数说明：**

| 参数          | 层级  | 约束                         | 说明                                                         |
| ------------- | ----- | ---------------------------- | ------------------------------------------------------------ |
| **header**    | **1** | 必选                         |                                                              |
| version       | 2     | 必选                         | 版本号,初始版本号1.0,有升级后续调整                          |
| msgId         | 2     | 必选                         | 使用UUID标识请求的唯一性                                     |
| timestamp     | 2     | 必选                         | 请求消息发送的系统时间，精确到毫秒，共17位，格式：20121227180001165 |
| appId         | 2     | 必选                         | 应用ID                                                       |
| **body**      | **1** | 必选                         |                                                              |
| openType      | 2     | 否，requestertype字段为0时是 | 运营商类型：</br>1:移动;</br>2:联通;</br>3:电信;</br>0:未知  |
| requesterType | 2     | 是                           | 请求方类型：</br>0:APP；</br>1:WAP                           |
| message       | 2     | 否                           | 接入方预留参数，该参数会透传给通知接口，此参数需urlencode编码 |
| expandParams  | 2     | 否                           | 扩展参数格式：param1=value1\|param2=value2  方式传递，参数以竖线 \| 间隔方式传递，此参数需urlencode编码。 |
| phoneNum      | 2     | 是                           | 待校验的手机号码的64位sha256值，字母大写。（手机号码 + appKey + timestamp， “+”号为合并意思）（注：建议开发者对用户输入的手机号码的格式进行校验，增加校验通过的概率） |
| token         | 2     | 是                           | 身份标识，字符串形式的token                                  |
| sign          | 2     | 是                           | 签名，HMACSHA256( appId +     msgId + phonNum + timestamp + token + version)，输出64位大写字母 （注：“+”号为合并意思，不包含在被加密的字符串中,appkey为秘钥, 参数名做自然排序（Java是用TreeMap进行的自然排序）） |

**返回说明：**

| 参数         | 层级  | 类型   | 约束 | 说明                                                         |
| ------------ | ----- | :----- | :--- | :----------------------------------------------------------- |
| **header**   | **1** |        | 必选 |                                                              |
| msgId        | 2     | string | 必选 | 对应的请求消息中的msgid                                      |
| timestamp    | 2     | string | 必选 | 响应消息发送的系统时间，精确到毫秒，共17位，格式：20121227180001165 |
| appId        | 2     | string | 必选 | 应用ID                                                       |
| resultCode   | 2     | string | 必选 | 平台返回码                                                   |
| **body**     | **1** |        | 必选 |                                                              |
| resultDesc   | 2     | String | 必选 | 平台返回码                                                   |
| message      | 2     | String | 否   | 接入方预留参数，该参数会透传给通知接口，此参数需urlencode编码 |
| expandParams | 2     | String | 否   | 扩展参数格式：param1=value1\|param2=value2  方式传递，参数以竖线 \| 间隔方式传递，此参数需urlencode编码。 |

<div STYLE="page-break-after: always;"></div>

# 4. SDK方法说明

## 4.1. 获取管理类的实例对象

### 4.1.1. 方法描述

获取管理类的实例对象

</br>

**原型**

```java
public static AuthnHelper getInstance(Context context)
```

</br>

### 4.1.2. 参数说明

| 参数      | 类型      | 说明                              |
| ------- | ------- | ------------------------------- |
| context | Context | 调用者的上下文环境，其中activity中this即可以代表。 |

</br>

## 4.2. 取号

### 4.2.1. 方法描述

**功能**

本方法用于发起取号请求，并返回用户当前网络环境是否具备取号条件的结果。SDK将在后台完成网络判断、数据网络切换等内部操作并向网关请求申请获取用户本机号码。取号请求成功后，开发者就可以调用并弹出由开发者自定义布局的授权页面。**注意：取号请求前，开发者需提前申请`READ_PHONE_STATE`权限，否则取号会失败！**

**调用建议**

建议开发者在应用开启时调用本方法，有助于提升登录的成功率和用户登录体验。

</br>

### 4.2.2. 参数说明

**请求参数**

| 参数       | 类型            | 说明                                       |
| :------- | :------------ | :--------------------------------------- |
| appId    | String        | 应用的AppID                                 |
| appkey   | String        | 应用密钥                                     |
| listener | TokenListener | TokenListener为回调监听器，是一个java接口，需要调用者自己实现；TokenListener是接口中的认证登录token回调接口，OnGetTokenComplete是该接口中唯一的抽象方法，即void OnGetTokenComplete(JSONObject  jsonobj) |

</br>

**响应参数**

OnGetTokenComplete的参数JSONObject，含义如下：

| 字段          | 类型   | 含义                          |
| ------------- | ------ | ----------------------------- |
| resultCode    | String | 接口返回码，“103000”为成功。  |
| desc          | String | 成功标识，true为成功。        |
| securityPhone | String | 手机号码掩码，如“138XXXX0000” |

</br>

### 4.2.3. 示例

**请求示例代码**

```java
/***
判断和获取READ_PHONE_STATE权限逻辑
***/   

//创建AuthnHelper实例
public void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    mContext = this;    
    ……
    mAuthnHelper = AuthnHelper.getInstance(mContext);
    }

//实现取号回调
mListener = new TokenListener() {
    @Override
    public void onGetTokenComplete(JSONObject jObj) {
        …………	// 应用接收到回调后的处理逻辑
    }
};

//调用取号方法
mAuthnHelper.getPhoneInfo(APP_ID, APP_KEY, mListener);
```

**响应示例代码**

```
{
	"resultCode":"103000",
	"desc":"true",
	"securityPhone:"138XXXX0000"
}
```

## 4.3. 授权

### 4.3.1. 方法描述

**功能**

在应用弹出授权页的情况下，调用本方法可以成功获取取号凭证token。如果需要**获取用户完整的手机号码**，调用该方法时，需要将正在运行的授权页面activity传入并获取相对应的token；如果需要做**本机号码校验**，调用该方法时，activity参数传null即可。

**注：**若要实现一键登录，必须在授权页的activity中调用本方法，否则会报错

</br>

**原型**

```java
public void loginAuth(final Activity activity, 
                      final String appId, 
                      final String appKey, 
                      final TokenListener listener)
```

</br>

### 4.3.2. 参数说明

**请求参数**

| 参数     | 类型          | 说明                                                         |
| :------- | :------------ | :----------------------------------------------------------- |
| appId    | String        | 应用的AppID                                                  |
| appkey   | String        | 应用密钥                                                     |
| activity | Activity      | 授权页面，由开发者完成页面的设计布局。**当activity传值为null时，将不弹出授权页，登录方式为隐式登录** |
| listener | TokenListener | TokenListener为回调监听器，是一个java接口，需要调用者自己实现；TokenListener是接口中的认证登录token回调接口，OnGetTokenComplete是该接口中唯一的抽象方法，即void OnGetTokenComplete(JSONObject  jsonobj) |

</br>

**响应参数**

TokenListener的参数JSONObject，含义如下：

| 字段        | 类型   | 含义                                                         |
| ----------- | ------ | ------------------------------------------------------------ |
| resultCode  | String | 接口返回码，“103000”为成功。具体响应码见4.1 SDK返回码        |
| authType    | String | 登录方式：1：WIFI下网关鉴权</br>2：网关鉴权</br>3：其他      |
| authTypeDes | String | 登录方式描述                                                 |
| token       | String | 成功时返回：临时凭证，token有效期2min，一次有效；同一用户（手机号）10分钟内获取token且未使用的数量不超过30个 |
| openId      | String | 成功时返回：用户身份唯一标识。（activity传入null时，不返回） |

</br>

### 4.3.3. 示例

**请求示例代码**

```java
// 在调用取号方法getPhoneInfo并返回正确成功时，可以提前知道网络是否可取号并获取到手机号掩码；
// 在弹出授权页前不调用getPhoneInfo方法，无法获取手机号掩码，用户点击授权登录可能会失败


//创建AuthnHelper实例
public void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    mContext = this;    
    ……
    mAuthnHelper = AuthnHelper.getInstance(mContext);
    }

//实现回调
mListener = new TokenListener() {
    @Override
    public void onGetTokenComplete(JSONObject jObj) {
        …………	// 应用接收到回调后的处理逻辑
    }
};

/***
1. 应用构造activity页面逻辑
2. 应用弹出授权页面activity逻辑
***/

//用户点击授权按钮时，调用loginAuth方法，获取token
private void Login() {
    
/***
（调用loginAuth方法前，如果还未获取到READ_PHONE_STATE权限，需提前申请）
判断和获取READ_PHONE_STATE权限逻辑
***/   
        
//调用授权登录，传入应用定义的activity（授权页面） 
    mAuthnHelper.loginAuth(activity, Constant.APP_ID, Constant.APP_KEY, mListener);
    ……
}
```

**响应示例代码**

```
{
	"resultCode": "103000",
	"authType": "0",
	"authTypeDes": "其他",
	"token": "STsid0000001526349949452XU2gWQFcgWnoMmwwbK7ijolJcsQsH1Ws",
	“openId":""
}
```

</br>

## 4.5. 设置取号超时

###4.5.1. 方法描述

设置取号超时时间，默认为8秒，适用于取号和隐式登录阶段。建议配置值2~4秒

**原型**

```java
public void setTimeOut(long timeOut)
```

###4.5.2. 参数说明

**请求参数**

| 参数    | 类型 | 说明                       |
| ------- | ---- | -------------------------- |
| timeOut | long | 设置超时时间（单位：毫秒） |

**响应参数**

无



## 4.6. 获取网络状态和运营商类型

### 4.6.1. 方法描述

本方法用于获取用户当前的网络环境和运营商

**原型**

```java
public void getNetworkType(Context context,
                           TokenListener listener)
```

### 4.6.2. 参数说明

**请求参数**

| 参数     | 类型          | 说明       |
| -------- | ------------- | ---------- |
| context  | Context       | 上下文对象 |
| listener | TokenListener | 回调       |

**响应参数**

TokenListener的参数JSONObject，含义如下：

| 参数     | 类型   | 说明                                                         |
| -------- | ------ | ------------------------------------------------------------ |
| operator | String | 运营商类型：</br>1.移动流量；</br>2.联通流量网络；</br>3.电信流量网络 |
| netType  | String | 网络类型：</br>0.未知；</br>1.流量；</br>2.wifi；</br>3.数据流量+wifi |

### 4.6.3. 示例

```java
mAuthnHelper.getNetworkType (context, new TokenListener() {
    @Override
	public void onGetTokenComplete(JSONObject jsonobj) {
	        // do something
    }
}
);
```



## 4.7. 删除临时取号凭证

### 4.7.1. 方法描述

开发者调用取号方法`getPhoneInfo`成功后，SDK将取号的一个临时凭证缓存在本地（凭证使用keystore加密），有效期30天。开发者可以根据产品的场景、风险控制级别，确定何时删除凭证。

SDK将在2个地方会更新生成新的凭证，一个是开发者调用取号成功后，会缓存凭证；一个是开发者调用授权方法成功后，会缓存凭证。本地若保存临时凭证，将允许应用在纯wifi下成功获取token，取号时间将缩短，成功率会提升。

**特殊情况：**

基于安全考虑，在两种情况下，凭证将不保存在本地，而只会保存在正在运行的内存中：

1. keystore加密失败；
2. 手机获取了root权限。

这2种情况下，凭证的有效时间将与应用程序内存的生命周期有关。

**原型**

```java
public void delScrip()
```

### 4.7.2. 参数说明

**请求参数**

无

**响应参数**

无

<div STYLE="page-break-after: always;"></div>

# 5. 服务端接口说明

## 5.1. 获取手机号码接口

业务平台或服务端携带用户授权成功后的token来调用统一认证服务端获取用户手机号码等信息。

### 5.1.1. 业务流程

SDK在获取token过程中，用户手机必须在打开数据网络情况下才能获取成功，纯wifi环境下会自动跳转到SDK的短信验证码页面（如果有配置）或者返回错误码

![](image/19.png)

### 5.1.2. 接口说明

**请求地址：**https://www.cmpassport.com/unisdk/rsapi/loginTokenValidate

**协议：** HTTPS 

**请求方法：** POST+json,Content-type设置为application/json

**注意：开发者需到开发者社区填写服务端出口IP地址后才能正常使用**

</br>

### 5.1.3. 参数说明

**请求参数**

| 参数                  |   类型   |  约束  | 说明                                       |
| :------------------ | :----: | :--: | :--------------------------------------- |
| version             | string |  必选  | 填2.0                                     |
| msgid               | string |  必选  | 标识请求的随机数即可(1-36位)                        |
| systemtime          | string |  必选  | 请求消息发送的系统时间，精确到毫秒，共17位，格式：20121227180001165 |
| strictcheck         | string |  必选  | 暂时填写"0"                                  |
| appid               | string |  必选  | 业务在统一认证申请的应用id                           |
| expandparams        | string |  可选  | 扩展参数                                     |
| token               | string |  必选  | 需要解析的凭证值。                                |
| sign                | string |  必选  | 当**encryptionalgorithm≠"RSA"**时，sign = MD5（appid + version + msgid + systemtime + strictcheck + token + appkey)（注：“+”号为合并意思，不包含在被加密的字符串中），输出32位大写字母；</br>当**encryptionalgorithm="RSA"**，业务端RSA私钥签名（appid+token）, 服务端使用业务端提供的公钥验证签名（公钥可以在开发者社区配置）。 |
| encryptionalgorithm | string |  可选  | 开发者如果需要使用非对称加密算法时，填写“RSA”。（当该值不设置为“RSA”时，执行MD5签名校验） |

</br>

**响应参数**

| 参数         | 类型   | 约束 | 说明                                                         |
| ------------ | ------ | ---- | ------------------------------------------------------------ |
| inresponseto | string | 必选 | 对应的请求消息中的msgid                                      |
| systemtime   | string | 必选 | 响应消息发送的系统时间，精确到毫秒，共17位，格式：20121227180001165 |
| resultcode   | string | 必选 | 返回码                                                       |
| msisdn       | string | 必选 | 表示手机号码，如果加密方式为RSA，应用需要用私钥进行解密      |

</br>

### 3.1.3. 示例

**请求示例**

```
{
    appid = 3000******76; 
    msgid = 335e06a28f064b999d6a25e403991e4c;
    sign = 213EF8D0CC71548945A83166575DFA68;
    strictcheck = 0;
    systemtime = 20180129112955435;
    token = STsid0000001517196594066OHmZvPMBwn2MkFxwvWkV12JixwuZuyDU;
    version = "2.0";
}
```

**响应示例**

```
{
    inresponseto = 335e06a28f064b999d6a25e403991e4c;
    msisdn = 14700000000;
    resultCode = 103000;
    systemtime = 20180129112955477;
}
```

<div STYLE="page-break-after: always;"></div>

## 5.2. 本机号码校验接口

校验用户输入的号码是否本机号码。
应用将手机号码传给统一认证SDK，统一认证SDK向统一认证服务端发起本机号码校验请求，统一认证服务端通过网关或者短信上行获取本机手机号码和第三方应用传输的手机号码进行校验，返回校验结果。</br>

### 5.2.1. 业务流程

SDK在获取token过程中，用户手机必须在打开数据网络情况下才能获取成功，纯wifi环境下会自动跳转到SDK的短信验证码页面（如果有配置）或者返回错误码。**注：本业务目前仅支持中国移动号码，建议开发者在调用SDK的获取token方法前，判断当前用户手机运营商**

![1](image\1.1.png)

</br>

### 5.2.2. 接口说明

**调用次数说明：**本产品属于收费业务，开发者未签订服务合同前，每天总调用次数有限，详情可咨询商务。

**请求地址：** https://www.cmpassport.com/openapi/rs/tokenValidate

**协议：** HTTPS

**请求方法：** POST+json,Content-type设置为application/json

**回调地址：**请参考开发者接入流程文档

</br>

### 5.2.3.  参数说明

**请求参数**

| 参数            | 类型     | 层级    | 约束                    | 说明                                       |
| ------------- | ------ | ----- | --------------------- | ---------------------------------------- |
| **header**    |        | **1** | 必选                    |                                          |
| version       | string | 2     | 必选                    | 版本号,初始版本号1.0,有升级后续调整                     |
| msgId         | string | 2     | 必选                    | 使用UUID标识请求的唯一性                           |
| timestamp     | string | 2     | 必选                    | 请求消息发送的系统时间，精确到毫秒，共17位，格式：20121227180001165 |
| appId         | string | 2     | 必选                    | 应用ID                                     |
| **body**      |        | **1** | 必选                    |                                          |
| openType      | String | 2     | 否，requestertype字段为0时是 | 运营商类型：</br>1:移动;</br>2:联通;</br>3:电信;</br>0:未知 |
| requesterType | String | 2     | 是                     | 请求方类型：</br>0:APP；</br>1:WAP              |
| message       | String | 2     | 否                     | 接入方预留参数，该参数会透传给通知接口，此参数需urlencode编码      |
| expandParams  | String | 2     | 否                     | 扩展参数格式：param1=value1\|param2=value2  方式传递，参数以竖线 \| 间隔方式传递，此参数需urlencode编码。 |
| phoneNum      | String | 2     | 是                     | 待校验的手机号码的64位sha256值，字母大写。（手机号码 + appKey + timestamp， “+”号为合并意思）（注：建议开发者对用户输入的手机号码的格式进行校验，增加校验通过的概率） |
| token         | String | 2     | 是                     | 身份标识，字符串形式的token                         |
| sign          | String | 2     | 是                     | 签名，HMACSHA256( appId +     msgId + phonNum + timestamp + token + version)，输出64位大写字母 （注：“+”号为合并意思，不包含在被加密的字符串中,appkey为秘钥, 参数名做自然排序（Java是用TreeMap进行的自然排序）） |
|               |        |       |                       |                                          |

**响应参数**

| 参数           | 层级    | 类型     | 约束   | 说明                                       |
| ------------ | ----- | :----- | :--- | :--------------------------------------- |
| **header**   | **1** |        | 必选   |                                          |
| msgId        | 2     | string | 必选   | 对应的请求消息中的msgid                           |
| timestamp    | 2     | string | 必选   | 响应消息发送的系统时间，精确到毫秒，共17位，格式：20121227180001165 |
| appId        | 2     | string | 必选   | 应用ID                                     |
| resultCode   | 2     | string | 必选   | 规则参见4.3平台返回码                             |
| **body**     | **1** |        | 必选   |                                          |
| resultDesc   | 2     | String | 必选   | 描述参见4.3平台返回码                             |
| message      | 2     | String | 否    | 接入方预留参数，该参数会透传给通知接口，此参数需urlencode编码      |
| expandParams | 2     | String | 否    | 扩展参数格式：param1=value1\|param2=value2  方式传递，参数以竖线 \| 间隔方式传递，此参数需urlencode编码。 |

</br>

### 5.2.4. 示例

**请求示例**

```
{
    body =     {
        openType = 1;
        phoneNum =0A2050AC434A32DE684745C829B3DE570590683FAA1C9374016EF60390E6CE76;
        requesterType = 0;
        sign = 87FCAC97BCF4B0B0D741FE1A85E4DF9603FD301CB3D7100BFB5763CCF61A1488;
        token = STsid0000001517194515125yghlPllAetv4YXx0v6vW2grV1v0votvD;
    };
    header =     {
        appId = 3000******76;
        msgId = f11585580266414fbde9f755451fb7a7;
        timestamp = 20180129105523519;
        version = "1.0";
    };
}
```



**响应示例**

```
{
    body =     {
        message = "";
        resultDesc = "\U662f\U672c\U673a\U53f7\U7801";
    };
    header =     {
        appId = 3000******76;
        msgId = f11585580266414fbde9f755451fb7a7;
        resultCode = 000;
        timestamp = 20180129105523701;
    };
}
```

<div STYLE="page-break-after: always;"></div>

# 6. 返回码说明

##6.1. SDK返回码

使用SDK时，SDK会在认证结束后将结果回调给开发者，其中结果为JSONObject对象，其中resultCode为结果响应码，103000代表成功，其他为失败。成功时在根据token字段取出身份标识。失败时根据resultCode定位失败原因。

| 返回码 | 返回码描述                                     |
| ------ | ---------------------------------------------- |
| 103000 | 成功                                           |
| 102101 | 无网络                                         |
| 102102 | 网络异常                                       |
| 102103 | 未开启数据网络                                 |
| 102121 | 用户取消登录                                   |
| 102223 | 数据解析异常                                   |
| 102203 | 输入参数错误                                   |
| 102507 | 请求超时，预取号、buffer页取号、登录时请求超时 |
| 102508 | 数据网络切换失败                               |
| 200002 | 手机未安装sim卡                                |
| 200005 | 用户未授权（READ_PHONE_STATE）                 |
| 200006 | 用户未授权（SEND_SMS）                         |
| 200007 | authType仅使用短信验证码认证                   |
| 200008 | 1. authType参数为空；2. authType参数不合法；   |
| 200009 | 应用合法性校验失败（包名包签名未填写正确）     |
| 200010 | 预取号时imsi获取失败或者没有sim卡              |
| 200012 | 	取号失败，跳短信验证码登录|
| 200013 | 	短信上行发送短信失败|
| 200014| 	手机号码格式错误|
| 200015| 	短信验证码格式错误|
| 200016| 	更新KS失败|
| 200017	|  非移动卡不支持短信上行|
| 200018 | 不支持网关登录|
| 200019 | 不支持短信验证码登录（authtype没有短信验证码登录方式）|
| 200020| 	用户取消认证|
| 200021| 	数据解析异常（服务器异常可重新尝试）|
| 200022| 	无网络状态|
| 200023| 	请求超时|
| 200024| 	数据网络切换失败|
| 200025	| 未知错误一般出现在线程捕获异常，请配合异常打印分析|
| 200026| 	输入参数错误|
| 200027	| 预取号只开启WIFI（预取号需要数据网络）|
| 200028	| 网络请求出错（根据日志分析）|
| 200029| 请求出错,上次请求未完成|
| 200030| 没有初始化参数|
| 200031| 生成token失败|
| 200032| 	KS缓存不存在|
| 200033	|  复用中间件获取Token失败|
| 200034	| 预取号token失效|
| 200035	 | 协商ks失败|
| 200036	| 预取号失败|
| 200037	| 获取不到openid|
| 200038	| 电信重定向失败|
| 200039	| 电信取号接口返回失败|
| 200040	| UI资源加载异常|
| 200042	| 授权页弹出异常 |
</br>

##6.2. 获取用户信息接口返回码

| 返回码    | 返回码描述                           |
| ------ | ------------------------------- |
| 103000 | 成功                              |
| 103101 | 签名错误                            |
| 103103 | 用户不存在                           |
| 103104 | 用户不支持这种登录方式                     |
| 103105 | 密码错误                            |
| 103106 | 用户名错误                           |
| 103107 | 已存在相同的随机数                       |
| 103108 | 短信验证码错误                         |
| 103109 | 短信验证码超时                         |
| 103111 | wap  网关IP错误                     |
| 103112 | 错误的请求                           |
| 103113 | Token内容错误                       |
| 103114 | token验证KS过期                     |
| 103115 | token验证KS不存在                    |
| 103116 | token验证sqn错误                    |
| 103117 | mac异常                           |
| 103118 | sourceid不存在                     |
| 103119 | appid不存在                        |
| 103120 | clientauth不存在                   |
| 103121 | passid不存在                       |
| 103122 | btid不存在                         |
| 103123 | redisinfo不存在                    |
| 103124 | ksnaf校验不一致                      |
| 103125 | 手机号格式错误                         |
| 103127 | 证书验证：版本过期                       |
| 103128 | gba:webservice  error           |
| 103129 | 获取短信验证码的msgtype异常               |
| 103130 | 新密码不能与当前密码相同                    |
| 103131 | 密码过于简单                          |
| 103132 | 用户注册失败                          |
| 103133 | sourceid不合法                     |
| 103134 | wap方式手机号码为空                     |
| 103135 | 昵称非法                            |
| 103136 | 邮箱非法                            |
| 103138 | appid已存在                        |
| 103139 | sourceid已存在                     |
| 103200 | 不需要更新ks错误                       |
| 103202 | 缓存用户不存在或者验证短信输入失败次数过多           |
| 103203 | 缓存用户不存在                         |
| 103204 | 缓存随机数不存                         |
| 103205 | 服务器异常                           |
| 103207 | 发送短信失败                          |
| 103210 | 修改密码失败                          |
| 103211 | 其他错误                            |
| 103212 | 校验密码失败                          |
| 103213 | 旧密码失败                           |
| 103214 | 访问缓存或数据库错误                      |
| 103226 | sqn过小或过大                        |
| 103265 | 用户已存在                           |
| 103270 | 随机校验凭证过期                        |
| 103271 | 随机校验凭证错误                        |
| 103272 | 随机校验凭证不存在                       |
| 103303 | sip  用户未开户（获取应用密码）              |
| 103304 | sip  用户未开户（注销用户）                |
| 103305 | sip  开户用户名错误                    |
| 103306 | sip  用户名不能为空（获取应用密码）            |
| 103307 | sip  用户名不能为空（注销用户）              |
| 103308 | sip  手机号不合法                     |
| 103309 | sip  opertype 为空                |
| 103310 | sip  sourceid 不存在               |
| 103311 | sip  sourceid 不合法               |
| 103312 | sip  btid 不存在                   |
| 103313 | sip  ks 不存在                     |
| 103314 | sip密码变更失败                       |
| 103315 | sip密码推送失败                       |
| 103399 | sip  sys错误                      |
| 103400 | authorization  为空               |
| 103401 | 签名消息为空                          |
| 103402 | 无效的  authWay                    |
| 103404 | 加密失败                            |
| 103405 | 保存数据短信手机号为空                     |
| 103406 | 保存数据短信短信内容为空                    |
| 103407 | 此sourceId,  appPackage, sign已注册 |
| 103408 | 此sourceId注册已达上限   99次           |
| 103409 | query  为空                       |
| 103412 | 无效的请求                           |
| 103413 | 系统异常                            |
| 103414 | 参数效验异常                          |
| 103505 | 重放攻击                            |
| 103511 | 源IP不合法                          |
| 103810 | 校验失败，接口token版本不一致               |
| 103811 | token为空                         |
| 103899 | aoi  token 其他错误                 |
| 103901 | 短信验证码下发次数已达上限                   |
| 103902 | 凭证校验失败                          |
| 103903 | 调用webservice错误                  |
| 103904 | 配置不存在                           |
| 103905 | 获取手机号码错误                        |
| 103906 | 平台迁移访问错误  - （访问旧地址）             |
| 103911 | 请求过于频繁                          |
| 103920 | 没有存在的版本更新                       |
| 103921 | 下载时间戳超时                         |
| 103922 | 自动升级文件没找到                       |
| 104001 | APPID和APPKEY已存在                 |
| 104201 | 凭证已失效或不存在                       |
| 104202 | 短信验证失败过多                        |
| 105001 | 联通网关取号失败                        |
| 105002 | 移动网关取号失败                        |
| 105003 | 电信网关取号失败                        |
| 105004 | 短信上行ip检测不合法                     |
| 105005 | 短信上行发送信息为空                      |
| 105006 | 手机号码为空                          |
| 105007 | 手机号码格式错误                        |
| 105008 | 短信内容为空                          |
| 105009 | 解析失败                            |
| 105010 | phonescript失效或者非法               |
| 105011 | getPhonescript参数加密的私钥失效或者非法     |
| 105012 | 不支持电信取号                         |
| 105013 | 不支持联通取号                         |
| 105014 | 校验本机号码失败                        |
| 105015 | 校验有数三要素失败                       |
| 105018 | 用户权限不够                          |
| 105019 | 应用未授权                           |
## 6.3. 本机号码校验接口返回码

本返回码表仅针对`本机号码校验接口`使用

| 返回码    | 说明              |
| ------ | --------------- |
| 000    | 是本机号码（纳入计费次数）   |
| 001    | 非本机号码（纳入计费次数）   |
| 002    | 取号失败            |
| 003    | 调用内部token校验接口失败 |
| 004    | 加密手机号码错误        |
| 102    | 参数无效            |
| 124    | 白名单校验失败         |
| 302    | sign校验失败        |
| 303    | 参数解析错误          |
| 606    | 验证Token失败       |
| 999    | 系统异常            |
| 102315 | 次数已用完           |
