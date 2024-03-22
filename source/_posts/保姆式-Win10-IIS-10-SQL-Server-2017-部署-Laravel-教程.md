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
![install-iis-01.png](https://chenlijun.com/usr/uploads/2020/03/4228691737.png)
2. 点击添加角色和功能
![install-iis-02.png](https://chenlijun.com/usr/uploads/2020/03/1219185588.png)
3. 点击下一步
![install-iis-03.png](https://chenlijun.com/usr/uploads/2020/03/3501490930.png)
4. 选择基于角色或基于功能的安装，点击下一步
![install-iis-04.png](https://chenlijun.com/usr/uploads/2020/03/67576602.png)
5. 选择服务器，点击下一步
![install-iis-05.png](https://chenlijun.com/usr/uploads/2020/03/3493739113.png)
6. 勾选 Web 服务器(IIS)
![install-iis-06.png](https://chenlijun.com/usr/uploads/2020/03/598145813.png)
7. 弹出如图所示对话框，点击添加功能
![install-iis-07.png](https://chenlijun.com/usr/uploads/2020/03/3295301337.png)
8. 点击下一步
![install-iis-08.png](https://chenlijun.com/usr/uploads/2020/03/917935682.png)
9. 再点下一步
![install-iis-09.png](https://chenlijun.com/usr/uploads/2020/03/969053236.png)
10. 再点下一步
![install-iis-10.png](https://chenlijun.com/usr/uploads/2020/03/693675246.png)
12. 最后，点击安装
![install-iis-12.png](https://chenlijun.com/usr/uploads/2020/03/3689811225.png)
14. 等待安装，安装完成后，关闭安装向导
![install-iis-14.png](https://chenlijun.com/usr/uploads/2020/03/1732699476.png)
1.  启用CGI与fastCGI扩展
IIS 并不能理解php代码，他需要通过CGI或fastCGI将代码转发，然后获取执行结果。
	1. 在服务器管理器的菜单项中点击管理-添加角色和功能
	![install-iis-15.png](https://chenlijun.com/usr/uploads/2020/03/2373795247.png)
	2. 打开对话框后，一直下一步到服务器角色菜单项，依次展开Web服务器(IIS)，Web服务器，应用程序开发，勾选CGI。
	![install-iis-16.png](https://chenlijun.com/usr/uploads/2020/03/2046959120.png)
	3. 一直点击下一步，等待安装完成
	![install-iis-17.png](https://chenlijun.com/usr/uploads/2020/03/2026349333.png)
16. CGI 安装完成后，需要重启一下IIS（也可以和下一步合并）
	1. 打开 IIS 管理器
	方式一：开始-Windows 管理工具-IIS 管理器
	![install-iis-15-1.png](https://chenlijun.com/usr/uploads/2020/03/2804544865.png)
	方式二：点击搜索-输入IIS
	![install-iis-15-2.png](https://chenlijun.com/usr/uploads/2020/03/3489344981.png)
	2. 选中服务器，点击重新重启
	![install-iis-18.png](https://chenlijun.com/usr/uploads/2020/03/3942404755.png)
17. 添加url重写(rewrite)模块，IIS 10 默认不带此扩展，因此需要手动安装。
	1. 从[官网](https://www.iis.net/downloads/microsoft/url-rewrite "官网")下载扩展安装程序
	2. 打开安装程序，点击安装
	![install_iis_19.png](https://chenlijun.com/usr/uploads/2020/03/599280732.png)
	3. 选择我接受，然后等待安装完成
	![install-iis-20.png](https://chenlijun.com/usr/uploads/2020/03/2788565187.png)
	4. 安装完成后退出安装程序
	5. 点击IIS管理器右上角的x，彻底退出后，再次打开管理器，选择服务器后，就可以看到中间的Url重写功能了。
	![install-iis-21.png](https://chenlijun.com/usr/uploads/2020/03/2116514974.png)

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
![install-php-01.png](https://chenlijun.com/usr/uploads/2020/03/2090106342.png)
2. 解压后获得如下文件列表
![install-php-02.png](https://chenlijun.com/usr/uploads/2020/03/2707627912.png)
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
![install-php-03.png](https://chenlijun.com/usr/uploads/2020/03/3349906342.png)
1. **变量生效需要系统重启，但可以等后面安装全部完成后统一重启**

## 安装composer
发布的源码并不携带依赖包，所以用它来安装依赖项。
1. 下载 [windows 安装包](https://getcomposer.org/Composer-Setup.exe "windows 安装包")
2. 运行安装包，选择为所有用户安装
![install-composer-01.png](https://chenlijun.com/usr/uploads/2020/03/4137860733.png)
2. 点击下一步
![install-composer-02.png](https://chenlijun.com/usr/uploads/2020/03/2682482199.png)
3. 选择php的安装目录
![install-composer-03.png](https://chenlijun.com/usr/uploads/2020/03/3031044638.png)
4. 一路下一步，完成安装
5. 配置 composer 全局使用阿里云镜像，加速下载
```
composer config -g repo.packagist composer https://mirrors.aliyun.com/composer/
```
## 安装Redis
1. 从[github](https://github.com/microsoftarchive/redis/releases "github")下载windows版本的redis安装包
2. 双击打开安装包
![install-redis-01.png](https://chenlijun.com/usr/uploads/2020/03/561271387.png)
3. 接受协议，点击下一步
![install-redis-02.png](https://chenlijun.com/usr/uploads/2020/03/1697638377.png)
4. 选择安装路径，点击下一步
![install-redis-03.png](https://chenlijun.com/usr/uploads/2020/03/2494113392.png)
5. 选择redis运行端口（这里采用默认端口），并勾选允许redis通过防火墙，点击下一步
![install-redis-04.png](https://chenlijun.com/usr/uploads/2020/03/963779662.png)
1. 配置redis最大使用内存，这里抱持默认配置，不限制，点击下一步
![install-redis-05.png](https://chenlijun.com/usr/uploads/2020/03/3174737001.png)
1. 点击下一步开始安装
![install-redis-06.png](https://chenlijun.com/usr/uploads/2020/03/1791481284.png)
1. 安装完成后点击结束，完成安装

## 安装SQL Server
SQL Server 有很多版本，我选用了适合小型站点的免费版本-Express。
1. [下载安装包](https://go.microsoft.com/fwlink/?linkid=866658 "下载")
2. 打开下载的安装程序，点击基本安装
![install-sqlserver-01.png](https://chenlijun.com/usr/uploads/2020/03/2617352243.png)
3. 点击接受协议
![install-sqlserver-02.png](https://chenlijun.com/usr/uploads/2020/03/1716980621.png)
4. 选个风水宝地（或直接用默认路径），然后点安装
![install-sqlserver-03.png](https://chenlijun.com/usr/uploads/2020/03/2511441359.png)
5. 泡杯枸杞，边养生边等安装，安装完后展示如下页面
![install-sqlserver-04.png](https://chenlijun.com/usr/uploads/2020/03/245065774.png)
6. 点击关闭退出

## 安装SSMI（可视化数据库管理工具）
0. 如果数据库已存在，你可以让管理员只给帮你新建一个数据库，并给你一个连接用的账号密码
1. 下载安装程序
2. 打开安装程序，点击install
![install-sqlserver-05.png](https://chenlijun.com/usr/uploads/2020/03/1687724530.png)
3. 一路下一步，安装完成后提示需要重启操作系统完成安装，点击重启。
![install-sqlserver-06.png](https://chenlijun.com/usr/uploads/2020/03/619442740.png)
4. 开始菜单打开刚安装的管理工具，连接方式选择Windows 认证，点击连接(Connect)
![install-ssmi-01.png](https://chenlijun.com/usr/uploads/2020/03/12700406.png)
5. 在左侧数据库上右键，新建数据库
![install-ssmi-02.png](https://chenlijun.com/usr/uploads/2020/03/3343510415.png)
6. 填写数据库名称，点击确定
![install-ssmi-03.png](https://chenlijun.com/usr/uploads/2020/03/1120047302.png)
7. 在左侧依次展开安全性-登陆名，在用户sa上右键，属性
![install-ssmi-04.png](https://chenlijun.com/usr/uploads/2020/03/4161148525.png)
8. 输入两边密码，修改当前的默认密码
![install-ssmi-05.png](https://chenlijun.com/usr/uploads/2020/03/4293188312.png)
9. 再点击左侧的状态，允许sa用户登录，点击确认
![install-ssmi-06.png](https://chenlijun.com/usr/uploads/2020/03/3572877470.png)
10. 在左侧数据库上右键，属性
![install-ssmi-07.png](https://chenlijun.com/usr/uploads/2020/03/3163236334.png)
11. 左侧选中安全性，右侧选中SQL Server和Windows身份验证模式，点击确定
![install-ssmi-08.png](https://chenlijun.com/usr/uploads/2020/03/815454209.png)
12. 点击开始，打开SQL Server 配置管理器
![install-ssmi-09.png](https://chenlijun.com/usr/uploads/2020/03/1950380249.png)
11. 左侧选中MSSQLSERVER的协议，在右侧的TCP/IP上右键，点击启用
![install-ssmi-10.png](https://chenlijun.com/usr/uploads/2020/03/1777519581.png)
12. 会看到如下提示，点击确定关闭它
![install-ssmi-11.png](https://chenlijun.com/usr/uploads/2020/03/1482576974.png)
13. 点击左侧的SQL Server服务，在右侧的 SQL Server (MSSQLSERVER)上右键，点击重新启动
![install-ssmi-12.png](https://chenlijun.com/usr/uploads/2020/03/3614743299.png)
1. 至此应该可以用sa账号和修改后的密码登陆数据库了，如果你觉得直接用sa账号权限过大，可以选择新建一个数据库账号，并为此账号授予上面创建的数据库的权限。具体操作请google一下。

这个玩意非常吃性能，我测试用的阿里云1核2G卡的一匹，不得不感叹一句，用Windows服务器的都是老板啊。

# 配置站点
因为项目采用的是前后端分离的方式开发的，所以创建了两个站点。一个前端，一个后端。

## 配置后端站点
0. Laravel 站点依赖服务器的url重写功能，而Laravel项目的public目录中已默认提供一个web.config配置文件，后续操作的模块映射和默认文档都会体现在默认配置文件上。
1. 按照前文所述的方式打开IIS，在左侧选择服务器，然后展开站点，在网站菜单上右键，添加站点。
![add_site_fe_01.png](https://chenlijun.com/usr/uploads/2020/03/2636506236.png)
2. 在弹出对话框中输入站点名称，网站物理路径和域名，最后点击确定创建站点
![add_site_fe_02.png](https://chenlijun.com/usr/uploads/2020/03/3823486944.png)
4. 单击刚创建的站点，在中间双击打开处理程序映射
![add_site_fe_03.png](https://chenlijun.com/usr/uploads/2020/03/3909166300.png)
5. 点击右侧的添加模块映射
![add_site_fe_04.png](https://chenlijun.com/usr/uploads/2020/03/1420251131.png)
6. 按下图配置，第三步选择可执行文件时，默认仅显示.dll的文件，需要手动改成.exe，才能找到php-cgi.exe
![add_site_fe_06.png](https://chenlijun.com/usr/uploads/2020/03/297277552.png)
7. 点击确定时会弹出如下对话框，请选择是
![add_site_fe_07.png](https://chenlijun.com/usr/uploads/2020/03/1878798261.png)
8. 再在左侧点击当前站点，在中间双击打开默认文档
![add_site_fe_08.png](https://chenlijun.com/usr/uploads/2020/03/2607627079.png)
9. 进入如下页面，在右侧点击添加，输入index.php，点击确认
![add_site_fe_09.png](https://chenlijun.com/usr/uploads/2020/03/1054952558.png)
10. 授予IIS项目目录权限
	1. 在项目所在目录的上一级目录上右键-属性
	![add_site_fe_14.png](https://chenlijun.com/usr/uploads/2020/03/2214488603.png)
	1. 点击安全选项卡，点击编辑
	![add_site_fe_15.png](https://chenlijun.com/usr/uploads/2020/03/282897080.png)
	1. 再点击添加
	![add_site_fe_16.png](https://chenlijun.com/usr/uploads/2020/03/1263366631.png)
	1. 点击高级
	![add_site_fe_17.png](https://chenlijun.com/usr/uploads/2020/03/1291902903.png)
	1. 点击立即查找，双击选中用户IIS_IUSERS
	![add_site_fe_18.png](https://chenlijun.com/usr/uploads/2020/03/2410533883.png)
	1. 点击确定
	![add_site_fe_19.png](https://chenlijun.com/usr/uploads/2020/03/2887660283.png)
	1. 赋予所有权限，点击确定
	![add_site_fe_20.png](https://chenlijun.com/usr/uploads/2020/03/1797509765.png)
	1. 再点击确定
	![add_site_fe_21.png](https://chenlijun.com/usr/uploads/2020/03/627787676.png)
	1. 重复上面步骤，为IUSER也提供完全权限
11. 打开项目文件所在目录，复制.env.example，重命名为.env，打开编辑，如下图所示
	![add_site_fe_12.png](https://chenlijun.com/usr/uploads/2020/03/183325214.png)
12. composer安装项目依赖
	1. 在站点目录空白处按住Shift同时点击右键，点击在此处打开 PowerShell 窗口。
	![add_site_fe_10.png](https://chenlijun.com/usr/uploads/2020/03/3542630571.png)
	2. 执行composer install安装项目依赖
	![add_site_fe_11.png](https://chenlijun.com/usr/uploads/2020/03/90000590.png)
	3. 执行php .\artisan key:generate 生成加密key
	![add_site_fe_13.png](https://chenlijun.com/usr/uploads/2020/03/1734028954.png)
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
![add_site_fe_22.png](https://chenlijun.com/usr/uploads/2020/03/862876512.png)
2. 点击右侧的导入规则
![add_site_fe_23.png](https://chenlijun.com/usr/uploads/2020/03/4285019925.png)
3. 将上面的Apache重写规则粘贴在重写规则区域，点击右上角的应用
![add_site_fe_24.png](https://chenlijun.com/usr/uploads/2020/03/2318009505.png)

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
![add_site_fe_25.png](https://chenlijun.com/usr/uploads/2020/03/2768838995.png)