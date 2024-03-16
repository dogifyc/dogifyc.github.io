---
title: Laravel 利用 doctrine/annotations 在注解中实现角色权限控制
date: 2024-03-16 22:28:29
tags:
  - laravel
  - rbac
---

## 基本术语

项目中基于常见的角色和权限方式实现权限控制，下面介绍一些术语。

- 用户
	- 登录使用系统的自然人
- 角色
	- 系统中用户的身份，比如管理员，普通用户，不同的角色权限不同，同一用户可以有任意多个角色
- 权限
	- 用户行为，具体为做某件事的能力，有权限即能做此事。一个角色可以有任意项权限，拥有多个角色的用户即拥有这些所有权限的并集。

## 简单用法

本文实现的权限控制只要是针对API进行权限控制，控制力度分3级。具体的实现方法是在控制器的方法注释中添加注解。注解会在 Laravel 的中间件中被读取，然后判断是否有权限。没有权限则会返回统一的报错。

<!--more-->
以下是三级力度的控制方法。
1. 公开API，未填写权限注解的是公开API，无需登录即可访问
    ```php
    /**
     * 获取icp备案号
     * @param Request $request
     * @return \Illuminate\Http\JsonResponse
     */
    public function icp(Request $request)
    {
        ...
    }
    ```
2. 登录后才可访问的 API，@Permission()注解指定当前路由需要登录后才可访问
    ```php
    /**
     * 省
     * @Permission()
     * @param Request $request
     * @return \Illuminate\Http\JsonResponse
     */
    public function states(Request $request)
    {
        ...
    }
    ```
3. 登录用户必须具有特定权限才可访问，当前路由要求登录用户必须具有add_customer权限。
    ```php
    /**
     * 新建客户
     * @Permission(action="add_customer")
     * @param Request $request
     * @return \Illuminate\Http\JsonResponse
     * @throws \Illuminate\Validation\ValidationException
     */
    public function add(Request $request)
    {
        ...
    }
    ```

## 表结构

```sql
/*
*用户表
*/
create table users
(
    id                bigint unsigned auto_increment
        primary key,
    name              varchar(12)   default '' not null comment '昵称',
    password          varchar(128)             not null comment '密码',
    avatar_path       varchar(2048) default '' not null comment '头像存储地址',
    is_super_admin    tinyint(1)    default 0  not null comment '是否超级管理员',
    is_enable         tinyint(1)    default 1  not null comment '是否启用',
    remark            varchar(1024) default '' not null comment '备注',
    created_at        timestamp                null,
    updated_at        timestamp                null,
    deleted_at        timestamp                null,
    constraint users_phone_unique
        unique (phone)
) collate = utf8mb4_unicode_ci;

/*
*角色表
*/
create table roles
(
    id             bigint unsigned auto_increment
        primary key,
    name           varchar(32)              not null comment '名称',
    `desc`         varchar(1024) default '' not null comment '简介',
    create_user_id int           default 0  not null comment '创建人id',
    created_at     timestamp                null,
    updated_at     timestamp                null,
    deleted_at     timestamp                null
) collate = utf8mb4_unicode_ci;

/*
* 权限表
*/
create table permissions
(
    id         bigint unsigned auto_increment
        primary key,
    action     varchar(255)  not null comment '权限标识',
    name       varchar(255)  not null unique comment '名称',
    parent_id  int default 0 not null comment '权限id',
    created_at timestamp     null,
    updated_at timestamp     null,
    constraint permissions_action_unique
        unique (action),
    constraint permissions_name_unique
        unique (name)
) collate = utf8mb4_unicode_ci;

/*
* 角色权限中间表
*/
create table role_permission_pivot
(
    role_id       int not null comment '角色id',
    permission_id int not null comment '权限id'
) collate = utf8mb4_unicode_ci;

/*
* 用户角色中间表
*/
create table user_role_pivot
(
    role_id int not null comment '角色id',
    user_id int not null comment '用户id'
) collate = utf8mb4_unicode_ci;
```

## 核心实现
Laravel 提供了中间件的概念，可以为统一为某些url添加前置或后置函数处理。本文使用了前置中间件。

1. 引入 doctrine/annotations 依赖
    `composer require doctrine/annotations ^1.10`
2. 创建前置中间件
    `php artisan make:middleware AnnotationCheck
3. 在handle 函数中添加处理逻辑
    ```php
    public function handle($request, Closure $next)
    {
        $route = $request->route();
        if (empty($route)) {
            return $this->respond(500, '路由信息为空');
        }
        $controller = $route->getController();
        try {
            // 反射获取目标控制器对象
            $class = new \ReflectionClass($controller);
            // 反射获取目标控制器的目标方法
            $method = $class->getMethod($route->getActionMethod());
            AnnotationRegistry::registerFile(app_path('Annotations/Permission.php'));
            $reader = new AnnotationReader();
            foreach ($reader->getMethodAnnotations($method) as $annotation) {
                if ($annotation instanceof Permission) {
                    if (!auth()->check()) {
                        return $this->respond(401, '用户未登录');
                    }
                    if (strlen($annotation->action) == 0) {
                        return $next($request);
                    }
                    return auth()->user()->can($annotation->action) ? $next($request) : $this->respond(403, '权限不足');
                }
            }
            return $next($request);
        } catch (\Exception $e) {
            return $this->respond(500, $e->getMessage());
        }
    }
    ```
5. 在 `app\Http\Kernel.php` 中的 `$routeMiddleware` 数组中添加注解中间件的注册，这样就可以在路由配置文件中使用此中间件了。
```php
protected $routeMiddleware = [
        ...
        'verified' => \Illuminate\Auth\Middleware\EnsureEmailIsVerified::class,
        'annotation' => AnnotationCheck::class, // 添加这一行
    ];
```

6. 在 `routes/api.php` 中使用此中间件
```php
Route::prefix('admin')->namespace('Admin')->middleware('annotation')->group(function () {
    Route::post('user/resetPassword', 'UserController@resetPassword');
    // 省略其它路由
}
```
