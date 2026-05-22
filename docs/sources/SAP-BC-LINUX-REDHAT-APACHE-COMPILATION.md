---
Author: Tomás Vírseda
Category: Procedure
DocType: How-to guide
Scope: Linux Administration, SAP Basis
Topic: Installation
Tag: apache, webserver, compilation, httpd
Product: SAP Content Server, DMS, Apache, RHEL
User: sapcs
OS: Linux
Command: yum, wget, rpm, make, mkdir, configure, apachectl,
Date: 2019-10-07
---

# Apache compilation and installation in RedHat

## Preparation
Before the SAP Content Server installation, Apache Web Server must be compiled and installed in `/sapdb/apache2`-

Mandatory packages:

```shell
yum install uuidd uuid mod_ssl openssl-devel libapreq2 krb5-devel keyutils-libs-devel zlib-devel pcre-devel openldap-devel libverto-devel libsepol-devel libselinux-devel libdb-devel expat-devel libcom_err-devel cyrus-sasl-devel gcc-c++ libstdc++-devel gcc glibc-headers glibc-devel kernel-headers cpp mpfr libmpc
```

Then, download the sources to a temporary directory:

`wget -c https://dlcdn.apache.org/httpd/httpd-2.4.54.tar.bz2`

> **🔥 CAUTION**\
Uninstall Apache from RedHat distribution

`rpm -e --nodeps httpd`

## Apache compilation

Compile it and install it (as `sapcs` user or `root`? Check):

```shell
mkdir –p /app/sapcs/software/apache
export LDFLAGS="-L/lib64"
export CFLAGS="-D_LARGEFILE_SOURCE -D_FILE_OFFSET_BITS=64 -g"
./configure --prefix=/app/sapcs/software/apache --enable-load-all-modules --enable-mods-shared=most --with-mpm=prefork --enable-ssl --with-expat=builtin
make
```

## Installation

`make install`

## Check installation

Apache 2.0.59 Compilation flags:

`/global/sapcs/software/apache2/bin/apachectl -V`

```
Server version: Apache/2.0.59
Server built:   Aug  6 2007 16:40:45
Server's Module Magic Number: 20020903:12
Server loaded:  APR 0.9.12, APR-UTIL 0.9.12
Compiled using: APR 0.9.12, APR-UTIL 0.9.12
Architecture:   64-bit
Server compiled with....
 -D APACHE_MPM_DIR="server/mpm/prefork"
 -D APR_HAS_SENDFILE
 -D APR_HAS_MMAP
 -D APR_HAVE_IPV6 (IPv4-mapped addresses enabled)
 -D APR_USE_FCNTL_SERIALIZE
 -D APR_USE_PTHREAD_SERIALIZE
 -D SINGLE_LISTEN_UNSERIALIZED_ACCEPT
 -D APR_HAS_OTHER_CHILD
 -D AP_HAVE_RELIABLE_PIPED_LOGS
 -D HTTPD_ROOT="/global/sapcs/software/apache2"
 -D SUEXEC_BIN="/global/sapcs/software/apache2/bin/suexec"
 -D DEFAULT_PIDLOG="logs/httpd.pid"
 -D DEFAULT_SCOREBOARD="logs/apache_runtime_status"
 -D DEFAULT_LOCKFILE="logs/accept.lock"
 -D DEFAULT_ERRORLOG="logs/error_log"
 -D AP_TYPES_CONFIG_FILE="conf/mime.types"
 -D SERVER_CONFIG_FILE="conf/httpd.conf"
```
