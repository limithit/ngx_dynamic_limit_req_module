# ngx_dynamic_limit_req_module
 
## Introduction

The *ngx_dynamic_limit_req_module* module is used to dynamically lock IP and release it periodically.

Table of Contents
=================
* [dynamic_limit_req_zone](#dynamic_limit_req_zone)
* [dynamic_limit_req](#dynamic_limit_req)
* [dynamic_limit_req_log_level](#dynamic_limit_req_log_level)
* [dynamic_limit_req_status](#dynamic_limit_req_status)
* [black-and-white-list](#black-and-white-list)
* [principle](#principle)
* [Installation](#Installation)
* [About](#About)
* [Donate](#Donate)
* [Extend](#Extend)
* [Api-count](#Api-count)

## dynamic_limit_req_zone
Sets parameters for a shared memory zone that will keep states for various keys. In particular, the state stores the current number of excessive requests. The key can contain text, variables, and their combination. Requests with an empty key value are not accounted. 
```
 Syntax:  dynamic_limit_req_zone key zone=name:size rate=rate [sync]  redis=127.0.0.1 block_second=time;
 Default: —
 Context: http
 ```
 
## dynamic_limit_req
Sets the shared memory zone and the maximum burst size of requests. If the requests rate exceeds the rate configured for a zone, their processing is delayed such that requests are processed at a defined rate. Excessive requests are delayed until their number exceeds the maximum burst size in which case the request is terminated with an error. By default, the maximum burst size is equal to zero.
```
 Syntax:  dynamic_limit_req zone=name [burst=number] [nodelay | delay=number];
 Default: —
 Context: http, server, location, if
```

## dynamic_limit_req_log_level
Sets the desired logging level for cases when the server refuses to process requests due to rate exceeding, or delays request processing. Logging level for delays is one point less than for refusals; for example, if “dynamic_limit_req_log_level notice” is specified, delays are logged with the info level.
```
 Syntax:  dynamic_limit_req_log_level info | notice | warn | error;
 Default: dynamic_limit_req_log_level error;
 Context: http, server, location
```

## dynamic_limit_req_status 
Sets the status code to return in response to rejected requests.
```
 Syntax:  dynamic_limit_req_status code;
 Default: dynamic_limit_req_status 503;
 Context: http, server, location, if
```

     

## Configuration example：
```nginx

    worker_processes  2;
    events {
        worker_connections  1024;
    }
    http {
        include       mime.types;
        default_type  application/octet-stream;
        sendfile        on;
        keepalive_timeout  65;
        
   dynamic_limit_req_zone $binary_remote_addr zone=one:10m rate=100r/s redis=127.0.0.1 block_second=300;
   dynamic_limit_req_zone $binary_remote_addr zone=two:10m rate=50r/s redis=127.0.0.1 block_second=600;
   dynamic_limit_req_zone $binary_remote_addr zone=sms:5m rate=5r/m redis=127.0.0.1 block_second=1800;
        
        
        server {
            listen       80;
            server_name  localhost;
            location / {
                
                if ($http_x_forwarded_for) {
                 return 400;
                }
                root   html;
                index  index.html index.htm;
                dynamic_limit_req zone=one burst=100 nodelay;
                dynamic_limit_req_status 403;
            }
            error_page   403 500 502 503 504  /50x.html;
            location = /50x.html {
                root   html;
            }
        }
        server {
            listen       80;
            server_name  localhost2;
            location / {
                root   html;
                index  index.html index.htm; 
                
                    set $flag 0;
                   if ($document_uri ~* "regist"){
                      set $flag "${flag}1";
                        }
                  if ($request_method = POST ) {
                        set $flag "${flag}2";
                          }
                      if ($flag = "012"){
                      dynamic_limit_req zone=sms burst=3 nodelay;
                      dynamic_limit_req_status 403;
                      }

                
                      if ($document_uri ~* "getSmsVerifyCode.do"){
                      dynamic_limit_req zone=sms burst=5 nodelay;
                      dynamic_limit_req_status 444;
                }

                dynamic_limit_req zone=two burst=50 nodelay;
                dynamic_limit_req_status 403;
            }
            error_page   403 502 503 504  /50x.html;
            location = /50x.html {
                root   html;
            }
        }
    }
   
```
    
## If you use CDN at the source station :
```nginx
 worker_processes  2;
    events {
        worker_connections  1024;
    }
    http {
        include       mime.types;
        default_type  application/octet-stream;
        sendfile        on;
        keepalive_timeout  65;
        
       ####--with-http_realip_module  
       
       set_real_ip_from 192.168.16.0/24;
       real_ip_header X-Forwarded-For;
       real_ip_recursive on;
        
   #### $http_x_forwarded_for or $binary_remote_addr
  dynamic_limit_req_zone $http_x_forwarded_for zone=one:10m rate=100r/s redis=127.0.0.1 block_second=300;
        server {
            listen       80;
            server_name  localhost;
            location / {
                root   html;
                index  index.html index.htm;
                dynamic_limit_req zone=one burst=100 nodelay;
                dynamic_limit_req_status 403;
            }
            error_page   403 500 502 503 504  /50x.html;
            location = /50x.html {
                root   html;
            }
        }

    }
```

About [ngx_http_realip_module](http://nginx.org/en/docs/http/ngx_http_realip_module.html)

## black-and-white-list

###  White list rules
 ```redis-cli set whiteip ip```
 
 example：
 ```redis-cli set white192.168.1.1 192.168.1.1```
###  Black list rules 
 ```redis-cli set ip ip ```
 
 example：
 ```redis-cli set 192.168.1.2 192.168.1.2```
 

## Installation

###  Option #1: Compile Nginx with module bundled
    cd redis-4.0**version**/deps/hiredis
    make 
    make install 
    echo /usr/local/lib >> /etc/ld.so.conf
    ldconfig
    
    cd nginx-**version**
    ./configure --add-module=/path/to/this/ngx_dynamic_limit_req_module 
    make
    make install


###  Option #2: Compile dynamic module for Nginx

Starting from NGINX 1.9.11, you can also compile this module as a dynamic module, by using the ```--add-dynamic-module=PATH``` option instead of ```--add-module=PATH``` on the ```./configure``` command line above. And then you can explicitly load the module in your ```nginx.conf``` via the [load_module](http://nginx.org/en/docs/ngx_core_module.html#load_module) directive, for example,

```nginx
    load_module /path/to/modules/ngx_dynamic_limit_req_module.so;
```
## principle
The ngx_dynamic_limit_req_module module is used to limit the request processing rate per a defined key, in particular, the processing rate of requests coming from a single IP address. The limitation is done using the “leaky bucket” method.

## About
This module is an extension based on [ngx_http_limit_req_module](http://nginx.org/en/docs/http/ngx_http_limit_req_module.html).

## Donate
The developers work tirelessly to improve and develop ngx_dynamic_limit_req_module. Many hours have been put in to provide the software as it is today, but this is an extremely time-consuming process with no financial reward. If you enjoy using the software, please consider donating to the devs, so they can spend more time implementing improvements.

 ### Alipay:
![Alipay](https://github.com/limithit/shellcode/blob/master/alipay.png)

## Extend
This module can be works with [RedisPushIptables](https://github.com/limithit/RedisPushIptables),  the application layer matches then the network layer to intercept. Although network layer interception will save resources, there are also deficiencies. Assuming that only one specific interface is filtered and no other interfaces are filtered, those that do not need to be filtered will also be inaccessible. Although precise control is not possible at the network layer or the transport layer, it can be precisely controlled at the application layer. Users need to weigh which solution is more suitable for the event at the time.

## Api-count
### If you want to use the api counting function, please use [limithit-API_alerts](https://github.com/limithit/ngx_dynamic_limit_req_module/tree/limithit-API_alerts). Because not everyone needs this feature, so it doesn't merge into the trunk. Users who do not need this feature can skip this paragraph description.

``` 
git clone https://github.com/limithit/ngx_dynamic_limit_req_module.git
cd ngx_dynamic_limit_req_module
git checkout limithit-API_alerts
```
``` 
root@debian:~# redis-cli 
127.0.0.1:6379> SELECT 3
127.0.0.1:6379[3]> scan 0 match *12/Dec/2018* count 10000 
127.0.0.1:6379[3]> scan 0 match *PV count 10000
1) "0"
2) 1) "[13/Dec/2018]PV"
   2) "[12/Dec/2018]PV"
127.0.0.1:6379[3]> get [12/Dec/2018]PV
"9144"
127.0.0.1:6379[3]> get [13/Dec/2018]PV
"8066"
127.0.0.1:6379[3]> get [13/Dec/2018]UV
"214"

```

This module is compatible with following nginx releases:

Author
Gandalf zhibu1991@gmail.com
