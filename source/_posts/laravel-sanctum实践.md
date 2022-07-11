---
title: laravel-sanctum实践
date: 2022-07-11 23:29:25
tags: [laravel, sanctum, auth]
categories: laravel
---

## sanctum是官方新推出的认证组件

### laravel/sanctum

文档上表示，这个轻量化的 token 面向两类认证场景而生，思路起源为 Github 的 personal token 。

一是 为 api 令牌，发放长期 token ，认证时 在 header 头部传入。二为 基于 laravel api 的 spa 单页，将信息存入 cookie 。

**对比下 jwt 与 sanctum 或者叫自定义 token 的 方式**

| 认证方式 | 不需要数据库支持 | 多租户支持 | 便于管理 |  高性能   | 
| -------- | ---------------- | ---------- | -------- | --- |
| jwt      | ✔️               | ✔️         | ❌       |  ✔️   |
| sanctum  | ❌               | ✔️         | ✔️       |  ❌   |

总之对于体量不大的应用内部使用 sanctum 没什么问题，要是做对外开放相关的业务还是建议 jwt 


#### token 加解密实现

很想吐槽的就是这个 token 的生成方式，如
`Bearer 195|JswKUO3O9Jsmh9ks5fNQoS1Qlk6Ub6KvJ137g00q`
为什么这么设计，其实看头部就知道，用主键ID开头来加速 查询，但是 缺点也很明显，一个是对外暴露了内部信息的主键，第二点是 默认使用 bigint为主键导致的 token 长度不固定。

看下内部的加解密实现
```php
# 加密 vendor/laravel/sanctum/src/HasApiTokens.php 
public function createToken(string $name, array $abilities = ['*'])  
{  
    $token = $this->tokens()->create([  
        'name' => $name,  
        'token' => hash('sha256', $plainTextToken = Str::random(40)),  
        'abilities' => $abilities,  
    ]);  
    
	return new NewAccessToken($token, $token->getKey().'|'.$plainTextToken);  
}

# 解密 vendor/laravel/sanctum/src/PersonalAccessToken.php
public static function findToken($token)  
{  
    if (strpos($token, '|') === false) {  
        return static::where('token', hash('sha256', $token))->first();  
    }  
    [$id, $token] = explode('|', $token, 2);  
  
    if ($instance = static::find($id)) {  
        return hash_equals($instance->token, hash('sha256', $token)) ? $instance : null;  
    }}

```

其核心 代码 也就是
`hash('sha256', $plainTextToken = Str::random(40))` 与  `hash_equals`

数据库存的 token 是 随机字符串后 hash 出来的 64位字符串，对外暴露的为 `主键ID + | + 40位随机字符串`


#### 可选优化思路

##### token 多端通用

根据其生成方法 `createToken(string $name, array $abilities = ['*'])` 可知，数据库字段 `name` 与 `abilities` 都可以实现类似于多租户/多端 的功能，特别是 `abilities` 这个字段，可以做到更加精细的权限控制
比如 生成的时候
```php
# 生成 一个能控制 order 与 goods 模块的 admin.token
$user->createToken('admin',["order","goods"]);

# 验证 这个属性
auth('admin')->user()->currentAccessToken()->abilities;

```
简单一点的直接用 `$name`  控制 多端也ok, 因为使用了数据库，所以扩展性也还 OK

##### 发放 token 有效期控制

因为 官方 给出的过期时间很死板，没法做到 不同类型的 token 有不同的有效期，但还好也提供了一个回调函数来弥补

```php
#vendor/laravel/sanctum/src/Guard.php
protected function isValidAccessToken($accessToken): bool  
{  
	// ...... 重点在这
  
    if (is_callable(Sanctum::$accessTokenAuthenticationCallback)) {  
        $isValid = (bool) (Sanctum::$accessTokenAuthenticationCallback)($accessToken, $isValid);  
    }  
    return $isValid;  
}
```
重点就在 `Sanctum::$accessTokenAuthenticationCallback` 这个回调的属性上，
我们可以自定义这个回调函数 ，比如在 `AuthServiceProvider>boot()` 内，通过
`Sanctum::authenticateAccessTokensUsing` 来注入我们自定义的有效期检查逻辑

##### hook 每次请求的更新操作 , 来减少数据库消耗

我们可以通过 sql 监听发现，每次使用 sanctum , 都会产生两条 sql 记录，一条 通过id查询的sql,另一条为
```sql
update `personal_access_tokens` set `last_used_at` = '2022-07-01 19:03:02', `personal_access_tokens`.`updated_at` = '2022-07-01 19:03:02' where `id` = 195 {"time":29.06}
```
看到这个执行时间就知道，这个 sql 肯定会成为整个系统的性能消耗点，现在让我们看看，这是在哪里触发的

```php
#vendor/laravel/sanctum/src/Guard.php
public function __invoke(Request $request)  
{

//................... 调用触发点
  
        if (method_exists($accessToken->getConnection(), 'hasModifiedRecords') &&  
            method_exists($accessToken->getConnection(), 'setRecordModificationState')) {  
            tap($accessToken->getConnection()->hasModifiedRecords(), function ($hasModifiedRecords) use ($accessToken) {  
                $accessToken->forceFill(['last_used_at' => now()])->save();  
  
                $accessToken->getConnection()->setRecordModificationState($hasModifiedRecords);  
            });        } else {  
            $accessToken->forceFill(['last_used_at' => now()])->save();  
        }  
        return $tokenable;  
}
```

问题就转换成了 对于
`$accessToken->forceFill(['last_used_at' => now()])->save()` 
这个代码的优化，很显然，好多时候我们并不需要 通过这种方式去，更新 `last_used_at` 字段，甚至不客气的说，这有点过度设计，因为这个字段基本用不到，提供一个可选的回调属性，来替换这一个操作更加合理。

个人提供的思路为 ，替换原来 model
```php
# app/Providers/AppServiceProvider.php > boot() 中 替换原来 model

Sanctum::usePersonalAccessTokenModel(\App\Models\PersonalAccessToken::class);

#然后在 新建的 model 中 继承重写 forceFill()
public function forceFill(array $attributes)  
{  
    //hook $accessToken->forceFill(['last_used_at' => now()])->save(); 这个操作  
    if (count($attributes) === 1 && isset($attributes['last_used_at'])){  
		return $this;  
    }
    
    return static::unguarded(function () use ($attributes) {  
        return $this->fill($attributes);  
    });
}
```

这样，去掉了对于 `last_used_at` 的更新操作，每次请求就只有一次 sql 操作了，至于需要属性的，可以自行调整更新频率

- 最后可能要去优化的点，对于每次的通过ID查整条token的操作，很明显是一个读多写少的场景，可以考虑上缓存去替换这个查询 sql ，但是不是极限场景（秒杀之类的），我不建议这么去做，收益太少了~