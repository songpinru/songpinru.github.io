# 安装

1） yum安装依赖包
```bash
sudo yum -y install    openssl openssl-devel pcre pcre-devel    zlib zlib-devel gcc gcc-c++
```
2） 安装依赖包
解压缩nginx-xx.tar.gz
进入解压缩目录，执行
```bash
./configure --prefix=/opt/module/nginx
make && make install
```
3） 启动、关闭命令nginx（root用户）
```bash
#启动命令:
#在/opt/module/nginx/sbin目录下执行  
./nginx
#关闭命令:
#在/opt/module/nginx/sbin目录下执行  
./nginx  -s  stop 
#重新加载命令:
#在/opt/module/nginx/sbin目录下执行  
./nginx  -s reload
#注意：如果启动时报错，则执行
ln -s /usr/local/lib/libpcre.so.1 /lib64
```

4）修改配置文件

/opt/module/nginx/conf/nginx.conf

# Nginx 服务的基本配置

Nginx在运行时候，至少要加载几个核心模块和一个事件类模块。这些模块运行时所支持的配置项称为基本配置——所有其他模块执行时都依赖的配置项。

由于配置项较多，所以把它们按照用户使用时的预期功能分成以下4类：

* 用于调试、定位问题的配置项；
* 正常运行的必备配置项；
* 优化性能的配置项；
* 事件类配置项（有些事件类配置项归纳到优化性能类，这是因为它们虽然也属于event{}块，但作用是优化性能）

有一些配置项，几十没有显式的进行配置，他们会有默认的值，如：daemon，即是在nginx.conf中没有对它进行配置，也相当于打开了这个功能，这点需要注意。

```yml
##代码块中的events、http、server、location、upstream等都是块配置项##
##块配置项可以嵌套。内层块直接继承外层快，例如：server块里的任意配置都是基于http块里的已有配置的##
 
##Nginx worker进程运行的用户及用户组 
#语法：user username[groupname]    默认：user nobody nobody
#user用于设置master进程启动后，fork出的worker进程运行在那个用户和用户组下。当按照"user username;"设置时，用户组名与用户名相同。
#若用户在configure命令执行时，使用了参数--user=usergroup 和 --group=groupname,此时nginx.conf将使用参数中指定的用户和用户组。
#user  nobody;
 
##Nginx worker进程个数：其数量直接影响性能。
#每个worker进程都是单线程的进程，他们会调用各个模块以实现多种多样的功能。如果这些模块不会出现阻塞式的调用，那么，有多少CPU内核就应该配置多少个进程，反之，有可能出现阻塞式调用，那么，需要配置稍多一些的worker进程。
worker_processes  1;
 
##ssl硬件加速。
#用户可以用OpneSSL提供的命令来查看是否有ssl硬件加速设备：openssl engine -t
#ssl_engine device;
 
##守护进程(daemon)。是脱离终端在后台允许的进程。它脱离终端是为了避免进程执行过程中的信息在任何终端上显示。这样一来，进程也不会被任何终端所产生的信息所打断。##
##关闭守护进程的模式，之所以提供这种模式，是为了放便跟踪调试nginx，毕竟用gdb调试进程时最繁琐的就是如何继续跟进fork出的子进程了。##
##如果用off关闭了master_proccess方式，就不会fork出worker子进程来处理请求，而是用master进程自身来处理请求
#daemon off;   #查看是否以守护进程的方式运行Nginx 默认是on 
#master_process off; #是否以master/worker方式工作 默认是on
 
##error日志的设置#
#语法： error_log /path/file level;
#默认： error_log / log/error.log error;
#当path/file 的值为 /dev/null时，这样就不会输出任何日志了，这也是关闭error日志的唯一手段；
#leve的取值范围是debug、info、notice、warn、error、crit、alert、emerg从左至右级别依次增大。
#当level的级别为error时，error、crit、alert、emerg级别的日志就都会输出。大于等于该级别会输出，小于该级别的不会输出。
#如果设定的日志级别是debug，则会输出所有的日志，这一数据量会很大，需要预先确保/path/file所在的磁盘有足够的磁盘空间。级别设定到debug，必须在configure时加入 --with-debug配置项。
#error_log  logs/error.log;
#error_log  logs/error.log  notice;
#error_log  logs/error.log  info;
 
##pid文件（master进程ID的pid文件存放路径）的路径
#pid        logs/nginx.pid;
 
 
events {
 #仅对指定的客户端输出debug级别的日志： 语法：debug_connection[IP|CIDR]
 #这个设置项实际上属于事件类配置，因此必须放在events{……}中才会生效。它的值可以是IP地址或者是CIRD地址。
 	#debug_connection 10.224.66.14;  #或是debug_connection 10.224.57.0/24
 #这样，仅仅以上IP地址的请求才会输出debug级别的日志，其他请求仍然沿用error_log中配置的日志级别。
 #注意：在使用debug_connection前，需确保在执行configure时已经加入了--with-debug参数，否则不会生效。
	worker_connections  1024;
}
 
##核心转储(coredump):在Linux系统中，当进程发生错误或收到信号而终止时，系统会将进程执行时的内存内容(核心映像)写入一个文件(core文件)，以作为调试只用，这就是所谓的核心转储(coredump).
 
http {
##嵌入其他配置文件 语法：include /path/file
#参数既可以是绝对路径也可以是相对路径（相对于Nginx的配置目录，即nginx.conf所在的目录）
    include       mime.types;
    default_type  application/octet-stream;
 
    #log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
    #                  '$status $body_bytes_sent "$http_referer" '
    #                  '"$http_user_agent" "$http_x_forwarded_for"';
 
    #access_log  logs/access.log  main;
 
    sendfile        on;
    #tcp_nopush     on;
 
    #keepalive_timeout  0;
    keepalive_timeout  65;
 
    #gzip  on;
 
    server {
##listen监听的端口
#语法：listen address:port [ default(deprecated in 0.8.21) | default_server | [ backlog=num | rcvbuf=size | sndbuf=size | accept_filter=filter | deferred | bind | ssl ] ]
#default_server: 如果没有设置这个参数，那么将会以在nginx.conf中找到的第一个server块作为默认server块
	listen       8080;
 
#主机名称：其后可以跟多个主机名称，开始处理一个HTTP请求时，nginx会取出header头中的Host，与每个server中的server_name进行匹配，以此决定到底由那一个server来处理这个请求。有可能一个Host与多个server块中的server_name都匹配，这时会根据匹配优先级来选择实际处理的server块。server_name与Host的匹配优先级见文末。
	 server_name  localhost;
 
        #charset koi8-r;
 
        #access_log  logs/host.access.log  main;
 
        #location / {
        #    root   html;
        #    index  index.html index.htm;
        #}
 
##location 语法： location [=|~|~*|^~] /uri/ { ... }
# location的使用实例见文末。
#注意：location时有顺序的，当一个请求有可能匹配多个location时，实际上这个请求会被第一个location处理。
	location / {
	proxy_pass http://192.168.1.60;
        }
 
        #error_page  404              /404.html;
 
        # redirect server error pages to the static page /50x.html
        #
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }
 
        # proxy the PHP scripts to Apache listening on 127.0.0.1:80
        #
        #location ~ \.php$ {
        #    proxy_pass   http://127.0.0.1;
        #}
 
        # pass the PHP scripts to FastCGI server listening on 127.0.0.1:9000
        #
        #location ~ \.php$ {
        #    root           html;
        #    fastcgi_pass   127.0.0.1:9000;
        #    fastcgi_index  index.php;
        #    fastcgi_param  SCRIPT_FILENAME  /scripts$fastcgi_script_name;
        #    include        fastcgi_params;
        #}
 
        # deny access to .htaccess files, if Apache's document root
        # concurs with nginx's one
        #
        #location ~ /\.ht {
        #    deny  all;
        #}
    }
 
 
 
    # another virtual host using mix of IP-, name-, and port-based configuration
    #
    #server {
    #    listen       8000;
    #    listen       somename:8080;
    #    server_name  somename  alias  another.alias;
 
    #    location / {
    #        root   html;
    #        index  index.html index.htm;
    #    }
    #}
 
 
    # HTTPS server
    #
    #server {
    #    listen       443 ssl;
    #    server_name  localhost;
 
    #    ssl_certificate      cert.pem;
    #    ssl_certificate_key  cert.key;
 
    #    ssl_session_cache    shared:SSL:1m;
    #    ssl_session_timeout  5m;
 
    #    ssl_ciphers  HIGH:!aNULL:!MD5;
    #    ssl_prefer_server_ciphers  on;
 
    #    location / {
    #        root   html;
    #        index  index.html index.htm;
    #    }
    #}
 
}
```

**server_name与Host的匹配优先级**  
| server_name                               | Host                   |
| ----------------------------------------- | ---------------------- |
| 先选择所有字符串完全匹配的server_name     | 如：www.testwab.com    |
| 其次选择通配符在前面的server_name         | 如：\*.testwab.com     |
| 其次选择通配符在后面的server_name         | 如：www.testwab.\*     |
| 最后选择使用正在表达式才匹配的server_name | 如：\~^\.testwab\.com$ |

```
最基本的区别
alias 指定的目录是准确的，给location指定一个目录。
root 指定目录的上级目录，并且该上级目录要含有locatoin指定名称的同名目录。
以root方式设置资源路径：

语法: root path;
配置块: http、server、location、if
以alias 方式设置资源路径

语法: alias path;
配置块: location
Example:

location /img/ {
	alias /var/www/image/;
}
#若按照上述配置的话，则访问/img/目录里面的文件时，ningx会自动去/var/www/image/目录找文件
location /img/ {
	root /var/www/image;
}
#若按照这种配置的话，则访问/img/目录下的文件时，nginx会去/var/www/image/img/目录下找文件

注意： 

1.使用alias时，目录名后面一定要加”/“。
2.使用alias标签的目录块中不能使用rewrite的break。
3.alias在使用正则匹配时，必须捕捉要匹配的内容并在指定的内容处使用。
4.alias只能位于location块中
```



### location的使用实例 —— 以root方式设置资源路径

```
location /download/ {

root    /opt/wab/html/;

}          [[[意思是有一个请求的URL是 /download/index/test.html， 那么Web服务器就会返回服务器上 /opt/wab/html/download/index/test.html 文件的内容]]]
```
### location的使用实例 —— 以alias方式设置资源路径
alias也是用来设置文件资源路径的，它与root不同点主要在于如何解读紧跟location后面的uri参数，这将会致使alias与root以不同的方式将用户请求映射到真正的磁盘文件上。

例如：如果有一个请求的URI是/conf/nginx.conf，而用户实际想访问的是 /usr/local/nginx/conf/nginx.conf，则两种方式如下：
```
alias：
	location   /conf {
         alias    /usr/local/nginx/conf
        }
root:
	location    /conf{
    	root    /usr/local/nginx
      }
```
使用alias时，在URI向实际文件路径的映射过程中，已经把location后配置的 /conf这部分字符串丢弃掉了，因此若path中不加/conf这部分，直接映射回的地址是/usr/local/nginx/nginx.conf 与用户实际想访问的路径不符。root可以放置在http、server、location或if块中，而alias只能放置在location块中。

### location的使用实例 —— 以index方式访问首页
有时，访问站点时的URI是/ ，这时返回网站的首页，而这与root和alias都不同。这里用ngx_http_index_module模块提供的index配置实现。index后可以跟多个文件参数，Nginx将会按照顺序来访问这些文件。

```
location  /　｛
　　　root    path;
          index   /index.html　　　/html/index.php          /index.php
｝
```
接受到请求后，Nginx首先会尝试访问path/index.php 文件，如果可以访问，就直接返回文件内容结束请求，否则再试图返回path/html/index.php 文件的内容，以此类推。（从后向前）