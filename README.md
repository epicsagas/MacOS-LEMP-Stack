# Install LEMP stack on MacOS (with phpenv)

## Install Homebrew

- https://brew.sh

``` /usr/bin/ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)" ```

## mariadb

- Install mariadb
``` brew install mariadb ```

- Init mysql
``` sudo mysql_secure_installation ```

- Add user root@'%' for using dbms tools like Datagrip, DBeaver, Sequel Pro, etc
``` mysql -uroot -p ```
``` MariaDB> create user root@'%' identified by 'your password'; ```
``` MariaDB> grant all privileges on *.* to root@'%' ```
``` MariaDB> flush privileges; ```
``` MariaDB> \q ```

- `my.cnf` example
```
[mysqld]
innodb_log_file_size = 32M
innodb_buffer_pool_size = 1024M
innodb_log_buffer_size = 4M
slow_query_log = 1
query_cache_limit = 512K
query_cache_size = 128M
skip-name-resolve
slow_query_log
slow_query_log_file=/usr/local/var/log/mariadb/mariadb-slow.log
```

## phpenv

- Install phpenv
``` git clone https://github.com/madumlao/phpenv.git ~/.phpenv ```

- Install php-build
``` git clone https://github.com/php-build/php-build.git ~/.phpenv/plugins/php-build ```

- Install libraries
``` brew install re2c openssl bison libxml2 autoconf automake icu4c libjpeg libpng libmcrypt libzip ```

- Fix Missing Headers Compiling on macOS Mojave : https://donatstudios.com/MojaveMissingHeaderFiles
``` sudo installer -pkg /Library/Developer/CommandLineTools/Packages/macOS_SDK_headers_for_macOS_10.14.pkg -target / ```

- MacOS 10.15 Catalina build error could handle with [this solution](https://github.com/phpenv/phpenv/issues/98#issuecomment-552384823) 

- export path to `.bash_profile`
```
# phpenv
export PATH="/usr/local/opt/bison/bin:$PATH"
export PHPENV_ROOT="$HOME/.phpenv"
export PATH="$PHPENV_ROOT/bin:$PATH"
eval "$(phpenv init -)"
```

- load `.bash_profile`
``` source ~/.bash_profile ```

- using composer
``` git clone https://github.com/ngyuki/phpenv-composer.git ~/.phpenv/plugins/phpenv-composer && phpenv rehash ```

- phpenv commands

	1. check installed versions : `phpenv versions`
	2. install specific version : `phpenv install 7.3.9 && phpenv rehash`
	3. set global version : `phpenv global 7.3.9`
	4. set local version : `phpenv local 7.3.9`
	5. update phpenv : `cd $(phpenv root) && git pull`
	6. update php-build : `cd $(phpenv root)/plugins/php-build && git pull`

- php-fpm setting : ~/.phpenv/versions/{php version}/etc/php-fpm.d/www.conf

```
catch_workers_output = yes
php_flag[display_errors] = On
php_admin_value[error_log] = /usr/local/var/log/php/fpm7.3-php.www.log 
slowlog = /usr/local/var/log/php/fpm7.3-php.slow.log 
php_admin_flag[log_errors] = On
php_admin_value[memory_limit] = 1024M
request_slowlog_timeout = 10
php_admin_value[upload_max_filesize] = 100M
php_admin_value[post_max_size] = 100M
```

## nginx

- install nginx
``` brew install nginx ```

- global config /usr/local/nginx/global/common.conf
```
    add_header X-Frame-Options "SAMEORIGIN";
    add_header X-XSS-Protection "1; mode=block";
    add_header X-Content-Type-Options "nosniff";
    client_body_in_file_only clean;
    client_body_buffer_size 32K;
    client_max_body_size 100M;
    send_timeout 300s;

    location = /favicon.ico { access_log off; log_not_found off; }
    location = /robots.txt  { access_log off; log_not_found off; }

    charset utf-8;

    error_page 404 /index.php;

    gzip on;
    gzip_vary on;
    gzip_disable "msie6";
    gzip_comp_level 6;
    gzip_min_length 1100;
    gzip_buffers 16 8k;
    gzip_proxied any;
    gzip_types
        text/plain
        text/css
        text/js
        text/xml
        text/javascript
        application/javascript
        application/x-javascript
        application/json
        application/xml
        application/xml+rss;

    location ~* \.(?:jpg|jpeg|gif|png|ico|cur|gz|svg|svgz|mp4|ogg|ogv|webm|htc|svg|woff|woff2|ttf)$ {
        expires max;
        access_log off;
        add_header Cache-Control "public";
    }

    location ~* \.(?:css|js)$ {
      expires 7d;
      access_log off;
      add_header Cache-Control "public";
    }

    location ~ /\.ht {
        deny  all;
    }

    location ~ /\.env* {
        deny  all;
    }

    location ~ /\.(?!well-known).* {
        deny all;
    }
```

- php.conf example `/usr/local/nginx/php7.3.conf`
```
location ~ \.php$ {
  proxy_intercept_errors on;
  try_files $uri /index.php;
  fastcgi_split_path_info ^(.+\.php)(/.+)$;
  include fastcgi_params;
  fastcgi_read_timeout 300;
  fastcgi_buffer_size 128k;
  fastcgi_buffers 8 128k;
  fastcgi_busy_buffers_size 128k;
  fastcgi_temp_file_write_size 128k;
  fastcgi_index index.php;
  fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
  fastcgi_pass 127.0.0.1:9073;
}
```

- site setting example including alias for subdirectory
```
server {
    listen 80;
    server_name main.test;
    root /Users/name/workspace/www/main/public;
    index index.html index.htm index.php;

    access_log /usr/local/var/log/nginx/main.test-access.log combined;
    error_log /usr/local/var/log/nginx/main.test-error.log;

    include global/common.conf;

    # sub1 laravel
    location ^~ /sub1 {
        alias ~/workspace/www/sub1/public;
        try_files $uri $uri/ @sub1;

        location ~ \.php$ {
            fastcgi_split_path_info ^(.+\.php)(/.+)$;
            fastcgi_pass 127.0.0.1:9071;
            include fastcgi_params;
            fastcgi_param SCRIPT_FILENAME ~/workspace/www/sub1/public/index.php;
        }
    }

    location @sub1 {
        rewrite /sub1/(.*)$ /sub1/index.php?/$1 last;
    }
    # end sub1

    location / {
        autoindex on;
        try_files $uri $uri/ /index.php?$query_string;
        include php7.3.conf;
    }
}
```

## Tips
- cli shortcuts `~/.bash_profile`

```
alias nginx.start='brew services start nginx'
alias nginx.stop='brew services stop nginx'
alias nginx.restart='nginx.stop && nginx.start'
alias fpm71.start='~/.phpenv/versions/7.1.8/etc/init.d/php-fpm start'
alias fpm71.stop='~/.phpenv/versions/7.1.8/etc/init.d/php-fpm stop'
alias fpm71.restart='fpm71.stop && fpm71.start'
alias fpm73.start='~/.phpenv/versions/7.3.9/etc/init.d/php-fpm start'
alias fpm73.stop='~/.phpenv/versions/7.3.9/etc/init.d/php-fpm stop'
alias fpm73.restart='fpm73.stop && fpm73.start'
alias mysql.start='brew services start mariadb'
alias mysql.stop='brew services stop mariadb'
alias mysql.restart='mysql.stop && mysql.start'
alias localserver.stop='mysql.stop && nginx.stop && php-fpm71.stop && php-fpm73.stop'
alias localserver.start='mysql.start && nginx.start && php-fpm71.start && fpm73.start'
alias localserver.restart='localserver.stop && localserver.start'
```


## References
- https://brew.sh/
- https://github.com/phpenv/phpenv
- https://donatstudios.com/MojaveMissingHeaderFiles
- https://mariadb.com/kb/en/library/installing-mariadb-on-macos-using-homebrew/
