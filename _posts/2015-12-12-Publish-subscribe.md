---
layout: post
title:  "Publish Subscribe"
subtitle: "emtf"
categories: [design]
---


在讲解发布订阅的操作之前，首先一定要了解这几点：
(1) SQL Management Express 是不支持发布的，他只能操作订阅。参考微软地址：https://technet.microsoft.com/en-us/library/ms165686(v=sql.105).aspx
<img src="{{ site.baseurl }}/_posts/images/1.jpg">
(2) 发布和订阅操作必须在服务器上操作，远程是没办法操作的（也有可能是本人没找到正确的方法，如果有，希望告诉我sessionliang@outlook.com）。
(3) 发布和订阅如果不在同一台机器上，那么需要知道服务器的名称，然后做ip映射（后面详细说明为什么这么做，怎么操作）。

上面几点切记，都是泪啊。

下面开始。
SQL的订阅发布功能，在我理解，就是为了数据同步（当然，这只是在我的项目中是只做了这部分，其他的功能，读者自行了解）。

第一步，创建发布。
其实创建发布很简单，打开sql management，菜单：复制-本地发布-右键-添加发布，然后按照他的操作说明，下一步就可以了。
下一步的过程中可以设置：
（1）对那个数据库进行发布
（2）发布方式
（3）发布数据库中的那些表
（4）过滤条件（没有使用到）
（5）创建定时任务（可选）
<img src="{{ site.baseurl }}/_posts/images/2.jpg">

上面操作的过程中，代理安全性设置的时候，如果选择的是windows账户，那么需要写域\账户。域怎么获取？有个简答办法，打开sql management，使用windows身份登录服务器名称就是域。

<img src="{{ site.baseurl }}/_posts/images/3.jpg">
上面操作完成之后，发布就大功告成了。

第二步，创建订阅。

订阅的操作其实也很简单，使用windows身份认证登录sql menegement，打开菜单-复制-本地订阅-右键-新建订阅，下一步直接往下走就可以了。
这里又进行了哪些设置呢？思考一下，其实挺简单：
（1）我订阅谁的内容？（选择发布服务器）
<img src="{{ site.baseurl }}/_posts/images/4.jpg">
（2）订阅方式（推送，请求）这个自己研究有什么区别
（3）我自己的哪个数据库需要订阅他的内容？（选择订阅数据库）
<img src="{{ site.baseurl }}/_posts/images/5.jpg">
（4）然后设置好代理使用哪个账户登录
        这里有一个小问题，当你设置好账户之后，想确定完成操作，但是你会发现，在最下面的三个按钮是被挡着的，你完全没明白哪个是确定，哪个是取消。也有可能是我这边机器的原因，小问题不深究了。
        记住，最左边的按钮是确定，中间是取消，最右边呢，我也没用到。读者有时间可以自己试试。

<img src="{{ site.baseurl }}/_posts/images/6.jpg">
上面的操作完成，那么订阅也ok了。

如果是同一台服务器的话，那么现在应该是完成90%的工作了，剩余的10%就是测试数据了。

问题1：
但是如果订阅和发布不在同一台服务器的话，那么在查找远程服务器的时候，这里需要设置一个服务器名称和IP的映射，因为当你选择远程发布服务器的时候，会弹出一个错误，无法连接到 【ip地址】，请指定实际的服务器名称，如下图：

<img src="{{ site.baseurl }}/_posts/images/7.jpg">
<img src="{{ site.baseurl }}/_posts/images/8.jpg">

解决办法：
在服务器的host文件中，做一个服务器名称的ip映射就可以了。

最后提供一个创建发布的sql语句（从微软网站说明找到的），他的作用和上面发布的操作其实是一样的。


{% highlight ruby %}
-- To avoid storing the login and password in the script file, the values 
-- are passed into SQLCMD as scripting variables. For information about 
-- how to use scripting variables on the command line and in SQL Server
-- Management Studio, see the "Executing Replication Scripts" section in
-- the topic "Programming Replication Using System Stored Procedures".

USE [AdventureWorks2008R2]  --这里应该设置一下数据库
DECLARE @publicationDB AS sysname;
DECLARE @publication AS sysname;
DECLARE @login AS sysname;
DECLARE @password AS sysname;
SET @publicationDB = N'AdventureWorks2008R2';  -- 发布的数据库名称
SET @publication = N'AdvWorksProductTran';  --发布任务的名称
-- Windows account used to run the Log Reader and Snapshot Agents.
SET @login = $(Login); --设置登录账户，域\账户
-- This should be passed at runtime.
SET @password = $(Password); --设置密码

-- Enable transactional or snapshot replication on the publication database.
EXEC sp_replicationdboption 
	@dbname=@publicationDB, 
	@optname=N'publish',
	@value = N'true';

-- Execute sp_addlogreader_agent to create the agent job. 
EXEC sp_addlogreader_agent 
	@job_login = @login, 
	@job_password = @password,
	-- Explicitly specify the use of Windows Integrated Authentication (default) 
	-- when connecting to the Publisher.
	@publisher_security_mode = 1;

-- Create a new transactional publication with the required properties. 
EXEC sp_addpublication 
	@publication = @publication, 
	@status = N'active',
	@allow_push = N'true',
	@allow_pull = N'true',
	@independent_agent = N'true';

-- Create a new snapshot job for the publication, using a default schedule.
EXEC sp_addpublication_snapshot 
	@publication = @publication, 
	@job_login = @login, 
	@job_password = @password,
	-- Explicitly specify the use of Windows Integrated Authentication (default) 
	-- when connecting to the Publisher.
	@publisher_security_mode = 1;
GO
{% endhighlight %}

上面就是这次用到的东西，希望会对像本人一样的开发一个提示，帮助。

