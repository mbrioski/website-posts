---
title: "Delta migration of products from Magento1 to Magento2"
date: 2020-06-03T11:06:46+02:00
draft: false
tags: [ "Magento2", "Data migration"  ]
categories: [ "Magento2" ]
---

It's very common  to handle projects where you need to migrate a Magento1 webshop to a Magento2 webshop: this article describe how achieve this task using [Magento2 data migration tool](https://github.com/magento/data-migration-tool), and in particular how:

* Migrate configurations
* Migrate data
* How exclude some tables from data migration

## Set up docker environment for migration

Create a docker enviroment with 3 containers, i.e:
 ```
  version: '2.1'
  
  services:
    db:
      image: 'mariadb:10.3.22'
      volumes:
        - /var/lib/mysql
        - './.docker/mysql/docker-entrypoint-initdb.d:/docker-entrypoint-initdb.d'
      environment:
        - MYSQL_ROOT_PASSWORD=magento2
        - MYSQL_DATABASE=magento2
        - MYSQL_USER=magento2
        - MYSQL_PASSWORD=magento2
      ports:
      - 3306:3306
      healthcheck:
        test: ["CMD", "mysqladmin" ,"ping", "-h", "localhost"]
        timeout: 10s
        retries: 50
      networks:
        - chromsystems
  
    magento1-db:
      image: 'mariadb:10.3.22'
      volumes:
        - /var/lib/mysql
        - './.docker/magento1-db/docker-entrypoint-initdb.d:/docker-entrypoint-initdb.d'
      environment:
        - MYSQL_ROOT_PASSWORD=magento1
        - MYSQL_DATABASE=magento1
        - MYSQL_USER=magento1
        - MYSQL_PASSWORD=magento1
      ports:
        - 8888:3306
      networks:
        - chromsystems
  
    m2-cli:
      image: magento/magento-cloud-docker-php:7.3-cli
      depends_on:
         db:
          condition: service_healthy
      extends: generic
      volumes: 
        - 'magento2:/app'
        - 'magento-vendor:/app/vendor:delegated'
        - 'magento-generated:/app/generated:delegated'
        - 'magento-setup:/app/setup:delegated'
        - 'magento-static:/app/pub/static:delegated'
        - 'magento-media:/app/pub/media:delegated'
        
    generic:
      image: alpine
      networks:
        - chromsystems
      environment:
        - PHP_MEMORY_LIMIT=2048M
        - UPLOAD_MAX_FILESIZE=64M
        - MAGENTO_ROOT=/app
        - MAGENTO_RUN_MODE=developer
        - PHP_IDE_CONFIG=serverName=chromsystems_local_xdebug
        - XDEBUG_CONFIG=remote_host=host.docker.internal idekey=PHPSTORM remote_port=9002 remote_log=/tmp/xdebug.log remote_enable=1 remote_connect_back=0
        - 'PHP_EXTENSIONS=bcmath bz2 calendar exif gd gettext intl mysqli pcntl pdo_mysql soap sockets sysvmsg sysvsem sysvshm opcache zip redis xsl'
  
  volumes:
    magento2:
      driver: local
      driver_opts:
        type: nfs
        device: ':${PWD}/magento2'
        o: addr=host.docker.internal,rw,nolock,hard,nointr,nfsvers=3
    magento-vendor: {  }
    magento-generated: {  }
    magento-setup: {  }
    magento-var: {  }
    magento-etc: {  }
    magento-static: {  }
    magento-media: {  }
  
 ```

The docker-compose file is just a possible example containing:

* mysql containing magento1 db
* mysql containing magento2 db
* Php container with Magento2 (in the example above Magento 2 is inside a directory *magento2*)

## Install data migration tool

You need to install data migration tool inside your magento2 installation using composer:

```
composer config repositories.data-migration-tool git https://github.com/magento/data-migration-tool
composer require magento/data-migration-tool:<version>
```

Where *<version>* is the version of your current Magento2 version.

## Configure data migration tool

Now that we have data migration tool in place, we need to configure it.

We surf trough *vendor/magento/data-migration-tool/etc/{versionFor-to-versionTo}/{magento1-version}* and here i rename the file **config.xml.dist** file to **config.xml** and  i set up the database connections:

```
    <source>
        <database host="magento1-db" port="3306" name="magento1" user="root" password="magento1"/>
    </source>
    <destination>
        <database host="db" port="3306" name="magento2" user="root" password="magento2"/>
    </destination>
```

 ### Steps

Inside config.xml file, it is possible to configure the data should be migrated for each **steps**: steps are the three types of migration is possible to execute:

* settings
* data
* delta

```
 <steps mode="settings">
        <step title="Settings Step">
            <integrity>Migration\Step\Settings\Integrity</integrity>
            <data>Migration\Step\Settings\Data</data>
        </step>
        <step title="Stores Step">
            <integrity>Migration\Step\Stores\Integrity</integrity>
            <data>Migration\Step\Stores\Data</data>
            <volume>Migration\Step\Stores\Volume</volume>
        </step>
    </steps>
    <steps mode="data">
        <step title="Data Integrity Step">
            <integrity>Migration\Step\DataIntegrity\Integrity</integrity>
        </step>
        <step title="EAV Step">
            <integrity>Migration\Step\Eav\Integrity</integrity>
            <data>Migration\Step\Eav\Data</data>
            <volume>Migration\Step\Eav\Volume</volume>
        </step>
```

Every step tag define the class inside data migration tool where the data for migration is defined.

**The delta migration is only possible if the same data was already migrated in the data migration: migration tool infact tracks the data migrated inside the magento1 database, creating new tables that starts with m2_: with the default configuration is only possible to migrate customer data** 

### Exclude tables from data migration

It's possibile to exclude some tables or data from datamigration using the file **max.xml** inside the folder of the version of your magento1 installation, i.e.:

```
<?xml version="1.0" encoding="UTF-8"?>
<!--
/**
 * Copyright © Magento, Inc. All rights reserved.
 * See COPYING.txt for license details.
 */
-->
<map xmlns:xs="http://www.w3.org/2001/XMLSchema-instance"
     xs:noNamespaceSchemaLocation="urn:magento:module:Magento_DataMigrationTool:etc/map.xsd">
    <source>
        <document_rules>
            <ignore>
                <document>mageworx_downloads_categories</document>
            </ignore>
            <ignore>
                <document>mageworx_downloads_customer</document>
            </ignore>
            <ignore>
                <document>mageworx_downloads_files</document>
            </ignore>
            <ignore>
                <document>mageworx_downloads_relation</document>
            </ignore>
            ...
            </document_rules>
    </source>
</map>
```



## Run data migration

Now that data migration tool is configurared i can use the tool from command line inside *php container*.

```bin/magento``` will display all possible commands for the tool:

*  ```migrate:data {full-path-to-main-configuration-file.xml} ``` to migrate the data  
* ```migrate:delta {full-path-to-main-configuration-file.xml}``` to migrate the data is added into Magento after the main migration  
* ```migrate:settings {full-path-to-main-configuration-file.xml}``` to migrate system configurations

i.e. ```php bin/magento migrate:data -r vendor/magento/data-migration-tool/etc/opensource-to-commerce/1.9.4.4/config.xml```





