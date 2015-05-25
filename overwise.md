# 什么是 Nginx

## Nginx

Nginx是俄罗斯人编写的十分轻量级的HTTP服务器,Nginx，它的发音为“engine X”，是一个高性能的HTTP和反向代理服务器，同时也是一个IMAP/POP3/SMTP 代理服务器。Nginx是由俄罗斯人 Igor Sysoev为俄罗斯访问量第二的 Rambler.ru站点开发的，它已经在该站点运行超过两年半了。Igor Sysoev在建立的项目时,使用基于BSD许可。

英文主页：[http://nginx.net](http://nginx.net) 。

到2013年，目前有很多国内网站采用Nginx作为Web服务器，如国内知名的新浪、163、腾讯、Discuz、豆瓣等。据netcraft统计，Nginx排名第3，约占15%的份额(参见：http://news.netcraft.com/archives/category/web-server-survey/ )

Nginx以事件驱动的方式编写，所以有非常好的性能，同时也是一个非常高效的反向代理、负载平衡。其拥有匹配Lighttpd的性能，同时还没有Lighttpd的内存泄漏问题，而且Lighttpd的mod_proxy也有一些问题并且很久没有更新。

现在，Igor将源代码以类BSD许可证的形式发布。Nginx因为它的稳定性、丰富的模块库、灵活的配置和低系统资源的消耗而闻名．业界一致认为它是Apache2.2＋mod_proxy_balancer的轻量级代替者，不仅是因为响应静态页面的速度非常快，而且它的模块数量达到Apache的近2/3。对proxy和rewrite模块的支持很彻底，还支持mod_fcgi、ssl、vhosts ，适合用来做mongrel clusters的前端HTTP响应。

## Nginx 特点

Nginx 做为HTTP服务器，有以下几项基本特性：

- 处理静态文件，索引文件以及自动索引；打开文件描述符缓冲．

- 无缓存的反向代理加速，简单的负载均衡和容错．

- FastCGI，简单的负载均衡和容错．

- 模块化的结构。包括gzipping, byte ranges, chunked responses,以及 SSI-filter等filter。如果由FastCGI或其它代理服务器处理单页中存在的多个SSI，则这项处理可以并行运行，而不需要相互等待。

- 支持SSL 和 TLSSNI．

Nginx专为性能优化而开发，性能是其最重要的考量,实现上非常注重效率 。它支持内核Poll模型，能经受高负载的考验,有报告表明能支持高达 50,000个并发连接数。

Nginx具有很高的稳定性。其它HTTP服务器，当遇到访问的峰值，或者有人恶意发起慢速连接时，也很可能会导致服务器物理内存耗尽频繁交换，失去响应，只能重启服务器。例如当前apache一旦上到200个以上进程，web响应速度就明显非常缓慢了。而Nginx采取了分阶段资源分配技术，使得它的CPU与内存占用率非常低。nginx官方表示保持10,000个没有活动的连接，它只占2.5M内存，所以类似DOS这样的攻击对nginx来说基本上是毫无用处的。就稳定性而言,nginx比lighthttpd更胜一筹。

Nginx支持热部署。它的启动特别容易, 并且几乎可以做到7*24不间断运行，即使运行数个月也不需要重新启动。你还能够在不间断服务的情况下，对软件版本进行进行升级。

Nginx采用master-slave模型,能够充分利用SMP的优势，且能够减少工作进程在磁盘I/O的阻塞延迟。当采用select()/poll()调用时，还可以限制每个进程的连接数。

Nginx代码质量非常高，代码很规范，手法成熟， 模块扩展也很容易。特别值得一提的是强大的Upstream与Filter链。Upstream为诸如reverse proxy,与其他服务器通信模块的编写奠定了很好的基础。而Filter链最酷的部分就是各个filter不必等待前一个filter执行完毕。它可以把前一个filter的输出做为当前filter的输入，这有点像Unix的管线。这意味着，一个模块可以开始压缩从后端服务器发送过来的请求，且可以在模块接收完后端服务器的整个请求之前把压缩流转向客户端。

Nginx采用了一些os提供的最新特性如对sendfile (Linux2.2+)，accept-filter (FreeBSD4.1+)，TCP_DEFER_ACCEPT (Linux 2.4+)的支持，从而大大提高了性能。

当然，Nginx 还很年轻，多多少少存在一些问题，比如：Nginx是俄罗斯人创建，虽然前几年文档比较少，但是目前文档方面比较全面，英文资料居多，中文的资料也比较多，而且有专门的书籍和资料可供查找。

Nginx 的作者和社区都在不断的努力完善，我们有理由相信Nginx将继续以高速的增长率来分享轻量级HTTP服务器市场，会有一个更美好的未来。