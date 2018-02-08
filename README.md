# 背景
> 大多数的系统都会涉及缩略图的处理，比如新闻系统和电商系统，特别是电商系统，每个商品大图都会对应一系列尺寸的缩略图用于不同业务场景的使用。部分系统也会生成不同尺寸的缩略图以供PC、手机端、ipad端使用。

**解决方案探索：** 
1. 直接加载原图，使用css样式表来控制图片的宽高。显然不太合适，大家也尽量不要这样做。
2. web程序在上传成功后，同时生成相应缩略图。这种做法效率较低，如果遇到批量导入的业务时严重影响性能。有些图片的缩略图很少使用到，如果能按需生成岂不更好？
3. 使用[七牛](https://www.qiniu.com/)、[阿里云](https://www.aliyun.com/)提供的云存储及数据处理服务，解决图片的处理、存储、多节点访问速度的问题，这种方式优点是方案成熟，相应的有一定费用和开发工作，另外有一些小概率的风险，比如云服务挂掉影响本站访问。
4. 使用第三方的图片处理程序，比如[zimg](http://blog.buaa.us/zimg-v2-release/)，点击查看[使用手册](http://zimg.buaa.us/guidebook.html)，[@招牌疯子](http://weibo.com/buaazp)开发。zimg的性能和扩展性不错，文档也很完善，会继续保持关注。

-----

本文使用的是Nginx+Lua+GraphicsMagick实现缩略图功能，图片的上传及删除还是交由web服务处理，缩略图由单独的模块去完成。最终效果类似淘宝图片，实现自定义图片尺寸功能，可根据图片加后缀_100x100.jpg(固定高宽),_-100.jpg(定高),_100-.jpg(定宽)形式实现自定义输出图片大小。
# 说明
**文件夹规划**
```
img.xxx.com(如/usr/local/filebase)
├─upload
│  └─img
│    ├─001.jpg
│    └─002.jpg
```
**自定义尺寸后的路径**
```
thumb(/tmp/thumb,可在conf文件里面更改)
├─upload
│  └─img
│    ├─001.jpg_100x100.jpg #固定高和宽
│    ├─001.jpg_-100.jpg #定高
│    ├─001.jpg_200-.jpg #定宽
│    └─002.jpg_300x300.jpg #固定高和宽
```
* 其中img.xxx.com为图片站点根目录，img目录是原图目录
* 缩略图目录根据保持原有结构，并单独设置目录，可定时清理。

**链接地址对应关系**

原图访问地址：http://img.xxx.com/upload/img/001.jpg <br>
缩略图访问地址：http://img.xxx.com/upload/img/001.jpg_100x100.jpg 即为宽100,高100 <br>
自动宽地址: http://img.xxx.com/upload/img/001.jpg_-100.jpg 用"-"表示自动,即为高100,宽自动 <br>
自动高地址: http://img.xxx.com/upload/img/001.jpg_200-.jpg 用"-"表示自动,即为宽200,高自动

**访问流程** <br>
* 首先判断缩略图是否存在，如存在则直接显示缩略图；
* 缩略图不存在,则判断原图是否存在，如原图存在则拼接graphicsmagick(gm)命令,生成并显示缩略图,否则返回404

# 安装
**系统环境**
centOS7 X64 虚拟机内最小化安装<br>
以下操作均在此系统中操作，仅供参考 <br>
**1、环境准备**
```
yum install -y wget  git 
yum install -y gcc gcc-c++ zlib zlib-devel openssl openssl-devel pcre pcre-devel
yum install -y libpng libjpeg libpng-devel libjpeg-devel ghostscript libtiff libtiff-devel freetype freetype-devel
yum install -y GraphicsMagick GraphicsMagick-devel
```
如果提示没有GraphicsMagick的可用安装包，请自行安装GraphicsMagick，具体可参考我的另一篇文章：[CentOS7下安装GraphicsMagick1.3.21](http://blog.51cto.com/zhaobotao/2070287)。<br>
**2、下载相关应用**
```
cd /usr/local/src
wget http://nginx.org/download/nginx-1.8.0.tar.gz
wget http://luajit.org/download/LuaJIT-2.0.4.tar.gz
wget http://zlib.net/fossils/zlib-1.2.8.tar.gz
```
**3、下载nginx组件**
```
git clone https://github.com/alibaba/nginx-http-concat.git
git clone https://github.com/simpl/ngx_devel_kit.git
git clone https://github.com/openresty/echo-nginx-module.git
git clone https://github.com/openresty/lua-nginx-module.git
```
**解压安装**
```
tar -zxf nginx-1.8.0.tar.gz
tar -zxf LuaJIT-2.0.4.tar.gz
tar -zxf zlib-1.2.8.tar.gz
```
**1、安装LuaJIT**
```
cd LuaJIT-2.0.4
make -j8
make install 
export LUAJIT_LIB=/usr/local/lib
export LUAJIT_INC=/usr/local/include/luajit-2.0
ln -s /usr/local/lib/libluajit-5.1.so.2 /lib64/libluajit-5.1.so.2
cd ..
```
**2、安装nginx**
```
cd nginx-1.8.0
./configure --prefix=/usr/local/nginx \
--sbin-path=/usr/local/nginx/sbin/nginx \
--conf-path=/usr/local/nginx/conf/nginx.conf \
--pid-path=/usr/local/nginx/pid/nginx.pid  \
--lock-path=/usr/local/nginx/pid/nginx.lock \
--error-log-path=/usr/local/nginx/logs/error.log \
--http-log-path=/usr/local/nginx/logs/access.log \
--with-http_ssl_module \
--with-http_realip_module \
--with-http_sub_module \
--with-http_flv_module \
--with-http_dav_module \
--with-http_gzip_static_module \
--with-http_stub_status_module \
--with-http_addition_module \
--with-http_spdy_module \
--with-pcre \
--with-zlib=../zlib-1.2.8 \
--add-module=../nginx-http-concat \
--add-module=../lua-nginx-module \
--add-module=../ngx_devel_kit 
make -j8
make install
```

# 配置
相关配置文件结构位置

```
/usr/local/nginx
│  ├─conf
│      ├─......
│      └─nginx.conf
│  ├─html
│  ├─logs
│  ├─lua
│      ├─autoSize.lua
│      └─cropSize.lua
│  ├─pid
│  ├─sbin
│  └─vhost
│      └─demo.conf
```

**修改nginx配置文件**
```
cd /usr/local/nginx/
vi conf/nginx.conf
```
```
user root;
worker_processes 4;
worker_cpu_affinity 1000 0100 0010 0001;
error_log /usr/local/nginx/logs/error.log error;
pid /usr/local/nginx/pid/nginx.pid;
worker_rlimit_nofile 65535;
events
{
    use epoll;
    worker_connections 65535;
}
http
{
    limit_conn_zone $binary_remote_addr zone=one:10m;
    limit_conn_zone $server_name zone=perserver:10m;
    include mime.types;
    include fastcgi.conf;
    default_type application/octet-stream;
    charset utf-8;
    server_names_hash_bucket_size 128;
    client_header_buffer_size 32k;
    large_client_header_buffers 4 64k;
    sendfile on;
    autoindex off;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 120;


    fastcgi_connect_timeout 60;
    fastcgi_send_timeout 60;
    fastcgi_read_timeout 60;
    fastcgi_buffer_size 128k;
    fastcgi_buffers 8 128k;
    fastcgi_busy_buffers_size 128k;
    fastcgi_temp_file_write_size 128k;


    gzip on;
    gzip_min_length 1k;
    gzip_buffers 4 16k;
    gzip_http_version 1.0;
    gzip_comp_level 2;
    gzip_types text/plain application/x-javascript text/css application/xml;
    gzip_vary on;

    log_format main '$remote_addr - $remote_user [$time_local] "$request" '
    '$status $body_bytes_sent "$http_referer" '
    '"$http_user_agent" $http_x_forwarded_for';

    client_max_body_size 200m;

    #lua_package_path "/etc/nginx/lua/?.lua";

    include /usr/local/nginx/vhost/*.conf;
}
```
**修改站点配置** <br>
普通站点的配置文件，包含固定高宽和定高,定宽两种模式配置
```
cd /usr/local/nginx/
mkdir vhost 
vi vhost/demo.conf
```
```
server {
    listen   80;
    index index.php index.html index.htm;

    set $root_path '/var/www';
    root $root_path;


    location /lua {
        default_type 'text/plain';
        content_by_lua 'ngx.say("hello, ttlsa lua")';
    }

    location / {
        try_files $uri $uri/ /index.php?$args;

        # add support for img which has query params,
        # like:  xxx.jpg?a=b&c=d_750x750.jpg
        if ($args ~* "^([^_]+)_(\d+)+x(\d+)+\.(jpg|jpeg|gif|png)$") {
            set $w $2;
            set $h $3;
            set $img_ext $4;

            # rewrite ^\?(.*)$ _${w}x${h}.$img_ext? last;
            rewrite ([^.]*).(jpg|jpeg|png|gif)$  $1.$2_${w}x${h}.$img_ext? permanent;
        }
    }

    # set var for thumb pic
    set $upload_path /usr/local/filebase;
    set $img_original_root  $upload_path;# original root;
    set $img_thumbnail_root $upload_path/cache/thumb;
    set $img_file $img_thumbnail_root$uri;

    # like：/xx/xx/xx.jpg_100-.jpg or /xx/xx/xx.jpg_-100.jpg
    location ~* ^(.+\.(jpg|jpeg|gif|png))_((\d+\-)|(\-\d+))\.(jpg|jpeg|gif|png)$ {
            root $img_thumbnail_root;    # root path for croped img
            set $img_size $3;

            if (!-f $img_file) {    # if file not exists
                    add_header X-Powered-By 'Nginx+Lua+GraphicsMagick By Botao';  #  header for test
                    add_header file-path $request_filename;    #  header for test
                    set $request_filepath $img_original_root$1;    # origin_img full path：/document_root/1.gif
                    set $img_size $3;    # img width or height size depends on uri
                    set $img_ext $2;    # file ext
                    content_by_lua_file /usr/local/nginx/lua/autoSize.lua;    # load lua
            }
    }

    # like: /xx/xx/xx.jpg_100x100.jpg
    location ~* ^(.+\.(jpg|jpeg|gif|png))_(\d+)+x(\d+)+\.(jpg|jpeg|gif|png)$ {
            root $img_thumbnail_root;    # root path for croped img

            if (!-f $img_file) {    # if file not exists
                    add_header X-Powered-By 'Nginx+Lua+GraphicsMagick By Botao';  #  header for test
                    add_header file-path $request_filename;    #  header for test
                    set $request_filepath $img_original_root$1;    # origin_img file path
                    set $img_width $3;    # img width
                    set $img_height $4;    # height
                    set $img_ext $5;    # file ext
                    content_by_lua_file /usr/local/nginx/lua/cropSize.lua;    # load lua
            }
    }

    # if need (all go there)
    location ~* /upload {
            root $img_original_root;
    }


    location ~ /\.ht {
        deny all;
    }
}
```
**裁切图片lua工具**
```
cd /usr/local/nginx/
mkdir lua 
```
lua文件夹下需要两个文件 <br>
* [autoSize.lua](https://github.com/botaozhao/nginx-lua-GraphicsMagick/blob/master/lua/autoSize.lua) 定高或定宽模式裁切图片处理lua脚本 <br>
* [cropSize.lua](https://github.com/botaozhao/nginx-lua-GraphicsMagick/blob/master/lua/cropSize.lua) 固定高宽模式裁切图片处理lua脚本 <br>

# 访问
**开启80端口**
```
firewall-cmd --permanent --zone=public --add-port=80/tcp
firewall-cmd --reload
```
**启动nginx **
```
cd /usr/local/nginx/
./sbin/nginx 
```
**访问查看图片** <br>
分别访问下面几个地址，测试能否查看及生成缩略图 <br>
http://XXX/upload/img/001.jpg <br>
http://XXX/upload/img/001.jpg_-200.jpg <br>
http://XXX/upload/img/001.jpg_200X200.jpg <br>
效果如下：
![](https://github.com/botaozhao/image/blob/master/gm_img/001.png)
<br> 此时服务器端也已经在相应路径下生成了缩略图文件：
![](https://github.com/botaozhao/image/blob/master/gm_img/002.png)
<br> 至此，nginx+lua+GraphicsMagick生成实时缩略图完成！
