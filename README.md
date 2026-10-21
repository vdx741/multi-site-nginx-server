# nginx部署多个网站：一台服务器配多个 server block，静态站和反向代理站各管各的

一台服务器上放多个网站，靠的是 Nginx 的 server block。每个站点单独写一个配置文件，用 `server_name` 声明它负责哪个域名，用 `root` 或 `proxy_pass` 决定请求最终去哪。请求进来时，Nginx 读 HTTP 头里的 Host 字段，跟各个 server block 的 server_name 逐个比对，匹配上就交给对应的站点处理。一台机器跑十个八个站在运维里属于常规操作。

要准备的东西只有三样：域名解析、每个站独立的目录、每个站一个 conf 文件。

> 服务器还没买的话，如果只是放几个静态站和小应用，腾讯云轻量应用服务器够用，套餐把 CPU、内存、带宽和月流量打包在一起，不用像 CVM 那样把实例和带宽分开算。机型档位和当前活动价看这里：[👉 查看腾讯云轻量应用服务器当前活动价](https://bit.ly/yunhuodong)

## 多个网站是靠什么区分开的

靠域名。Nginx 不看目录、不看端口，只看请求头里的 Host 跟你写的 `server_name` 是否匹配。匹配不到的请求会落到默认站点，所以真正决定"这台机器上有几个网站"的，是你写了几个 server block。

Debian / Ubuntu 系的目录组织方式是这样的：


/etc/nginx/
├── nginx.conf
├── sites-available/          # 每个站一个文件，写在这里
│   ├── example.com.conf
│   └── blog.example.com.conf
└── sites-enabled/            # 指向上面的软链接，只有这里的会被加载


CentOS 和部分云镜像用 `/etc/nginx/conf.d/*.conf`，效果一样，只是少了 available 和 enabled 两层。分两层的好处是下线一个站只需要删掉软链接，配置文件还留着，以后要恢复直接再链一次。

站点目录同样分开：


/var/www/example.com/
/var/www/blog.example.com/


别让两个站共用一个 root。共用的后果是改 A 站的文件，B 站跟着变，排查起来很费时间。

## 从加解析到页面能打开，六步

第一步，加 A 记录。每个要用的域名（包括 www）都指向服务器公网 IP。子域多的话用 `*.example.com` 泛解析，省得以后每加一个站都回来改 DNS。

第二步，建目录，放一个 index.html 当测试页。

第三步，写配置文件：

nginx
server {
    listen 80;
    listen [::]:80;
    server_name example.com www.example.com;

    root /var/www/example.com;
    index index.html;

    access_log /var/log/nginx/example.com.access.log;
    error_log  /var/log/nginx/example.com.error.log;

    location / {
        try_files $uri $uri/ =404;
    }
}


第四步，链到 sites-enabled：

bash
sudo ln -s /etc/nginx/sites-available/example.com.conf /etc/nginx/sites-enabled/


第五步，`sudo nginx -t` 检查语法。第六步，`sudo systemctl reload nginx`。reload 是平滑重载，不会断开已有连接；restart 会断，多站点环境下没必要用。

第二个站照抄这份配置，改 `server_name` 和 `root` 两个地方就行。

## 静态站、前端 SPA、后端应用，配置写法不一样

三种站点的差别在 location 里。静态站用 `root` 加 `try_files` 找文件，SPA 要把找不到的路径回落到 index.html，后端应用用 `proxy_pass` 转给本机端口。

前端用了 history 路由的（React Router、Vue Router 的默认模式），最后一项必须回落：

nginx
location / {
    root /var/www/app.example.com;
    try_files $uri $uri/ /index.html;
}


少了这个回落，用户刷新子页面就是 404。

反向代理站：

nginx
server {
    listen 80;
    server_name api.example.com;

    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}


最常见的其实是混搭：一个域名下前端静态文件，`/api/` 转发给后端。这里有个细节容易栽跟头——`proxy_pass http://127.0.0.1:8080/;` 末尾带斜杠会把 `/api/` 前缀剥掉再传给后端，不带斜杠则原样传过去。后端路由是按 `/api/xxx` 写的还是按 `/xxx` 写的，决定你该用哪种。

## 多站点最容易翻车的几个地方

**两个配置文件都写了 `default_server`，Nginx 直接起不来。**报错是 `nginx: [emerg] a duplicate default server for 0.0.0.0:80`。同一个端口只能有一个默认站点，想指定的话只在一个 conf 里写 `listen 80 default_server;`，其他全部写 `listen 80;`。

**没配 default_server 时，第一个被加载的 server block 会兜底。**这意味着别人拿 IP 直接访问你的服务器，看到的是按文件名排序最先加载的那个站。如果机器上有测试环境或者内部后台，最好单独写一个默认站点，直接返回 444 掐掉连接：

nginx
server {
    listen 80 default_server;
    server_name _;
    return 444;
}


**403 Forbidden 基本都是目录权限问题。**Nginx 的 worker 进程以 `www-data`（Debian 系）或 `nginx` 用户跑，读不了 `/root/` 下面的东西。站点放 `/var/www`，并确保从根到站点目录的每一级都有执行权限。

**HTTPS 不需要给每个站配独立 IP。**老教程里"SSL 要独立 IP"是 SNI 出现之前的事，现在 Nginx 靠 SNI 按域名选证书，几十个站共用一个 443 端口没问题。顺带提一句，`listen 443 ssl http2;` 这种写法在 Nginx 1.25.1 之后改成了 `listen 443 ssl;` 加单独一行 `http2 on;`，写老写法会有警告。

**证书是按域名签的，不是按服务器签的。**`certbot --nginx -d example.com -d www.example.com` 一次可以带多个 `-d`，但不同域名对应不同证书。站点多了之后，`certbot renew --dry-run` 值得先跑一遍确认续签链路通。certbot 会自己往你的 conf 里插 443 段和 HTTP 跳转，改完回头看一眼它写了什么，别让它在多站点配置里插错位置。

排查的时候 `sudo nginx -T` 比逐个翻文件快得多，它会把 include 之后的完整配置全部打印出来。

## 一台机器放几个站，资源怎么估

静态站的资源开销基本可以忽略，真正卡住你的是带宽和每个站背后的应用进程。

Nginx 自身空闲时内存几十 MB 级别，一个 worker 进程能扛几千并发连接，静态文件走 sendfile，CPU 压力很小。纯 HTML/CSS 的站，几百 MB 磁盘就够，内存几乎不用算。

带应用的站就不一样了。Node、Python、PHP-FPM 每个进程按几百 MB 内存估，数据库另算。一台 1G 内存的机器上如果还要跑 MySQL，两三个应用站就能把它填满。

带宽是所有站共享的。这一点在选机器时比 CPU 更容易被忽略：

> 轻量应用服务器的月流量包是整台实例合并计算的，不是每个网站单独一份。挂三个站，流量算的是三个站加起来的出网总量。几个文字博客没什么感觉，一旦其中某个站开始跑图片或文件下载，月底就可能撞上套餐上限。超出后的处理方式（限速还是额外计费）以购买页说明为准。

所以资源规划的顺序应该是：先看哪个站最耗流量，再看有几个站带应用进程要占内存，最后才看 CPU。

## 轻量应用服务器和 CVM 该选哪个

如果这台机器上的站点都是静态站加少量小应用，轻量应用服务器更省事：套餐把 CPU、内存、系统盘、带宽和月流量打包成一个价格，买的时候不用在"实例多少钱、带宽多少钱"之间来回算。需要自定义 CPU 和内存配比、要接负载均衡或者跟其他云资源走内网的时候，CVM 更合适。

| 产品 | 适合的场景 | 计费方式 | 价格 | 开通 |
| --- | --- | --- | --- | --- |
| 轻量应用服务器 | 几个到十几个静态站、小型应用站，配置固定够用 | 套餐打包 CPU / 内存 / 系统盘 / 带宽 / 月流量，按月或按年 | 随配置档位和购买时长变化，以活动页为准 | [ 查看轻量服务器机型与当前价格](https://bit.ly/yunhuodong) |
| 云服务器 CVM | 站点多、需要自定义 CPU 内存配比、要接负载均衡或内网 | 实例规格与公网带宽 / 流量分别计费 | 同上 | [ 查看 CVM 机型与当前价格](https://bit.ly/yunhuodong) |

具体机型档位、可选购买时长和实时活动价以页面展示为准。这里不写死具体数字，是因为配置组合和促销一直在变，写下来反而容易误导——同一条产品线，选不同的 CPU 内存和时长，总价能差好几倍。

一个实操建议：多个站放在同一台机器上，先按"最坏情况"估算，也就是假设所有站的流量同时起来。流量不够是可以后面单独加带宽或者上 CDN 的，内存不够就得升配或者拆机器，后者麻烦得多。

## 几个实际会被问到的问题

**一台服务器最多能部署多少个网站？**
没有数量上限，server block 想写多少个写多少个，限制全部来自资源。纯静态站几十个也能跑；每个带应用进程的站按内存累加；带宽是所有站共享的。

**多个网站需要买多张 SSL 证书吗？**
不需要买。Let's Encrypt 免费，每个域名签一张，90 天有效，certbot 自动续期。子域很多的话一张通配符证书能覆盖 `*.example.com`，但签发要走 DNS-01 验证，需要 DNS 服务商的 API 权限。

**能不能用同一个 IP 的不同端口访问不同网站？**
可以，`listen` 后面写不同端口就行，但访问时得带端口号，实际很少这么用。正规做法还是每个站一个域名走 80 和 443。

**网站和配置怎么迁到新服务器？**
打包 `/var/www` 和 `/etc/nginx` 两个目录，新机器装同版本 Nginx，解包后 `nginx -t` 检查，再改 DNS 解析。证书目录 `/etc/letsencrypt` 一起带走，或者在新机器上重新签一遍，后者更干净。

**在浏览器输入 IP，为什么看到的是别人的站？**
没配 default_server，Nginx 用第一个加载的 server block 兜底了。按前面那节加一个返回 444 的默认站点就能解决。

## 下单前的最后几件事

站点数量在两三个、以静态内容为主，选轻量应用服务器的入门档就够，多出来的预算留给域名和备份。站点里有跑数据库或者多个应用进程的，内存往 2G 以上看，别在 1G 的机器上硬塞。

买之前有两点值得确认：套餐包含的月流量是否够你所有站加起来用，以及超出后的计费方式；另外系统镜像选跟你熟悉程度匹配的，Ubuntu 和 Debian 的 Nginx 目录结构一致，CentOS 系是 conf.d，照着本文的路径换成对应目录就行。

配置写完后先 `nginx -t` 再 reload，这个习惯能省掉大部分"改完一个站，所有站全挂"的场面。[👉 按你的站点规模挑一台合适的腾讯云服务器](https://bit.ly/yunhuodong)
