nginx-rtmp-win32
================

* Nginx: 1.13.12  
* Nginx-Rtmp-Module: 1.2.1  
* openssl-1.0.2o
* pcre-8.42
* zlib-1.2.11

# configure arguments
```
nginx version: nginx/1.13.12
built by cl 16.00.30319.01 for 80x86
built with OpenSSL 1.0.2o  27 Mar 2018
TLS SNI support enabled
configure arguments: --with-cc=cl --builddir=objs --prefix= --conf-path=conf/ngi
nx.conf --pid-path=logs/nginx.pid --http-log-path=logs/access.log --error-log-pa
th=logs/error.log --sbin-path=nginx.exe --http-client-body-temp-path=temp/client
_body_temp --http-proxy-temp-path=temp/proxy_temp --http-fastcgi-temp-path=temp/
fastcgi_temp --http-scgi-temp-path=temp/scgi_temp --http-uwsgi-temp-path=temp/uw
sgi_temp --with-cc-opt=-DFD_SETSIZE=1024 --with-openssl=objs/lib/openssl-1.0.2o
--with-openssl-opt=no-asm --with-pcre=objs/lib/pcre-8.42 --with-zlib=objs/lib/zl
ib-1.2.11 --with-select_module --with-http_v2_module --with-http_realip_module -
-with-http_addition_module --with-http_sub_module --with-http_dav_module --with-
http_stub_status_module --with-http_flv_module --with-http_mp4_module --with-htt
p_gunzip_module --with-http_gzip_static_module --with-http_auth_request_module -
-with-http_random_index_module --with-http_secure_link_module --with-http_slice_
module --with-mail --with-stream --with-stream_realip_module --with-http_ssl_mod
ule --with-mail_ssl_module --with-stream_ssl_module --add-module=objs/lib/nginx-
rtmp-module
```
Example nginx.conf

rtmp {

    server {

        listen 1935;

        chunk_size 4000;

        # TV mode: one publisher, many subscribers
        application mytv {

            # enable live streaming
            live on;

            # record first 1K of stream
            record all;
            record_path /tmp/av;
            record_max_size 1K;

            # append current timestamp to each flv
            record_unique on;

            # publish only from localhost
            allow publish 127.0.0.1;
            deny publish all;

            #allow play all;
        }

        # Transcoding (ffmpeg needed)
        application big {
            live on;

            # On every pusblished stream run this command (ffmpeg)
            # with substitutions: $app/${app}, $name/${name} for application & stream name.
            #
            # This ffmpeg call receives stream from this application &
            # reduces the resolution down to 32x32. The stream is the published to
            # 'small' application (see below) under the same name.
            #
            # ffmpeg can do anything with the stream like video/audio
            # transcoding, resizing, altering container/codec params etc
            #
            # Multiple exec lines can be specified.

            exec ffmpeg -re -i rtmp://localhost:1935/$app/$name -vcodec flv -acodec copy -s 32x32
                        -f flv rtmp://localhost:1935/small/${name};
        }

        application small {
            live on;
            # Video with reduced resolution comes here from ffmpeg
        }

        application webcam {
            live on;

            # Stream from local webcam
            exec_static ffmpeg -f video4linux2 -i /dev/video0 -c:v libx264 -an
                               -f flv rtmp://localhost:1935/webcam/mystream;
        }

        application mypush {
            live on;

            # Every stream published here
            # is automatically pushed to
            # these two machines
            push rtmp1.example.com;
            push rtmp2.example.com:1934;
        }

        application mypull {
            live on;

            # Pull all streams from remote machine
            # and play locally
            pull rtmp://rtmp3.example.com pageUrl=www.example.com/index.html;
        }

        application mystaticpull {
            live on;

            # Static pull is started at nginx start
            pull rtmp://rtmp4.example.com pageUrl=www.example.com/index.html name=mystream static;
        }

        # video on demand
        application vod {
            play /var/flvs;
        }

        application vod2 {
            play /var/mp4s;
        }

        # Many publishers, many subscribers
        # no checks, no recording
        application videochat {

            live on;

            # The following notifications receive all
            # the session variables as well as
            # particular call arguments in HTTP POST
            # request

            # Make HTTP request & use HTTP retcode
            # to decide whether to allow publishing
            # from this connection or not
            on_publish http://localhost:8080/publish;

            # Same with playing
            on_play http://localhost:8080/play;

            # Publish/play end (repeats on disconnect)
            on_done http://localhost:8080/done;

            # All above mentioned notifications receive
            # standard connect() arguments as well as
            # play/publish ones. If any arguments are sent
            # with GET-style syntax to play & publish
            # these are also included.
            # Example URL:
            #   rtmp://localhost/myapp/mystream?a=b&c=d

            # record 10 video keyframes (no audio) every 2 minutes
            record keyframes;
            record_path /tmp/vc;
            record_max_frames 10;
            record_interval 2m;

            # Async notify about an flv recorded
            on_record_done http://localhost:8080/record_done;

        }


        # HLS

        # For HLS to work please create a directory in tmpfs (/tmp/hls here)
        # for the fragments. The directory contents is served via HTTP (see
        # http{} section in config)
        #
        # Incoming stream must be in H264/AAC. For iPhones use baseline H264
        # profile (see ffmpeg example).
        # This example creates RTMP stream from movie ready for HLS:
        #
        # ffmpeg -loglevel verbose -re -i movie.avi  -vcodec libx264
        #    -vprofile baseline -acodec libmp3lame -ar 44100 -ac 1
        #    -f flv rtmp://localhost:1935/hls/movie
        #
        # If you need to transcode live stream use 'exec' feature.
        #
        application hls {
            live on;
            hls on;
            hls_path /tmp/hls;
        }

        # MPEG-DASH is similar to HLS

        application dash {
            live on;
            dash on;
            dash_path /tmp/dash;
        }
    }
}

# HTTP can be used for accessing RTMP stats
http {

    server {

        listen      8080;

        # This URL provides RTMP statistics in XML
        location /stat {
            rtmp_stat all;

            # Use this stylesheet to view XML as web page
            # in browser
            rtmp_stat_stylesheet stat.xsl;
        }

        location /stat.xsl {
            # XML stylesheet to view RTMP stats.
            # Copy stat.xsl wherever you want
            # and put the full directory path here
            root /path/to/stat.xsl/;
        }

        location /hls {
            # Serve HLS fragments
            types {
                application/vnd.apple.mpegurl m3u8;
                video/mp2t ts;
            }
            root /tmp;
            add_header Cache-Control no-cache;
        }

        location /dash {
            # Serve DASH fragments
            root /tmp;
            add_header Cache-Control no-cache;
        }
    }
}
