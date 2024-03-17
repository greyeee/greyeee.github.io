+++
title = '云存储订阅设计'
date = 2024-03-16T12:44:03+08:00
draft = false

+++

## 需求

1. 摄像头录像云存储套餐订阅，分月、年套餐，分基础版、高级版套餐，区别是最多可用设备数不同。
2. google play、app store内购采用升降级套餐，stripe、paypal平台采用购买多个套餐，用户可选择内购或stripe、paypal购买。
3. 全球有4大区（us, eu, ap, cn），各区订阅不互通。

## 领域建模

1. 套餐

```go
// Plan 套餐
type Plan struct {
	Id             int               // 套餐id
	Names          map[string]string // 各个语言的名称
	Descriptions   map[string]string // 各个语言的描述
	PeriodInDays   int               // 有效天数
	MaxDeviceCount int               // 最多可用设备数
	Products       []Product         // 各支付平台产品配置
}

// Product 支付平台产品配置
type Product struct {
	Id       string           // 支付平台的产品id，google play、app store的product id，stripe的price id，paypal的plan id
	Platform Platform         // 支付平台
	Prices   map[string]Price // 各个地区的价格
}

// Platform 支付平台
type Platform string

const (
	PlatformGooglePlay Platform = "google_play"
	PlatformAppStore   Platform = "app_store"
	PlatformStripe     Platform = "stripe"
	PlatformPaypal     Platform = "paypal"
)

// Price 价格
type Price struct {
	CurrencyCode string // 货币编码
	Value        string // 数额
}
```

2. 订阅

```go
// Subscription 维护订阅的生命周期
type Subscription struct {
	Id                     string             // 订阅id
	UserId                 string             // 用户id
	PlanId                 string             // 套餐id
	PendingPlanId          string             // 下一个周期变更的套餐id
	Platform               Platform           // 支付平台
	Status                 SubscriptionStatus // 订阅状态
	ExpiresAt              time.Time          // 过期时间
	PlatformSubscriptionId string             // 支付平台的订阅id，google play的purchase token，app store的original transaction id，stripe、paypal的subscription id
	CreatedAt              time.Time          // 创建时间
}

// SubscriptionStatus 订阅状态
type SubscriptionStatus string

const (
	SubscriptionPending  SubscriptionStatus = "pending"  // 未完成
	SubscriptionActive   SubscriptionStatus = "active"   // 订阅中
	SubscriptionCanceled SubscriptionStatus = "canceled" // 已取消
	SubscriptionPaused   SubscriptionStatus = "paused"   // 已暂停
	SubscriptionExpired  SubscriptionStatus = "expired"  // 已过期
)
```

## 领域建模设计细节

1. 套餐Plan的domain model与read model的不同。前端是分平台请求套餐列表的，只关心当前语言的名称和描述、当前地区的价格、当前平台的product id，即read model为

   ```go
   type Plan struct {
   	Id             int    // 套餐id
   	Name           string // 名称
   	Description    string // 描述
   	PeriodInDays   int    // 有效天数
   	MaxDeviceCount int    // 最多可用设备数
   	ProductId      string // 支付平台的产品id
   }
   ```

   套餐Plan domain model需要根据客户端的language, countryCode，platform转换填充read model。

2. 订阅Subscription的id采用string，不采用自增主键int64，原因是支付平台的通知只有一个入口，而全球4个区的订阅不互通，需要维护订阅id到region的映射。根据这个映射，将通知路由转发到各个区处理，这就要求各个区创建的订阅id不能相同。

## 如何关联购买到用户订阅

1. 透传用户订阅id。

   - google play：[`obfuscatedExternalAccountId`](https://developer.android.com/reference/com/android/billingclient/api/BillingFlowParams.Builder#setobfuscatedaccountid)透传用户订阅id。
   - app store：[`appAccountToken`](https://developer.apple.com/documentation/storekit/product/purchaseoption/3749440-appaccounttoken)透传用户订阅id。
   - stripe: [`subscriptionData.metaData["subscriptionId"]`](https://docs.stripe.com/api/checkout/sessions/create)透传用户订阅id。

   - 风险点：`obfuscatedExternalAccountId`和`appAccountToken`本意是用于关联购买到用户账号，传用户账号id更加合适。google play使用它来检测不规则活动，例如许多设备在短时间内使用同一帐户进行购买。

2. app在购买成功后关联用户订阅

   - google play：购买成功后的[回调](https://developer.android.com/google/play/billing/integrate)中，将[`purchaseToken`](https://developer.android.com/reference/com/android/billingclient/api/Purchase#getPurchaseToken())和用户订阅id提交给后台。官方例子[play-billing-samples](https://github.com/android/play-billing-samples)就是这么做的，不过关联的是用户。

   - app store：购买成功后的[回调](https://developer.apple.com/documentation/storekit/in-app_purchase/original_api_for_in-app_purchase/offering_completing_and_restoring_in-app_purchases)中，将[`transactionIdentifier`](https://developer.apple.com/documentation/storekit/skpaymenttransaction)和用户订阅id提交给后台。官方课程[Manage in-app purchases on your server](https://developer.apple.com/videos/play/wwdc2021/10174)中提到Send the original transaction id and other relevant fields to your server。

   - 风险点：拿到支付平台的订阅id的其他人也可以关联到他的订阅，出现可能性极低。

3. `Subscription.PlatformSubscriptionId`记录支付平台的订阅id，即google play的`purchase token`，app store的`original transaction id`，stripe、paypal的`subscription id`。支付平台的通知处理统一采用支付平台的订阅id查找`Subscription`，不采用透传的`subscriptionId`查找订阅。

4. google play在升降级或在订阅到期之前从应用中取消后又订阅，旧订阅会失效，并且新订阅会通过新的`purchaseToken`创建，`linkedPurchaseToken`表示用户升级、降级或重新订阅时所基于的旧购买交易的字段。业务需要更新`Subscription.PlatformSubscriptionId`为新的`purchaseToken`。
5. 如果后台配置了允许过期的订阅重新订阅，用户可以在 Google Play 商店中重新购买已到期的订阅。这是新的购买，全新的`purchaseToken`，后端会收到类型为 `SUBSCRIPTION_PURCHASED` 的通知，`linkedPurchaseToken`不会关联到原始的`purchaseToken`。建议配置成禁止重新订阅。

## 支付平台通知处理

主要是同步订阅（包括套餐id、订阅状态、过期时间、支付平台的订阅id），提供服务和停止提供服务。

### google play

https://developer.android.com/google/play/billing/lifecycle/subscriptions?hl=zh-cn#new-auto

先查询 [`purchases.subscriptionsv2.get`](https://developers.google.com/android-publisher/api-ref/rest/v3/purchases.subscriptionsv2/get?hl=zh-cn) 端点，获取包含最新订阅状态的[订阅资源](https://developers.google.com/android-publisher/api-ref/rest/v3/purchases.subscriptionsv2?hl=zh-cn)。

| NotificationType                    | 说明                                                         | 处理                                                         |
| ----------------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| SUBSCRIPTION_PURCHASED              | 购买订阅成功                                                 | 确保subscriptionState为`SUBSCRIPTION_STATE_ACTIVE`，检查`purchaseToken`是否使用过，如果`linkedPurchaseToken`有值，撤销对应的订阅权益。提供服务，然后调用`acknowledgePurchase`确认发货。 |
| SUBSCRIPTION_RENEWED                | 续订成功                                                     | 提供服务                                                     |
| SUBSCRIPTION_STATE_IN_GRACE_PERIOD  | 宽限期                                                       | 还能继续使用，延长过期时间                                   |
| SUBSCRIPTION_STATE_ON_HOLD          | 宽限期结束后进入保留期                                       | 停止提供服务                                                 |
| SUBSCRIPTION_RECOVERED              | 恢复                                                         | 提供服务                                                     |
| SUBSCRIPTION_CANCELED               | 保留期结束没有付款，或者用户主动取消                         | 如果expiryTime已经过去，撤销访问权限，否则可继续访问         |
| SUBSCRIPTION_EXPIRED                | 保留期期间用户主动取消                                       | 停止提供服务                                                 |
| SUBSCRIPTION_REVOKED                | 撤销订阅，后端使用 `purchases.subscriptions.revoke` 撤消订阅或购买交易被退款。 | 停止提供服务                                                 |
| SUBSCRIPTION_DEFERRED               | 延迟过期时间，后端使用`purchases.subscriptions.defer`延长结算日期 | 提供服务                                                     |
| SUBSCRIPTION_PAUSE_SCHEDULE_CHANGED | 用户发起订阅暂停                                             | 不处理                                                       |
| SUBSCRIPTION_PAUSED                 | 暂停生效                                                     | 停止提供服务                                                 |
| SUBSCRIPTION_RESTARTED              | 在到期之前恢复订阅                                           | 提供服务                                                     |

### app store

https://developer.apple.com/documentation/appstoreservernotifications/notificationtype

先JWT校验签名。

| notificationType                                          | 说明                                                         | 处理                 |
| --------------------------------------------------------- | ------------------------------------------------------------ | -------------------- |
| CONSUMPTION_REQUEST                                       | 用户发起退款                                                 | 不处理               |
| DID_CHANGE_RENEWAL_PREF                                   | 升降级，subtype为UPGRADE、DOWNGRADE                          | 升降级               |
| DID_CHANGE_RENEWAL_STATUS 且 subtype为AUTO_RENEW_ENABLED  | 重新开启自动续费                                             | 提供服务             |
| DID_CHANGE_RENEWAL_STATUS 且 subtype为AUTO_RENEW_DISABLED | 用户关闭自动续费，或用户发起退款app store关闭订阅自动续费    | 继续提供服务直到过期 |
| DID_FAIL_TO_RENEW且subtype为GRACE_PERIOD                  | 宽限期                                                       | 继续提供服务直到过期 |
| DID_FAIL_TO_RENEW且subtype为空                            | 续订失败                                                     | 停止提供服务         |
| DID_RENEW且subtype为BILLING_RECOVERY                      | 之前续订失败的过期订阅已成功续订                             | 提供服务             |
| DID_RENEW且subtype为空                                    | 续订成功                                                     | 提供服务             |
| EXPIRED                                                   | 订阅过期                                                     | 停止提供服务         |
| GRACE_PERIOD_EXPIRED                                      | 宽限期结束，没有续订                                         | 停止提供服务         |
| OFFER_REDEEMED                                            | 用户兑换促销优惠或优惠码                                     | 不处理               |
| PRICE_INCREASE                                            | 价格变动                                                     | 不处理               |
| REFUND                                                    | 退款退款                                                     | 停止提供服务         |
| REFUND_DECLINED                                           | App Store拒绝了应用开发者发起的退款请求                      | 不处理               |
| REFUND_REVERSED                                           | 撤销退款                                                     | 恢复提供服务         |
| RENEWAL_EXTENDED                                          | 延长续订日期                                                 | 不处理               |
| RENEWAL_EXTENSION                                         | 申请延长续订日期                                             | 不处理               |
| REVOKE                                                    | 关掉家庭共享，或离开家庭组，或退款                           | 停止提供服务         |
| SUBSCRIBED                                                | 订阅了，或通过家庭共享首次获得订阅产品的访问权，重新订阅了，或订阅了同一订阅组内的另一个产品，或通过家庭共享重新获得订阅产品的访问权 | 提供服务             |
| TEST                                                      | 测试                                                         | 不处理               |

### stripe

https://docs.stripe.com/billing/subscriptions/overview

只订阅customer.subscription.updated事件

golang库提供了校验和反序列化的方法。

```go
event, err := webhook.ConstructEvent(payload, header, secret)
```

| subscription.status | 说明                                                         | 处理         |
| ------------------- | ------------------------------------------------------------ | ------------ |
| active              | 付款成功                                                     | 提供服务     |
| unpaid              | 付款失败，所有重试都失败后                                   | 停止提供服务 |
| canceled            | 取消了，最终状态，无法再付款                                 | 停止提供服务 |
| past_due            | 付款失败，如果所有重试都失败，状态变成canceled或unpaid，后台配置 | 不处理       |
| trialing            | 试用                                                         | 提供服务     |
| incomplete          | 付款未完成                                                   | 不处理       |
| incomplete_expired  | 首笔付款超时未完成                                           | 不处理       |
| paused              | 试用期结束，用户没有设置付款方式                             | 不处理       |

## 续订通知可靠性

可以启动一个定时器，在订阅过期时间之后一小段时间调用支付平台的查看订阅信息api，来更新业务的订阅。如果支付平台的续订通知准时到达并处理了，则取消这个定时器。

