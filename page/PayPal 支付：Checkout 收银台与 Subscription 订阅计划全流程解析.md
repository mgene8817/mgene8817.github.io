> 本文将详细分析请求的生命周期，逐步实现整个支付过程。

## 一、请求生命周期

### 1. Checkout - 收银台支付

支付流程概述如下（与支付宝的收银台过程相似）：

1. 本地应用组装参数并请求 Checkout 接口，接口同步返回一个支付 URL。
2. 本地应用重定向到此 URL，用户需登录 PayPal 账户并确认支付，完成后用户将被跳转至设定的本地应用地址。
3. 本地请求 PayPal 执行付款接口以发起扣款。
4. PayPal 发送异步通知至本地应用，本地应用收到数据包后进行验签操作。
5. 验签成功后，执行支付完成后的业务（如修改本地订单状态、增加销量、发送邮件等）。

### 2. Subscription - 订阅支付

订阅支付流程包括以下步骤：

1. 创建一个订阅计划。
2. 激活该计划。
3. 使用已激活的计划创建一个订阅申请。
4. 本地跳转至订阅申请链接获取用户授权并完成第一次付款，用户完成支付后，携带 token 跳转至设定的本地应用地址。
5. 回跳后请求执行订阅。
6. 收到订阅授权的异步回调结果，验证支付异步回调是否成功，并进行后续业务处理。

## 二、具体实现

了解了以上流程后，接下来开始编码。我们将使用官方提供的 SDK。

### Checkout

#### 在项目中安装扩展

bash
$ composer require paypal/rest-api-sdk-php:* // 这里使用的是最新版本


#### 创建 PayPal 配置文件

bash
$ touch config/paypal.php


配置内容如下（沙箱和生产环境的两个配置）：

php
<?php

return [
    'sandbox' => [
        'client_id' => env('PAYPAL_SANDBOX_CLIENT_ID', ''),
        'secret' => env('PAYPAL_SANDBOX_SECRET', ''),
        'notify_web_hook_id' => env('PAYPAL_SANDBOX_NOTIFY_WEB_HOOK_ID', ''),
        'checkout_notify_web_hook_id' => env('PAYPAL_SANDBOX_CHECKOUT_NOTIFY_WEB_HOOK_ID', ''),
        'subscription_notify_web_hook_id' => env('PAYPAL_SANDBOX_SUBSCRIPTION_NOTIFY_WEB_HOOK_ID', ''),
    ],

    'live' => [
        'client_id' => env('PAYPAL_CLIENT_ID', ''),
        'secret' => env('PAYPAL_SECRET', ''),
        'notify_web_hook_id' => env('PAYPAL_NOTIFY_WEB_HOOK_ID', ''),
        'checkout_notify_web_hook_id' => env('PAYPAL_CHECKOUT_NOTIFY_WEB_HOOK_ID', ''),
        'subscription_notify_web_hook_id' => env('PAYPAL_SUBSCRIPTION_NOTIFY_WEB_HOOK_ID', ''),
    ],
];


#### 创建 PayPal 服务类

bash
$ mkdir -p app/Services && touch app/Services/PayPalService.php


#### 编写Checkout的方法

可以参考官方给出的 [DEMO](https://github.com/paypal/PayPal-PHP-SDK) 。

php
<?php

namespace App\Services;

use App\Models\Order;
use Carbon\Carbon;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Log;
use PayPal\Api\Currency;
use PayPal\Auth\OAuthTokenCredential;
use PayPal\Rest\ApiContext;
use PayPal\Api\Amount;
use PayPal\Api\Details;
use PayPal\Api\Item;
use PayPal\Api\ItemList;
use PayPal\Api\Payer;
use PayPal\Api\Payment;
use PayPal\Api\RedirectUrls;
use PayPal\Api\Transaction;
use PayPal\Api\PaymentExecution;
use Symfony\Component\HttpKernel\Exception\HttpException;

class PayPalService
{
    protected $config;
    protected $notifyWebHookId;
    public $apiContext;

    public function __construct($config)
    {
        $this->config = $config;
        $this->notifyWebHookId = $this->config['web_hook_id'];
        $this->apiContext = new ApiContext(
            new OAuthTokenCredential(
                $this->config['client_id'],
                $this->config['secret']
            )
        );

        $this->apiContext->setConfig([
            'mode' => $this->config['mode'],
            'log.LogEnabled' => true,
            'log.FileName' => storage_path('logs/PayPal.log'),
            'log.LogLevel' => 'DEBUG',
            'cache.enabled' => true,
        ]);
    }

    public function checkout(Order $order)
    {
        try {
            // 构建支付相关逻辑
            
            // 返回支付链接
            return $payment->getApprovalLink();
        } catch (HttpException $e) {
            Log::error('PayPal Checkout Create Failed', ['msg' => $e->getMessage(), 'code' => $e->getStatusCode(), 'data' => ['order' => ['no' => $order->no]]]);
            return null;
        }
    }

    public function executePayment($paymentId)
    {
        try {
            // 执行付款逻辑
            
            return Payment::get($payment->getId(), $this->apiContext);
        } catch (HttpException $e) {
            return false;
        }
    }
}


#### 将 PayPal 服务类注册在容器中

打开文件 `app/Providers/AppServiceProvider.php` 进行修改：

php
<?php

namespace App\Providers;

use App\Services\PayPalService;

class AppServiceProvider extends ServiceProvider
{
    public function register()
    {
        $this->app->singleton('paypal', function () {
            // 根据环境选择配置
            $config = [];
            return new PayPalService($config);
        });
    }
}


#### 创建控制器

bash
$ php artisan make:controller PaymentsController


控制器代码示例：

php
<?php

namespace App\Http\Controllers;

use App\Models\Order;

class PaymentController extends Controller
{
    public function payByPayPalCheckout(Order $order)
    {
        if ($order->paid_at || $order->closed) {
            return json_encode(['code' => 422, 'msg' => 'Order Status Error.', 'url' => '']);
        }
        $approvalUrl = app('paypal')->checkout($order);
        if (!$approvalUrl) {
            return json_encode(['code' => 500, 'msg' => 'Interval Error.', 'url' => '']);
        }
        return json_encode(['code' => 201, 'msg' => 'success.', 'url' => $approvalUrl]);
    }
}


#### 支付后的回调方法

php
public function payPalReturn(Request $request)
{
    if ($request->has('success') && $request->success == 'true') {
        $payment = app('paypal')->executePayment($request->paymentId);
        // 处理支付完成后的业务
    }
}


### Testing Subscription

进一步实现定期订阅功能。

#### 创建计划并激活计划

php
public function createPlan(Order $order)
{
    try {
        // 创建并激活支付计划
    } catch (HttpException $e) {
        Log::error('PayPal Create Plan Failed', ['msg' => $e->getMessage(), 'code' => $e->getStatusCode(), 'data' => ['order' => ['no' => $order->no]]]);
        return false;
    }

    return $result;
}


#### 控制器调用及数据处理

创建一个 `SubscriptionsController` 来处理订阅相关的逻辑。

bash
$ php artisan make:controller SubscriptionsController


控制器中包括创建计划、订阅及异步回调的处理，涉及到的路由和中间件配置。

### 结论

通过上述步骤，我们详细解析了 PayPal 收银台与订阅模式的集成流程。希望本篇文章对开发者在实现支付系统的过程中有所帮助。

👉 [野卡 | 一分钟注册，轻松订阅海外线上服务](https://bit.ly/bewildcard)