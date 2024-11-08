# TMDOG的Gin学习笔记_02——Gin集成支付宝支付沙箱环境

## 博客地址：[TMDOG的博客](https://blog.tmdog114514.icu/)

**作者自述：**  
最近忙着整理自己的项目代码，终于有时间更新一下博客。这次的内容是关于如何在Gin框架下集成支付宝的支付沙箱环境，具体包括如何初始化支付宝的SDK、生成支付链接和处理回调请求。不同于NestJS笔记，Gin学习笔记更专注于业务逻辑，而支付宝支付集成是大多数电商项目中不可缺少的部分，所以这一篇笔记对我来说也非常重要。

## 学习目标

学习如何将支付宝支付集成到Gin框架下，实现在沙箱环境中进行支付流程的模拟。

**预期使用的技术栈：**  
Gin、支付宝SDK、Go

## 支付宝支付沙箱环境配置

### 1. 创建支付宝开放平台应用

首先，你需要在支付宝开放平台上创建一个应用：

- 访问[支付宝开放平台](https://open.alipay.com/api)。
![image](https://github.com/user-attachments/assets/7787281c-beb6-46a4-a9b3-16a9957048c1)

- 使用支付宝扫码登陆并进入控制台。
- 向下划，在页面下面进入沙箱环境。
![image](https://github.com/user-attachments/assets/4e0823f1-900a-4076-8a9d-667365553569)
![image](https://github.com/user-attachments/assets/866ce08e-c9d6-45eb-8965-8b73423f665c)

- 在`接口加签方式`中选择系统默认的证书模式，启用并查看
![image](https://github.com/user-attachments/assets/6d81a260-d427-48d3-b758-fe7d8761b1f6)

- 然后下载所有证书并，复制好应用私钥
![image](https://github.com/user-attachments/assets/f274f502-a995-40c6-b35a-92b06ba28725)

- 在`接口内容加密方式`中获取密钥并保存
![image](https://github.com/user-attachments/assets/513f1399-c54b-4d64-92a0-41b0956631dd)
![image](https://github.com/user-attachments/assets/b3743890-7afb-45c4-b300-abe2c010572e)


### 2. 安装支付宝SDK

支付宝官方提供了Go语言SDK，使用以下命令进行安装：

```bash
go get github.com/smartwalle/alipay/v3
```

### 3. 完整代码示例

我们将下载的证书放在项目目录下
![image](https://github.com/user-attachments/assets/c160a670-ffde-46e2-bbf3-c124f771b550)


```go
package main

import (
	"context"
	"fmt"
	"github.com/gin-contrib/cors"
	"github.com/gin-gonic/gin"
	"github.com/smartwalle/alipay/v3"
	"github.com/smartwalle/xid"
	"log"
	"net/http"
	"time"
)

var client *alipay.Client

const (
	kAppId        = "APPID"
	kPrivateKey   = "刚才保存的私钥"
	kServerPort   = "9989"                      //端口
	kServerDomain = "http://localhost:9989"//回调地址
	kSecretKey = "接口内容加密密钥"// 内容加密密钥
)

func main() {
	var err error
	
	// 初始化支付宝
	if client, err = alipay.New(kAppId, kPrivateKey, false); err != nil {
		log.Println("初始化支付宝失败", err)
		return
	}

	// 加载证书
	if err = client.LoadAppCertPublicKeyFromFile("appPublicCert.crt"); err != nil {
		log.Println("加载证书发生错误", err)
		return
	}
	if err = client.LoadAliPayRootCertFromFile("alipayRootCert.crt"); err != nil {
		log.Println("加载证书发生错误", err)
		return
	}
	if err = client.LoadAlipayCertPublicKeyFromFile("alipayPublicCert.crt"); err != nil {
		log.Println("加载证书发生错误", err)
		return
	}

	if err = client.SetEncryptKey(kSecretKey); err != nil {
		log.Println("加载内容加密密钥发生错误", err)
		return
	}

	// 配置 CORS 中间件
	corsConfig := cors.DefaultConfig()
	corsConfig.AllowAllOrigins = true                                   // 允许所有域名
	corsConfig.AllowMethods = []string{"GET", "POST", "OPTIONS"}        // 允许的 HTTP 方法
	corsConfig.AllowHeaders = []string{"Content-Type", "Authorization"} // 允许的请求头

	// 使用 gin 框架设置路由
	r := gin.Default()

	// 因为需要支付宝返回回调请求，是跨域的，所以需要处理跨域请求
	// 将 CORS 中间件应用到所有路由
	r.Use(cors.New(corsConfig))

	// 配置路由
	r.GET("/alipay/pay", pay)
	r.GET("/alipay/callback", callback)
	r.POST("/alipay/notify", notify)

	// 启动 HTTP 服务
	r.Run(":" + kServerPort)
}

func pay(c *gin.Context) {
	var tradeNo = fmt.Sprintf("%d", xid.Next())

	// 构造支付请求
	var p = alipay.TradePagePay{}
	p.NotifyURL = kServerDomain + "/alipay/notify"
	p.ReturnURL = kServerDomain + "/alipay/callback"
	p.Subject = "支付测试:" + tradeNo
	p.OutTradeNo = tradeNo
	p.TotalAmount = "100.00"
	p.ProductCode = "FAST_INSTANT_TRADE_PAY"

	// 打印支付请求数据
	log.Printf("支付请求数据: %+v", p)

	// 发起支付请求
	url, _ := client.TradePagePay(p)

	// 打印生成的支付链接
	log.Printf("支付链接: %s", url.String())

	// 重定向到支付链接
	c.Redirect(http.StatusTemporaryRedirect, url.String())
}

func callback(c *gin.Context) {
	// 解析请求参数
	if err := c.Request.ParseForm(); err != nil {
		c.JSON(http.StatusBadRequest, gin.H{
			"error": "解析表单数据失败",
		})
		return
	}

	// 打印回调的请求数据
	log.Printf("回调请求数据: %+v", c.Request.Form)

	// 获取支付宝回调数据中的签名字段
	sign := c.DefaultQuery("sign", "")
	if sign == "" {
		c.JSON(http.StatusBadRequest, gin.H{
			"error": "签名参数缺失",
		})
		return
	}

	// 验证签名
	if err := client.VerifySign(c.Request.Form); err != nil {
		log.Println("回调签名验证失败", err)
		c.JSON(http.StatusBadRequest, gin.H{
			"error": "签名验证失败",
		})
		return
	}

	log.Println("回调签名验证通过")

	// 获取回调中的订单信息
	outTradeNo := c.DefaultQuery("out_trade_no", "")
	totalAmount := c.DefaultQuery("total_amount", "")
	//tradeNo := c.DefaultPostForm("trade_no", "")

	// 验证支付金额
	if totalAmount != "100.00" { // 这里你可以根据自己的业务逻辑进行金额校验
		log.Printf("支付金额不匹配，expected: 100.00, received: %s", totalAmount)
		c.JSON(http.StatusBadRequest, gin.H{
			"error": "支付金额不匹配",
		})
		return
	}

	// 查询订单状态
	var p = alipay.TradeQuery{}
	p.OutTradeNo = outTradeNo

	// 调用支付宝查询接口
	rsp, err := client.TradeQuery(context.Background(), p)
	if err != nil {
		c.JSON(http.StatusBadRequest, gin.H{
			"error": fmt.Sprintf("查询订单 %s 时发生错误: %s", outTradeNo, err.Error()),
		})
		return
	}

	// 打印查询结果
	log.Printf("支付宝查询响应: %+v", rsp)

	// 检查支付状态
	if rsp.IsFailure() {
		c.JSON(http.StatusBadRequest, gin.H{
			"error": fmt.Sprintf("支付查询失败: %s-%s", rsp.Msg, rsp.SubMsg),
		})
		return
	}

	// 如果支付成功，处理业务逻辑，如更新数据库、通知用户等
	if rsp.TradeStatus == "TRADE_SUCCESS" {
		log.Printf("订单 %s 支付成功", outTradeNo)
		// 这里可以写入支付成功后的业务逻辑，如更新订单状态、发货等
	} else {
		log.Printf("订单 %s 支付失败", outTradeNo)
		// 如果支付失败，可以进行相应的处理，如通知管理员等
	}

	// 返回成功响应
	c.JSON(http.StatusOK, gin.H{
		"message": fmt.Sprintf("订单 %s 支付成功", outTradeNo),
	})
}

func notify(c *gin.Context) {
	// 解析表单数据
	if err := c.Request.ParseForm(); err != nil {
		c.JSON(http.StatusBadRequest, gin.H{
			"error": "解析表单数据失败",
		})
		return
	}

	// 打印接收到的异步通知数据
	log.Printf("异步通知请求数据: %+v", c.Request.Form)

	// 解析异步通知
	notification, err := client.DecodeNotification(c.Request.Form)
	if err != nil {
		log.Println("解析异步通知发生错误", err)
		c.JSON(http.StatusBadRequest, gin.H{
			"error": "解析异步通知发生错误",
		})
		return
	}

	// 打印解析后的异步通知数据
	log.Printf("解析后的异步通知数据: %+v", notification)

	// 使用自定义请求进行查询
	var p = alipay.NewPayload("alipay.trade.query")
	p.AddBizField("out_trade_no", notification.OutTradeNo)

	// 查询订单信息
	var rsp *alipay.TradeQueryRsp
	if err = client.Request(context.Background(), p, &rsp); err != nil {
		log.Printf("异步通知验证订单 %s 信息发生错误: %s \n", notification.OutTradeNo, err.Error())
		c.JSON(http.StatusBadRequest, gin.H{
			"error": fmt.Sprintf("验证订单 %s 信息发生错误: %s", notification.OutTradeNo, err.Error()),
		})
		return
	}

	// 打印支付宝的查询响应数据
	log.Printf("支付宝查询响应: %+v", rsp)

	if rsp.IsFailure() {
		log.Printf("异步通知验证订单 %s 信息发生错误: %s-%s \n", notification.OutTradeNo, rsp.Msg, rsp.SubMsg)
		c.JSON(http.StatusBadRequest, gin.H{
			"error": fmt.Sprintf("异步通知验证订单 %s 信息发生错误: %s-%s", notification.OutTradeNo, rsp.Msg, rsp.SubMsg),
		})
		return
	}

	log.Printf("订单 %s 支付成功 \n", notification.OutTradeNo)

	// 确认通知成功
	client.ACKNotification(c.Writer)
}

```

涉及了三个主要的请求方法，分别是：

#### 1. **`pay` 请求方法**
- **作用：** 这个方法是处理用户发起支付请求的地方。当用户请求 `GET /alipay/pay` 时，系统会生成一个支付链接，并将用户重定向到支付宝的支付页面。
- **具体步骤：**
  1. 创建一个唯一的订单号（`tradeNo`），这里通过 `xid` 包生成一个。
  2. 使用 `alipay.TradePagePay` 构建支付请求对象，设置支付相关的信息，如：通知地址（`NotifyURL`）、回调地址（`ReturnURL`）、商品描述（`Subject`）、订单号（`OutTradeNo`）和支付金额（`TotalAmount`）。
  3. 调用 `client.TradePagePay(p)` 向支付宝请求生成支付页面链接。
  4. 将生成的支付链接作为响应返回，并通过重定向将用户引导到支付宝的支付页面。
  
  该方法主要用于支付页面的生成和跳转，用户通过此链接进行支付操作。

#### 2. **`callback` 请求方法**
- **作用：** 这个方法用于处理支付宝支付完成后的同步回调请求。当用户完成支付，支付宝会根据配置的 `ReturnURL` 返回到这个接口，并携带支付结果。
- **具体步骤：**
  1. 从回调请求中解析支付宝返回的数据。
  2. 获取回调中的签名参数，进行签名验证，确保回调数据未被篡改。
  3. 验证支付金额是否与预期一致。
  4. 使用支付宝的查询接口（`TradeQuery`）查询支付订单状态。
  5. 如果支付成功，则执行相应的业务逻辑（如更新订单状态、发货等）。
  6. 最终返回处理结果，如支付成功或失败的消息。

  这个方法的目的是处理支付完成后的验证和后续操作，确保支付数据的安全性与一致性。

#### 3. **`notify` 请求方法**
- **作用：** 这个方法是处理支付宝支付后的异步通知请求。当用户支付完成后，支付宝会向 `NotifyURL` 发送异步通知，通知支付结果。支付宝会通过 POST 请求将支付结果发送到该接口。
- **具体步骤：**
  1. 解析支付宝发送的异步通知数据。
  2. 解码通知数据，得到支付结果。
  3. 查询支付订单的最新状态，验证订单是否支付成功。
  4. 确认支付成功后，可以进行进一步的处理，比如更新订单状态或进行其他业务逻辑。
  5. 最后返回一个确认响应（`client.ACKNotification`），告诉支付宝已经收到并处理完通知。

  这个方法用于处理支付完成后的异步通知，它可以确保系统及时接收到支付宝的支付通知并执行相应的后续处理。

这三个请求方法分别用于：
- **`pay`**：生成支付链接并跳转到支付宝支付页面。
- **`callback`**：同步回调，处理支付完成后的回调请求。
- **`notify`**：异步通知，处理支付宝发送的支付成功通知。


### 4. 测试

直接在浏览器中测试，我们输入`http://localhost:9989/alipay/pay`,将会重定向到支付宝支付页面
![image](https://github.com/user-attachments/assets/b8072794-6b83-4794-a3ac-0308999d5e0e)

在沙箱控制台右边栏当中点击沙箱账号可以填入买家邮箱和支付密码模拟支付
![image](https://github.com/user-attachments/assets/ec18aa98-aa16-4bc4-8010-3d349e4d213d)

也可以下载手机[支付宝沙箱版](https://open.alipay.com/develop/sandbox/tool/alipayclint),用沙箱账号登录后扫码支付

我是使用手机扫码
![image](https://github.com/user-attachments/assets/779064f4-575b-4f40-8a3b-7e184551c9b4)

<img width="1274" alt="c4fcdf494210ee0764b417b7025c3c6" src="https://github.com/user-attachments/assets/306692e9-837f-499d-8796-f0948b7cc416">


<img width="1274" alt="29a02f0bd5940b065a67b5128971a1e" src="https://github.com/user-attachments/assets/2d8aab05-2d61-448a-b5b8-e9624ffed0cf">

![image](https://github.com/user-attachments/assets/3c8c0e94-e575-48d1-8252-f6de40829b6c)

我们发现经过两次重定向后最后进入`http://localhost:9989/alipay/callback`显示支付成功

第一次重定向后支付宝服务方会异步发送alipay/notify到自己的服务器

第二次重定向会直接同步发送回调请求alipay/callback显示在用户界面通知支付成功


### 5. 查看gin控制台
```
2024/11/08 19:33:33 
支付请求数据: 
{Trade:{AuxParam:{} NotifyURL:http://localhost:9989/alipay/notify ReturnURL:http://localhost:9989/alipay/callback AppAuthToken: Subject:支付测试:3717435097532596224 Ou
tTradeNo:3717435097532596224 TotalAmount:100.00 ProductCode:FAST_INSTANT_TRADE_PAY Body: GoodsDetail:[] BusinessParams:[] DisablePayChannels: EnablePayChannels: SpecifiedChannel: ExtendParams:<nil> Agr
eementSignParams:<nil> GoodsType: InvoiceInfo: PassbackParams: PromoParams: RoyaltyInfo: SellerId: SettleInfo: StoreId: SubMerchant: TimeoutExpress: TimeExpire: MerchantOrderNo: ExtUserInfo:<nil> QueryOptions:[]} AuthToken: QRPayMode: QRCodeWidth:}
2024/11/08 19:33:33 

支付链接: https://openapi-sandbox.dl.alipaydev.com/gateway.do?alipay_root_cert_sn=687b59193f3f462dd5336e5abf83c5d8_02941eef3187dddf3d3b83462e1dfcf6&app_cert_sn=17811ce5cb5cf99a255aa
c61c6a11ae8&app_id=9021000141668546&biz_content=pYwkNjCG66efEQCRHGGZCO%2B46LHmZe15cel6apDQiiKhmB%2Bt%2B79id%2Fli5No%2B4PC%2F0CckV20dSx%2Fr0mL5JDwysgCtvijdgWWyiPffxrfv0iLsSXMqC7VAOY0Mr9cA32uL01mEH39nE9I
zTrcCjdJ9x1GN2tXhU57IFCy%2FbnSd3OfJcrLqJyKxxFlTtrjpdCCL1SBM1VxSo%2B9C8zL%2FlzfKDw%3D%3D&charset=utf-8&encrypt_type=AES&format=JSON&method=alipay.trade.page.pay&notify_url=http%3A%2F%2Flocalhost%3A9989%
2Falipay%2Fnotify&return_url=http%3A%2F%2Flocalhost%3A9989%2Falipay%2Fcallback&sign=buxn%2Fiwm3kBnCJbpIDi8Bz%2BYyJ1PFpYUR%2B%2Frh54aSzyDlyuMKlVHNO0AbSeAP5eyXXIK%2FbKJ8MMSETkfWEpDO%2Fuk4O4yzlBn0VMyZnoIT
vRDIvUJYuCpKXgBXplfuVI6YEzA6S2S0l6%2BQuzIRpqHlK0VJoDahiWisr95YOxb8ld8oH37UO%2F%2BvbrHMYWT7BLTWz%2FVqh0ELMKX%2BhsAeGBM1HlNAe1gpk3CB4%2FTIH%2BGUbicv9TcTOV2nLLKCLkqmnqOlmhK%2FB%2BnbP52j7%2FbuGaV5hx%2FSnKeARdMZJW1tr3A7cWKyBpGOGf%2BylvBGvySg046aeRDo5l6OEnX3J10zdflUg%3D%3D&sign_type=RSA2&timestamp=2024-11-08+19%3A33%3A33&version=1.0

[GIN] 2024/11/08 - 19:33:33 | 307 |     31.9174ms |             ::1 | GET      "/alipay/pay"

2024/11/08 19:34:09 
回调请求数据: 
map[app_id:[9021000141668546] auth_app_id:[9021000141668546] charset:[utf-8] method:[alipay.trade.page.pay.return] out_trade_no:[3717435097532596224] seller_id:[208872
1048676520] sign:[R2vAtE5H22B5Z+rVvlvQBiokQWl84bH2E9peZDON1LEg82SXVS4F1FWIPDtifhI+CvUQ0Kj+oLHDuaoWMeAZhjRpwWQ+Q1htfDuFLcHPYN9ZSFWglw0dCmZ8cjPBy5Y0A5NcH2qAJQcCpKqtr6t9nFsk/U4PT4WW0i614AoLod7J94bqJwm0IhC
pGxGCFyhlL8MnNXDRAg28Ne139n4O7GDNiw6PQtcmybGCmNW8YxEh1oP+tDj6+cATbxlwzeCmB4veowSS66JuAec3GqxLMFyXGOuLKUtrHAo6Uhth9pRKtVHVK8E5JLi97sB6N+hSX4h1qOBjybuvbRXao/RqjQ==] sign_type:[RSA2] timestamp:[2024-11-08 19:34:06] total_amount:[100.00] trade_no:[2024110822001476540504411933] version:[1.0]]

2024/11/08 19:34:09 
回调签名验证通过
HYBAmount: BKAgentRespInfo:<nil> ChargeInfoList:[] DiscountGoodsDetail: VoucherDetailList:[]} rminalId: FundBillList:[] StoreName: BuyerUserId:2088722048676545 B
2024/11/08 19:34:10 订单 3717435097532596224 支付成功

[GIN] 2024/11/08 - 19:34:10 | 200 |    805.5153ms |             ::1 | GET      "/alipay/callback?charset=utf-8&out_trade_no=3717435097532596224&method=alipay.trade.page.pay.return&total_amount=100.00&sign=R2vAtE5H22B5Z%2BrVvlvQBiokQWl84bH2E9peZDON1LEg82SXVS4F1FWIPDtifhI%2BCvUQ0Kj%2BoLHDuaoWMeAZhjRpwWQ%2BQ1htfDuFLcHPYN9ZSFWglw0dCmZ8cjPBy5Y0A5NcH2qAJQcCpKqtr6t9nFsk%2FU4PT4WW0i614AoLod7J94bqJwm0IhCpGxGCFyhlL8MnNXDRAg28Ne139n4O7GDNiw6PQtcmybGCmNW8YxEh1oP%2BtDj6%2BcATbxlwzeCmB4veowSS66JuAec3GqxLMFyXGOuLKUtrHAo6Uhth9pRKtVHVK8E5JLi97sB6N%2BhSX4h1qOBjybuvbRXao%2FRqjQ%3D%3D&trade_no=2024110822001476540504411933&auth_app_id=9021000141668546&version=1.0&app_id=9021000141668546&sign_type=RSA2&seller_id=2088721048676520&timestamp=2024-11-08+19%3A34%3A06" 19%3A34%3A06"

```
我们可以看到所有的参数
以及触发的请求

但是，我们发现并没有接收到notify请求、使用为我们没有在控制台设置回调地址，并且也是处于本地网络环境，设置了也接收不到。
要想正确收取notify请求就需要在公网环境中部署。

支付宝的沙箱环境支持模拟支付过程，因此可以避免涉及真实交易的风险。你可以在支付宝开放平台的沙箱环境中获取测试用的账户和支付信息。测试时，使用支付宝提供的沙箱账号来模拟支付。

## 总结

通过本篇学习笔记，我们成功地将支付宝支付集成到了Gin框架中，并在沙箱环境中模拟了支付流程。我们介绍了如何使用支付宝SDK创建支付订单，如何配置回调处理支付结果。集成支付宝支付是电商系统中重要的一部分，本篇笔记为后续功能扩展和真实支付接入打下了基础。接下来，我将继续深入研究其他支付方式的集成与优化，敬请期待。