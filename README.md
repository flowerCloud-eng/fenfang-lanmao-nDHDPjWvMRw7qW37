
用户购买某种产品时习惯一次性付款，但是对开发者而言，单次购买模式或需要用户频繁续订的服务可能会导致收入不稳定，无法获得持续稳定的收入。对于有视频、音乐等会员需求的用户，一旦体验到服务中断或需要频繁操作，可能会转向其他竞争产品，导致用户流失。


HarmonyOS SDK应用内支付服务（IAP Kit）为开发者提供应用内[自动续期订阅商品能力](https://github.com "自动续期订阅商品能力")，用户购买后在一段时间内允许访问增值功能或内容，周期结束后可以选择自动续期购买下一期的服务。此外，IAP Kit提供全面的[订阅服务管理](https://github.com "订阅服务管理")，涵盖订阅商品、订阅关系、周期、促销、价格和通知等方面，帮助您创造持续稳定的收入。


![image](https://img2024.cnblogs.com/blog/2396482/202412/2396482-20241220111138530-924638820.png)


### 开发步骤


在接入订阅前，开发者需在华为AppGallery Connect网站[配置自动续期订阅商品](https://github.com "配置自动续期订阅商品")，录入商品ID和商品价格等信息。用户在应用购买自动续期订阅商品时，应用需要调用createPurchase接口来拉起IAP Kit订阅收银台，收银台会展示商品名称、商品价格、商品续期计划等信息，用户可在收银台完成商品购买。


1\.判断当前登录的华为账号所在的服务地是否支持应用内支付。


在使用应用内支付之前，应用需要向IAP Kit发送queryEnvironmentStatus请求，以此判断用户当前登录的华为账号所在的服务地是否在IAP Kit支持结算的国家/地区中。



```
import { iap } from '@kit.IAPKit';
import { common } from '@kit.AbilityKit';
import { BusinessError } from '@kit.BasicServicesKit';

queryEnvironmentStatus() {
  const context: common.UIAbilityContext = getContext(this) as common.UIAbilityContext;
  iap.queryEnvironmentStatus(context).then(() => {
    // 请求成功
    console.info('Succeeded in querying environment status.');
  }).catch((err: BusinessError) => {
    // 请求失败
    console.error(`Failed to query environment status. Code is ${err.code}, message is ${err.message}`);
  });
}

```

2\.查询商品信息


通过queryProducts来获取在AppGallery Connect上配置的商品信息。发起请求时，需在请求参数QueryProductsParameter中携带相关的商品ID，并指定其productType为iap.ProductType.AUTORENEWABLE。


当接口请求成功时，IAP Kit将返回商品信息Product的列表。 应用可以使用Product包含的商品价格、名称和描述等信息，向用户展示可供购买的商品列表。



```
import { iap } from '@kit.IAPKit';
import { BusinessError } from '@kit.BasicServicesKit';

queryProducts() {
  const queryProductParam: iap.QueryProductsParameter = {
    productType: iap.ProductType.AUTORENEWABLE,
    productIds: ['product1', 'product2', 'product3']
  };
  const context: common.UIAbilityContext = getContext(this) as common.UIAbilityContext;
  iap.queryProducts(context, queryProductParam).then((result) => {
    // 请求成功
    console.info('Succeeded in querying products.');
    // 展示商品信息
    // ...
  }).catch((err: BusinessError) => {
    // 请求失败
    console.error(`Failed to query products. Code is ${err.code}, message is ${err.message}`);  
  });
}

```

3\.发起购买


用户发起购买时，应用可通过向IAP Kit发送createPurchase请求来拉起IAP Kit收银台。发起请求时，应用需在请求参数PurchaseParameter中携带此前已在华为AppGallery Connect网站上配置并生效的自动续期订阅的商品ID，并指定其productType为iap.ProductType.AUTORENEWABLE。


当用户购买成功时，应用将接收到一个CreatePurchaseResult对象，其purchaseData字段包括了此次购买的结果信息。可参见对返回结果验签对PurchaseData.jwsSubscriptionStatus进行解码验签，成功后可得到SubGroupStatusPayload的JSON字符串。


当用户购买失败时，需要针对code为iap.IAPErrorCode.PRODUCT\_OWNED和iap.IAPErrorCode.SYSTEM\_ERROR的场景，检查是否需要补发货，确保权益发放，具体请参见[权益发放](https://github.com "权益发放"):[悠兔机场](https://xinnongbo.com)。



```
import { iap } from '@kit.IAPKit';
import { BusinessError } from '@kit.BasicServicesKit';
// JWTUtil为自定义类，可参见Sample Code工程。
import { JWTUtil } from '../commom/JWTUtil';

subscribe() {
  const createPurchaseParam: iap.PurchaseParameter = {
    // 购买的商品必须是开发者在AppGallery Connect网站配置的商品
    productId: 'test001',
    productType: iap.ProductType.AUTORENEWABLE
  }
  const context: common.UIAbilityContext = getContext(this) as common.UIAbilityContext;
  iap.createPurchase(context, createPurchaseParam).then(async (result) => {
    console.info('Succeeded in creating purchase.');
    const jwsSubscriptionStatus: string = JSON.parse(result.purchaseData).jwsSubscriptionStatus;
    if (!jwsSubscriptionStatus) {
      return;
    }
    const subscriptionStatus: string = JWTUtil.decodeJwtObj(jwsSubscriptionStatus);
    if (!subscriptionStatus) {
       return;
    }
    // 需自定义SubGroupStatusPayload类，包含的信息请参见SubGroupStatusPayload
    const subGroupStatusPayload: SubGroupStatusPayload = JSON.parse(subscriptionStatus);
    const lastSubscriptionStatus = subGroupStatusPayload.lastSubscriptionStatus;
    if (!lastSubscriptionStatus || !lastSubscriptionStatus.status) {
      return;
    }
    const purchaseOrderPayload = lastSubscriptionStatus.lastPurchaseOrder;
    if (purchaseOrderPayload === undefined) {
      return;
    }
    // 处理发货
    // ...
    // 发货成功后向IAP Kit发送finishPurchase请求，确认发货，完成购买
    // finishPurchase请求的参数来源于lastSubscriptionStatus.lastPurchaseOrder
    // 发起finishPurchase请求
    // ...
  }).catch((err: BusinessError) => {
    // 请求失败
    console.error(`Failed to create purchase. Code is ${err.code}, message is ${err.message}`);
    if (err.code === iap.IAPErrorCode.PRODUCT_OWNED || err.code === iap.IAPErrorCode.SYSTEM_ERROR) {
      // 参见确保权益发放检查是否需要补发货，确保权益发放
      // ...
    }
  })
}

```

2\.完成购买


对PurchaseData.jwsSubscriptionStatus解码验签成功后，需要进一步判断订阅状态是否是生效中来决定是否发放权益。检查SubGroupStatusPayload.lastSubscriptionStatus.status是否为1（生效中），是则发放相关权益。如果开发者同时接入了[服务端关键事件通知](https://github.com "服务端关键事件通知")，为了避免重复发货，建议先检查此笔订单是否已发货，未发货再发放相关权益。发货成功后记录SubGroupStatusPayload.lastSubscriptionStatus.lastPurchaseOrder等信息，用于后续检查是否已发货。


发货成功后，应用需发送finishPurchase请求确认发货，以此通知IAP服务器更新商品的发货状态，完成购买流程。发送finishPurchase请求时，需在请求参数FinishPurchaseParameter中携带PurchaseOrderPayload中的productType、purchaseToken、purchaseOrderId，其中PurchaseOrderPayload为SubGroupStatusPayload.lastSubscriptionStatus.lastPurchaseOrder。请求成功后，IAP服务器会将相应商品标记为已发货。



```
import { iap } from '@kit.IAPKit';
import { BusinessError } from '@kit.BasicServicesKit';

/**
 * 确认发货，完成购买
 *
 * @param purchaseOrder 购买数据，来源于购买请求
 */
finishPurchase(purchaseOrder: PurchaseOrderPayload) {
  const finishPurchaseParam: iap.FinishPurchaseParameter = {
    productType: purchaseOrder.productType,
    purchaseToken: purchaseOrder.purchaseToken,
    purchaseOrderId: purchaseOrder.purchaseOrderId
  };
  const context: common.UIAbilityContext = getContext(this) as common.UIAbilityContext;
  iap.finishPurchase(context, finishPurchaseParam).then(() => {
    // 请求成功
    console.info('Succeeded in finishing purchase.');
  }).catch((err: BusinessError) => {
    // 请求失败
    console.error(`Failed to finish purchase. Code is ${err.code}, message is ${err.message}`);
  });
}

```

**了解更多详情\>\>**


访问[应用内支付服务联盟官网](https://github.com "应用内支付服务联盟官网")


获取[接入订阅开发指导文档](https://github.com "接入订阅开发指导文档")


