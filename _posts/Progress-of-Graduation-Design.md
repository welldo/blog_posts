---
title: 毕业设计记录
date: 2016-12-27 10:13:48
tags: [Frontend, Backend, Diary]
---

### 2016/12/27

今天是第一天正式开始做，目前分配的目标是平台搭建。// 考研终于结束了……

我做的是微信客户端（其实我猜应该是公众号吧，不然没有小程序的权限啊），那么任务可明确为“移动设备 HTML5 应用开发”，初期的平台搭建任务实际上是 Web 服务端搭建。按理说微信公众号的申请也算是平台搭建的任务，但我已经有自己的微信公众号了，所以就不在这里说明公众号的申请的细节了（都快要三年过去了呢）。此外，我也有自己的 VPS，因此服务器购买也不算在这里。

要做的事情就是搭建 Web 服务器了。

关于服务器选择问题，由于之前有搭建 nginx 的经历，此处就直接采用 nginx 了。

而关于动态语言的选择问题，我只会一点 PHP……虽然以前也在一些 Python / Django 的项目里挑过 bug，不过还是不太敢自己写新的项目用（不过 Python 用起来确实比较舒服，起码不容易小指骨折）

具体过程如下：

1. 登录到 VPS 上，看看源里有没有现成的程序可以用

 源里是有 nginx 和 OpenSSL 这样的程序，但 VPS 的系统是 CentOS 6, 源用的是 EPEL，其中 OpenSSL 的版本是 1.0.1e-fips，而 OpenSSL 官方网站上带 -fips 的版本是 2.0了。故选择自己编译 OpenSSL，同时 nginx 也要一起重新编译。（源里面的 nginx 是 1.10.2, 是最新的稳定版，这比前一个 VPS 的源里的版本新的多了，不知道前面那个 VPS 自带的上游源没有同步，我当时也懒得换源）

2. 下载相应的源码，按依赖顺序编译

 这很简单，去官网找到链接，复制到 shell 里面用 wget 就好了。这里选择的是 nginx 1.10.2, OpenSSL 1.0.2j，PHP 7.1.0. 分别 `tar xf` 解开，按 OpenSSL -> nginx -> PHP 的顺序编译就好。`configure` 参数命令行如下所示：

```
# OpenSSL
./config --prefix=/root/root --prefix=/root/root/ssl

# nginx
./configure --prefix=/root/root --with-openssl=/root/src/openssl-1.0.2j

# PHP
./configure --with-openssl=/root/root/ssl --enable-mbstring --enable-fpm --prefix=/root/root --with-mysqli --with-pdo-mysql --with-gd
```

 每个 `configure` 结束以后用 `make install` 来安装到 /root/root 目录下，以避免污染系统自带的版本。库的版本不能随便乱动的，免得整个系统崩掉，更何况 OpenSSL 的 branch 都变了。要体验完全源码编译的乐趣的话，像 Gentoo 那样自动编译整个系统比较好……

 顺便吐个槽，PHP 的 `--with-openssl` 的帮助就只说了一个 Include OpenSSL support (requires OpenSSL >= 1.0.1)，谁知道它到底要的是 OpenSSL 的安装路径还是源码路径？还是 nginx 做得好，set path to OpenSSL library sources. 蛋疼的是，PHP 要求的路径是安装路径，与 nginx 相反……

 （为什么 Sublime Text 下载得这么慢啊）

3. 测试 nginx

 直接打 IP 进去，提示 403. 看起来是 nginx 默认自己降到 nobody 用户。为了方便管理，给它新增一个 nginx 专用账户。

```
groupadd nginx
useradd -g nginx nginx
chsh -s /sbin/nologin nginx
```

 最后一行是防止这个账户通过 shell 登录。（在 `useradd` 里面加两个参数就能用一条命令做完三条的事，不过我不记得了……）

 然后把 nginx 的 html 文件夹移动到 ~nginx/ 底下，`chown` 一下，403 就没有了。

4. 配置 & 测试 PHP

 这里有些麻烦。先是尝试了几次才找对全部需要的参数（如上命令行，`--enable-fpm` 和扩展相关的参数一开始就没加，得到的是裸的 PHP），然后复制 php-fpm.conf.default 把 .default 去掉，设置 socket 路径，改启动用户为 nginx，启动后手动 chown nginx:nginx 它的 socket 文件。nginx 有自带 PHP-FPM 的配置，一行 `include fastcgi.conf` 就好了。

 要特别注意的是 nginx 的 fastcgi_temp 目录需要指定到 nginx 能够写入的位置，我这里 nginx 的程序放在 ~root 下面，所以它默认的 temp 目录也在这下面。需要用 `fastcgi_temp_path` 指定到 ~nginx 底下去然后 chown，否则能返回 HTTP 200 但 php 网页显示不全。

 至于 PHP 的探针，`phpinfo();` 就足够了～

5. 微信推送测试

 由于公众号权限不足，故只能通过回复消息的形式给出链接，菜单功能只能稍后处理。回复消息包含的链接文本会自动转化成可点击的链接。


### 2016/12/28

今天主要的任务是写一个 Demo. 打算先实现简单的登录功能，然后显示点简单的数据，比如显示目前登录到的变电站名称，目前该站拥有的设备及其最近一次故障记录详情。自己的测试服务端里要写死以上数据。

任务书还没下来，不知道要做微信登录还是用户名密码登录。微信登录个人感觉不是很必要，因为我们这里并不强调人与人的关系。所以，这个 Demo 还是按用户名和密码做。

此外有一点要注意的地方，就是这里面涉及的数据都比较私密，所以前端发送密码时应该加盐 hash 一下，并且做到每次发送的字符串都不一样。但是查了一下关于这个问题的讨论 [1]，发现还是有不少人认为只要有 HTTP 连接就没有加密的意义了，即便是一次一密，也能通过前端页面公开的算法甚至 XSS 做中间人攻击。但是我觉得如果服务器真的是在登录的时候发来 nonce，那么即便是监听到了也很难通过 hash 反推密码啊。一个 nonce 怎么也要十几个字符了，加用户名加密码得有二三十个字节长，用 bcrypt 之类的怎么反推得了……（当然 HTTPS 是最好了，但目前要上 HTTPS 的话要通过 CloudFlare，速度会很慢）

[1] https://www.zhihu.com/question/25539382

做的时候发现 Javascript 里似乎没有现成的散列函数，选择用 SHA256 [2] 好了。

[2] http://www.bichlmeier.info/sha256.js

PHP 是被访问时执行一次然后退出，以前以为是必须要把当前会话的状态存入数据库的，现在查到 PHP 自身支持 session 功能，不需要数据库也可以在不同的访问中保留状态。

（发现 PHP 已经忘得一干二净了）

（为什么 PHP 在使用全局变量的时候还要先用 `global` 啊）

HTML 从头开始写的话有点懵，最后从 MDN 找了 AJAX 请求代码修改了下来验证 SHA256 函数以及登录过程实现正确。最后是验证成功了，今天就到这里了，代码已经 push 到 GitHub 了，有心情再回去看吧。

（说起来，总觉得 PHP 里面的那些类实现得很奇怪，突然不知道类里应该要写哪些方法了……）

### 2016/12/29

今天要汇报了，想着给那个测试登录用的网页加点样式吧。不过这次时间有点赶，看书可能来不及，先口胡两句 CSS 上去……

CSS 水平居中好说，垂直居中的话如果用父元素 table、子元素 table-cell 的方法，要记得在 html {} 里就把 `height: 100%` 写好，否则后面 inherit 来的高度最大也就只有那么大了，不能覆盖全屏。

剩下的都慢慢调……不知道为什么 `position: absolute; left: 0; right: 0;` 的 copyright notice居然没有跟登录框水平对齐，明明登录框已经是 `text-align: center` 了。好像是登录框歪了，不知道为什么……暂时把 left 改成 2%，看起来舒服一点。

闪烁背景提示错误的功能用 CSS Animation 比较好实现。如果要在 JS 里触发动画，只要在对象的 style 属性里把 `animationName` （对应 CSS 里面的 `animation-name` 属性，style 里面的成员都是这样）设为空，然后用 `setTimeout(function(){}，0)` 把 `animationName` 设为动画名称就好了。至于动画时长、重复次数之类，都应该直接在 CSS 里面写好。

TODO 刚才想到，我在登录那里用了 nonce 和两次 SHA256，就是为了防止被窃听。但窃听还是能够拿到 PHPSESSID 啊，有了这个也一样能登录了。但是，考虑到这是微信端，用户 IP 随时可能变化，不能通过绑定 session 和用户 IP 的方式来做验证，而其他任何不是客户端固有的数据都能够通过窃听获取到，所以防登录是没有办法了。这样 hash 就只能是防撞库和明文密码泄漏……

顺便，Android 为什么比 iOS 好呢？因为它可以设定 `theme-color` 这个属性啊，比 iOS 除了白还是白的顶部不知道高到哪里去了。

### 2016/12/30

听说今天是最后一个工作日了呢……今天上午学长介绍了下本地网络的情况，需要加上无线接入功能，使得笔记本和手机能够通过无线读取数据，并且模拟某电厂内部的情况——WiFi只能够上内网。

我理解的本地网络是这样构造的：

有一台非网管交换机接入上游教育网，下面接正向和反向网络隔离设备，往下接另一个非网管交换机，再往下接各台服务器。内网没有 DHCP 和 DNS 服务，所有服务器都是手动配置的 192.168 开头的私有地址。昨晚还提到一个需求：做出来的移动端在公网和内网都要能用。

说起来不知道网络隔离设备是做什么的。我电脑插在直连公网的交换机上，但是直接配置 192.168 的地址是可以访问内网的，也不需要校园网认证（相当于我可以从上游访问）。但是根据学长的描述，隔壁实验室同是公网，配置 192.168 不能够访问这里。为什么同是公网却不能访问呢，是什么设备起了阻隔作用呢？我现在的一种推测是，其实网络隔离设备并没有起作用，实际上是因为我这里的上游交换机和隔壁实验室的上游交换机没有连到同一个路由器（至少不是同一个 VLAN），192.168 无法跨越。另一种推测是其实在隔壁是能够访问的，但因为要跨越上游交换机（而不是路由器），所以要通过认证来开启端口；但要做认证的话，锐捷又会分配公网 IP。这样一来，不过认证直接配 IP 会导致交换机端口不开，不通，过认证了又配不了 IP……TODO 我找个时间找认识的学长用 minieap 测试一下开认证 + 手动 IP 吧。万恶的锐捷。

（2017/01/04 回顾：实际上只能访问分给我的那台服务器。因为那台服务器跟我一样被接到公网但还是配置着 192.168 的内网 IP，所以我以为能访问整个 192.168 ……内网其他服务器是不能访问的，隔离设备实际上生效了）

好了，回到原始话题。网络结构说完了，接下来是网络里的数据流。

首先，有准实时查询功能：（个人理解）浏览器客户端每隔 10s 从服务器请求一次数据，现场客户端每隔 1s 上报一次数据。既然是定时的，那就不会是长连接，应该也不是 UDP，所以能够无压力跨越 NAT。

其次，有分析结果广播功能：（个人理解 x2）因为数据量比较大，在浏览器客户端分析计算会比较麻烦，所以后台会先分析一部分。在分析完成的时候，服务器会发送 UDP 广播来通知其他服务器，这个当然跨不了 NAT。不过，接入 WiFi 的设备不需要接收这个（要的话也不难啊，路由改 AP 就是了）

还有一种用客户端发起 TCP 连接的功能，不是很记得（历史数据查询？），也是查询类的，可以跨 NAT。

综上所述，WiFi 端可以采用静态 IP + NAT 这种最简单的配置方式，几乎什么路由都能满足。备选方案是路由改 AP，找一台服务器做 DHCP 和 DNS。这样也是什么路由都能满足的。

（说起来，昨晚提到的那个移动端内外网都能用的问题，可以靠内网 DNS 单独配置来解决。这样一来，无论如何都要在内网配置 DNS 了，不然给路由刷个改版固件吗……）

敲定了路由没有特殊要求以后，接下来是无线环境要求。

首先，用户量在 20 人。最大数据量是 20 人同时查询一天内的详细数据：一条数据是一个浮点值（假设是 8 bytes 的原始形式），每隔 1s 有一个，一天有 86400 * 8B = 691.2KB，20 人就是 691.2KB * 20 = 13.824MB。假设服务器在 1s 内能够查询完毕（我记得学长提到了上报时先写缓存，不知道查询时有没有缓存），考虑其他开销，取最高速度要求为 20MB/s 应该是足够了。

那么问题就是 20 人，总速度 20MB/s，覆盖面积是机柜附近，大约 30 平方米。有线千兆没得跑。预算……不好说，要是学长直接说不限价格的话，直接买 UBNT 啊。但是，并没有这么说……

（搞不好还是 20 个 2.4GHz 设备……家用路由真的撑得住？）

想了想，还是用 Netgear R6220 好了。（JD 没有 Newifi D1, 不然用它的话内网可以省一个 DNS）

好了，关于我自己的这方面，我想先写一个获取一个变电站最近数据的页面，把这个 AJAX 弄好。可以开始看书了。

TODO 现在有这么多 XHR 请求了，应该要做一个 XMLHttpRequest 池。不过是不是应该引入框架呢？如果后面还有人要维护这些代码的话，还是赶紧学学框架吧。

### 2016/12/30 - 2017/01/03

Happy new year!

元旦假期，这几天主要是在做别的服务器维护工作。被 Intel 的 FakeRAID 和 Seafile 坑了……

关于这几天的 JavaScript（细节）：

1. 1 == true, 1 !== true ((1 === true) is false)
2. 在函数里面用 `var` 可以声明一个局部变量（从而遮蔽掉同名的全局变量，不会导致错误）而在没有 `var` 也没有同名全局变量的情况下，会（在函数里面）创建一个别处能用的全局变量……
3. DOM 里面的节点有元素节点、文本节点和属性节点三种
   根据 W3Schools，child nodes 只包括文本、注释和元素节点。属性节点在 attributes 里面，且用NamedNodeMap 表示，而不是其他节点一样是 HTMLElement. 比如 `<h1 id="header">text</h1>` 里面的 text 是文本节点，要通过 `h1.firstChild` 来访问。id 属性节点要用 `h1.attributes` 访问，也可以用 `[get|set]Attribute(name[, value])`。
   要改变包含文本的内容，就要先找到外面的元素节点，然后 firstChild 得到文本节点，在上面用 nodeValue 改。
4. 三大经典方法 (HTMLElement.)getElementById, getElementsByTagName, getElementsByClassName.
   注意最后一个是 HTML5 加的，它在传入 "btn-close custom-btn" 这样的参数时，选择的是 **同时** 具有这两个 class 的元素，而非选择两个 class 各自的所有元素。getElementsByTagName 可以用 "*"做参数来取出所有节点。这两个都返回数组，记得 [0]！
5. HTML-DOM 说可以用 `image.src` 来修改 <img> 的 src 属性，而 DOM Core 说要用 `image.setAttribute('src', ...)`。前者可以在 HTML 里运行，后者可以通用于任何使用 DOM API 的标记语言。
6. JavaScript 的 event handler 在返回 false 的时候会停止继续冒泡（即可以阻断默认行为）。在 handler 里面用 `this` 可以取得引发事件的元素。
7. nodeType 里的 2（Node.ATTRIBUTE_NODE）在 DOM4 里已经被 deprecate 了
8. 平稳退化：假设 JavaScript 没有被浏览器支持或者被禁用，则网页的基本功能仍然需要保持正常（比如该在本页打开的在 href 也写一遍，就可以在 JS 关闭后仍能在新窗打开）特别注意在拦截 event 的时候，如果拦截下来了但操作失败，千万记得 `return true;` 让事件继续传递。
9. 渐进增强：（平稳退化 Plus）在 HTML 以外增加一层 JavaScript 而无需对 HTML 做太多修改（至少是不要在 HTML 标签里面写 onclick 了），像 CSS 一样 hook 上去生效。可以在 JS 里用 `onload` 和 `element.onclick` 之类的方法动态“hook”上去。要注意在 getElementById 的时候要检查是不是 null！
10. JS 文件里直接暴露在最外层的代码（不在函数里的）在 JS 文件加载时就会执行，所以要用 onload 延迟一下。
11. 减少请求数量是优化加载时间的首要问题（？）
12. 浏览器兼容最好用 `if (!window.XMLHttpRequest) return false;` 这样来做，不然缩进层级太多
13. Accessbility: `onclick` 也会在按下键盘回车的时候触发，所以不需要再处理 `onkeypress` （这名字一听就坑）
14. `document.createElement()` `document.createTextNode()`
15. XMLHttpRequest 的属性里有一个 responseXML，是把 responseText 处理成 DOM 树的结果（所以它叫 XMLHttpRequest）
16. Ajax 的同源策略！而且 Ajax 要特别注意平稳退化，比如登录的时候应该按先 POST 整个页面然后显示结果页面的方式做，然后在 JS 里 hook 掉 onsubmit 来做 Ajax 动态刷新，这样就可以平稳退化了。
   但是，对于我这里做的这个网站，我觉得是没有必要做到这么激进。首先，微信浏览器虽然辣鸡，但怎么着也是支持 JS 的。其次，后面页面要求 1s 刷新一次，要是全页面刷新的话流量要飞起来了，持续的 loading 白屏也很影响正常使用，所以 JS 是肯定要有的，无法妥协。没有 JS 的话直接拒绝服务吧。
17. `HTMLElement.style` 属性只能用于内联的样式，外部 CSS 文件的读取不到。但是从 DOM 写入 style 的样式可以再被读出来。（BTW CSS 转 JS 的驼峰命名法）

归纳起来就是：

* JavaScript 的设计思路：检查可用，层次分离；平稳退化，渐进增强
* DOM 的操作方法：三种元素的增删改查

几个问题：

* 书上的示例中，需要 JavaScript 来改变的部分都是不写入 HTML 的，而是由 JavaScript 动态添加。比如图片库例子里的 placeholder <img> 以及 Ajax 结果的 <p>。这在此元素仅用于 JS 的时候是没问题的，但所有元素都动态创建是不是太麻烦了？尤其是 Ajax 结果，它难道不是本来就应该在那里，然后由 JS 来改变的吗？
* `setTimeout()` 的第一个参数难道不是直接传入函数的吗，为什么书上是传入字符串……
  查了 MDN，说是传代码字符串就像 `eval()` 一样不建议使用了。

晚上开会讨论了下需求，发现现有的后台基本都把接口留好了，问了下学长也有接口文档，那应该不是很麻烦了。具体的可以看另一篇文档。

### 2017/01/04

买的路由器到了，把它接在公网交换机上，配置 192.168 的固定 IP，访问分给我的服务器毫无压力，但内网其他服务器不可以。后来发现是因为分的服务器也在公网上（我记得第一次看到它的时候它上面啥都没插）但还配置着固定 IP，所以路由器接公网配置固定 IP 访问它毫无压力。要访问内网的话，如果要接到内网交换机，双线不可避、刷机不可避。

我先在自己的那台 R6220 上试了下，新建一个 VLAN，新建一个 WAN 接口，设置成静态地址，插上连接到内网交换机的网线，可得到内网 + 公网访问。看起来应该是需要的了。

更新：需求更改，不需要做内网访问。

更新后的网络规划如下：

为了能够进行微信推送，分给我的服务器需要能够访问外网（推送到手机），并能够被外网访问（接受手机上行消息的推送）。因此，这台服务器将运行校园网认证，并配置路由服务。

其次，由于校园网认证是二层操作，最好将其配置在单独的网卡上，以免对内网数据产生干扰。但服务器上只插了一根网线，故使用 macvlan 划分为两个虚拟网卡，主网卡用来认证，副网卡配置内网。

```
ip link add link eth0 name macv1 type macvlan
ip link set macv1 up
ip addr add SERVER_INTERNAL_ADDRESS dev macv1
```

下面来编译（主要是自己写的）校园网认证客户端。

顺便注明为什么不用锐捷官方客户端：

* 锐捷会把 NetworkManager 服务器停掉，导致网络难以配置
* 锐捷可能会检测虚拟网卡之类并拒绝认证，对认证造成影响

虽然本校客户端已经尽力去除了它的流氓行为（以上第二条），但还是要记得锐捷是一家很恶心的公司。不要用技术无罪来开脱。

由于这个时候还没有外网访问，因此把 minieap 的源码用 tar 打包，scp 传上去。注意在 config.mk 里面添加自定义 CFLAGS `-std=gnu99`，因为服务器上的编译器太老了，是 gcc 4.4.7，默认不支持源码里的某些写法。

然后启动认证的命令就很简单：

```
minieap -u USERNAME -p PASSWORD -a 1 -d 2 -b 3 --module rjv3 --heartbeat 30 -w
```

要注意 `--heartbeat 30`，因为宿舍区用 60 就可以了，所以第一次用的时候总是 45 秒掉线，后来才发现是这里的问题。其他参数可参阅软件内置帮助。

在 NetworkManager 里面把 eth0 改为 DHCP，就可以看到公网地址出现了。

下面来配置 IPv4 NAT。由于客户端电脑连接这个网络的时候就不能再连接其他网络（只有一个网卡）所以这个网络虽然不要求公网访问，但也最好要配置到能上公网，否则不便于查资料。

先打开 IPv4 转发。

```
echo 1 > /proc/sys/net/ipv4/ip_forward
```

方便起见在 /etc/sysctl.conf 中把 net.ipv4.ip_forward 也改为 1.

再设置 NAT

```
iptables -t nat -A POSTROUTING -s R6220_WAN_ADDRESS/32 -j MASQUERADE
```

注意这里 -s 是具体的 IP 地址，而不是一个段。这样限制本服务器只能 NAT R6220 过来的流量，而不能为其他内网设备提供公网访问。

最后配置防火墙，禁止此服务器转发其他流量（比如从公网过来访问内网其他设备的）

```
iptables -P FORWARD DROP # 不符合以下规则的转发数据包全部禁止
iptables -A FORWARD -d R6220_WAN_ADDRESS -j ACCEPT # 允许转发到 R6220
iptables -A FORWARD -s R6220_WAN_ADDRESS -j ACCEPT # 允许转发来自 R6220 的数据包
```

我看了一下，R6220 上没有 IPv6 穿透的设置，实在是太可惜了，不然可以同时再配置一个 Google 和 Wikipedia 直连。服务器这里也不再配置 IPv6 了。

R6220 的 WAN 地址固定，网关设置为 SERVER_INTERNAL_ADDRESS 即可。

晚上，学长讲解了快速组态查询从点击到返回数据的流程。我发现后台有一个接口可以直接调用，学长也讲解了数据库的结构，根据这里来做就好了。详细的接口文档请出门右拐看专门的文件。

### 2017/01/05

早上来了，启动服务器上的 minieap，咦，还是没网？一看 dmesg，segfault 了。`ifconfig` 发现所用的网卡又被 NetworkManager 变成固定 IP 了。想起来昨天第一次用 `-d 1` 的时候也是会 segfault，那时候也是固定 IP，看来只要网卡是固定 IP 而认证时 DHCP 开启就会 segfault。（错误发生在 libc 里面，暂时不太想去改，先把网页写好再说吧。）

于是为了避免 NetworkManager 瞎搞，再划分一个新的 macvlan，在这个新接口上认证就好了。RHEL 6.4 上的 NetworkManager 不支持配置 macvlan 这种虚拟接口，正好。

```
ip link add link eth0 name macv2 type macvlan
ip link set macv2 up
dhclient -4 macv2
```

那么今天要做的就是根据昨晚说到的接口实现界面。

P.S. 学长说可以把现有的网页的代码粘贴下来，但是那些代码中数据处理逻辑和界面耦合得太紧密了（此外缩进感人），我这里不方便用……感觉还是自己写吧。

上午都拿来写接口文档了，下午也花了一点时间来写，现在取得快速概览数据所需要的几个接口都已经有文档了，可以开始写脚本了。

下午还写了一个 AJAX 库。暂时不太想引入 jQuery，不知道是为什么……

晚上开始写界面啦。

注意一点，div 的 border 要专门指定 `border-style` 才会显示出来，不然怎么调 width 都看不见。

问题：如何在 table 的每一行中间显示分割线？

答：table 元素的 rules 属性设置为 none 来禁止显示分割线（禁止占地方），把 tr 高度设为固定需要的值，设置 td 的 border-bottom 即可。这是 tr 高度能够固定的情况。

字体大小还是要慢慢调……

### 2017/01/06

上午主要是做文档工作，记录下从连接校园网到配置域名解析网络配置流程。下午有博士答辩，感觉去了会碍事，还是不去好了（但是队友说要去啊那就去吧）

下午来填充 JS 内容吧，今天希望能把这个页面的功能完成……

说到这里，我有一个疑问：既然每个服务器上的 web 界面都不一样，那么我最后要交的就是使用这台 web 服务器的作品吗？虽然各个变量看起来确实是跟发电机有关的（定转子 etc）但是这不是第一次给我们看的那个界面啊……不过反正表格布局、数据解析和 ajax 都有了，即使要重新做也应该很简单。

（结果下午博士论文答辩，我要去东边回不来，所以下午就没有去，在周末补了一点 JS）

### 2017/01/09

来写 JS 了。macOS 的全屏模式真好用，以后带鼠标用 macOS，不用 Debian 了。不想去折腾那下面的 Sublime Text……

JS 的 `for (var x in ...)` 是遍历属性用的，不是遍历数组（不然它会遍历出数组的属性）。要遍历数组还是用 `myarray.forEach` 好了。

`myarray.includes()` 的参数必须要是数组的内容，而不能是数组的下标。如果要查找下标是否定义，可以用 `if (key in myarray)`

正反向隔离的文件传输客户端会有问题，外网端获取数据内网要过很久才会回应，甚至直接超时，返回空数据，导致 API 挂掉。学姐看了以后说是反向端临时文件没有删掉，造成效率急剧降低。这个问题交给学长了。

后来这个问题修好了，但还是很慢……

为什么 onblur 是失去焦点时执行的……blur 这个词挺文艺啊。

在 Chrome 里要让 `<select>` 里面显示的已选择项靠右对齐，可以用 `text-align-last: right;`，但 Safari 不支持。

Chrome 里可以用 select 的 onclick 方法来做点击重试，而微信浏览器不行。看来还是弄个 span 好了。

多选框在 PC 端显示为多行，手机端显示为单行（“选择了 n 项”），手机端比较好看。

disabled 不需要加值，只要有个 disabled 在 tag 里面就会起作用。并且 CSS 里面可以用 `select[disabled]` 这样来设置禁止的样式，不需要专门弄个 class。

今天只是写了一些样式，做完了逐步加载参数选择的功能而已……

### 2017/01/10

今天是个值得纪念的日子，因为可以正确地画出图表了！

前几天一直在纠结用什么图表库，一开始看的是 CanvasJS，因为它的触摸手势缩放功能很好用（是单指拖动标记要放大的区域，像鼠标一样，很精准）。后来一看，免费试用 30 天，收费吓人，于是放弃。

后来看到 Dygraphs，手势缩放是自然的双指捏合缩放，而且免费。但是它的手势缩放太快了，经常不知道缩放到哪里去了……

最后想起来 Highcharts，目前的 PC 端也是用的这个，我去看了下居然对个人用户免费。那很好啊，而且这么有名的库，功能应该不会差吧？试了下，也是双指捏合，但增益没有 Dygraphs 那么夸张，用起来也还不错，那么就选它了。

然后就是看看 API 文档，初始化参数，然后根据返回的数据添加轴和系列。不要直接操作 chart.axis 属性，而是要通过 chart.addAxis。要注意一点，`addAxis` 和 `addSeries` 的第一个参数是 `options`，要传入的是类似于初始化函数里面的那种对象， **但是** 不要带 `{'series':{...}}` 这样的包装！直接传入 `{'name':'My Series', 'data': [[1 1], [2 4]]}` 就好了。被这个坑了一个多小时，数据都加上了，断点看传过来的数据都是对的，甚至还以为是不能直接支持 Date 做 X 轴，尝试了很多时间格式，结果图上都是空的……最后才发现是 addSeries 的参数不对，那个对象我是直接写在 addSeries 的函数调用里面的（`chart.addSeries({'series':{'name':name, 'data':data}}, true)`），所以断点看不到它的结构，也就没有往这里想。

还有一点就是用日期做 X 轴的时候，series 里面的 X 轴坐标是 Unix epoch 到目前的毫秒数，而不能直接用 Date object，否则画出来的数据点横坐标是 1970/01/01 00:00:00.001，1970/01/01 00:00:00.002 这样子……

好的，那么这个历史数据查询就剩下一些 CSS 工作了，还有一些显示 / 隐藏 loading wheel 和 div 的事情，这些相比画图要简单多了。

P.S. 随手记一下，今天支付宝爆出登录密码修改验证过于简单，导致熟人可以修改自己的密码的问题。具体是因为允许了在非常用设备上使用“最近购买的物品”“常用快递地址”“选择好友头像”“连接过的 Wi-Fi 名”这样的问题来验证身份。讲真，这不需要是熟人（关系好的），只需要天天能见面就可以了……我把余额宝和余额都转进银行卡了，然后挂失，吃个饭回来支付宝说升级了，以后那些措施只能在自己手机上用，于是又把钱转回去了……（感觉好奇妙）

### 2017/01/11

程序员节？

今天是要给图表加样式，图表出现了以后隐藏选单，然后给个返回按钮退回选单。没什么好记的，慢慢调吧……

嗯……还是有要记的：`display: none` 不能触发动画效果，毕竟变化太大已经不能算是 transition 了。找了一下，比较好的做法是给 `max-height: 0; overflow: hidden` 加动画，然后视情况决定要不要 `display: none` 掉。要的话，可以在 `addEventListener('transitionend', ...)` 里面操作。

### 2017/02/13

万万没想到程序员节的记录后面就要到情人节的记录了啊啊啊啊啊！当然我不过情人节。

然后是迟到的新年快乐啊~寒假又长胖了一公斤，害怕。

在家里做的工作主要是翻译，13 页的英文，还有很多图和表。本来以为可以很快翻译完的，结果翻译到中文还是有 1W 字了，看了备份记录，10 天才翻译完……在火车上两个小时肝完了最后 1.5k 到 2k 字，发现在家效率真低，一小时才 0.7k 字左右，不然五天就翻译完了。顺便一提，Markdown 用于论文除了公式以外的部分都挺舒服的，直觉的加粗和分级标题方式不知道比 Word 高到哪里去了，并且 [Typora](https://typora.io/) 也不知道比 Word 流畅到哪去了，还能带图片实时渲染，光标移走以后自动隐藏格式符，66666. 唯一的问题就是搜索的时候从搜索框点到文本区域，光标可能会跳走，需要重新点击搜出来的位置……

在代码方面，看完了 JS 与 DOM 的入门书，对平稳退化与渐进增强有很深的印象。还有一本响应式设计实践，看了一大半，主要记得的东西是要合理安置 HTML 文件中元素的先后顺序，这样即使 CSS 不控制位置和大小，过大的元素也能被浏览器直接排到下一行显示，即不依赖 CSS 实现“响应式”。当然这只是极端的 backup，只是说要遵循这一点来写网页罢了。

回到学校以后，开了次组会（同学迟到被挂了），更新了一点要求，可出门右转查看。然后把相关的服务器全部开起来，根据以前的文章把校园网认证啊 NAT 啊啥的都运行了，然后写了篇综合的 Boot Sequence 介绍，以后可以直接抄里面的命令了。BTW 算是见识了服务器开机有多么吵，开机瞬间风扇开始全速转，转个 5 秒 8 秒的讲真耳朵要爆炸，还好不是两台服务器一起开……服务器启动的流程也是长，BIOS 要等个一分多钟，看到 Scanning devices 都要等好久，比较有趣，有机会买 Gen 8 玩 #(滑稽)。

剩下的时间是把 Markdown 转换成 Word。我怀疑我用的是假 Word，要么是 macOS 上的 Word 有 bug, 因为我在做分级序号的时候它硬是不能找出自己的第一等级编号是多少。也就是说，大刚显示“2. CMFD 概念”下的段落，设置分级序号以后是从 1.1 开始的…… 原因未知。并且，本来在一级标题以后二级序号要自动重置的，这里也没有重置，即一级标题“2. CMFD 概念”以下有 1.1, 1.2, 1.3，而一级标题"3. CMFD 实践"下有 1.3, 1.4, 1.5. 简直丧心病狂。手动在每个一级标题下重置了序号，然后把“2. CMFD 概念”下的二级序号依次删掉，手动改成，结果改完了 2.1 和 2.2 以后 2.3 以及其后所有的二级序号突然都自动更正了，能读到自己所在的一级标题序号了，迷。

今天还开了一大堆 ticket，比如 SS 某线路 80% 丢包但一旦通了延迟就很低，再比如 Logitech Options 跟 macOS 的安全键盘输入不兼容导致滚轮失效。如果上午服务器没开导致半个上午没网然后叫学长来解决也算的话，那也是个 ticket……

最后，准备加实时数据显示了，把已有网站的 Flash 弄下来反编译，发现跨域安全策略服务器和实时数据推送服务器都没开机。尴尬。明天再去开这个 ticket 吧。

（总部好像也需要搞个 Boot Sequence 文档，不然没人能接我这一届的盘了）

### 2017/02/14

真的到情人节了诶，然而我还是要在这里写文档 23333

今天先把文字部分在 Word 里的格式排好，然后把图表抠出来翻译。还好 Science Direct 上可以下载到原图，表格也很方便复制，花时间做就是了。

结果两小时过去了全都在写其他的文档 [doge].

### 2017/02/15 - 20

总结一下这段时间，就是准备调剂资料为主，实验室工作没有那么主要了。

15 号起可以查考研成绩了。分数符合预期，于是要准备调剂材料。16 号顺手把攒了一个寒假没投的简历投出去了（我应该会记得第一个简历是投给谁的），在 20 号变成“已处理”状态，有点方方的。希望能成吧，但假设面试笔试都过了，拿了 offer 了，最后要不要去呢？我内心是有点希望读研，但是看到大部分研究生的样子，觉得要做导师的事吧往往没有什么特别好的事要做（相比自己要做的而言），要做自己的事吧又被导师的事逼得没有时间，而且还没多少工资。既然工作和读研都是一样的不那么理想的环境，为什么不工作呢？我真的怀疑上一辈人的经验在软件领域不那么适用。或许通信这样的会有点用吧。

我现在一听到别人在问要不要考研（甚至考研要准备什么）都会条件反射一样的问你真的喜欢你的专业吗，真的要走科研的路吗，你为什么要考研？感觉这么做不太好，但是，我总觉得不出于自己内心的选择的话，考研和读研都是十分痛苦的。毕竟这不像高考，周围很多同ji学you推着你走，而且要学的东西都是资料充足做到手熟就行，真的“题来伸手点来张口”。而研究生呢，我的话首先是没人陪，当时心情真的爆炸；其次研究生是要突破知识边界的，要付出的努力远非高考能比，即便是按照 18 岁和 22 岁的精力、学力来折算也不行。没有内在意志的强力驱动，我真觉得是做不到的。我不希望我在意的人因为这些原因走上并不适合自己的路，更不要出现“自己考的研跪着也要读完”这种情况。有个很要好的朋友（学弟）不是计算机的，打算按部就班地学完本科去考本专业的研，但是他也像我一样研究了很多计算机方面的东西。我也像上面那样问过他，他说他确实对他的专业有些兴趣。可能我管太宽，但我真的希望他不要不明不白地被众人裹挟去了不适合自己的方向。他现在在计算机领域表现出的能力比我当时强多了，说起来我有点羡慕他……

> 插播题外话：16 级的学弟们一上来就有 13 到 15 的老司机们带，路由啊 NAS 啊还有 Linux 啥的只要有问题就能立刻得到帮助，不知道接下去玩什么的时候也能立刻得到新的意见。如果我当时就有这么充足的资源，是不是可以提早一年开始进入 Linux 运维相关领域，然后学到更多东西，甚至于会有更多拿得出手的项目经验？
>
> 同样的，我也很羡慕 15 的学弟们（虽然就俩），不过对于他们的话，我更多是羡慕他们的技术吧…… 虽然技术是可以学来的，但是自己这会儿确实是没有啊。而且快要毕业了，不能再有他们那样的环境来全身心学习自己想要的技术了，心痛……

实验室方面的主要工作还是文字工作。文献翻译的图和表很多，尤其是图翻译起来很麻烦，翻译了两天才把图翻译完。（吐槽一下， PDF 里面的图比 Web 版论文里的不知道清晰到哪里去了，遗憾的是我直接拿 Web 版图片做的翻译底片，PS 放大以后边缘糊得乱七八糟，但是已经做出来了…… 还好图层留的比较好，只需要少许修改就能替换，但有时间再说吧）而表的话一天就翻译完了。大概在17号左右翻译工作全部结束。***TODO***现在把翻译的文献打印出来就可以直接上交了。

文献翻译结束以后是开题报告，目前无论是调剂还是工作都有很多东西压着（我总觉得简历写的不很好，后来把每个项目的 star 数加上了，不知道有用没）所以开题报告只是看了一天文献然后写了个框架，包含目前的发展情况以及一点点趋势，展开的话恐怕也就 1k 字，而总共需要 3k 字呢，剩下的都写我自己对本项目的预想架构？

### 2017/02/21

昨天还是上周用 Word 模板弄了个 CV，看起来清新淡雅，可以的（问题是 macOS 下的 Word 实在是太 bug 了：表格形式 N 行 2 列布置的文档，在一个单元格里插入另一个表格，随便改一下这个表格的宽度，外层 2 列的表格就会坍缩，第二列会把第一列挤得没地方。明明指定的宽度远小于外层第二列的内容宽度，不太懂为啥会让外层表格炸掉。）

CV 内容跟 AI 简历一样的，不知道老师会不会喜欢这种充满暴力实践的简历…… BTW 似乎可以把个人项目都写到招聘网站规范化简历的“项目经验”部分去，那就有好多能写了，开心。

现在发现的校招涉及的招聘网站有：

* 智联（zhaopin.com [doge]）
* WinTalent
* 大街（在职位页注册会失败 23333）
* 拉勾

要写那么多遍简历还是挺捉急的，复制粘贴也要好久。打那么多遍手机号和 QQ 号也比较方，万一泄露就去求了（我好像写了备用号？）此外，简历要知道我的身高体重干嘛？

现在要开始投简历了，工作和预先调剂的都要。有些网站还需要成绩单打印，但是能自助查询的成绩单都有那门没考试的公选课，很难看，或许应该去新放置的自助打印机看看。工作的还要额外去招聘会，手机的日历里记了好多了，接下来的日子会很忙呢……

什么？你说实验室工作？一点都不想做了啊，满脑子都是简历啊面试啊工作啊……

（有一点有趣的地方，有些手机公司第一次来我校招聘就赶上了诶）

### 2017/03/01

这段时间沉迷投简历不能自拔，先后投了 PPTV、vivo 和一加，拿到了 PPTV 面试通知，目前面试还没结束；而 vivo 今天完成简历筛选，状态变成已处理，但现在还没结果，估计药丸。一加昨晚笔试，专业部分 45 分钟要手写 KMP、有序链表合并，还有一堆写出运行结果，仔细做不可能做得完啊。通用部分题目很奇怪就不多说了。说是连夜改卷要出结果，但并没有收到通知，估计也 GG 了。还有 TP-LINK，投了简历变成已处理，估计跟小米一样挂掉了。所以找工作还是很难啊…… 所以到底应该去读研还是找新的工作呢？

昨天还有一个有趣的数据恢复工作，除了常规的解压缩以外还要从 lost+found 里面捞出一大堆东西。今天有空继续吧。

晚上还有公选课，233333 不过是基友的委托，还是要去的。

### 2017/03/02 - 03

1 号晚上那个公选课有点尴尬，被点上讲台做自我介绍，于是在前面有 5 个大一的和一个大二的做完自我介绍以后，我就只能说“说出来你可能不信，我是计算机系大四的”…… 场面一度十分尴尬。

（而且，以其他人的身份做自我介绍，好像也怪怪的）

（此处应有梁非凡）

关于简历的后续：

* 一加的面试通知是 1 号中午打电话的，约的下午面试，但因为有 vivo 的模拟就推到第二天
* vivo 的笔试通知是 1 号中午发短信的，笔试是当晚，下午有模拟测试
* TP-LINK 的笔试通知是 2 号晚上发短信的，并且认识了一个投 TP-LINK 的同系同学（当然他报的硬件方向）

面试一加的时候是在他们下榻的酒店，每个房间里有一个工程师进行面试。面试官问我要了简历（通知里没说，还好准备了），然后简历从头到尾都问了一遍，试卷上的没写的问题也问了。目前我记得的可能有些问题的地方是

* 我说了做过一个很简单的 STM32 NAND 编程器，他问我 NAND 的特征是什么，我一下子想不起来是 0 变 1 还是 1 变 0 变不回去…… 想了一下说是 0 变 1 后变不回去，对了。

* 问卷子上的有序链表插入，我说一时没想到递归怎么写，循环又没有时间写。但是后来口述了比较插入的循环逻辑，应该没问题。

* 问简历上的 MCS51 项目，说知不知道 51 单片机上电后是怎样启动的，我说不太确定，只知道会有一段汇编把 C 环境初始化好，然后再执行 C 语言的部分。再前面硬件的部分不是太会。

* 还是 MCS51 项目，问串口操作是阻塞性还是非阻塞性，我有点不明白为什么这么问。读串口不是轮询或中断的么，写入也只是写寄存器，一条指令的事，显然非阻塞，怎么会有阻塞的串口…… 问了一下终于弄清楚是什么意思，说是非阻塞，没问题。然后问知不知道怎么初始化串口，我想这么简单的项目当然是我一个人写，肯定自己初始化啊，但是他的意思是是不是用了库。这也是问了一遍才知道他的意思的。然后又问发送数据是写完寄存器还有什么操作（应该是硬件方面的问题），我也想不出来了……

* 总体说话可能有点啰嗦，因为我想把前因后果都说清楚。也许应该要直截了当地说出结果。

* 思考和说某些话的时候会摇头晃脑，可能对方不习惯。

* 问为什么昨天下午推迟了，我说有事，他问你有没有参加 vivo 的招聘？我赶紧说是去试了下，还好没来得及感觉到尴尬。要是真挂了，我感觉这里最可能是导致挂的地方……

  （突然开脑洞：他会不会觉得我这样子肯定能拿 vivo 的 offer，而 vivo 开出的工资比一加高，所以即使给了我 offer 我也可能不来，也就没有必要浪费名额？当时应该强调一下我更欣赏一加的……）

* 还有职业规划，我说我现在还是需要多学习的，做好自己本分的工作，然后努力学习，希望能成为负责 BSP 更多方面的人。不知道说起来是不是太低了点，或者听起来不清晰。

* 最后还有什么问题的时候问了工作用机的配置，是不是问得太早？

面试官全程 poker face，不像上一次面试的能看出喜怒。写到这里的时候，差不多 24 小时过去了，还没有通知，等到今天中午看看吧…… 还是挺喜欢一加的。

关于实验室和学校方面，昨天下午学校检查搞事情，详细的不说了。实验室方面，早上迟到了一点，也是不太顺。开题报告方面，昨天写了一部分，主要写的是课题来源 / 意义 / 要求，包含为什么微信适合用于推送。今天学长说研究现状要好好写，也就是文献综述。这样的话，要花很多时间…… 周末加班吧。

今天下午有 vivo 面试，要带很多证书啊证件啊，下午就不能来实验室了，得自己去收集起来复印。

实验室的内外网隔离又坏掉了，最后发现问题是外网请求能穿透到内网，但内网的那台机器上面有两个 IP，穿透进内网的请求从另一个 IP 发出来了，而这个 IP 没有被允许通过隔离，所以回应丢了。说起来我还不是很清楚穿透隔离的通信是什么原理，明明内外网 IP 都不是一个段的。难道是基于硬件地址进行 NAT？而且 TCP 连接怎么可能一个 IP 进，另一个 IP 出？四元组好好的在那。如果隔离设备是需要在内网主机上安装客户端软件，用 UDP 通信的话，倒是有可能……

### 2017/03/06

新的一周开始啦。

昨天 TP-LINK 第一次面试，面试官问的很专业，802.1x 什么的能问下去很深，但是不要不食人间烟火啊喂，锐捷的 802.1x 真的有很多修改啊😂 互相听不懂对方在讲什么甚至还有点小小的冲突，怕是要挂了。

还有点问题，对方问你会反汇编吗，我说会一点，最近是在破解网卡白名单的时候用到的。对方问白名单是什么，我说是 BIOS  里记录了允许使用的网卡 ID 的名单，如果网卡不在里面就不能开机。说完以后我看对方“请继续你的表演”的表情，就把为什么我需要换网卡、需要用反汇编这么麻烦的方式来修改都说了，结果说到一半被打断 233333

面试出来收到个 offer，第一个哟，感觉不错。但是这个命中率好低…… 总觉得没有三个 offer 都不敢跟人说话……

（告诉一加我还爱它）

实验室方面，为了写研究现状，需要看已有的综述。去谷歌搜了下主题，前三篇全是综述，引用次数几千几百的，不知道比百度高到哪里去了。

啃文献去了……

### 2017/03/07

结果昨天下午做招银网络科技的在线评测用了两个多小时，明明选的 Android 但题目在专业方向的全部是后端，SQL 啥的。要是整张卷子都是组原、算法、数据结构也没有任何问题，我报的 Android 考这么大一堆后端是怎样？当我的方向是闭着眼睛随便选的么？

然后没有心情再去看文献，然后还为这个辣鸡笔试跟 Zeus 吵了好久。（我确实希望你把自己看高一点，这样就不会被人欺负了还觉得别人是对的。）

今天上午 TP-LINK 第二次面试。面试官不太年轻。上来先走套路自我介绍，说到一半说你做了 802.1x 是么（历史重演），我说是的，然后开始讲。刚讲没两句他说他不是做技术的，直接跳过所有技术部分…… 于是就有了下面的对话：

- 面试官：你说的技术名词我都不知道，你直接给我讲讲认证是怎么回事

- 我：（那你是什么职位？）就像家里上网要拨号，要用户名和密码一样，学校也是

- 面试官：那都是要用户名和密码，学校和家里有什么区别？

- 我：（你不是不要听技术细节吗，那我怎么讲 802.1x 和 PPPoE 的区别，反正都是给你上网的）802.1x 认证后直接就可以用以太网通讯了，PPPoE 还需要建立 PPP 会话，把数据都封包到 PPP 帧中

- 面试官：那学校为什么不用已有的 PPPoE 而要用 802.1x？

- 我：（这种私密性的东西我哪知道？人家喜欢呗）我个人感觉是因为校内带宽过大，那么大量的数据包的封装和解封装对交换设备压力太大。

  （我还说了因为外面的带宽不高，所以包转发率要求不高，学校里带宽高，包转发率要求就高了，所以导致设备压力大，所以导致不能用 PPPoE。然而他并没能听懂，觉得带宽大小跟 PPPoE 需要封装没关系，我解释了好半天也不知道他听懂没有）

  （注：后来学弟说是因为锐捷这样的成套方案提供商用的是 802.1x，学校也没必要换）

- 面试官：这个说法是有数据支持的还是你猜的？1400 字节加两个字节看起来没什么问题？

- 我：（我要能知道具体数值那岂不是已经在网络中心上班了？）这是我根据某些系统 PPPoE 协议栈有效率低下的问题而猜想的。（也就是说，系统可能会处理不好 PPPoE 封装问题，那交换设备也不是没有可能性能不足）

  （注：后来确认了 PPP 有 8 字节额外开销而不是两字节）


- 面试官：那你为什么要自己写程序而不用标准的 802.1x 呢？

- 我：因为锐捷在标准的 802.1x 以外扩展了字段，需要用它的客户端才能认证，而我用的平台上没有对应的客户端。

- 面试官：锐捷为什么要用私有字段呢？

- 我：（我已经要炸了，我__怎么知道它为什么要用私有字段，人家商业机密是让我知道的吗）因为锐捷的设备在校园网中很多，用私有字段可以限制用户全部用它的设备和服务（所以可以多赚钱）

- 面试官：锐捷为什么能覆盖那么多学校？

- 我：我！哪！知！道！

- 面试官：所以说你没有通过网络中心（我：网中不干事）论坛或者其他渠道了解？

- 我：我只是急着要写个客户端让我的机器通过认证，我管那么多为什么干嘛，能解决认证问题吗？！

  （当然实际上是“好气哦但还是要保持微笑”这种状态来回答的）

真的是好气，我__要能知道人家商业决策我为什么还在这里面试，我早就去人家公司入职了好吗！

最后就只问了这一串 802.1x 的问题，面试官就不想问了。问我还有要问的没，我大概就问了

1. 贵司哪方面职位缺口最大？

   答：你自己的定位是什么？/ BSP

   我司确实是 BSP 需求最大。/ 但我对上层的协议也有些了解，比如 IPv6 啥的

   会有你的用武之地的。

2. 贵司是注重代码可维护性还是运行速度？（对速度优化的要求有多严格）

   答：这个问题很好，但不好把握。但是如果赶上产品上市，就顾不上了。（我感觉他理解错了。此外，他难道是以前坚持代码可维护性，然后多次被 DDL 压迫从而放弃？）

3. 贵司为什么不在国内市场上高端型号？国外品牌把国内高端市场都占完了

   答：市场决定。/ 但是高端机单个利润很大，用户群也不小，尤其高校用户，带宽大，性能要求高。

   你没有具体数据，是不能让市场部觉得你对的。

这面试肯定药丸。

下午早点走，去拿三方寄给唯一一家发了 offer 的公司吧。我本来还觉得 TP-LINK 会更适合我的，结果两场面试都在肛，恐怕还是算了吧。

### 2017/03/08 - 11

祝天下母亲妇女节快乐~

工作的事情终于尘埃落定了（诶，听说腾讯还会来？）短期内应该可以搞实验室的事情了。

顺便吐槽下把微信群当工作交流工具的，你们是不知道不带文件管理、不带发言引用的群有多难用吗？不过这样倒也好，降低效率，可以放假。

（其实还有一个槽点，在这样的微信群里面不敢水啊，我还怎么认识同事）

考研调剂方面，先前投了一堆简历，但都没回应。感觉真的去上的话拿不到现在这一份工作，但别家会不会好一点呢？（我在某群里看到的了来自中科院大学的同事——我现在对于“同事”这个称呼还是有点不习惯）调剂意向采集系统开了，但是真正的填报还需要等等。先填几个喜欢的吧。

实验室工作方面，这几天都是在写 & 改开题报告。说要具体到我研究的设备而不要只讲到电站整体，我也想啊，但是我真的没找到能写得那么具体包含什么故障要用什么变量怎么设定阈值怎么进行变换的论文啊，是不是实验室的成果还没有公开整理发表😂 最后要改的部分几乎就成了学姐的报听写，辛苦学姐了~ （虽然了解一下背后的原理是好，但是单纯为了项目似乎要求就可以不要那么高？先设计操作逻辑不是更好吗）

不过值得庆幸的是，只要涉及到计算机技术的部分都不需要改，加了几个图就好了哈哈哈哈哈

最后周六加了一上午班，开题报告算是结束了。希望能开始专心写代码了。Flex 布局好像很厉害，试试吧。（可是我还是更习惯 Android 的布局）

（什么，又加需求了？）

有一点其他事情要说的，周五晚去了总部，感觉学弟 / 朋友们现在开始见一次少一次了…… 希望在走之前能合个影啥的，一直保留下来…… 这是大事😂

### 2017/03/13 - 18

周一，下雨，66666

周二，公益劳动拔草，西十二的草抠半天抠不掉，半天就手疼，干了一天以后戳手机都疼。西十二中庭靠前那片草地边缘起码有两碗不知道多久了的热干面倒在那里，拔草拔到那附近真的是…… 顺便翻出了个 2011 年的瓶盖，上面写着“谢谢惠顾”。它在六年的公益劳动中都存活下来了，很厉害。

周三，还是拔草，去青年园及西体附近。有了锄头和锹（但是明显是锄头更好用吧）做起来比较方便，但工具不够，可以划划水。早上下雨了，老师说去找个楼避雨，结果下了半小时居然停了……

周四，继续拔草，去设计学系附近。有锄头，但主要还是用手。这边的草好拔多了。第一次知道那些跟草长得一样高度、一样形状但更绿的植物也是杂草。感觉可以做一个杂草种类分布分析。那么多人在一片草地里拔，正常的草都踩得差不多了。下午同样经历下雨、躲雨、雨停这样的“大起大落”哈哈哈哈哈哈

周五，TGIF 但还是拔草。已经是闭上眼就能看到拔草的画面了。下午真的下雨了，66666

总结：累成狗

### 2017/03/20 - 23

回实验室，一开始辅助找了两个数据接口。周二开题答辩，结果老师说做的方向完全不对！？本来是任务书都说的是做一个读取、绘制历史数据和实时数据曲线的程序，结果现在变成了人员和设备考评，类似于问卷系统。完全是两个方向好吗！虽然老师说一开始是学长和我们都没理解清楚，但是既然同意了任务书怪谁咯？实际上大家心里都懂。要求本周内改好开题报告，可以的很强势，所有方案全部推翻重写。之前写那么多文档和代码也基本就废掉了。切身体会 PM 改需求的感觉。我的任务从写网站变成后端（推送 + 问卷），嗯这跨度可以也是很可以很强势的。

周三要交人员评价系统的设计方案，做完了，发给学长了，不知道学长给老师没有。反正现在周四了没得回应。

肝 Django 文档去了。Django 方面有大腿抱，不然我后端真的毛都不会……

### 2017/03/24

这周一直下雨。我本来觉得我不会因为下雨而导致心情不好，但是回想起来，觉得这个星期心情确实一直都不好啊…… 今天跟老师又讨论了一些东西，定下了模型，最终拟定出了基本的接口文档给客户端的小伙伴们。最初设计部分应该是完成了，下周开始码吧。

### 2017/03/27

周末搞搞了一天 Gen8，好像这两台都有问题，1 和 2 槽插 4T 硬盘都有很大概率不认，设置 AHCI 和 RAID 模式都是这样。拿另外两块 2T 盘一起插，就没问题了。三块盘每次都全认，两种模式都可以。随后发现 4T 盘插 3 号位能稳定认，也是两种模式都行。然后想想反正机械硬盘 SATA2 也无所谓，就插在 3 算了。终于能稳定认出硬盘了。

去 ESXi 下面新建虚拟机，装 DSM 6.0.2 怎么都是卡在“文件可能损坏”。查到是 SataPortmap 的问题，但是忘了怎么改，放弃。然后装 DSM 5.2，能装上，但套件中心新加社群无限 loading，官方源一直提示“Synology 服务器正忙碌，请稍候再试”，估计是一定要洗一下了。晚上再说吧。

回寝室以后要用某特殊方式给服务器和软路由配置备用管理 IP，于是走之前插好交换机的 Console。在寝室配的时候要把路由的 WAN 口从 access 改成 trunk，哦豁，断了。后来想想好像没什么办法能保证改 trunk 时不断，把 Uplink 的 PVID 设成原先网段的？在改 trunk 的时候还是来不及敲 trunk PVID 就断了啊。

这里有点问题，我还不知道怎么在交换机上设置某 port 在某 VLAN 内的 tag 和 untag 选项。用 PVID 是能够实现一部分 tag 和 untag 的功能（本接口传出包自动打 tag），但是如果是外面进来的带 tag 的包转发到这个口上，那个 tag 会不会自动 strip 掉呢？不会的话，那 untag 的实现就不完整了。还是得再问问学弟或者查查文档……（补充：根据华为论坛上的描述，在 trunk 模式下转发出符合 PVID 的数据包时会自动 strip 掉 tag，那就完全符合 untag 的概念了，可以的）（注意：交换机内部都是带 tag 处理的，不存在“进来的包不带 tag”的情况。进交换机时不带 tag 的包也会被打上相应端口的 PVID 进行处理。所以这个问题看起来就很幼稚了——已知外面不带 tag 的包从 Uplink trunk 进来，能从同 PVID 的另一 trunk 不带 tag 出去，而交换机内部无论是什么包都打 tag，也就是说带 tag 的包出去时没有 tag 了，那就证明同 PVID 时会自动 strip 掉 tag）

PyCharm 好像对中文注释不太友好，PEP-8 要求 # 后面空一格再加注释，但是写成 `# 设备地址` 这样的话就提示PEP-8 不满足，“inline comments should start with #”。写成 `# Device address` 就好了……

Django 里面 `OneToOneField(SomeModel)` 和 `ForeignKey(SomeModel, unique=True)` 的区别在于，从 SomeModel 反查对应的此模型的对象时，`OneToOneField` 使用 `SomeModel.thismodel` 直接取得对象，而 `ForeignKey` 使用 `SomeModel.thismodel_set.all()` 取回一个集合，里面有一个 `SomeModel` 的对象。

### 2017/03/28

沉迷反编译不能自拔。一些 RAW 形式的 binary，载入 IDA 以后如果发现跳转地址都不对，比如相比正常偏移量多了 0xF00000，那么可以在 Edit -> Segments -> Rebase program 里面设定 image base 为 0xF00000，重新分析以后跳转就正常了。

无法跳转到的偏移量一般会以红色标记，统计一下所有跳转位置的规律就能看出 base 在哪里了。

还有一个比较坑爹的地方，就是看着那里自动标注了符号（比如 `unk_13ef0`）但 List xref to 就是没有，全程序反编译了也搜不到这个符号，全部汇编搜 `0x13EF0` 也搜不到。这可能是因为 IDA 尊重原始指令，原始指令是相对寻址就显示成相对寻址，即使能够推断出那块内存被访问了也不会把相对寻址替换成变量标号，可以在附近找个别的函数，看看调用全局变量的时候用的是什么寻址方式。比如看到调用 `dword_12896` 的时候用的是 0x2896 这个地址，那么可以搜一下 0x3ef0 这个立即数看是否会搜到调用点。 **（前面的原理完全是在口胡，别信，我只是被减掉 0x10000 的情况坑了所以才这么想）** 

接着，反编译插件有时候也不靠谱。一个函数可能会少参数，那么汇编里有的变量或者子程序调用在反编译里就没了。比如看到 `MD5_Final();` 这一行，就会想到缺了参数，至少要 `MD5_CTX*` 这样的参数吧。因为是从 OpenSSL import 进来的，可以使用 OpenSSL 的定义：`MD5_Final(unsigned char*, MD5_CTX*);` ，按 Y 更正以后就变成了 `MD5_Final(&unk_13ef0, &ctx);`，那个不见的变量就出来了。

（相对寻址真神奇）

Django 这方面，调整了下 admin，感觉大体能用了，admin 细节再优化一下就开始写 JSON 借口吧。

### 2017/03/29

昨晚反编译看到了一些奇妙的东西…… 然后花了半个上午实现，中午借来路由测试了一下，稳了，然后水了一帖，感觉不错。

（果然在 *nix 下写东西舒服多了）

继续优化 Django 后台，调动顺序啊，显示 ForeignKey 啊……

后台一般要做这些改变：

* 详情页显示自动填充的字段，比如 `auto_now_add`
* 详情页显示 ForeignKey 字段
* 列表页上显示更多字段（默认只有一个标题）

`auto_add` 的字段要用 `readonly_fields` 来让它显示，默认还显示在最末尾，可用 `fields` 控制顺序。

可以 override `save()` 来做自动填充 hash 字段之类的功能。要检测变更的话，在 `__init__()` 里面读取原先的值，保存到一个属性里，在 `save()` 里判断就好了。但是不知道这个实现是不是足够优雅……

有些时候需要综合输出一个字符串，而不单是模型的字段，这种时候可以用定义一个类函数 `def func(self, obj)` 其中 `obj` 是模型实例，函数返回一个字符串。在 `readonly_fields` 中加入这个函数的名字就会显示此函数的返回值。

可用 `post_save` `post_delete` 之类的信号来实现 OneToOneField 与原模型的同步新建与删除。这里有一个包含 OneToOneField(User) 的 UserExtra 模型，用于来扩展 User 模型，可以用这两个信号来实现 UserExtra 和 User 同增同删。

### 2017/03/30

改 & 加需求啦！

改的需求比较简单，是把被测评人这个字段从 QuestionTemplate 移动到 Project 下了，因为一个 Template 只能测评某一方面，而 Project 是针对同一被测评人的各方面 Template 的集合，所以被测评人字段按逻辑是放在 Project 下的。之前文档里没说到 Project 是集合各方面 Template 的含义，我以为 Project 是很粗糙的分类，底下 Template 可以有不同的被测评人，所以就放在 Template 里了。

加的需求就比较 233，需要用用户选择控制答题进程。比如

```
1. 您是否愿意参加本次调查？
  Y -> 转下题
  N -> 提交

2. 您是男性还是女性？
  a. 男性 -> 转第 3 题
  b. 女性 -> 转第 4 题
```

目前的想法是，每**选项**设置一个“退出去向”的字段即可。“退出去向”包含“转入下一题”、“转入指定题”两种。

当用户完成此题时，需执行此字段的逻辑。仅需这一个字段就可以完成复杂的逻辑了。

### 2017/03/31 - 04/01

时间太久远了，不太记得这时候发生了什么…… 可能继续沉迷反编译了？

想起来三方和就业推荐表是 4 月 1 日下午寄出的。稳了。

### 2017/04/02 - 04/04

清明节假期。虽然不符合本文标题，但是现在既然已经变成日记了，那还是可以写写的。（而且我也想写写）

我自己的路由在第一天到了，前一天晚上就已经写好了 DTS（我记得是从看 NVRAM 里面的 reset_gpio 开始改 DTS 的），固件也编译完了。之前已经发现 TFTP 传过来的固件如果不是 DTS 格式就直接解压执行，那就很美好了。编译出 initramfs 的，然后提取其中 kernel 那部分（`file` 看的话是 LZMA-compressed data），TFTP 发过去就好了。首先看到一堆 mtd_read error，后面大意是数据校验失败，感觉可能是之前一直怀疑的 NAND ECC 配置有误。这个错误还引发 `/dev/nvram` 找不到的问题（这种情况下都不会重启，LEDE 也是稳）。试着读了下 mtd0，发现读出来的数据是正常的，那就真的只是 ECC 配置问题了。大喜。反正 initramfs 也不会写 Flash，随便改 NAND 配置都不会有问题。于是把 include  ECC strength = 8 的文件改成了 4，好了，居然不用改成 2 和 1 诶。`/dev/nvram` 也跟着出来了。那么基本系统就到这里了。

第二天搞无线。按理说 BCM4366 是有 brcmfmac 支持的，可是没人说 BCM4366 还分 B1 和 C0 两个版本啊卧槽。B1 版有公开固件，C0 没有。而网卡必须要固件才能运转。于是找 C0 固件提取方法，从各种路由器里面提取（参见另一篇专门讨论固件性能的文章）然后测试。最开始看 GitHub 的 issue 说 EA9500 用它自己的固件可以让 WiFi 跑的很欢快，而我用路由自己的固件则全面超时，所以我也用了 EA9500 的固件。但是这个固件只有在手动启动 hostapd 时广播出的 SSID 才能被客户端连上，LEDE 自己的 WiFi 控制按钮怎么开都是能广播但连不上（密码错误或者拒绝关联），配置文件都一样，很奇怪。于是干脆手动启动 hostapd 做了性能测试，此后所有固件都是如此。在总部跑了整整一天，跟学弟一起测了很多数据，最后发现 EA9500 的固件性能最均衡（信号强度和速率均为第二，而两方面各自的第一在另一方面都不怎么样），于是选择它加入 LEDE Buildroot。加的时候觉得光一个 EA9500 是不是太少了，于是把 R8500 的固件也顺手加进选项了，反正是复制粘贴的活儿。

编译完以后，无线（当然）还是要手动开 hostapd。这不行啊，跟 LuCI 就分离了，不好控制。尝试了很多办法，试了很久 brcmfmac 的调试（吐槽下谷歌，它的调试是往 `/sys/module/brcmfmac/parameters/debug` 写 mask，而不是带 `debug=XXX` 参数加载 kmod），启用了很多日志，逐行对比了都没能看出什么差别——在客户端连接之前的日志打一大堆都一毛一样，客户端一连，bia，hostapd 立即蹦出 disassociated，brcmfmac 一言不发。强行要说区别的话，就是客户端连接后手动启动的 hostapd 那边 brcmfmac 会报告建立了新空间流，而自动启动的那边不会。可是这又不是初始化阶段的区别，我怎么知道为什么没新建空间流……

binwalk 都上了，发现固件是 VxWorks 的，放弃。这时候已经晚上十点多了，最后想，会不会是 EA9500 固件有问题？拿 strings 看了固件版本号，发现路由器自己是 10.10.69.x，而 EA9500 是 10.10.122.x. 换个其他 10.10.69.x 的试试，比如 R8500 的？scp 进去，重启，看着手机从“正在连接”跳到了“正在获取 IP 地址”，卧槽连上了！卧槽！卧槽！连上了！贼激动！原来固件能 load 进去不代表能用啊……

（后来发现 EA9500 的固件会出现 160MHz 选项，而 R8500 不会。EA9500 的固件版本号里面也有 -dyn160 这样的字眼，怕是这种问题导致了不兼容）

（写到这里忽然想起之前在哪搜到过一个补丁，说是 160MHz 和 80+80MHz 的频段上可能有某些客户端连不上，于是按照 80MHz 处理，后期如果发现主信道和 sideband 差距太大就表示实际上是 160 或 80+80，于是就可以还原了。不知道需不需要重新打上这个补丁。但是这是路由器用 Reason 7 （Better AP available）主动 Deauth 了客户端，应该不是吧……）

那天晚上就很兴奋地把 config 里面的 BCM4366C0 固件选项从 EA9500 换到了 R8500，然后编译了个新固件，刷到自己路由里，没问题，顶着困意写了帖子，但考虑到公布 TFTP 的方法似乎不太好，就只是存了草稿，睡了。

（顺便测试了下 BCM4709 和 BCM47094 两个 dtsi，发现 USB 3.0 都能用，就不是很懂有什么区别了）

第三天打算发布了，但还是觉得不想那么高调地把 TFTP 宣扬出去，万一厂家把 telnetd_startup 的 debug flag 和 CFE 的自动 tftp 一起封了，那还玩啥？所以想办法逆向了原厂固件升级的套路，做了个原厂能直刷的固件（参见另一篇文章）。拿另一个学弟的路由器测试了，能刷，然后发布。贼爽。

这三天都跟学弟一起在总部搞这些事，感觉现在应该开香槟庆祝了。

但是到了晚上就有比较坑爹的事情了，帮同学的 Gen8 加硬盘，先在 ESXi 里把存储空间扩展到两块盘上（对虚拟机而言就是一块盘了），然后在虚拟群晖里把 RAID 卷扩大到占用全部存储空间。还剩最后一步，扩大 RAID 里面的文件系统。但是这个文件系统在群晖里面是 online 的，无法卸载，于是开了个 Debian 虚拟机，使用同一个虚拟磁盘文件，然后组装 RAID 卷并扩展文件系统。一切顺利，关掉 Debian 打开群晖，发现文件系统剩余 4T 可用，很完美。于是关掉群晖，删除 Debian。点了删除，整个 ESXi 都卡了。在外面听到了硬盘起飞，感觉可能是 ESXi 哪里又 bug 了。等了几分钟居然还不停下来，我在想会不会是在删虚拟盘？但是之前新建了那么多虚拟机，机器都删了而硬盘全留下来，害得我手删删半天才删掉，所以删虚拟机不会删盘吧。又过了十几分钟，硬盘消停了，一看 ESXi 显示卷可用 7.4T，哦豁，虚拟硬盘真的被带着删掉了。学弟昨晚才把第二块盘的数据拷进去的，应该没有备份了。真·心痛。

然而班主任请吃饭，不能再留在总部。路上查了很多方法，都说不支持 VMFS6，能用的价格也贵的吓人。走到一半突然想起可以用 `testdisk` 恢复第二块盘上需要备份的数据。因为记起来把它格式化为 VMFS 的时候速度很快，应该没写多少数据，所以恢复数据的希望是很大的。赶紧告诉学弟让他试试。吃到一半的时候学弟发消息说文件都还在，勉强安心。但是前一块盘 4T 的种子还是没了，心疼下载量。

这个故事告诉我们，没有真正的虚拟化需求别去瞎掰 ESXi（而如果有这种需求，往往备份已经到位了，所以瞎掰不怕坏）

### 2017/04/05

开会成本真的很高啊，老师是不知道还是怎么地？现有工作讨论能直接搞一下午，我讲完了我的就不管后面的人了，直接码我自己的代码去了。说好的周末演示，要我们有紧迫感，怎么感觉上面那层人一点紧迫感都没有呢？这次新加的需求是评分时各个用户组所占百分比不同，以及支持 19 项选 10 项作答这种问卷类型。先记着，周末演示肯定只能做到问卷下发和结果提交，分析做不了。

### 2017/04/06

上午又是莫名其妙的讨论，我模型都写完了（或者说 JSON 结构都定了）还在说需要哪些功能，这不是早就定好了吗？于是又浪费大半个小时。

### 2017/04/07

后面这几天就是在助攻 Android 了。代码写到头晕，赶 deadline 代码质量不能看，Activity 里塞一大坨…… 还好 Django 比较稳（不过现在不限制权限，随便读写，以后肯定是不行的），后台一直都没出什么问题。

一天过去了最终还是只做到题目显示，做不到提交，丢那里了。

### 2017/04/08

加班加班！说是要周末演示，但没说时间，只能先来了。一上午把答案提交做了（开一个数组，监听每一个控件的变化（感人的 `implements` 列表），变化的时候就存进来，提交时 JSON 化）下午起床的时候说演示时间待定，于是就不去了，看串口玩。// 明明一样的初始化参数，`stty` 在官方和 LEDE 里都是一样的，device tree 里 UART1 需要的 clock-frequency 也加上了，/proc/tty 里的属性也比较过了，为毛 LEDE 上的 UART1 就是不能用？

### 2017/04/09

上午进行演示。客户点着要看从服务端下发数据到手机，害怕。不过最后看起来还算成功的。新增需求包括未授权设备不能登录、按设备设置权重。

继续看串口，官方固件拆包修改加了 strace，自己编译了个静态链接的大型 busybox，看了 uhmi 的初始化阶段的调用，并不能发现什么。不过确实可以看到在开机后的默认状态下把 GPIO 8 拉高会导致屏幕重启。

### 2017/04/10

下周出差，我也被要求去？（此处应该说 6 了，但是……）

官方固件内核好辣鸡，cmdline 上加了 `log_buf_len={31,10485760}` 两种方式都不能实现扩大 dmesg 缓冲区，硬改了 dhd.ko 把刷屏的信息都 zero 掉了，最多也只能看到 PCI 初始化的最后一部分。不过串口的三行初始化信息倒是都有了，可是相比 LEDE 的区别除了最开始的 tag 不同（LEDE 是 `18000400.0`，官方是 `serial8250.0`）后面的 MMIO 地址和 16550 类型都一毛一样，而且 UART0 两者都能用…… 我开始怀疑是不是官方内核有魔改。

此外，把固件里的 uhmi 替换成 stub，可以发现即使没有 uhmi 的 GPIO 操作，自己写的程序也一样能读串口。这说明屏幕不需要特定的 GPIO 就能操作，好气啊。当然，会不会是 uhmi 被调起来之前就已经有程序做了初始化呢？然而搜索 ttyS1 就只有 uhmi 有……

### 2017/04/11

Django 要实现在自带 admin 新建时中自动填充 ForeignKey(User) 为当前 user 的话，可以在 XXXAdmin 的 `get_form` 中这样写：

```python
    def get_form(self, request, obj=None, **kwargs):
        form = super(QuestionTemplateAdmin, self).get_form(request, obj, **kwargs)
        if obj is None:  # Default to current user when adding templates
            form.base_fields['foreignKeyUser'].initial = request.user
        return form
```

这样在新建时就会把这个字段默认设置成当前 user. 如果要更进一步，使其在新建时虽然有默认值但不可编辑，并且在查看已有对象时直接变成 readonly，可以继续加：

```python
    def get_readonly_fields(self, request, obj=None):
        readonly_fields = ('createdOn', 'reviewee_text')
        if obj is not None:  # Disable editing createdBy in existing templates
            readonly_fields += ('foreignKeyUser',)
        return readonly_fields
```

然后新建 JavaScript，使其在 window.onload 的时候把 `#id_foreignKeyUser` disable 掉就好。右边三个编辑按钮也可以 remove. 上传的时候记得 collectstatic.

如果把这个字段直接列入 readonly_fields 的话，form.fields 里面就不会有这一项，可能会造成非 NULL 字段未填写的问题。

新的需求（重要度从高到低）：

1. 为每个设备和人员指定所属权重组，但是每个组的权重不是固定的，每个项目里每个组的权重会不一样
2. 被评价人 5 选 3 评价，问题里 19 选 10 评价
3. 为每个模板增加回答说明和评价标准

### 2017/04/12

今天没下雨，但是比原来下雨的时候还要困……

上面说到“被评价人 5 选 3 评价，问题里 19 选 10 评价”这个需求，选择被评价人倒是好说，选择问题就比较尴尬了。问卷里并不是只有这一组 19 个问题，可能存在前 6 题全做，后 19 题选 10 个这种情况。按理说应该是在 Template 和 Question 中间加一层 QuestionGroup 隔开，但Django 不支持 nested inline，那就只能把 Group 单独加一个 inline 了，然后在 Question 里加一个 Group 字段来选择。

而“权重组”这个概念比较有趣，原本是有 WeighGroup 的，每个人和设备都需要选择一个（否则默认权重为 1）。我原来以为每个组的权重数值是固定的，所以数值直接写在组里。现在组没有固定的权重数值，而是要由项目决定每个组有多少权重，应该怎么表示呢？现在的方法是新建一个叫 Weigh 的模型，一个字段放组，一个字段放权重数值，一个字段放关联项目，然后 inline 到 Project 下面。有点绕，但是能用的样子。

关于那个路由器屏幕的问题，我以为是 LEDE 的内核没实现好，于是强行拿 AC88U 的预编译内核拆出来给它 tftp 启动了，发现状况与 LEDE 一样，我自己的测试程序只能收到一个 0x00 就再也没有了。而官方内核不用 uhmi 做初始化的话，用我的测试程序一样能收到结果。那么问题来了，是官方内核做了 hack，还是 userspace 有其他程序做了串口以外的初始化操作？如果是的话，是 GPIO 吗？屏幕的排线有 14 针，串口两三针差不多了，已知的 GPIO 7、8 两针，供电两针，还有一大堆针脚都不知道干嘛的…… 难道是说有额外的 GPIO？说到这里想起来这路由器没有灯，也就是说不像别的路由一样要初始化一大堆关于灯的 GPIO，那么固件里只要出现 GPIO 操作，很可能就是跟屏幕有关了（当然还可能跟 USB 电源开关和 RESET 有关——不过这两个就很少出现了）也许可以从这里入手试试。