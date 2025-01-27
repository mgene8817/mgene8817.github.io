## 1. 在 PayPal 后台创建产品与计划

### 正式与沙盒环境链接

- **正式环境创建订阅计划地址**：  
  [访问 PayPal 正式环境](https://www.paypal.com/merchantapps/appcenter/acceptpayments/subscriptions)

- **沙盒环境创建订阅计划**：  
  [访问 PayPal 沙盒环境](https://www.sandbox.paypal.com/billing/plans)

您可以通过上面的链接进入 PayPal 后台，创建您的产品与订阅计划。请注意，您在此处创建的产品及计划仅限于正式环境，沙盒计划不会在此显示。

### 创建产品步骤
请根据相关流程创建产品及计划。

### API 创建产品及计划
您也可以通过 API 创建产品与计划。

## 2. API 创建产品与计划

### API 地址
[PayPal API 文档](https://developer.paypal.com/api/rest/)

### 1. 生成 Token
在您的 PayPal 账户中获取 `clientId` 和 `Secret`，然后获取 `access_token`。  
请求地址：  
- **沙盒**: `https://api.sandbox.paypal.com/v1/oauth2/token`  
- **正式**: `https://api.paypal.com/v1/oauth2/token`

### 2. 创建产品
使用以下 `curl` 命令创建产品：

bash
curl -v -X POST https://api-m.sandbox.paypal.com/v1/catalogs/products \
-H "Content-Type: application/json" \
-H "Authorization: Bearer Access-Token" \
-H "PayPal-Request-Id: PRODUCT-18062020-001" \
-d '{
  "name": "Video Streaming Service",
  "description": "Video streaming service",
  "type": "SERVICE",
  "category": "SOFTWARE",
  "image_url": "https://example.com/streaming.jpg",
  "home_url": "https://example.com/home"
}'


**参数说明**：
- `name`: 产品名称  
- `description`: 产品说明  
- `type`: 产品类型（实物、数字或服务）  
- `category`: 产品类别  
- `image_url`: 产品 logo  
- `home_url`: 产品主页地址  

### 3. 创建计划
使用以下命令创建计划：

bash
curl -v -X POST https://api-m.sandbox.paypal.com/v1/billing/plans \
-H "Content-Type: application/json" \
-H "Authorization: Bearer Access-Token" \
-H "PayPal-Request-Id: PLAN-18062019-001" \
-d '{
  "product_id": "PROD-XXCD1234QWER65782",
  "name": "Video Streaming Service Plan",
  "description": "Video Streaming Service basic plan",
  "status": "ACTIVE",
  "billing_cycles": [
    {
      "frequency": {
        "interval_unit": "MONTH",
        "interval_count": 1
      },
      "tenure_type": "REGULAR",
      "sequence": 1,
      "total_cycles": 12,
      "pricing_scheme": {
        "fixed_price": {
          "value": "6",
          "currency_code": "USD"
        }
      }
    }
  ],
  "payment_preferences": {
    "auto_bill_outstanding": true,
    "setup_fee": {
      "value": "6",
      "currency_code": "USD"
    },
    "setup_fee_failure_action": "CONTINUE",
    "payment_failure_threshold": 3
  },
  "taxes": {
    "percentage": "0",
    "inclusive": false
  }
}'


**参数说明**：
- `product_id`: 产品 ID  
- `name`: 计划名称  
- `description`: 计划说明  
- `status`: 计划状态  
- `billing_cycles`: 计费周期说明  
- `payment_preferences`: 付款首选项内容  
- `taxes`: 与税务相关的设置  

### 5. 创建订阅
使用以下命令创建订阅：

bash
curl -v -X POST https://api-m.sandbox.paypal.com/v1/billing/subscriptions \
-H "Content-Type: application/json" \
-H "Authorization: Bearer &lt;Access-Token&gt;" \
-H "PayPal-Request-Id: SUBSCRIPTION-21092019-001" \
-d '{
  "plan_id": "P-5ML4271244454362WXNWU5NQ",
  "start_time": "2022-07-21T00:00:00Z",
  "quantity": "20",
  "shipping_amount": {
    "currency_code": "USD",
    "value": "10.00"
  },
  "application_context": {
    "brand_name": "walmart",
    "locale": "en-US",
    "shipping_preference": "SET_PROVIDED_ADDRESS",
    "user_action": "SUBSCRIBE_NOW",
    "payment_method": {
      "payer_selected": "PAYPAL",
      "payee_preferred": "IMMEDIATE_PAYMENT_REQUIRED"
    },
    "return_url": "https://example.com/returnUrl",
    "cancel_url": "https://example.com/cancelUrl"
  }
}'


**参数说明**：
- `plan_id`: 计划 ID  
- `start_time`: 开始时间（设置为第二次扣款时间）  
- `quantity`: 订阅中产品数量  
- `shipping_amount`: 交易的货币及金额   
- `application_context`: 应用上下文相关设置  

根据获取的结果选择 "links" 下第一个 "href" 进行支付订阅。

### 6. 订阅详情
使用以下命令查询订阅详情：

bash
curl -v -X GET https://api-m.sandbox.paypal.com/v1/billing/subscriptions/I-BW452GLLEP1G \
-H "Content-Type: application/json" \
-H "Authorization: Bearer Access-Token"


### 7. 配置 Webhook
您可以选择适合您的 webhook 配置方式。

👉 [野卡 | 一分钟注册，轻松订阅海外线上服务](https://bit.ly/bewildcard)