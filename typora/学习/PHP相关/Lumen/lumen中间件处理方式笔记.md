# lumen中间件处理方式笔记

路由表添加中间件路由
```php
$router->group(['middleware' => ['api:oa']], function () use ($router) {
    $router->get('/transferApply', 'TransferApplyController@Index');
});
```
`api`代表中间件的名字:
```php
$app->routeMiddleware([
     'api' => App\Http\Middleware\Authenticate::class,
     'admin' => App\Http\Middleware\AdminMiddleware::class,
     'auth' => App\Http\Middleware\AuthMiddleware::class,
 ]);
```
`oa`代表配置的auth的键:
```php
'guards' => [
        'api'   => ['driver' => 'api'],
        'admin' => ['driver' => 'admin'],
        'oa'    => ['driver' => 'oa'],
    ],
```

```php
//$guard代表驱动类型
public function handle($request, Closure $next, $guard = null)
    {
        if ($this->auth->guard($guard)->guest()) {
            return response('无城市oa访问权限', 401);
        }

        return $next($request);
    }
```

```php
//oa代表驱动类型
$this->app['auth']->viaRequest('oa', function ($request) {
            $userInfo = app('authCommon')->checkLoginCookie($request);
            if ($userInfo) {
                $request['userInfo'] = [
                    'passid' => $userInfo['i'],
                    'uid' => $userInfo['u'],
                    'username' => $userInfo['n'],
                ];
                return User::where('uid', $userInfo['u'])->first();
            }
        });
```
