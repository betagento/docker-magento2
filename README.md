# Docker Magento 2.4: Varnish 6 + PHP7.4 + Redis + Elasticsearch 7.6 + SSL Letsencrypt + cluster ready docker-compose infrastructure

## Infrastructure overview
* Container 1: MariaDB 8.0
* Container 2: Redis (volatile, for Magento's cache)
* Container 3: Redis (for Magento's sessions)
* Container 4: Apache 2.4 + PHP 7.4 (modphp)
* Container 5: Cron
* Container 6: Varnish 6
* Container 7: Redis 5 (volatile, cluster nodes autodiscovery)
* Container 8: Nginx SSL with Letsencrypt
* Container 9: Elasticsearch 7.6

## Setup Magento 2 project

* Download Magento 2 in any way you want (zip/tgz from website, composer, etc) and extract into your project directory

* Locate to the project and copy "docker-compose.override.yml.dist" to "docker-compose.override.yml"

* Change the default "magento2" to your project's name in the "docker-compose.override.yml" (there are 2 references, under the "cron" section and under the "apache" section)

* Add `"127.0.0.1  betagento.com"` to `/etc/hosts` on host machine

## Add utilities 
  1. Edit `~/.zshrc` or `~/.bashrc` with your preferred editor
  2. Append `"export PATH=$PATH:<path-to-folder>/betagento-docker/bin"` to the file
  3. Type `source ~/.zshrc` or `~/.bashrc` to apply the changes

## Starting all docker containers
``` 
docker-compose up -d 
```
OR
```
m2-up
```

> The fist time you run this command it's gonna take some time to download all the required images from docker hub.

## Install Magento2
Access to Apache container by
```
docker exec -it $(docker ps | grep _apache | cut -f 1 -d " ") bash
```
OR
```
m2-shell
```

And then
```
$ php bin/magento setup:install \
  --base-url='https://betagento.com/' \
  --db-host=betagento_db \
  --db-name=magento2 \
  --db-user=magento2 \
  --db-password=magento2  \
  --admin-user=admin \
  --timezone='Asia/Ho_Chi_Minh' \
  --language=en_US \
  --currency=USD \
  --use-rewrites=1 \
  --cleanup-database \
  --backend-frontname=admin \
  --admin-firstname=AdminFirstName \
  --admin-lastname=AdminLastName \
  --admin-email='hello@betagento.com' \
  --admin-password='ChangeThisPassword1' \
  --session-save=redis \
  --session-save-redis-host=sessions \
  --session-save-redis-port=6379 \
  --session-save-redis-db=0 \
  --session-save-redis-password='' \
  --cache-backend=redis \
  --cache-backend-redis-server=cache \
  --cache-backend-redis-port=6379 \
  --cache-backend-redis-db=0 \
  --page-cache=redis \
  --page-cache-redis-server=cache \
  --page-cache-redis-port=6379 \
  --page-cache-redis-db=1 \
  --search-engine=elasticsearch7 \
  --elasticsearch-host=elasticsearch
```

## Set developer mode if you are on localhost
```
$ php bin/magento deploy:mode:set developer
```

## Deploy static files
```
$ php bin/magento dev:source-theme:deploy
$ php bin/magento setup:static-content:deploy -f
$ php bin/magento setup:di:compile
```
OR
```
m2-flush-cache
```

## Enable Redis for Magento's cache
If you installed Magento via CLI then Redis is already configured, otherwise open magento2/app/etc/env.php and add these lines:
```php
'cache' => [
  'frontend' => [
    'default' => [
      'backend' => 'Cm_Cache_Backend_Redis',
      'backend_options' => [
        'server' => 'cache',
        'port' => '6379',
        'persistent' => '', // Specify a unique string like "cache-db0" to enable persistent connections.
        'database' => '0',
        'password' => '',
        'force_standalone' => '0', // 0 for phpredis, 1 for standalone PHP
        'connect_retries' => '1', // Reduces errors due to random connection failures
        'read_timeout' => '10', // Set read timeout duration
        'automatic_cleaning_factor' => '0', // Disabled by default
        'compress_data' => '1', // 0-9 for compression level, recommended: 0 or 1
        'compress_tags' => '1', // 0-9 for compression level, recommended: 0 or 1
        'compress_threshold' => '20480', // Strings below this size will not be compressed
        'compression_lib' => 'gzip', // Supports gzip, lzf and snappy,
        'use_lua' => '0' // Lua scripts should be used for some operations
      ]
    ],
    'page_cache' => [
      'backend' => 'Cm_Cache_Backend_Redis',
      'backend_options' => [
        'server' => 'cache',
        'port' => '6379',
        'persistent' => '', // Specify a unique string like "cache-db0" to enable persistent connections.
        'database' => '1', // Separate database 1 to keep FPC separately
        'password' => '',
        'force_standalone' => '0', // 0 for phpredis, 1 for standalone PHP
        'connect_retries' => '1', // Reduces errors due to random connection failures
        'lifetimelimit' => '57600', // 16 hours of lifetime for cache record
        'compress_data' => '0' // DISABLE compression for EE FPC since it already uses compression
      ]
    ]
  ]
],
```
and delete all Magento's cache with
```
$ rm -rf var/cache/*
```
from now on the var/cache directory should stay empty cause all the caches should be stored in Redis.

## Enable Redis for Magento's sessions
If you installed Magento via CLI then Redis is already configured, otherwise open magento2/app/etc/env.php and replace these lines:
```php
'session' => [
  'save' => 'redis',
  'redis' => [
    'host' => 'sessions',
    'port' => '6379',
    'password' => '',
    'timeout' => '2.5',
    'persistent_identifier' => '',
    'database' => '0',
    'compression_threshold' => '2048',
    'compression_library' => 'gzip',
    'log_level' => '3',
    'max_concurrency' => '6',
    'break_after_frontend' => '5',
    'break_after_adminhtml' => '30',
    'first_lifetime' => '600',
    'bot_first_lifetime' => '60',
    'bot_lifetime' => '7200',
    'disable_locking' => '0',
    'min_lifetime' => '60',
    'max_lifetime' => '2592000'
  ]
],
```
and delete old Magento's sessions with
```
$ rm -rf var/session/*
```

## Enable Varnish
Varnish Full Page Cache should already be enabled out of the box (we startup Varnish with the default VCL file generated by Magento2) but you could anyway go to "stores -> configuration -> advanced -> system -> full page cache" and:
* Select Varnish in the "caching application" combobox
* Type "apache" in both "access list" and "backend host" fields
* Type 80 in the "backend port" field
* Save

Configure Magento to purge Varnish:

```
$ php bin/magento setup:config:set --http-cache-hosts=varnish
```

https://devdocs.magento.com/guides/v2.3/config-guide/varnish/use-varnish-cache.html

## Enable SSL Support
Add this line to the-project-directory/.htaccess
```
SetEnvIf X-Forwarded-Proto https HTTPS=on
```
Then you can configure Magento as you wish to support secure urls.

If you need to generate new self signed certificates use this command
```
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /etc/nginx/ssl/nginx.key -out /etc/nginx/ssl/nginx.crt
```
then you can mount them into the nginx-ssl container using the "volumes" instruction in the docker-compose.override.yml file. Same thing goes if you need to use custom nginx configurations (you can mount them into /etc/nginx/conf.d). Check the source code of https://github.com/fballiano/docker-nginx-ssl-for-magento2 to better understand where are the configuration stored inside the image/container.

## Scaling apache containers
If you need more horsepower you can
```
docker-compose scale apache=X
```
where X is the number of apache containers you want.

The cron container will check how many apache containers we have (broadcast/discovery service is stored on the redis_clusterdata container) and will update Varnish's VCL.

You can start your system with just one apache container, then scale it afterward, autodiscovery will reconfigure the load balancing on the fly.

Also, the cron container (which updates Varnish's VCL) sets a "probe" to "/fb_host_probe.txt" every 5 seconds, if 1 fails (container has been shut down) the container is considered sick.

## Custom php.ini
We already have a personalized php.ini inside this project: https://github.com/fballiano/docker-magento2-apache-php/blob/master/php.ini but if you want to further customize your settings:
* Edit the conf/php.ini file in this project
* Edit the "docker-compose.override.yml" file, look for the 2 commented lines (under the "cron" section and under the "apache" section) referencing the php.ini
* Start/restart the docker stack

Please note that your php.ini will be the last parsed thus you can override any setting.

___
## TODO
* RabbitMQ?

___
## Solve common __Errors__

### Error 1
> _This happens due to insufficient permissions on the project folder and files. Also www-data must be owner of the project if using Apache as web-server. Please execute commands given below:_

Solved by
```
$ sudo chown -R www-data:www-data [path to magento directory]
```

### Error 2
> _Error `The directory "/var/www/html/generated/code/Magento" cannot be deleted Warning!rmdir(/var/www/html/generated/code/Magento): Directory not empty`_

Solved by
```
$ find . -type f -exec chmod 664 {} \;
$ find . -type d -exec chmod 775 {} \;
$ find ./var -type d -exec chmod 777 {} \;
$ find ./pub/media -type d -exec chmod 777 {} \;
$ find ./pub/static -type d -exec chmod 777 {} \;
$ chmod 777 ./app/etc
$ chmod 644 ./app/etc/*.xml
$ chmod u+x bin/magento
```
OR
```
$ find var vendor pub/static pub/media app/etc -type f -exec chmod g+w {} \;
$ find var vendor pub/static pub/media app/etc -type d -exec chmod g+w {} \;
$ chmod u+x bin/magento
```

## Disable redundant third party modules
```
$ php bin/magento module:status | grep -v Magento | grep -v List | grep -v None | grep -v -e '^$'| xargs php bin/magento module:disable

$ php bin/magento module:status | grep -v Magento | grep -v List | grep -v None | grep -v -e '^$'| xargs php bin/magento module:disable -f
```
