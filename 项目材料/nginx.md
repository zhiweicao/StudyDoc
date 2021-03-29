



## 提供静态资源的配置方式

```nginx
http {
  # main是格式的别名
  log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                    '$status $body_bytes_sent "$http_referer" '
                    '"$http_user_agent" "$http_x_forwarded_for"';
  server{
    listen 8080;						#127.0.0.1:8080就将这个服务转化为上游服务
    server_name localhost;
    
    # main是log的格式
    access_log logs/test_access.log main;
    
    location / {
    	alias dlib/; 					#将dlib下的所有文件一一映射到url地址后
      autoindex on;					#类似于maven的文件下载页面
      set $limit_rate 1k.		#设置传输速度
    }
  }
}
```





## 配置反向代理的方式

配置在另一个nginx执行程序中。（e.g. 静态资源部署在nginx, 反向代理部署在openresty中。)

```nginx
http {
  upstream local {
    server 127.0.0.1:8080;
  } 
  server_name geektime.taohui.pub;
  listen 80;
  
  location / {
    proxy_set_header Host $host;					#将浏览器发送来的包中的host，放入反向代理到上游服务的包中
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    
    #将缓存服务器加入到nginx中
    proxy_cache my_cache;
    
    #设置缓存的key，这样只有同一个用户访问才会碰撞cache
    proxy_cache_key $host$url$is_args$args;
    proxy_cache_valid 200 304 302 1d;
    
    proxy_pass http://local;
  }
}

```



## 配置缓存服务器

```nginx
http {
  include       mime.types;
  default_type  application/octet-stream;
  include       custom.conf;
  #log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
  #                  '$status $body_bytes_sent "$http_referer" '
  #                  '"$http_user_agent" "$http_x_forwarded_for"';
  
  client_max_body_size 60M;
  
  proxy_cache_path /tmp/nginxcache levels=1:2

}
```

