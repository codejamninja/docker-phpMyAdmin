# osixia/phpmyadmin

[![Docker Pulls](https://img.shields.io/docker/pulls/osixia/phpmyadmin.svg)][hub]
[![Docker Stars](https://img.shields.io/docker/stars/osixia/phpmyadmin.svg)][hub]
[![](https://images.microbadger.com/badges/image/osixia/phpmyadmin.svg)](http://microbadger.com/images/osixia/phpmyadmin "Get your own image badge on microbadger.com")

[hub]: https://hub.docker.com/r/osixia/phpmyadmin/

Latest release: 4.8.0.1 - phpMyAdmin 4.8.0 - [Changelog](CHANGELOG.md) | [Docker Hub](https://hub.docker.com/r/osixia/phpmyadmin/) 

**A docker image to run phpMyAdmin.**
> [phpmyadmin.net](https://www.phpmyadmin.net)

- [Quick start](#quick-start)
	- [MariaDB & phpMyAdmin in 1'](#mariadb--phpmyadmin-in-1)
- [Beginner Guide](#beginner-guide)
	- [Use your own phpMyAdmin config](#use-your-own-phpmyadmin-config)
	- [HTTPS](#https)
		- [Use autogenerated certificate](#use-autogenerated-certificate)
		- [Use your own certificate](#use-your-own-certificate)
		- [Disable HTTPS](#disable-https)
	- [Fix docker mounted file problems](#fix-docker-mounted-file-problems)
	- [Debug](#debug)
- [Environment Variables](#environment-variables)
	- [Set your own environment variables](#set-your-own environment-variables)
		- [Use command line argument](#use-command-line-argument)
		- [Link environment file](#link-environment-file)
		- [Make your own image or extend this image](#make-your-own image-or-extend-this-image)
- [Advanced User Guide](#advanced-user-guide)
	- [Extend osixia/phpmyadmin:4.8.0.1 image](#extend-osixiaphpmyadmin472-image)
	- [Make your own phpMyAdmin image](#make-your-own-phpmyadmin-image)
	- [Tests](#tests)
	- [Kubernetes](#kubernetes)
	- [Under the hood: osixia/web-baseimage](#under-the-hood-osixiaweb-baseimage)
- [Security](#security)
- [Changelog](#changelog)

## Quick start

Run a phpMyAdmin docker image by replacing `db.example.com` with your mysql host or IP :

    docker run -p 6443:443 \
           --env PHPMYADMIN_DB_HOSTS=db.example.com \
           --detach osixia/phpmyadmin:4.8.0.1

That's it :) you can access phpMyAdmin on [https://localhost:6443](https://localhost:6443)

### MariaDB & phpMyAdmin in 1'

Example script:

		#!/bin/bash -e
		docker run --name bdd-service --hostname bdd-service --env MARIADB_ROOT_ALLOWED_NETWORKS="#PYTHON2BASH:['172.17.%.%', 'localhost', '127.0.0.1', '::1']" --detach osixia/mariadb:0.2.9

		docker run --name phpmyadmin-service --hostname phpmyadmin-service --link bdd-service:bdd-host --env PHPMYADMIN_DB_HOSTS=bdd-host --detach osixia/phpmyadmin:4.8.0.1

		PHPMY_IP=$(docker inspect -f "{{ .NetworkSettings.IPAddress }}" phpmyadmin-service)

		echo "Go to: https://$PHPMY_IP"
		echo "Login: admin"
		echo "Password: admin"

## Beginner Guide

### Use your own phpMyAdmin config
This image comes with a phpMyAdmin config.inc.php file that can be easily customized via environment variables for a quick bootstrap, but setting your own config.inc.php is possible. 2 options:

- Link your config file at run time to `/container/service/phpmyadmin/assets/config/config.inc.php` :

      docker run --volume /data/my-config.inc.php:/container/service/phpmyadmin/assets/config/config.inc.php --detach osixia/phpmyadmin:4.8.0.1

- Add your config file by extending or cloning this image, please refer to the [Advanced User Guide](#advanced-user-guide)

### HTTPS

#### Use autogenerated certificate
By default HTTPS is enable, a certificate is created with the container hostname (it can be set by docker run --hostname option eg: phpmyadmin.my-company.com).

	docker run --hostname phpmyadmin.my-company.com --detach osixia/phpmyadmin:4.8.0.1

#### Use your own certificate

You can set your custom certificate at run time, by mounting a directory containing those files to **/container/service/phpmyadmin/assets/apache2/certs** and adjust their name with the following environment variables:

	docker run --volume /path/to/certifates:/container/service/phpmyadmin/assets/apache2/certs \
	--env PHPMYADMIN_HTTPS_CRT_FILENAME=my-cert.crt \
	--env PHPMYADMIN_HTTPS_KEY_FILENAME=my-cert.key \
	--env PHPMYADMIN_HTTPS_CA_CRT_FILENAME=the-ca.crt \
	--detach osixia/phpmyadmin:4.8.0.1

Other solutions are available please refer to the [Advanced User Guide](#advanced-user-guide)

#### Disable HTTPS
Add --env PHPMYADMIN_HTTPS=false to the run command :

    docker run --env PHPMYADMIN_HTTPS=false --detach osixia/phpmyadmin:4.8.0.1

### Fix docker mounted file problems

You may have some problems with mounted files on some systems. The startup script try to make some file adjustment and fix files owner and permissions, this can result in multiple errors. See [Docker documentation](https://docs.docker.com/v1.4/userguide/dockervolumes/#mount-a-host-file-as-a-data-volume).

To fix that run the container with `--copy-service` argument :

		docker run [your options] osixia/phpmyadmin:4.8.0.1 --copy-service

### Debug

The container default log level is **info**.
Available levels are: `none`, `error`, `warning`, `info`, `debug` and `trace`.

Example command to run the container in `debug` mode:

	docker run --detach osixia/phpmyadmin:4.8.0.1 --loglevel debug

See all command line options:

	docker run osixia/phpmyadmin:4.8.0.1 --help

## Environment Variables

Environment variables defaults are set in **image/environment/default.yaml**

See how to [set your own environment variables](#set-your-own-environment-variables)

- **PHPMYADMIN_CONFIG_ABSOLUTE_URI**: Sets here the complete URL (with full path) to your phpMyAdmin installation’s directory. E.g. http://www.example.net/path_to_your_phpMyAdmin_directory/. Note also that the URL on most of web servers are case–sensitive. Don’t forget the trailing slash at the end. it is advisable to try leaving this blank. In most cases phpMyAdmin automatically detects the proper setting. Defaults to `empty`.

- **PHPMYADMIN_DB_HOSTS**: Set phpMyAdmin server config. Defaults to :

  ```yaml
  - db1.example.org:
    - port: 3306
    - connect_type: tcp
    - auth_type: cookie
    - ssl: true
    - ssl_ca: /container/service/mariadb-client/assets/certs/ca.crt
    - ssl_cert: /container/service/mariadb-client/assets/certs/cert.crt
    - ssl_key: /container/service/mariadb-client/assets/certs/cert.key
  - db2.example.org
  - db3.example.org
  ```
  This will be converted in the phpmyadmin config.inc.php file to :
  ```php7
	$cfg['Servers'][1]['host'] = 'db1.example.org';
	$cfg['Servers'][1]['port']='3306';
	$cfg['Servers'][1]['connect_type']='tcp';
	$cfg['Servers'][1]['auth_type']='cookie';
	$cfg['Servers'][1]['ssl']=true;
	$cfg['Servers'][1]['ssl_ca']='/container/service/mariadb-client/assets/certs/ca.crt';
	$cfg['Servers'][1]['ssl_cert']='/container/service/mariadb-client/assets/certs/cert.crt';
	$cfg['Servers'][1]['ssl_key']='/container/service/mariadb-client/assets/certs/cert.key';
	$cfg['Servers'][2]['host'] = 'db2.example.org';
	$cfg['Servers'][3]['host'] = 'db3.example.org';
  ```
  All server configuration are available, just add the needed entries, for example:  
  ```yaml
	- db1.example.org:
    - port: 3306
    - connect_type: tcp
    - auth_type: cookie
		- compress: false
		- user: dbuser
		- nopassword: true
  - db2.example.org
  - db3.example.org
  ```

  See complete list: http://docs.phpmyadmin.net/fr/latest/config.html, $cfg['Servers'][$i] configs.

  If you want to set this variable at docker run command add the tag `#PYTHON2BASH:` and convert the yaml in python:

		docker run --env PHPMYADMIN_DB_HOSTS="#PYTHON2BASH:[{'db1.example.org': [{'port': 3306},{'connect_type': 'tcp'},{'auth_type': 'cookie'},{'ssl': True},{'ssl_ca': '/container/service/mariadb-client/assets/certs/ca.crt'},{'ssl_cert': '/container/service/mariadb-client/assets/certs/cert.crt'},{'ssl_key': '/container/service/mariadb-client/assets/certs/cert.key'}]},'db2.example.org','db3.example.org']" --detach osixia/phpmyadmin:4.8.0.1

  To convert yaml to python online: http://yaml-online-parser.appspot.com/

- **PHPMYADMIN_CONFIG_DB_HOST**: Set $cfg['Servers'][$i]['controlhost']. Defaults to `empty`.
- **PHPMYADMIN_CONFIG_DB_PORT**: Set $cfg['Servers'][$i]['controlport']. Defaults to `empty`.
- **PHPMYADMIN_CONFIG_DB_NAME**: Set $cfg['Servers'][$i]['pmadb']. Defaults to `empty`.
- **PHPMYADMIN_CONFIG_DB_USER**:  Set $cfg['Servers'][$i]['controluser']. Defaults to `empty`.
- **PHPMYADMIN_CONFIG_DB_PASSWORD**:  Set $cfg['Servers'][$i]['controlpass']. Defaults to `empty`.

- **PHPMYADMIN_CONFIG_DB_TABLES**: phpMyadmin config database tables names added to each server config. Defaults to:
  ```yaml
	- bookmarktable: pma__bookmark
  - relation: pma__relation
  - table_info: pma__table_info
  - table_coords: pma__table_coords
  - pdf_pages: pma__pdf_pages
  - column_info: pma__column_info
  - history: pma__history
  - table_uiprefs: pma__table_uiprefs
  - tracking: pma__tracking
  - userconfig: pma__userconfig
  - recent: pma__recent
  - favorite: pma__favorite
  - users: pma__users
  - usergroups: pma__usergroups
  - navigationhiding: pma__navigationhiding
  - savedsearches: pma__savedsearches
  - central_columns: pma__central_columns
	```

Apache :
- **PHPMYADMIN_SERVER_ADMIN**: Server admin email. Defaults to `webmaster@example.org`
- **PHPMYADMIN_SERVER_PATH**: Server path (usefull if behind a reverse proxy). Defaults to `/phpmyadmin`

HTTPS :
- **PHPMYADMIN_HTTPS**: Use apache ssl config. Defaults to `true`
- **PHPMYADMIN_HTTPS_CRT_FILENAME**: Apache ssl certificate filename. Defaults to `phpmyadmin.crt`
- **PHPMYADMIN_HTTPS_KEY_FILENAME**: Apache ssl certificate private key filename. Defaults to `phpmyadmin.key`
- **PHPMYADMIN_HTTPS_CA_CRT_FILENAME**: Apache ssl CA certificate filename. Defaults to `ca.crt`

Reverse proxy HTTPS :
- **PHPMYADMIN_TRUST_PROXY_SSL**: Set to `true` to trust X-Forwarded-Proto header

Other environment variables:
- **PHPMYADMIN_SSL_HELPER_PREFIX**: ssl-helper environment variables prefix. Defaults to `phpmyadmin`, ssl-helper first search config from PHPMYADMIN_SSL_HELPER_* variables, before SSL_HELPER_* variables.
- **MARIADB_CLIENT_SSL_HELPER_PREFIX**: ssl-helper environment variables prefix. Defaults to `database`, ssl-helper first search config from DATABASE_SSL_HELPER_* variables, before SSL_HELPER_* variables.

### Set your own environment variables

#### Use command line argument
Environment variables can be set by adding the --env argument in the command line, for example:

	docker run --env PHPMYADMIN_DB_HOSTS="db.example.org" \
	--detach osixia/phpmyadmin:4.8.0.1

#### Link environment file

For example if your environment file is in :  /data/environment/my-env.yaml

	docker run --volume /data/environment/my-env.yaml:/container/environment/01-custom/env.yaml \
	--detach osixia/phpmyadmin:4.8.0.1

Take care to link your environment file to `/container/environment/XX-somedir` (with XX < 99 so they will be processed before default environment files) and not  directly to `/container/environment` because this directory contains predefined baseimage environment files to fix container environment (INITRD, LANG, LANGUAGE and LC_CTYPE).

#### Make your own image or extend this image

This is the best solution if you have a private registry. Please refer to the [Advanced User Guide](#advanced-user-guide) just below.

## Advanced User Guide

### Extend osixia/phpmyadmin:4.8.0.1 image

If you need to add your custom TLS certificate, bootstrap config or environment files the easiest way is to extends this image.

Dockerfile example:

    FROM osixia/phpmyadmin:4.8.0.1
    MAINTAINER Your Name <your@name.com>

    ADD https-certs /container/service/phpmyadmin/assets/apache2/certs
    ADD database-certs /container/service/mariadb-client/assets/certs
    ADD my-config.inc.php /container/service/phpmyadmin/assets/config/config.inc.php
    ADD environment /container/environment/01-custom


### Make your own phpMyAdmin image

Clone this project :

	git clone https://github.com/osixia/docker-phpMyAdmin
	cd docker-phpMyAdmin

Adapt Makefile, set your image NAME and VERSION, for example :

	NAME = osixia/phpmyadmin
	VERSION = 4.7.2

	becomes :
	NAME = billy-the-king/phpmyadmin
	VERSION = 0.1.0

Add your custom certificate, environment files, config.inc.php ...

Build your image :

	make build

Run your image :

	docker run -d billy-the-king/phpmyadmin:0.1.0

### Tests

We use **Bats** (Bash Automated Testing System) to test this image:

> [https://github.com/sstephenson/bats](https://github.com/sstephenson/bats)

Install Bats, and in this project directory run :

	make test

### Kubernetes

Kubernetes is an open source system for managing containerized applications across multiple hosts, providing basic mechanisms for deployment, maintenance, and scaling of applications.

More information:
- http://kubernetes.io
- https://github.com/kubernetes/kubernetes

A kubernetes example is available in **example/kubernetes**

### Under the hood: osixia/web-baseimage

This image is based on osixia/web-baseimage.
More info: https://github.com/osixia/docker-web-baseimage

## Security
If you discover a security vulnerability within this docker image, please send an email to the Osixia! team at security@osixia.net. For minor vulnerabilities feel free to add an issue here on github.

Please include as many details as possible.

## Changelog

Please refer to: [CHANGELOG.md](CHANGELOG.md)
