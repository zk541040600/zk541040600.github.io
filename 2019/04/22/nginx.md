#  2019.4.22：  #
由于需求，需要将connect连接的逻辑修改为最新的逻辑，即利用nginx反响代理。为什么要这样呢？因为我们以前的那套逻辑在一次微信api更新后有的用户进不去了。

步骤我在这里全部用<font color=#9932CC size=3 face="黑体">紫色</font>表示

##  nginx是什么？  ##
一个web服务器，通过http协议提供各种网络的服务，可以作为反向代理进行负载均衡的处理
（消息通过nginx按照一定的策略转发给不同的服务器）

##  反向代理和正向代理：  ##
<font color=#0099ff size=3 face="黑体">正向代理</font>：客户端明确要访问的服务器地址，客户端必须设置正向代理服务器，服务器只知道代理服务器，隐藏客户端信息客户端信息。

<font color=#0099ff size=3 face="黑体">反向代理</font>：客户端不知道代理的存在，访问者不知道自己访问的是一个代理。不需要任何配置即可访问。反向代理隐藏了服务器的信息。

## 在部署nginx之前： ##

我们首先需要有一个方向，我是谁，我在哪里，我该干什么，目的是什么？（<font color=#DC143C size=3 face="黑体">优先想一下结果</font>）

<font color=#DC143C size=3 face="黑体">我是谁</font>：我是一个客户端，一个用户

<font color=#DC143C size=3 face="黑体">我在哪</font>：服务器不需要知道

<font color=#DC143C size=3 face="黑体">目的是什么</font>：发送包给反向代理的服务器，与服务器交互

<font color=#DC143C size=3 face="黑体">我该干什么</font>：连接一个域名上的端口

搞清楚这些，我们应该有点逼数了，我连接一个域名上的某个端口，这个域名在用户看来是服务器的域名，但实际上代理的服务器域名，所以是代理转发给服务器。那么代理转发，我们以nginx服务器为自己。

<font color=#DC143C size=3 face="黑体">用户转发到我的哪个端口？我应该发送给谁？端口是多少？</font>

这些数据，我一个nginx怎么知道，当然是**读配置**，那么我们接下来就是主要工作就是在设置配置上。

部署nginx配置：

<font size="5" face="黑体">1.部署的第一件事，<font color=#9932CC size=5 face="黑体">**下载文件包**</font>来安装：</font><br />

    wget http://mirrors.linuxeye.com/oneinstack-full.tar.gz

这个是应该是panda他们那边的一个备份

<font size="5" face="黑体">2.安装nginx</font><br />

该文件是一个tar.gz文件，直接解压（tar -zcvf ...），运行里面的<font color=#9932CC size=3 face="黑体">**install.sh脚本**</font>即可安装，一步一步照着做就可以了。

<font size="5" face="黑体">3.然后就是配置文件了</font><br />

这里首先要配置多个虚拟主机来形成port的重定向（就是你把a重定向到b，之后你访问a端口就是访问b端口），需要干什么呢？


当然是配置conf，我们很容易能够在nginx的文件夹中找到conf文件夹，我们打开<font color=#9932CC size=3 face="黑体">**nginx.conf**</font>查看，在nginx.conf中的最后，我们能找到如下配置：

	######################## default ############################
	  server {
	  listen 80;
	  server_name _;
	  access_log off;
	  index index.html index.htm index.php;
	  #include /usr/local/nginx/conf/rewrite/thinkphp.conf;
	
	  set $root /arthur/html;
	  root $root;
	
	  #error_page 404 = /404.html;
	  #error_page 502 = /502.html;
	
	  location @rewrite {
	    rewrite ^(.*)$ index.php/$1 last;
	  }
	
	  location /{
	      if ( !-e $request_filename) {
	          rewrite ^(.*)$ /index.php/$1 last;
	          break;
	      }
	  }
	
	  location ~ \.php {
	      root    $root;
	      fastcgi_pass unix:/dev/shm/php-cgi.sock;
	      fastcgi_index  index.php;
	      fastcgi_split_path_info  ^(.+\.php)(.*)$;
	      fastcgi_param PATH_INFO $fastcgi_path_info;
	      fastcgi_param SCRIPT_FILENAME $root$fastcgi_script_name;
	      include        fastcgi_params;
	  }
	
	  location ~* /runtime/ {
	     deny all;
	  }
	
	  }
	
	########################## vhost #############################
	  include vhost/*.conf;
	}


上面是一个默认的配置，我们之后的代理设置是在这个默认的配置上改，下面是一个incude，也就是说这个配置包含了vhost文件夹下面的任意配置（这里就是专门来存放我们自己的反响代理配置）

**我们先看######################## default ############################，分析一下几行重要的配置**

	listen 80

这里很明显的<font color=#DC143C size=3 face="黑体">80端口</font>，有没有觉得很熟？它是http的默认端口，这里我猜了一下它的用途，应该是用来给其他用户访问用的，而这个是<font color=#DC143C size=3 face="黑体">http协议</font>，不是https，所以不需要证书，任意用户都可以访问。

	server_name _;

server_name配置域名，不多解释

	set $root /arthur/html

然后看set $root /arthur/html这行配置，配置了其他用户访问后，在我们服务器中到达的路径，当然你可以改成想要的，例如zhuke/html，应该是没问题的吧，大概。

**再来看########################## vhost #############################**

很简单，就是一个单纯的include，配置不止这些，还包括vhost下的配置

ok，我们就按照上面的来，<font color=#9932CC size=3 face="黑体">**在conf里面新建vhost文件夹，再新建一个****.conf文件**</font>（名字开心就好）。

<font size="5" face="黑体">4.在文件中的copy内容如下：</font><br />

	server {
	  listen 80;
	  server_name h5jrmjgame1.arthur-tech.com;
	  access_log off;
	  index index.html index.htm index.php;
	  #include /usr/local/nginx/conf/rewrite/thinkphp.conf;
	
	  set $root /arthur/html;
	  root $root;
	
	  #error_page 404 = /404.html;
	  #error_page 502 = /502.html;
	
	  location @rewrite {
	    rewrite ^(.*)$ index.php/$1 last;
	  }
	
	  location /{
	      if ( !-e $request_filename) {
	          rewrite ^(.*)$ /index.php/$1 last;
	          break;
	      }
	  }
	
	  location ~ \.php {
	      root    $root;
	      fastcgi_pass unix:/dev/shm/php-cgi.sock;
	      fastcgi_index  index.php;
	      fastcgi_split_path_info  ^(.+\.php)(.*)$;
	      fastcgi_param PATH_INFO $fastcgi_path_info;
	      fastcgi_param SCRIPT_FILENAME $root$fastcgi_script_name;
	      include        fastcgi_params;
	  }
	
	   location ~* /runtime/ {
	         deny all;
	    }
	}


好了，80端口配置完毕。我们是不是应该配置转发配置了呢？
不是，我们发现我们是https连接，这是<font color=#DC143C size=3 face="黑体">加密的</font>，服务器和客户端都要相互预定解密口令。<font color=#DC143C size=3 face="黑体">这个口令是密钥</font>，而密钥在一个叫证书的里面，我们没有证书啊，在这之前我们需要先申请一个证书，不然配置的时候，证书的配置没法填。

<font size="5" face="黑体">5.ok，o(*￣▽￣*)ブ申请证书吧</font><br />
科普一下，
证书：有权威部门颁发，证明我是我自己，相当与身份证吧。

**加密方式：**

<font color=#0099ff size=3 face="黑体">1.对称加密：</font>
双方密钥是一样的，快，但是这个口令怎么给呢？给不了，怎么给都需要加密，死循环了。

<font color=#0099ff size=3 face="黑体">2.非对称加密：</font>
双方密钥是不一样的，自己这里有<font color=#DC143C size=3 face="黑体">一个私钥用来加密，对面用共钥进行解密（这仅仅是一对，反过来还有另外一对）</font>。效率 肯定是没上面的对称加密快的，但是我们可以通过非对称加密传输，把对称加密的密钥传过去。再利用对称加密的密钥瞎倒腾。

所以，证书有2个，一个含有上面自己的<font color=#DC143C size=3 face="黑体">私钥</font>，一个含有自己给别人的<font color=#DC143C size=3 face="黑体">共钥</font>（还有一个上级权威部门的公钥（CA（一直到rootCA）），因为要取上级权威部门验证公钥的真实性）。

客户端会验证，并且访问域名来获得含有公钥的那个证书，服务器会把包含公钥的证书和计算随机数给客户端，客户端拿到后验证，会传计算它的随机数给服务器，2方都会计算出相同的密钥，（非对称传输就到这里），之后客户端和服务器就会通过这个密钥来进行通信（对称传输，为什么要对称传输呢？因为这样省事啊！）。

不扯多了，回到证书申请，首先需要<font color=#9932CC size=3 face="黑体">**买一个域名**</font>（买爆！！），然后申请证书，但是阿里云需要验证是这个域名来申请证书，怎么办呢？这时候就需**要使用第4步中配置的80端口**来可以让外界连接到这里（里面配置已经好了），那边给我们一个验证文件，我门就需要<font color=#9932CC size=3 face="黑体">**把那个验证文件放在配置的root /arthur/html，把对应的校验文件放在对应的位置（.well-known/pki-validation）**</font>，root路径在上面代码中的/arthur/html：

	set $root /arthur/html;
	root $root;

至于.well-known/pki-validation这个路径，阿里云要求的，如图：

完成上面的步骤后，访问一下这个路径（<font color=#DC143C size=3 face="黑体">/arthur/html/.well-known/pki-validation</font>），能够看到看到你放的校验文件就证明可以了，给阿里云验证吧。

<font size="5" face="黑体">6.证书申请完毕后，我门有证书了，开始重定向我们的端口吧，conf一下的内容在我们创建的conf中，copy吧：</font><br />

	server {
	  listen 35002 ssl;# 2710
	  server_name h5jrmjgame1.arthur-tech.com;
	  access_log off;
	  index index.html index.htm index.php;
	
	  ssl_certificate   cert/h5jrmjgame1.arthur-tech.com.pem;
	    ssl_certificate_key  cert/h5jrmjgame1.arthur-tech.com.key;
	    ssl_session_timeout 5m;
	    ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:ECDHE:ECDH:AES:HIGH:!NULL:!aNULL:!MD5:!ADH:!RC4;
	    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
	    ssl_prefer_server_ciphers on;
	
	  set $root /arthur/html;
	  root $root;
	
	  location /{
	          proxy_pass http://127.0.0.1:35012;
	    proxy_http_version 1.1;
	    proxy_set_header Upgrade $http_upgrade;
	    proxy_set_header Connection "Upgrade";
	    proxy_set_header X-Real-IP $remote_addr;
	
	      if ( !-e $request_filename) {
	          rewrite ^(.*)$ /index.php/$1 last;
	          break;
	      }
	  }
	}


**几个重要的配置：**

<font color=#9932CC size=3 face="黑体">**1.监听的端口**</font>

<font color=#9932CC size=3 face="黑体">**2.转发的端口**</font>

<font color=#9932CC size=3 face="黑体">**3.域名**</font>

<font color=#9932CC size=3 face="黑体">**4.证书**</font>


<font size="5" face="黑体">7.配置文件</font><br />
配置域名server_name，2个ssl的证书栏，但是证书栏应该怎么配呢？其实就是2个证书的连接的地址，怎么配就不多讲了，(ln -s建立软连接或者绝对路径，一般在cert里面建立软连接，随便你们)。嗖，嗖，嗖，别忘记了nginx需要监听的端口，和转发的端口（ip随意）。
软连接建立如下图：

命令：<font color=#9932CC size=3 face="黑体">**n -s /ar..... h5jrmj...**</font>

<font size="5" face="黑体">8.配置完成，重启<font color=#9932CC size=5 face="黑体">**nginx。service nginx restart**</font>  !!!!!（别忘了，游戏服还需要配置证书哦</font><br />

<font size="5" face="黑体">9.启动游戏服的链接服的进程。</font><br />

<font size="5" face="黑体">10.<font color=#9932CC size=5 face="黑体">**web->f12->new WssSocket("域名",port)**</font>，验证一下是否配置完了。成功！</font><br />

<font size="5" face="黑体">11.带耳机，听歌去(ง •_•)ง</font><br />

其实我挺想这样写的，但实际上只能去做傻吊需求。( ´･･)ﾉ(._.`)

完毕！
