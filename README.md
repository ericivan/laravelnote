# 我在使用laravel passport 挖的坑

### laravel 版本:5.6 

> 安装

```shell
composer install laravel/passport
```

> 接着就是执行包的指令,安装passport以及做数据库的迁移

```shell
 php artisan passport:install

 php artisan migrate
```


> 然后修改auth.php的配置，把认证修改成passport,主要是做api的认证配置

```php
'api' => [
            'driver' => 'passport',
            'provider' => 'users',
        ],
```

#### 官方文档涉及到的东西太多，但是其实我不需要使用到那么多的内容，所以我自己挑了一部分出来

> 认证中间件

```php
  namespace App\Http\Middleware;

use App\Traits\ResponseTrait;
use Closure;
use Illuminate\Contracts\Auth\Factory as Auth;

class Authenticate
{
    use ResponseTrait;
    /**
     * The authentication factory instance.
     *
     * @var \Illuminate\Contracts\Auth\Factory
     */
    protected $auth;

    /**
     * Create a new middleware instance.
     *
     * @param  \Illuminate\Contracts\Auth\Factory  $auth
     * @return void
     */
    public function __construct(Auth $auth)
    {
        $this->auth = $auth;
    }

    /**
     * Handle an incoming request.
     *
     * @param  \Illuminate\Http\Request  $request
     * @param  \Closure  $next
     * @param  string[]  ...$guards
     * @return mixed
     *
     * @throws \Illuminate\Auth\AuthenticationException
     */
    public function handle($request, Closure $next, ...$guards)
    {
        $this->authenticate($guards);

        return $next($request);
    }

    /**
     * Determine if the user is logged in to any of the given guards.
     *
     * @param  array  $guards
     * @return void
     *
     * @throws \Illuminate\Auth\AuthenticationException
     */
    protected function authenticate(array $guards)
    {
        if (empty($guards)) {
            return $this->auth->authenticate();
        }

        foreach ($guards as $guard) {
            if ($guard == 'api') {
                $token = request()->header('Authorization');

                if (is_null($token) && request('token')) {
                    request()->headers->set('Authorization', 'Bearer ' . request('token'));
                }
            }

            if ($this->auth->guard($guard)->check()) {
                $this->auth->shouldUse($guard);

                return true;
            }
        }

        $this->error(10000, 401, 'Unauthenticated');
    }
}
```

> 注入了Auth对象，使用里面的写好的认证方法认证，具体操作可以自己跳到里面看方法,里面东西还是挺多的


### 接着是，控制器,下面的代码只是简单的认证，创建token

```php

namespace App\Http\Controllers\Auth;

use App\Http\Controllers\Controller;

class AuthController extends Controller{

  public function login(){
    if (Auth::attempt([
                'email' => request('email'),
                'password' => request('password')]
        )) {
            $user = Auth::user();
            $token = $user->createToken('Bitizens');
        } else {

            $this->error(10000, 401, 'Unauthenticated');
        }

        return $this->success([
            'token' => $token->accessToken,
        ]);
  }
}

```

### 上面的使用方式简单暴力，但是过期时间，refresh token的东西都没有，因为passport这个包本来就包含了控制器跟路由，所以我参考了一下它们里面的代码，自己封装了一层，只用于token创建这一块，其他功能可以自己自行设计

```php


namespace App\Http\Controllers\Auth;

use App\Http\Controllers\Controller;
use Illuminate\Support\Facades\Auth;
use Laravel\Passport\Http\Controllers\AccessTokenController;
use Psr\Http\Message\ServerRequestInterface;

class LoginController extends Controller
{
    /*
    |--------------------------------------------------------------------------
    | Login Controller
    |--------------------------------------------------------------------------
    |
    | This controller handles authenticating users for the application and
    | redirecting them to your home screen. The controller uses a trait
    | to conveniently provide its functionality to your applications.
    |
    */

    /**
     * Where to redirect users after login.
     *
     * @var string
     */
    protected $redirectTo = '/home';

    private $server;

    /**
     * Create a new controller instance.
     *
     * @return void
     */
    public function __construct(
        AccessTokenController $server
    )
    {
        $this->server = $server;
        $this->middleware('auth:api')->except('login','login2');
    }


    public function login2(ServerRequestInterface $request)
    {
        $request = $request->withParsedBody([
            'username' => array_get($request->getParsedBody(), 'address', 'no-username'),
            'password' =>  array_get($request->getParsedBody(), 'password', 'no-password'),
            'grant_type' => 'password',
            'client_id' => 1,
            'client_secret' =>'8BsHzOpt6om63slHKp8wKdd9FEvYWcgoCilMORSH',
            'scope' => ''
        ]);

        $issutToken= $this->server->issueToken($request);

        if ($issutToken->getStatusCode() == 200) {
            return $this->success(json_decode($issutToken->getContent(),true));
        }

        return $this->error(10000);


    }

    public function getUser()
    {
        dd('getUser Cross Auth');
    }
}

```

> 上面的接口会返回完整的token信息，token的过期时间可以再AuthServiceProvider里面

```php
namespace App\Providers;
use Illuminate\Foundation\Support\Providers\AuthServiceProvider as ServiceProvider;
use Laravel\Passport\Passport;

class AuthServiceProvider extends ServiceProvider
{
    /**
     * The policy mappings for the application.
     *
     * @var array
     */
    protected $policies = [
        'App\Model' => 'App\Policies\ModelPolicy',
    ];

    /**
     * Register any authentication / authorization services.
     *
     * @return void
     */
    public function boot()
    {
        $this->registerPolicies();

        Passport::tokensExpireIn(now()->addHour(48));

        Passport::refreshTokensExpireIn(now()->addDays(30));
    }
}

```

### 个人而言，没有用到refreshtoken这一块，passport自带的路由里面也是有这段逻辑的

### 最后一个就是 provider model的问题，有时候一些验证方法我们想自己自定义，首先说一下model的基本定义

> 首先，model里面要使用一个trait HasApiTokens，然后Model 继承的是 Authenticatable，并不是Model基类

#### 然后我想自定义查找用户的方法，这时候需要在model里面写一个方法，public function findForPassport($username),这是我看源码的时候发现他会检测一下模型里面是否有这个方法，有的话就会执行这个方法来查找用户

```php
 public function findForPassport($username)
    {
        return self::where('address', $username)->first();
    }

```

### 获取密码字段，有时候我们的密码字段并不是叫password，model里面重写这么一个方法

```php
 public function getAuthPassword()
    {
        return $this->attributes['hash'];
    }
```


### 最后就是密码验证这一段，我们也可以自己写，我这里比较暴力，直接return true了

```php
 public function validateForPassportPasswordGrant($password)
    {
        return true;
    }
    
```

#####
