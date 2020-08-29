---
title: 理解laravel中的Ioc容器,依赖注入
categories:
- 设计模式
date: 2019-07-26 14:37:20
tags:
- laravel,设计模式
---

**反射:**   
PHP 5 开始 具有完整的反射 API，添加了对类、接口、函数、方法和扩展进行反向工程的能力。 此外，反射 API 提供了方法来取出函数、类和方法中的文档注释。
反射是一切框架的基础;     
参考 : https://www.php.net/manual/zh/book.reflection.php


首先我们来下下面这个例子:    
在laravel框架中,很多对象实例通过方法参数定义就能传递进来，调用的时候不需要我们自己去手动传入对象实例。
类似这样:     
```php
// routes/web.php
Route::get('/post/store', 'PostController@store');

// App\Http\Controllers
class PostController extends Controller {

    public function store(Illuminate\Http\Request $request)
    {
        $this->validate($request, [
            'category_id' => 'required',
            'title' => 'required|max:255|min:4',
            'body' => 'required|min:6',
        ]);
    }

}
```
如上, 我们在调用控制器中store的时候, Request类的实例是什么时候传入的了, 我们并没有手动传入该对象实例;

这便是依赖注入,Ioc容器和php反射来实现的;


看下面的利用ioc容器技术来实现记录用户登录日志的例子:
```sql
<?php

// 定义写日志的接口规范
interface Log
{
    public function write();
}

// 文件记录日志
class FileLog implements Log
{
    public function write()
    {
        echo 'file log write...' . PHP_EOL;
    }
}

// 数据库记录日志
class DatabaseLog implements Log
{
    public function write()
    {
        echo 'database log write...' . PHP_EOL;
    }
}

class User
{
    protected $fileLog;
    protected $time;
    protected $account;

    public function __construct(Log $log, $account, $time)
    {
        $this->fileLog = $log;
        $this->time = $time;
        $this->account = $account;
    }

    public function login()
    {
        echo $this->account . ' login success... at ' . $this->time . PHP_EOL;
        $this->fileLog->write();
    }
}

class Ioc
{
    public $binding = [];

    public function bind($abstract, $concrete)
    {
        $this->binding[$abstract]['concrete'] = function ($ioc, array $parameters) use ($concrete) {
            return $ioc->build($concrete, $parameters);
        };
    }

    public function make($abstract, $parameters = [])
    {
        $concrete = $this->binding[$abstract]['concrete'];
        return $concrete($this, $parameters);
    }

    public function build($concrete, array $parameters)
    {
        $reflector = new ReflectionClass($concrete); //https://www.php.net/manual/zh/book.reflection.php
        $constructor = $reflector->getConstructor();
        if (is_null($constructor)) {
            return $reflector->newInstance();
        } else {
            $dependencies = $constructor->getParameters();
            $instances = $this->getDependencies($dependencies, $parameters);
            return $reflector->newInstanceArgs($instances);
        }
    }

    protected function getDependencies($paramters, array $parameters)
    {
        $dependencies = [];
        foreach ($paramters as $paramter) {
            $class = $paramter->getClass()->name; //https://www.php.net/manual/en/reflectionparameter.getclass.php
            if (is_null($class)) {
                $dependencies[] = array_shift($parameters);
            } else {
                $dependencies[] = $this->make($class);
            }
        }
        return $dependencies;
    }

}

$ioc = new Ioc();
$ioc->bind('User', 'User');
$ioc->bind('Log', 'FileLog');
$user = $ioc->make('User', ['123@qq.com', date('Y-m-d H:i:s')]);
$user->login();

$ioc->bind('Log', 'DatabaseLog');
$user = $ioc->make('User', ['456@qq.com', date('Y-m-d H:i:s')]);
$user->login();
```