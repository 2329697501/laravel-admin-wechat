# Laravel-admin-wechat

Laravel admin 的微信扩展、支持多公众号、多小程序、多微信支付的后台管理，并提供小程序、微信支付的基础接口，在此基础上通过[事件](https://learnku.com/docs/laravel/6.x/events/5162)、继承等形式完成自定义。

本扩展使用了 [EasyWeChat](https://www.easywechat.com/)，微信实例使用可移步到 [EasyWeChat 文档](https://www.easywechat.com/docs)

## 安装

安装依赖

`composer require hanson/laravel-admin-wechat:dev-master -vvv`

安装

`php artisan wechat:install -m`

此命令将：
* 发布 WeChat 所需资源
* 生成微信相关后台菜单
* 创建各个微信相关数据表
* 创建路由文件 `routes/wechat_admin` 与 `routes/wechat_api` 
* 创建 `database/migrations` 的相关微信数据库（可自行根据需求做对应修改，可以加字段，不建议删减字段）
* 执行 `migrate` 操作（去掉 `-m` 可不执行）

## 配置

修改 `config/auth.php` （用于小程序登录等接口，如果不需要可以不加）

```php
<?php

return [
    'guards' => [
        // ...
        'mini' => [
            'driver' => 'jwt',
            'provider' => 'wechat_user',
        ]
    ],
    'providers' => [
        // ...
        'wechat_user' => [
            'driver' => 'eloquent',
            'model' => Hanson\LaravelAdminWechat\Models\WechatUser::class, // 你也可以自己继承此 model 后修改为自己的 model
        ],
    ]
];
```

## TO DO

### 后台

- [x] 公众号与小程序配置
- [x] 公众号用户 
- [x] 公众号菜单 
- [x] 小程序用户 
- [x] 微信支付配置 
- [ ] 公众号卡券 
- [ ] 公众号门店
- [ ] 公众号模板消息
- [ ] 公众号素材
- [ ] 公众号客服
- [ ] 开放平台
- [ ] 微信支付订单
- [ ] 微信支付退款
- [ ] 微信支付红包
- [ ] 小程序其他解密接口

## 接口

对于本人来说， `laravel-admin-wechat` 另一个有价值的点在于自带的接口，尽管内容不多，但因为做项目比较多经常要新建用户表，写登录逻辑，但实际上代码基本都一样，这也是为什么会提供基础的接口

 * `post` `api/wechat/mini/check-token` 检查token是否过期 
 
 * `post` `api/wechat/mini/login` 使用 code 登录 
 
 |  参数 |  备注 |
 |---|---|
 |  app_id |  小程序的 app id |
 |  code |  登录的 code |

 * `post` `api/wechat/mini/decrypt-mobile` 解密手机号码 
 
 * `post` `api/wechat/mini/decrypt-user-info` 解密用户信息 
 
 |  参数 |  备注 |
 |---|---|
 |  app_id |  小程序的 app id |
 |  iv |  微信参数 |
 |  encrypted_data |  微信参数 |

## 高级

此扩展只提供了最基础的业务，但很多情况下企业需要更多样化的业务需求，`laravel-admin-wechat` 同样提供了十分灵活的自定义方案。

### 自定义后台

后台路由在 `routes/wechat_admin.php` 中，你可以自由修改

当你需要对进行细微调整时，可以通过 `php artisan admin:controller` 自行创建控制器，并修改其继承的类为原来的类，覆盖方法做调整

### 通用方法

`laravel-admin-wechat` 的通用函数均在 `Hanson\LaravelAdminWechat\Services` 内,并提供 `Facade` 方式进行调用

```php
<?php
use \Hanson\LaravelAdminWechat\Facades\ConfigService;
use \Hanson\LaravelAdminWechat\Facades\MerchantService;
use \Hanson\LaravelAdminWechat\Facades\OrderService;

// ConfigService 可获取 公众号/小程序 实例
ConfigService::getCurrent(); // 获取后台操作中的 WechatConfig 对象
ConfigService::getAdminCurrentApp(); // 获取后台操作中的微信实例
ConfigService::getInstanceByAppId('app id'); // 根据 appid 获取微信实例

// MerchantService 可获取 微信支付实例
MerchantService::getInstanceByMchId('mch id'); // 根据 mch id 获取微信支付实例

// OrderService 订单相关服务
OrderService::unify('mch id', 'JSAPI', array $data); // 统一下单并创建微信订单 data 为统一下单参数，与微信支付文档一致
OrderService::jsConfig('mch id', 'JSAPI', array $data); // 返回 js sdk 所需参数（其中包括统一下单，创建订单）
```

### 事件

为了能够实现基础业务外，也能更好的适应各种自定义需求，本扩展使用了事件去实现自定义

在你的 `app/Providers/ServiceProvider.php` 中

```php
<?php

protected $listen = [
    \Hanson\LaravelAdminWechat\Events\DecryptUserInfo::class => [
        'App\Listeners\AfterSaveUserInfo',
    ],
    \Hanson\LaravelAdminWechat\Events\DecryptMobile::class => [
        'App\Listeners\SaveMobile',        
    ],
    \Hanson\LaravelAdminWechat\Events\OrderPaid::class => [
        'App\Listeners\ChangeOrderStatus',
    ]
];
```

```php
<?php
use \Hanson\LaravelAdminWechat\Events\DecryptMobile;

class SaveMobile 
{
    public function handle(DecryptMobile $event)
    {
        $event->wechatUser->user()->update([
            'phone' => $event->decryptedData['purePhoneNumber'],
            'country_code' => $event->decryptedData['countryCode'],
        ]);
    }
}
```

```php
<?php
use \Hanson\LaravelAdminWechat\Events\DecryptUserInfo;

class AfterSaveUserInfo 
{
    public function handle(DecryptUserInfo $event)
    {
        // 你的业务
        $event->decryptedData['nickname'];
        $event->wechatUser;
    }
}
```

```php
<?php
use \Hanson\LaravelAdminWechat\Events\OrderPaid;

class AfterSaveUserInfo 
{
    public function handle(OrderPaid $event)
    {
        // 你的业务
        $wechatOrder = $event->order;
        
        $order = $wechatOrder->order()->update(['status' => 'paid']);
        
        $openId = $wechatOrder->openid;
    }
}
```

### 微信支付

`laravel-admin-wechat` 提供了微信订单表、创建订单以及生成 js 参数等方法，但并没有相关业务参数 `地址`、`商品` 等，建议自身生成 `orders` 表并关联 `wechat_orders`

```php
// 支付接口示例
<?php

class OrderController extends Controller
{
    public function pay()
    {
        // some validate
        
        $data = [
            'body' => '商品标题', 
            'total_fee' => 100,
            'openid' => auth('mini')->user()->openid,
            // 'out_trade_no' => 'xxx', 选填，如不填写时会自动创建一个订单号
        ];
        
        /**
        * $result['config'] jssdk 所需参数
        * $result['order'] WechatOrder 的 model 对象
        * $result['unify'] unify 接口返回的结果
        */
        $result = \Hanson\LaravelAdminWechat\Facades\OrderService::jsConfig('mch id', 'JSAPI', $data);
        
        App\Models\Order::create([
            'wechat_order_id' => $result['order']->id,
            'status' => 'not paid',
            'goods_id' => '...',
            'address_id' => '...',
        ]);
        
        return $result;
    }
}
```

## 特别鸣谢

[EasyWeChat 微信开发包](https://github.com/overtrue/wechat)

[yisonli/wxmenu 微信菜单的代码来源](https://github.com/yisonli/wxmenu)

## 定制

如需找我定制，可加我微信 524291355
