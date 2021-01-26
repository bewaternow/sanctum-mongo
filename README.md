# sanctum-mongo

## 前言

官方自带的已经很好，但是我需要一个对 mongo 友好的。鉴于发授令牌的方法有各种限制，我需要重新写一个类来继承 `Jenssegers\Mongodb\Eloquent\Model` ，所以我网站找了一圈，都没有，索性自己搞一个了。

## 安装 mongo 的拓展

```
composer require jenssegers/mongodb
```

添加服务容器到 config/app.php

```
<?php
Jenssegers\Mongodb\MongodbServiceProvider::class,

//  alias
'Mongo'  => Jenssegers\Mongodb\MongodbServiceProvider::class

//  修改database.php
'default' => env('DB_CONNECTION', 'mongodb'),    //修改

'your_mongodb_name' => [
    'driver'   => 'mongodb',
    'host'     => 'localhost',
    'port'     => 27017,
    'database' => env('DB_DATABASE', 'forge'),
    'username' => env('DB_USERNAME', 'forge'),
    'password' => env('DB_PASSWORD', ''),
],     
```

## 安装 zach/sanctum-mongo

```
composer require zach/sanctum-mongo
```

接下来，你需要使用 `vendor:publish` Artisan 命令发布 Sanctum 的配置和迁移文件。Sanctum 的配置文件将会保存在 `config` 文件夹中：

```
php artisan vendor:publish --provider="Zach\Sanctum\SanctumServiceProvider"
```

最后，你需要执行数据库迁移文件。Sanctum 将创建一个数据库表用于存储 API 令牌：

```
php artisan migrate
```

假如你需要使用 Sanctum 来验证 SPA，你需要在 `app/Http/Kernel.php` 文件中将 Sanctum 的中间件添加到你的 `api` 中间件组中：

```
<?php

use Zach\Sanctum\Http\Middleware\EnsureFrontendRequestsAreStateful;

'api' => [
    EnsureFrontendRequestsAreStateful::class,
    'throttle:60,1',
    \Illuminate\Routing\Middleware\SubstituteBindings::class,
],
```

## 发行 API 令牌

可以使用 Sanctum 发行 API 令牌 / 个人访问令牌 对你的 API 请求进行认证。 当使用 API 令牌进行请求的时，令牌可以以 Bearer 的形式包含在 Authorization header 头里。



给用户发行令牌的时候，User 模型里应该使用 HasApiTokens trait :

```
<?php

namespace App\Models;

use Illuminate\Contracts\Auth\MustVerifyEmail;
use Illuminate\Database\Eloquent\Factories\HasFactory;
use Jenssegers\Mongodb\Auth\User as Authenticatable;
use Illuminate\Notifications\Notifiable;
use Zach\Sanctum\HasApiTokens;

class User extends Authenticatable
{
    use HasApiTokens, HasFactory, Notifiable;

    /**
     * The attributes that are mass assignable.
     *
     * @var array
     */
    protected $connection = 'your_mongodb_name';
}
```

## 方法的重写

### 第一步 创建模型

下面是一个示例，我创建在了 `App\Models` 下面，文件名 `SanctumPersonalAccessModel` ：

```
<?php

namespace App\Models;
use Zach\Sanctum\PersonalAccessToken;

class SanctumPersonalAccessModel extends PersonalAccessToken
{
    protected $connection = 'your_mongodb_name';
    protected $table = 'personal_access_tokens';
}
```

### 第二步 模型替换

在 `Providers` 目录下，找到 `AuthServiceProvider` ，添加模型替换的代码，如下：

```
<?php

namespace App\Providers;

use Illuminate\Foundation\Support\Providers\AuthServiceProvider as ServiceProvider;
use Illuminate\Support\Facades\Gate;
use Zach\Sanctum\Sanctum;
use App\Models\SanctumPersonalAccessModel as PersonalAccessModel;

class AuthServiceProvider extends ServiceProvider
{
    /**
     * The policy mappings for the application.
     *
     * @var array
     */
    protected $policies = [
        // 'App\Models\Model' => 'App\Policies\ModelPolicy',
    ];

    /**
     * Register any authentication / authorization services.
     *
     * @return void
     */
    public function boot()
    {
        $this->registerPolicies();
        //  自定义模型
        Sanctum::usePersonalAccessTokenModel(PersonalAccessModel::class);
    }
}
```

## 最重要的事

希望各位一起参与到维护中来，不然一个人搞的精力不足。
