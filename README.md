# Http live streaming using nginx with authentication.

##### nginx.conf file code:

    worker_processes  1;
    
    error_log  logs/error.log info;
    
    events {
        worker_connections  1024;
    }
    
    rtmp {
        server {
            listen 1935;
	    chunk_size 4096;
	    allow publish 127.0.0.1;
	    deny publish all;
    
            application live {
                live on;
		record off;
		# Edit and enable line below to push incoming stream to YouTube
		# push rtmp://x.rtmp.youtube.com/live2/<your-Stream-Key-Copied-From-YouTube>;
            }
            
            application hls {
                live on;
                hls on;  
                hls_path temp/hls;  
                hls_fragment 8s;  
            }
        }
    }
    
    http {
        server {
            listen      8080;
            
            location / {
                root html;
		add_header Access-Control-Allow-Orgin *;
            }
    
            #for set header start
            #location =/ {
               # Disable cache
               # add_header 'Cache-Control' 'no-cache';
               # CORS setup
               # add_header 'Access-Control-Allow-Origin' 'https://example.com' always;
               # add_header 'Access-Control-Expose-Headers' 'Content-Length' always;
               # add_header 'X-Frame-Options' 'DENY' always;
           # }
           #for set header end
            
            location /stat {
                rtmp_stat all;
                rtmp_stat_stylesheet stat.xsl;
            }
    
            location /stat.xsl {
                root html;
            }
            
            location /hls {  
                #server hls fragments  
                types{  
                    application/vnd.apple.mpegurl m3u8;  
                    video/mp2t ts;  
                }  
                alias temp/hls;  
                #expires -1;
    
                #for auth start
                secure_link $arg_auth,$arg_expires;
                secure_link_md5 "$secure_link_expires$uri$remote_addr secret";
    
                location ~ \.m3u8$ { #for two file replace m3u8$ to (m3u8|ts)
                   if ($secure_link = "")  { return 403; } # deny direct links
                   if ($secure_link = "0") { return 410; } # deny expired links
                }
                #for auth end
            }  
        }
    }
    
url stracture :http://127.0.0.1/hls/stream.m3u8?auth=token&expires=time

- **listen 1935** means that RTMP will be listening for connections on port 1935, which is standard.
- **chunk_size 4096** means that RTMP will be sending data in 4KB blocks, which is also standard.
- **allow publish 127.0.0.1** and **deny publish all** mean that the server will only allow video to be published from the same server, to avoid any other users pushing their own streams.
- **application live** defines an application block that will be available at the /live URL path.
- **live on** enables live mode so that multiple users can connect to your stream concurrently, a baseline assumption of video streaming.
- **record off** disables Nginx-RTMP’s recording functionality, so that all streams are not separately saved to disk by default.
- **push** means push incoming stream to other platform.

generate auth hash in php

    <?php
    
    /**
     * @param $baseUrl - non protected part of the URL including hostname, e.g. http://example.com
     * @param $path - protected path to the file, e.g. /downloads/myfile.zip
     * @param $secret - the shared secret with the nginx server. Keep this info secure!!!
     * @param $ttl - the number of seconds until this link expires
     * @param $userIp - ip of the user allowed to download
     * @return string
     */
    function buildSecureLink($baseUrl, $path, $secret, $ttl, $userIp)
    {
        $expires = time() + $ttl;
        $md5 = md5("$expires$path$userIp $secret", true);
        $md5 = base64_encode($md5);
        $md5 = strtr($md5, '+/', '-_');
        $md5 = str_replace('=', '', $md5);
    
        return $baseUrl . $path . '?auth=' . $md5 . '&expires=' . $expires;
    }
    
    
    // example usage
    $secret = 'secret';
    $baseUrl = 'http://127.0.0.1:8080';
    $path = '/hls/stream.m3u8';
    $ttl = 600; //no of seconds this link is active
    $userIp = '127.0.0.1'; // normally you would read this from something like //$_SERVER['REMOTE_ADDR'];
    
    echo buildSecureLink($baseUrl, $path, $secret, $ttl, $userIp);
    

optional:

    location /hls {
            # Referrer protection
            valid_referers server_names;
            if ($invalid_referer) {
            return 403;
            }
    
            # allow CORS preflight requests
            if ($request_method = 'OPTIONS') {
            add_header 'Access-Control-Allow-Origin' 'https://example.com';
            add_header 'Access-Control-Max-Age' 1728000;
            add_header 'Content-Type' 'text/plain charset=UTF-8';
            add_header 'Content-Length' 0;
            return 204;
            }
            }

More link:
[1](https://gist.github.com/mrbar42/09c149059f72da2f09e652d4c5079919 "1")
[2](https://gist.github.com/goregrish/a169b0a72fcc7c30d374d1a2f8a772d2 "2")
