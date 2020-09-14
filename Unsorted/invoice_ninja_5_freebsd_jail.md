# Installing Invoice Ninja 5 into a FreeBSD jail

Taken from several different sources:
* <https://invoiceninja.github.io/selfhost.html#migrating-from-v4>
* <https://github.com/invoiceninja/invoiceninja/wiki/FreeNAS-11.2-(FreeBSD)>
* <https://forum.invoiceninja.com/t/install-invoice-ninja-v5-on-centos-8/4293>
* <https://www.gitmemory.com/issue/paulmaunders/delivery-slot-bot/69/623181525>

## Create your jail

## Install necessary packages

Install the following packages:

```tcsh
pkg install nginx openssl python3 npm mariadb104-server php74 php74-{ctype,pdo,pdo_mysql,session,iconv,filter,openssl,phar,mysqli,simplexml,xmlreader,xmlwriter,fileinfo,pear-PHP_Parser,tokenizer,gd,curl,gmp,json,zip,xml,readline,opcache,mbstring,bcmath,curl,partisan,extensions,dom,exif}
```

Enable the services necessary, and configure `mariadb` with the appropriate configuration file. Also, for some odd reason when `mariadb` is configured to use a Unix socket, it neglects to set the permissions on the folder (`/var/run/mysql`) properly, so we'll need to `chown` that folder.

```tcsh
sysrc mysql_enable=YES mysql_optionfile=/usr/local/etc/mysql/my.cnf nginx_enable=YES php_fpm_enable=YES
service mysql-server start          # This will create any necessary folders
chown -R 88:88 /var/run/mysql       # UID/GID 88 corresponds to the mysql user on FreeBSD
service mysql-server start
```

With `mariadb-server` running now, we can tighten up the installation with `mysql_secure_installation`. Once that is done, we will configure `mariadb` with the user and table necessary.

```tcsh

```



```tcsh
sed -i '' -e 's?listen = 127.0.0.1:9000?listen = /var/run/php-fpm.sock?g' /usr/local/etc/php-fpm.d/www.conf
sed -i '' -e 's/;listen.owner = www/listen.owner = www/g' /usr/local/etc/php-fpm.d/www.conf
sed -i '' -e 's/;listen.group = www/listen.group = www/g' /usr/local/etc/php-fpm.d/www.conf
sed -i '' -e 's/;listen.mode = 0660/listen.mode = 0600/g' /usr/local/etc/php-fpm.d/www.conf
sed -i '' -e 's/;env=[/env=[/g' /usr/local/etc/php-fpm.d/www.conf
cp /usr/local/etc/php.ini-production /usr/local/etc/php.ini
sed -i '' -e 's?;cgi.fix_pathinfo=1?cgi.fix_pathinfo=0?g' /usr/local/etc/php.ini
```

Let's create the standard `nginx.conf` per Invoice Ninja's instructions for v5:

```tcsh
cat <<EOF >> /usr/local/etc/nginx/conf.d/ininja.conf
server {
    listen=                 80;
	server_name				invoiceninja.com	;root					/usr/local/www/invoiceninja/public;
	index					index.html index.htm index.php;
	client_max_body_size	20M;
	charset					utf-8;

	gzip					on;
	gzip_types				application/javascript application/x-javascript text/javascript text/plain application/xml application/json;
	gzip_proxied			no-cache no-store private expired auth;
	gzip_min_length			1000;

	location = /favicon.ico { access_log off; log_not_found off; }
	location = /robots.txt { access_log off; log_not_found off; return 200 "User-agent: *\nDisallow: /\n";}

	access_log				/var/log/nginx/ininja.acc.log;
	error_log				/var/log/nginx/ininja.err.log;

	location / {
		try_files			$uri $uri/ /index.php?$query_string;
	}

	if (!-e $request_filename) {
		rewrite ^(.+)$ /index.php?q= last;
	}

	location ~ \.php$ {
		fastcgi_split_path_info		^(.+\.php)(/.+)$;
		fastcgi_pass				unix:/var/run/php-fpm.sock;
		fastcgi_index				index.php;
		include						fastcgi_params;
		fastcgi_param				SCRIPT_FILENAME $document_root$fastcgi_script_name;
		fastcgi_intercept_errors	off;
		fastcgi_buffer_size			16k;
		fastcgi_buffers				4 16k;
	}

	location ~ /\.ht {
		deny all;
	}
}
EOF
```

At let's edit the default `nginx.conf` so that way we just use an `include` directive to bring the configuration in:

```tcsh
cat <<EOF >> /usr/local/etc/nginx/nginx.conf
worker_processes		1;
pid						/var/run/nginx.pid;

events {
	worker_connections	1024;
}

http {
	include				mime.types;
	default_type		application/octet-stream;

	keepalive_timeout	65;

	include				/usr/local/etc/nginx/conf.d/*.conf;
}
EOF
```

Fetch the Invoice Ninja source:

```tcsh
git clone https://github.com/invoiceninja/invoiceninja.git /usr/local/www/invoiceninja
git checkout v2 /usr/local/www/invoiceninja
cd /usr/local/www/invoiceninja
php -d memory_limit=-1 /usr/local/bin/composer update --no-plugins --no-scripts
# puppeteer 1.20.0 really doesn't want to work on FreeBSD, so let's try 3.0.2
sed -i '' -e 's/"puppeteer": "^1.20.0"/"puppeteer": "^3.0.2"/g' /usr/local/www/invoiceninja/package.json
```

