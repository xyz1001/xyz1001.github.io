---
title: 软件折腾笔记之WordPress
author: 张帆
tags:
  - WordPress
date: 2016-04-11 18:57:00
---

WordPress是一个优秀的个人博客框架，用起来相当简单，功能强大。但世上没有完美无缺的东西，任何事物总存在或多或少的瑕疵。在折腾WordPress的过程中，我遇到了一些问题，有WordPress自身的BUG，也有其功能上的不足。但由于WordPress本身开源，而且提供了数量丰富的插件供我们使用，因此所有的问题也都不是问题了。以下是本人对WordPress的配置及优化过程。

<!--more-->

## 主题安装

安装好WordPress并启用后，首先要做的是选取一款合适的主题。WordPress自带了几款主题，并提供了数量庞大的主题库供大家免费下载。推荐选择使用人数较多的主题，这类主题往往兼容性较好，不会影响插件的效果。我之前使用的一款主题就和WP-postviews这款插件冲突了，导致无法显示访客数等。

## 基本设置

打开设置，可根据自己的需要进行设置，包括常规、撰写、阅读、讨论、多媒体和固定连接。

- 常规设置中，如果需要开放本站的注册资格，需要在成员资格中将“任何人都可注册选”中。
- WordPress系统提供了更新通知服务，当博客有更新时，就会以XML-RPC协议发送ping通知到网络更新服务商。这些更新服务商包括Google博客搜索，百度，雅虎等搜索引擎，以便更新内容更快速的收录。我们可以在撰写设置中的更新服务中添加自己希望通知的服务商，推荐保留自带的`http://rpc.pingomatic.com/`，以下是部分服务商的地址。

 > 在线Ping服务地址

 ```
 百度：http://ping.baidu.com/ping.html
 Google：http://blogsearch.google.cn/ping
 有道：http://tellbot.youdao.com/report?type=BLOG
 搜狗：http://www.sogou.com/feedback/blogfeedback.php
 ```

 > RPC服务地址（SAE可用）

 ```
 百度：http://ping.baidu.com/ping/RPC2
 谷歌：http://blogsearch.google.com/ping/RPC2
 有道：http://blog.youdao.com/ping/RPC2
 Bloglines：http://www.bloglines.com/ping
 PingOMatic：http://rpc.pingomatic.com
 Feedsky：http://www.feedsky.com/api/RPC2</p>
 ```

- 在阅读设置中，推荐将“对于feed中的每篇文章，显示”设置为“摘要”，否则会导致主页面上显示全文，带来阅读上的不变。
- 讨论设置中，部分主题可能不支持嵌套评论，设置后并没有效果。
- 多媒体设置中，推荐去除“总是裁剪缩略图到这个尺寸”的选中，这样图片显示更正常。
- 固定链接设置中，可根据自己的习惯设置文章地址的显示方式，个人设置为“自定义结构，参数为`/%year%%monthnum%%day%/%post_id%`

## 插件安装

正如Chrome，插件也是WordPress功能强大的一个核心原因。在此推荐几款插件：

- `Akismet`：Akismet是WordPress自带的一款反垃圾评论的插件，该插件配合其强大的数据库后端，可有效地检测评论的有效性。一旦检测到垃圾评论，可以自动屏蔽该评论。目前在WordPress上有100万+的活跃度。
- `WP Super Cache`：该插件是静态缓存插件，通过把整个网页直接生成 HTML 文件，这样 Web 服务器就不用解析 PHP 脚本，使得你的 WordPress 博客页面打开速度加快。启用该插件后，手动选择“启用缓存功能*(推荐)*”，并在高级中勾选所有标有“推荐”的选项即可。
- `All in one SEO pack`(多合一seo包)：该插件提供了SEO优化方面的支持，可以提升网页对搜索引擎的友好度，加快网页被收录速度，是站长的必备插件之一。安装后进入“首页设置”按自己情况填写一下即可，发布文章时可通过文章下方的SEO设置进行相关设置。
- `Add Post URL`(添加文章固定连接)：该插件可以允许你在文章开头、结尾添加固定文字，如版权，广告等。
- `BackUpWordPress`(备份WordPress)：数据库及网页自动备份插件，该插件可以定时（每天、周、月）备份数据库或文件，并可通过邮件通知站长，当数据不超过限定的大小时还可将备份文件通过附件发送。
- `Disable Google Fonts`(禁用谷歌字体)：由于某些原因，WordPress自带的谷歌字体在国内无法访问，导致网站会卡在加载谷歌字体(`fonts.googleapis.com`)上，由于谷歌字体是英文字体，在国内一般不需要，因此可以直接安装该插件禁用，或者安装`Useso take over Google`插件，将谷歌字体替换为国内的360前端公共库Useso也可解决。
- `Google XML Sitemaps`(谷歌XML网站地图)：生成XML格式的网站地图，方便搜索引擎爬虫检索网站，加快收录速度。
- `No Category Base`(WPML)：去除文章网址中的Category
- `WP-Mail-SMTP`：由于大部分服务器（阿里云等）均不提供`mail()`函数给WordPress使用，导致WordPress无法向外发送邮件，使得用户注册、评论通知以及部分插件的正常功能无法使用，因此需要改用SMTP服务器进行邮件发送。设置如下：

 ![WordPress 启用SMTP](wordpress-smtp.png)

- `WP-PostViews`：可以记录每篇文章展示次数、根据展示次数显示历史最热或最冷的文章排行等
- `crayon-syntax highlight`：程序猿必备插件，提供代码高亮显示，极大地方便的代码的阅读。使用方式为在编辑文章的“文本”状态下，点击crayon标签，将代码复制进去，选择代码语言即可。在设置中勾选“在代码中进行 HTML 转义”
- `百度分享按钮`：提供分享功能，需前往百度分享获取定制的分享代码，理论上支持其他网站提供的分享代码，未测试。该插件长时间（2年）未更新，可能存在一定的兼容性问题。另推荐另一款分享插件AddToAny。
- `JP Markdown`：为Wordpress添加Markdown支持，语法链接：[Markdown Extra](https://en.support.WordPress.com/markdown-quick-reference/)

## BUG修补

WordPress自身存在一定BUG，需要通过手动修改代码的方式修复。

- 注册邮件及找回密码邮件中地址错误：当用户注册或找回密码时，WordPress会发送一封邮件到注册邮箱中，邮件中包括修改密码的网页链接。WordPress为了使链接显示更直观，会自动在链接前后加上`<>`，然而邮件显示时会将`>`错误的加入地址中，从而导致链接地址错误，打开后显示“链接无效”。

 修改方式：复制服务器上WordPress根目录`/wp-content/themes/(你的主题目录)/functions.php`至本地，用文本处理器打开，在文件结尾添加以下代码：

 ``` php
 //修复WordPress找回密码提示“抱歉，该key似乎无效”问题
 function reset_password_message( $message, $key ) {
 if ( strpos($_POST['user_login'], '@') ) {
 $user_data = get_user_by('email', trim($_POST['user_login']));
 } else {
 $login = trim($_POST['user_login']);
 $user_data = get_user_by('login', $login);
 }
 $user_login = $user_data->user_login;
 $msg = __('有人要求重设如下帐号的密码：'). "\r\n\r\n";
 $msg .= network_site_url() . "\r\n\r\n";
 $msg .= sprintf(__('用户名：%s'), $user_login) . "\r\n\r\n";
 $msg .= __('若这不是您本人要求的，请忽略本邮件，一切如常。') . "\r\n\r\n";
 $msg .= __('要重置您的密码，请打开下面的链接：'). "\r\n\r\n";
 $msg .= network_site_url("wp-login.php?action=rp&&key=$key&&login=" . rawurlencode($user_login), 'login') ;
 return $msg;
 }
 add_filter('retrieve_password_message', reset_password_message, null, 2);
 ```

 保存，将原文件重名为`functions.php.bak`，作为备份，然后将修改后的文件复制进去即可。该方法在更换主题或升级主题后失效，需重新修改。另一种方法是修改WordPress根目录下`wp-login.php`文件，将其约381行的代码

 ``` php
 $message .= '<' . network_site_url("wp-login.php?action=rp&&key=$key&&login=" . rawurlencode($user_login), 'login') . ">\r\n";
 ```

 改为

 ``` php
 $message .= network_site_url("wp-login.php?action=rp&&key=$key&&login=" . rawurlencode($user_login), 'login');
 ```

- 大部分默认主题提供的404页面十分简陋，我们可以根据自己的需要，修改themes目录下对应主题目录下的`404.php`文件，自由定制。
- WordPress默认将图片保存至WordPress根目录`/wp-content/uploads`文件夹中，备份时注意备份。
- 文章中插入的图片，WordPress会默认以缩略图显示，往往会导致部分大图显示十分模糊，因此需要在插入图片时选择合适的显示大小，对于部分特别大的图片，可设置图片超链接至原图。
