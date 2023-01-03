# 使用 OpenWRT 的白名单 SSH 访问

> 原文：<http://web.archive.org/web/20220810161336/https://imrannazar.com/Whitelist-SSH-Access-with-OpenWRT>

有时，我会冒昧地查看服务器的日志文件。不变的是，在底部有类似下面的东西:

#### SSH 日志文件示例

```
May  8 01:46:43 adhocbox sshd[28514]: Invalid user tanta from 61.100.x.x
May  8 01:46:46 adhocbox sshd[28516]: Invalid user cornel from 61.100.x.x
May  8 01:46:49 adhocbox sshd[28518]: Invalid user ronaldo from 61.100.x.x
May  8 01:46:51 adhocbox sshd[28520]: Invalid user wave from 61.100.x.x
May  8 01:46:54 adhocbox sshd[28522]: Invalid user vanilla from 61.100.x.x
May  8 01:46:57 adhocbox sshd[28524]: Invalid user ice from 61.100.x.x
May  8 01:47:02 adhocbox sshd[28526]: Invalid user mason from 61.100.x.x
```

这种情况重复了几百行，几个小时后又有另一批来自另一个 IP 的攻击。这些登录尝试占用了一个人的带宽，这很烦人，尤其是当服务器只有一个有效用户时，我的设置就是这种情况。

我决定通过实施白名单来结束这种情况:允许特定 IP 通过防火墙，阻止所有其他 IP。幸运的是，我在我的互联网路由器上运行了 OpenWRT 安装，它为操纵防火墙规则提供了 Linux 的`iptables`基础设施。在这篇文章中，我将详细介绍我如何建立我的白名单系统，以及你如何做同样的事情。

### 你需要什么

我的网络由一个 Linksys WRT54G 无线路由器托管防火墙和一个运行 Linux 发行版的 web 服务器组成。对于这个设置，web 服务器的细节不是问题，但是您需要:

OpenWRT:

My Linksys router has been reflashed with OpenWRT Whiterussian, which provides the `iptables` firewall, along with simplification scripts for the firewall rules. Any Linux box can act as the firewall, as long as you can pull the appropriate formatting together for the rules.

PHP with PECL-SSH2:

OpenWRT provides SSH access to the router, which allows for direct editing of the firewall configuration file. We'll be using this to our advantage, by programmatically adding IPs to the whitelist using PHP.

An external computer:

The easiest way to test the whitelist setup is by using a computer that's outside the LAN; this will allow you to check that packets are being appropriately blocked at the router, which will not necessarily be the case if you're going between computers inside the LAN.

### OpenWRT 防火墙

OpenWRT 在 Linux `iptables`接口上提供了一个简单的包装器，使用`awk`将配置文件的内容重写为过滤和 NAT 规则，然后由 init 脚本应用这些规则。在此之上还有一个包装器，它构成了防火墙的 Web 接口；大多数人将这个接口与 OpenWRT 防火墙联系在一起。

Web 界面的主要问题是它相对笨拙，尤其是在改变防火墙规则的顺序时；新规则被添加到列表的底部，将它们移动到顶部需要一系列费力的点击和页面加载。在大多数情况下，直接编辑配置文件更有意义。

简单配置的规则中可能包含以下内容:

#### OpenWRT 的/etc/config/firewall:一个示例

```
accept:dport=113 src=192.168.0.0/24
forward:proto=tcp dport=22:192.168.0.1:22
forward:proto=tcp dport=80:192.168.0.1:80
drop
```

这个示例脚本将允许防火墙`accept`识别来自局域网内部的请求、`forward` SSH 和 HTTP 到位于 192.168.0.1 的服务器，以及`drop`其他一切。每个规则的参数由 init 脚本解析出来，并构建到`iptables`规则中。

与`iptables`一样，这些规则是按顺序处理的，并且应用第一个匹配传入数据包的规则。通过使用这一原则，很容易将一个规则集放在一起，作为 SSH 的白名单:

#### 将 SSH 列入白名单:防火墙规则集

```
forward:proto=tcp src=[IP #1] dport=22:192.168.0.1:22
forward:proto=tcp src=[IP #2] dport=22:192.168.0.1:22
drop:proto=tcp dport=22
```

在本例中，来自特定外部 IP 的任何 SSH 数据包都将被转发到 SSH 服务器，任何其他 SSH 数据包都将在防火墙处被丢弃。这是允许白名单的行为:下一个问题是如何将 IP 添加到名单中。

### 将 IP 添加到白名单

有两种方法可以将地址添加到此防火墙规则集。第一种是通过 SSH 进入 OpenWRT 路由器，编辑配置文件以添加适当的规则，然后重新启动防火墙服务:

#### 手动更新白名单

```
ssh -l root 192.168.0.254
vim /etc/config/firewall
/etc/init.d/S45firewall restart
```

这种方法的问题是双重的:

Ease of use:

This manual method of updating the list doesn't constitute the most user-friendly interface to addition of IPs, and it can get tiresome to add IPs months or years after the whitelist is initially put into place.

Access:

Almost exclusively, access to the router's SSH port is only available from inside the LAN. From an external viewpoint, this availability will only exist by connecting from the accessible SSH server residing on the LAN. This in turn is governed by the whitelist, held on the router. The eponymous Catch-22 situation is an apt description of this problem.

除了使用手动流程来更新列表，还可以提供一个外部可访问的界面来添加 IP。在我的例子中，我有一个 Web 服务器(恰好是我的 SSH 服务器)，所以我可以使用一个 Web 脚本来提供这个接口；出于本文的目的，PHP 被用作完成这项工作的语言。

PHP 默认没有 SSH 版本 2 的接口:这是由一个名为`ssh2`的 PECL 扩展提供的。一旦设置好了，就可以使用各种方法来建立 SSH 连接。这些可用于在 OpenWRT 路由器上执行工作:

#### 使用 PECL_ssh2 连接到路由器

```
$ssh = ssh2_connect('192.168.0.254');
if(ssh2_auth_password($ssh, 'root', '[router root passwd]'))
{
	$stream = ssh2_shell($ssh);
	fwrite($stream, 'touch /tmp/newfile');
}
```

顺便说一下，如果您不喜欢将路由器的根密码放在 PHP 文件中，PECL 的 ssh2 扩展也提供了公钥认证机制，OpenWRT 安装上的 ssh 服务器允许以与 OpenSSH 相同的方式添加公钥。

### 使用 PHP 自动添加 IP

用`ssh2_shell`打开一个交互式 shell 允许执行不止一个命令，这意味着我们可以进行向列表添加地址所需的文件操作。我们可以结合一切，产生下面的脚本。

#### ssh.php:将 IP 添加到路由器的白名单中

```
<?php
if(isset($_POST['add'])):
	$ssh = ssh2_connect('192.168.0.254');
	if(ssh2_auth_password($ssh, 'root', '[router root passwd]'))
	{
		$fp = ssh2_shell($ssh);
		fwrite($fp, 'echo "forward:proto=tcp src='.$_POST['ip'].' dport=22:192.168.0.1:22" > /tmp/1'."\n");
		fwrite($fp, "cp /etc/config/firewall /tmp/2\n");
		fwrite($fp, "cat /tmp/1 /tmp/2 > /etc/config/firewall\n");
		fwrite($fp, "/bin/sh /etc/init.d/S45firewall\n");

		// Provide enough time for the firewall to restart
		sleep(10);
	}
	echo "Done.";
else:
?>
<form method="post">
 <input type="text" name="ip">
 <input type="submit" name="add" value="Add">
</form>
<?php
endif;
?>
```

现在需要做的就是导航到这个脚本，在框中输入一个 IP，然后等待 10 秒钟。当此过程完成后，IP 会自动添加到防火墙脚本的顶部，防火墙会重新启动。

这应该是您为 SSH 建立自己的白名单访问列表所需的一切。不再对您的服务器进行暴力攻击！

*版权所有伊姆兰·纳扎尔<tf@oopsilon.com2008 年*

*文章日期:2008 年 5 月 9 日*