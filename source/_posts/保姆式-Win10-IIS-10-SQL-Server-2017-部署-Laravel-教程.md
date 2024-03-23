---
title: 保姆式 Win10 + IIS 10 + SQL Server 2017 部署 Laravel 教程
date: 2024-03-22 13:26:04
tags:
  - php
  - laravel
  - 运维
categories:
  - php
---

# 简介
做了个知识库管理系统，开发时服务器使用的是宝塔LAMP架构，结果开发完要求部署到客户的服务器上。客户的服务器架构是 Win10 + IIS 10 + SQL Server 2017。
折腾了一整天，终于装上了，这里记录一下。
<!--more-->

# 依赖
1. [项目 Laravel 依赖项 （5.5）](https://learnku.com/docs/laravel/5.5/installation/1282#df1b28 "项目 Laravel 依赖项 （5.5）")
	- PHP >= 7.0.0
	- PHP OpenSSL 扩展
	- PHP PDO 扩展
	- PHP Mbstring 扩展
	- PHP Tokenizer 扩展
	- PHP XML 扩展
2. [redis](https://github.com/microsoftarchive/redis/releases "redis")
	- 用来存token
3. IIS 10
	- Url Rewrite 模块
	- fastCgi 模块
4. SQL Server 2017

# 环境搭建
服务器采用阿里云购买的windows Server 2019，在阿里云面板安装成功后，通过 windows 远程桌面连接。由于windows Server版的浏览器安全配置，无法下载文件，且修改配置较麻烦，建议后续需要下载的文件全部先在本机下载完成后通过远程桌面上传。

## 安装 IIS 10
1. 点击开始-服务器管理器
{% asset_img iis-01.png "安装iis-01" %}
2. 点击添加角色和功能
{% asset_img iis-02.png "安装iis-02" %}
3. 点击下一步
{% asset_img iis-03.png "安装iis-03" %}
4. 选择基于角色或基于功能的安装，点击下一步
{% asset_img iis-04.png "安装iis-04" %}
5. 选择服务器，点击下一步
{% asset_img iis-05.png "安装iis-05" %}
6. 勾选 Web 服务器(IIS)
{% asset_img iis-06.png "安装iis-06" %}
7. 弹出如图所示对话框，点击添加功能
{% asset_img iis-07.png "安装iis-07" %}
8. 点击下一步
{% asset_img iis-08.png "安装iis-08" %}
9. 再点下一步
{% asset_img iis-09.png "安装iis-09" %}
10. 再点下一步
{% asset_img iis-10.png "安装iis-10" %}
11. 最后，点击安装
{% asset_img iis-11.png "安装iis-11" %}
12. 等待安装，安装完成后，关闭安装向导
{% asset_img iis-12.png "安装iis-12" %}
13.  启用CGI与fastCGI扩展
IIS 并不能理解php代码，他需要通过CGI或fastCGI将代码转发，然后获取执行结果。
	1. 在服务器管理器的菜单项中点击管理-添加角色和功能
	{% asset_img iis-13-1.png "安装iis-13-1" %}
	2. 打开对话框后，一直下一步到服务器角色菜单项，依次展开Web服务器(IIS)，Web服务器，应用程序开发，勾选CGI。
	{% asset_img iis-13-2.png "安装iis-13-2" %}
	3. 一直点击下一步，等待安装完成
	{% asset_img iis-13-3.png "安装iis-13-3" %}
14. CGI 安装完成后，需要重启一下IIS（也可以和下一步合并）
	1. 打开 IIS 管理器
	方式一：开始-Windows 管理工具-IIS 管理器
	{% asset_img iis-14-1-1.png "安装iis-14-1-1" %}
	方式二：点击搜索-输入IIS
	{% asset_img iis-14-1-2.png "安装iis-14-1-2" %}
	2. 选中服务器，点击重新重启
	{% asset_img iis-14-1-3.png "安装iis-14-1-3" %}
15. 添加url重写(rewrite)模块，IIS 10 默认不带此扩展，因此需要手动安装。
	1. 从[官网](https://www.iis.net/downloads/microsoft/url-rewrite "官网")下载扩展安装程序
	2. 打开安装程序，点击安装
	{% asset_img iis-15-1.png "安装iis-15-1" %}
	3. 选择我接受，然后等待安装完成
	{% asset_img iis-15-2.png "安装iis-15-2" %}
	4. 安装完成后退出安装程序
	5. 点击IIS管理器右上角的x，彻底退出后，再次打开管理器，选择服务器后，就可以看到中间的Url重写功能了。
	{% asset_img iis-15-3.png "安装iis-15-3" %}

## 安装PHP
### 下载PHP
因为项目依赖的 php 版本要求7.0+，我就去php官网上下了个php7.2的nts版本。注意要选择nts版本，nts是 Non-Thread-Safe 的缩写，即线程不安全。因为我们需要用IIS+fastCgi的方式执行php脚本，所以必须选择nts版本。
官方下载地址：[地址](https://windows.php.net/download/ "地址")
下载时请注意区分64位和32位，首选64位。
下载后是个压缩包，找个地方解压，比如C:\php7.2。

### 配置php.ini
打开C:\php7.2，将php.ini-production复制一份，并重命名为php.ini。
搜索;extension，可以找到一片被注释的扩展项。
去掉最前面的;可以启用该扩展，根据需要，我开启了以下扩展。
```
extension=curl
extension=fileinfo
extension=mbstring
extension=odbc
extension=openssl
extension=pdo_sqlsrv ;安装 SQL server 扩展后添加
```
找到以下代码段，将extension_dir = "ext"前面的分号去掉并保存
```
; Directory in which the loadable extensions (modules) reside.
; http://php.net/extension-dir
; extension_dir = "./"
; On windows:
extension_dir = "ext" ;将这行前面的分号去掉
```

### 添加 SQL server 扩展
因为数据库采用SQL Server，所以需要添加扩展。
1. 从微软网站上[下载](https://go.microsoft.com/fwlink/?linkid=2120362 "下载")扩展包安装程序后打开，选个地方解压，如C:\ext。
{% asset_img php-01.png "安装php-01" %}
2. 解压后获得如下文件列表
{% asset_img php-02.png "安装php-02" %}
因为我本地安装的php版本是7.2 nts 64位，所以将php_pdo_sqlsrv_72_nts_x64.dll复制到php安装目录下的ext目录中，并重命名为php_pdo_sqlsrv.dll。
这一步请根据自己安装的php版本选择拷贝对应的扩展文件，如果你安装的php版本过低，请选择下载[历史版本](https://docs.microsoft.com/zh-cn/sql/connect/php/release-notes-php-sql-driver?view=sql-server-ver15#previous-releases "历史版本")的扩展包。
3. 在上一步的php.ini配置文件中添加一项配置，如配置php.ini小结最后一行的代码所示。

### 配置openssl.cnf
因为我的项目使用到了openssl，所以需要额外配置。
1. 打开控制面板-系统和安全-系统
1. 点击高级系统设置
1. 点击环境变量
1. 点击添加系统变量
1. 弹出对话框中，变量名填入 OPENSSL_CONF
1. 变量值填入C:\php7.2\extras\ssl\openssl.cnf，这个路径需要根据你php的安装目录调整。
1. 逐级点击确定
具体操作可以参考下图：
{% asset_img php-03.png "安装php-03" %}
1. **变量生效需要系统重启，但可以等后面安装全部完成后统一重启**

## 安装composer
发布的源码并不携带依赖包，所以用它来安装依赖项。
1. 下载 [windows 安装包](https://getcomposer.org/Composer-Setup.exe "windows 安装包")
2. 运行安装包，选择为所有用户安装
{% asset_img composer-01.png "安装composer-01" %}
2. 点击下一步
{% asset_img composer-02.png "安装composer-02" %}
3. 选择php的安装目录
{% asset_img composer-03.png "安装composer-03" %}
4. 一路下一步，完成安装
5. 配置 composer 全局使用阿里云镜像，加速下载
```
composer config -g repo.packagist composer https://mirrors.aliyun.com/composer/
```
## 安装Redis
1. 从[github](https://github.com/microsoftarchive/redis/releases "github")下载windows版本的redis安装包
2. 双击打开安装包
{% asset_img redis-01.png "安装redis-01" %}
3. 接受协议，点击下一步
{% asset_img redis-02.png "安装redis-02" %}
4. 选择安装路径，点击下一步
{% asset_img redis-03.png "安装redis-03" %}
5. 选择redis运行端口（这里采用默认端口），并勾选允许redis通过防火墙，点击下一步
{% asset_img redis-04.png "安装redis-04" %}
6. 配置redis最大使用内存，这里抱持默认配置，不限制，点击下一步
{% asset_img redis-05.png "安装redis-05" %}
7. 点击下一步开始安装
{% asset_img redis-06.png "安装redis-06" %}
8. 安装完成后点击结束，完成安装

## 安装SQL Server
SQL Server 有很多版本，我选用了适合小型站点的免费版本-Express。
1. [下载安装包](https://go.microsoft.com/fwlink/?linkid=866658 "下载")
2. 打开下载的安装程序，点击基本安装
{% asset_img sql-server-01.png "安装sql-server-01" %}
3. 点击接受协议
{% asset_img sql-server-02.png "安装sql-server-02" %}
4. 选个风水宝地（或直接用默认路径），然后点安装
{% asset_img sql-server-03.png "安装sql-server-03" %}
5. 泡杯枸杞，边养生边等安装，安装完后展示如下页面
{% asset_img sql-server-04.png "安装sql-server-04" %}
6. 点击关闭退出

## 安装SSMI（可视化数据库管理工具）
0. 如果数据库已存在，你可以让管理员只给帮你新建一个数据库，并给你一个连接用的账号密码
1. 下载安装程序
2. 打开安装程序，点击install
{% asset_img sql-server-05.png "安装sql-server-05" %}
3. 一路下一步，安装完成后提示需要重启操作系统完成安装，点击重启。
{% asset_img sql-server-06.png "安装sql-server-06" %}
4. 开始菜单打开刚安装的管理工具，连接方式选择Windows 认证，点击连接(Connect)
{% asset_img sql-server-07.png "安装sql-server-07" %}
5. 在左侧数据库上右键，新建数据库
{% asset_img sql-server-08.png "安装sql-server-08" %}
6. 填写数据库名称，点击确定
{% asset_img sql-server-09.png "安装sql-server-09" %}
7. 在左侧依次展开安全性-登陆名，在用户sa上右键，属性
{% asset_img sql-server-10.png "安装sql-server-10" %}
8. 输入两遍密码，修改当前的默认密码
{% asset_img sql-server-11.png "安装sql-server-11" %}
9. 再点击左侧的状态，允许sa用户登录，点击确认
{% asset_img sql-server-12.png "安装sql-server-12" %}
10. 在左侧数据库上右键，属性
{% asset_img sql-server-13.png "安装sql-server-13" %}
11. 左侧选中安全性，右侧选中SQL Server和Windows身份验证模式，点击确定
{% asset_img sql-server-14.png "安装sql-server-14" %}
12. 点击开始，打开SQL Server 配置管理器
{% asset_img sql-server-15.png "安装sql-server-15" %}
11. 左侧选中MSSQLSERVER的协议，在右侧的TCP/IP上右键，点击启用
{% asset_img sql-server-16.png "安装sql-server-16" %}
12. 会看到如下提示，点击确定关闭它
{% asset_img sql-server-17.png "安装sql-server-17" %}
13. 点击左侧的SQL Server服务，在右侧的 SQL Server (MSSQLSERVER)上右键，点击重新启动
{% asset_img sql-server-18.png "安装sql-server-18" %}
14. 至此应该可以用sa账号和修改后的密码登陆数据库了，如果你觉得直接用sa账号权限过大，可以选择新建一个数据库账号，并为此账号授予上面创建的数据库的权限。具体操作请google一下。

这个玩意非常吃性能，我测试用的阿里云1核2G卡的一匹，不得不感叹一句，用Windows服务器的都是老板啊。

# 配置站点
因为项目采用的是前后端分离的方式开发的，所以创建了两个站点。一个前端，一个后端。

## 配置后端站点
1. Laravel 站点依赖服务器的url重写功能，而Laravel项目的public目录中已默认提供一个web.config配置文件，后续操作的模块映射和默认文档都会体现在默认配置文件上。
2. 按照前文所述的方式打开IIS，在左侧选择服务器，然后展开站点，在网站菜单上右键，添加站点。
{% asset_img config-site-01.png "安装config-site-01" %}
2. 在弹出对话框中输入站点名称，网站物理路径和域名，最后点击确定创建站点
{% asset_img config-site-02.png "安装config-site-02" %}
4. 单击刚创建的站点，在中间双击打开处理程序映射
{% asset_img config-site-03.png "安装config-site-03" %}
5. 点击右侧的添加模块映射
{% asset_img config-site-04.png "安装config-site-04" %}
6. 按下图配置，第三步选择可执行文件时，默认仅显示.dll的文件，需要手动改成.exe，才能找到php-cgi.exe
{% asset_img config-site-05.png "安装config-site-05" %}
7. 点击确定时会弹出如下对话框，请选择是
{% asset_img config-site-06.png "安装config-site-06" %}
8. 再在左侧点击当前站点，在中间双击打开默认文档
{% asset_img config-site-07.png "安装config-site-07" %}
9. 进入如下页面，在右侧点击添加，输入index.php，点击确认
{% asset_img config-site-08.png "安装config-site-08" %}
10. 授予IIS项目目录权限
	1. 在项目所在目录的上一级目录上右键-属性
	{% asset_img config-site-09.png "安装config-site-09" %}
	1. 点击安全选项卡，点击编辑
	{% asset_img config-site-10.png "安装config-site-10" %}
	1. 再点击添加
	{% asset_img config-site-11.png "安装config-site-11" %}
	1. 点击高级
	{% asset_img config-site-12.png "安装config-site-12" %}
	1. 点击立即查找，双击选中用户IIS_IUSERS
	{% asset_img config-site-13.png "安装config-site-13" %}
	1. 点击确定
	{% asset_img config-site-14.png "安装config-site-14" %}
	1. 赋予所有权限，点击确定
	{% asset_img config-site-15.png "安装config-site-15" %}
	1. 再点击确定
	{% asset_img config-site-16.png "安装config-site-16" %}
	1. 重复上面步骤，为IUSER也提供完全权限
11. 打开项目文件所在目录，复制.env.example，重命名为.env，打开编辑，如下图所示
	{% asset_img config-site-17.png "安装config-site-17" %}
12. composer安装项目依赖
	1. 在站点目录空白处按住Shift同时点击右键，点击在此处打开 PowerShell 窗口。
	{% asset_img config-site-18.png "安装config-site-18" %}
	2. 执行composer install安装项目依赖
	{% asset_img config-site-19.png "安装config-site-19" %}
	3. 执行php .\artisan key:generate 生成加密key
	{% asset_img config-site-20.png "安装config-site-20" %}
	4. 执行php .\artisan migrate:refresh --seed 生成数据库结构，并填充基础数据
	5. 指定php .\artisan storage:link 创建public\storage\ 到 storage\app\public 的快捷方式

## 配置前端站点
与后端一样新建站点，配置域名，站点名称和目录，此处不再赘述。
这里将另一个问题，因为前端站点是基于ant design pro搭建的，出现了站点刷新后会404的问题。根据其[官网介绍](https://pro.ant.design/docs/deploy-cn#%E5%89%8D%E7%AB%AF%E8%B7%AF%E7%94%B1%E4%B8%8E%E6%9C%8D%E5%8A%A1%E7%AB%AF%E7%9A%84%E7%BB%93%E5%90%88 "官网介绍")，需要服务器重写路由，将请求转发至index.html。
原来所使用的Apache重写规则如下：
```
 <IfModule mod_rewrite.c>
  RewriteEngine On
  RewriteRule ^index\.html$ - [L]
  RewriteCond %{REQUEST_FILENAME} !-f
  RewriteCond %{REQUEST_FILENAME} !-d
  RewriteRule . /index.html [L]
</IfModule>
```
而我们现在的服务器是IIS，上面的重写规则是针对Apache的，所以我们需要在IIS中导入此规则。

1. IIS管理器中选中前端站点，双击打开url重写
{% asset_img config-site-21.png "安装config-site-21" %}
2. 点击右侧的导入规则
{% asset_img config-site-22.png "安装config-site-22" %}
3. 将上面的Apache重写规则粘贴在重写规则区域，点击右上角的应用
{% asset_img config-site-23.png "安装config-site-23" %}

## SQL Server 查询总是返回字符串类型
项目上线后发现数据库中定义为 int 类型的字段，查询返回的永远都是字符串，这使得用 vue 架构的前端需要增加额外的适配工作。

经过多方查询，发现是因为php的原因，laravel 中的模型等数据库操作方式，实际上是对 php 提供的 PDO 的封装。PDO 在连接数据库时需要使用对应的驱动，早期版本（php 5.3 之前）的 PDO 使用的连接驱动仅能获取到字段名称，无法获取到字段的类型，导致其返回的就是字符串，因为字符串可以包容万象。

而高版本的 php 实际可以获取到对应的字段类型，但可能需要额外配置一下（mysql无需额外操作， sql server 需要）。

打开config/database.php，在sql server 连接配置中添加选项配置即可解决问题。
```php
'sqlsrv' => [
            'driver' => 'sqlsrv',
            'host' => env('DB_HOST', 'localhost'),
            'port' => env('DB_PORT', '1433'),
            'database' => env('DB_DATABASE', 'forge'),
            'username' => env('DB_USERNAME', 'forge'),
            'password' => env('DB_PASSWORD', ''),
            'charset' => 'utf8',
            'prefix' => '',
            'options' => array(
                PDO::ATTR_STRINGIFY_FETCHES => false,
                PDO::SQLSRV_ATTR_FETCHES_NUMERIC_TYPE => true
            )
        ],
```

# 完
{% asset_img config-site-24.png "安装config-site-24" %}