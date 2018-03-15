# pnds-windows
PowerDns windows版本编译。  pdns源码地址：https://github.com/PowerDNS/pdns
## 目的
获取Windows下可运行的PowerDns Server。
## 构建环境
>Windows10、Cygwin2.889（64bit）
## 依赖
  boost、openssl; gcc-g++、make、ragel、bison、flex、pip、virtualenv、libtool、automake、aotuconfig  
 注：所有依赖需在Cygwin上安装，其中除ragel和virtualenv，其余均可通过Cygwin安装程序直接安装。virtualenv可在Cygwin
 下通过 pip install virtualenv直接安装。ragel可下载源码，通过Cygwin编译后安装。ragel安装详情如下：
 ```
 1.ragel源码下载：http://www.colm.net/open-source/ragel/。
 2.解压后，在Cygwin下依次执行以下命令即可:  
   cd /ragel(/ragel为ragel解压后的根目录)
   ./configure
   make
   make install
 ```

## 编译脚本及源码修改说明
* 移除文档生成
```   
  /Makefile.am:1 -: SUBDIRS = ext modules codedocs docs  
  /Makefile.am:2 +: SUBDIRS = ext modules
```
* 所有动态库改为静态库
```
  ext/json11/Makefile.am:1 -: noinst_LTLIBRARIES = libjson11.la
  ext/json11/Makefile.am:1 +: noinst_LIBRARIES = libjson11.a
  ext/json11/Makefile.am:2 -: libjson11_la_SOURCES = json11.cpp json11.hpp
  ext/json11/Makefile.am:2 +: libjson11_a_SOURCES = json11.cpp json11.hpp
```
```
  etx/yahttp/yahttp/Makefile.am:1 -: noinst_LTLIBRARIES = libyahttp.la
  etx/yahttp/yahttp/Makefile.am:1 +: noinst_LIBRARIES = libyahttp.a
  etx/yahttp/yahttp/Makefile.am:3 -: libyahttp_la_SOURCES = \
  etx/yahttp/yahttp/Makefile.am:3 +: libyahttp_a_SOURCES = \
```
```
  modules/gmysqlbackend/Makefile.am:3 -: pkglib_LTLIBRARIES = libgmysqlbackend.la
  modules/gmysqlbackend/Makefile.am:3 +: pkglib_LIBRARIES = libgmysqlbackend.a
  modules/gmysqlbackend/Makefile.am:19 -: libgmysqlbackend_la_SOURCES = \
  modules/gmysqlbackend/Makefile.am:19 +: libgmysqlbackend_a_SOURCES = \
  modules/gmysqlbackend/Makefile.am:23 -: libgmysqlbackend_la_LDFLAGS = -module -avoid-version
  modules/gmysqlbackend/Makefile.am:24 -: libgmysqlbackend_la_LIBADD = $(MYSQL_LIBS)
  modules/gmysqlbackend/Makefile.am:23 +: libgmysqlbackend_a_LIBADD = /usr/lib/libmysqlclient.dll.a
```
```
  modules/pipebackend/Makefile.am:1 -: pkglib_LTLIBRARIES = libpipebackend.la
  modules/pipebackend/Makefile.am:1 +: pkglib_LIBRARIES = libpipebackend.a
  modules/pipebackend/Makefile.am:8 -: libpipebackend_la_SOURCES = \
  modules/pipebackend/Makefile.am:8 -: libpipebackend_a_SOURCES = \
  modules/pipebackend/Makefile.am:12 -: libpipebackend_la_LDFLAGS = -module -avoid-version
```
* configure.ac中增加 CPPFLAGS="-D_XOPEN_SOURCE=700"
```
  configure.ac:22 +: :${CPPFLAGS="-D_XOPEN_SOURCE=700"}
```
* 源码修改
```
  pdns/misc.cc:924 -: pkt->ipi_spec_dst = source->sin4.sin_addr;
```
```
屏蔽动态库的加载：
  pdns/dnsbackend.cc：107 -： load(entry->d_name);
  pdns/dnsbackend.cc：107 +： ;
  pdns/receiver.cc:513 -: loadModules();
```
```
  pdns/iputils.hh:81-82 源码如下：
    #ifdef __FreeBSD__
    #include <sys/endian.h>
  替换为如下内容：
    #if defined(__linux__) || defined(__CYGWIN__)
    /* Define necessary macros for the header to expose all fields. */
    #   define _BSD_SOURCE 
    #   define __USE_BSD
    #   define _DEFAULT_SOURCE
    #   include <endian.h>
    #   include <features.h>
    /* See http://linux.die.net/man/3/endian */
    #   if !defined(__GLIBC__) || !defined(__GLIBC_MINOR__) \
      || ((__GLIBC__ < 2) || ((__GLIBC__ == 2) && (__GLIBC_MINOR__ < 9))) 
    #       include <arpa/inet.h>
    #       if defined(__BYTE_ORDER) && (__BYTE_ORDER == __LITTLE_ENDIAN)
    #           define htobe16(x) htons(x)
    #           define htole16(x) (x)
    #           define be16toh(x) ntohs(x)
    #           define le16toh(x) (x)

    #           define htobe32(x) htonl(x)
    #           define htole32(x) (x)
    #           define be32toh(x) ntohl(x)
    #           define le32toh(x) (x)

    #           define htobe64(x) (((uint64_t)htonl(((uint32_t)(((uint64_t)(x)) >> 32)))) \
       | (((uint64_t)htonl(((uint32_t)(x)))) << 32))
    #           define htole64(x) (x)
    #           define be64toh(x) (((uint64_t)ntohl(((uint32_t)(((uint64_t)(x)) >> 32)))) \
       | (((uint64_t)ntohl(((uint32_t)(x)))) << 32))
    #           define le64toh(x) (x)
    #       elif defined(__BYTE_ORDER) && (__BYTE_ORDER == __BIG_ENDIAN)
    #           define htobe16(x) (x)
    #           define htole16(x) ((((((uint16_t)(x)) >> 8))|((((uint16_t)(x)) << 8)))
    #           define be16toh(x) (x)
    #           define le16toh(x) ((((((uint16_t)(x)) >> 8))|((((uint16_t)(x)) << 8)))

    #           define htobe32(x) (x)
    #           define htole32(x) (((uint32_t)htole16(((uint16_t)(((uint32_t)(x)) >> 16)))) \
       | (((uint32_t)htole16(((uint16_t)(x)))) << 16))
    #           define be32toh(x) (x)
    #           define le32toh(x) (((uint32_t)le16toh(((uint16_t)(((uint32_t)(x)) >> 16)))) \
       | (((uint32_t)le16toh(((uint16_t)(x)))) << 16))

    #           define htobe64(x) (x)
    #           define htole64(x) (((uint64_t)htole32(((uint32_t)(((uint64_t)(x)) >> 32)))) \
       | (((uint64_t)htole32(((uint32_t)(x)))) << 32))
    #           define be64toh(x) (x)
    #           define le64toh(x) (((uint64_t)le32toh(((uint32_t)(((uint64_t)(x)) >> 32)))) \
       | (((uint64_t)le32toh(((uint32_t)(x)))) << 32))
    #       else
    #           error Byte Order not supported or not defined.
    #       endif
    #   endif
  注：此处修改引自: https://gist.github.com/panzi/6856583
```
若需打包为不依赖Cygwin目录结构的可执行程序，还需做如下修改：
```
修改配置文件pdns.conf所在目录：（注：该目录下必须包含pdns.conf,这里仅简单指定为当前运行目录，仅作参考） 
  pdns/receiver.cc:453 -: string configname=::arg()["config-dir"]+"/"+s_programname+".conf";
  pdns/receiver.cc:453 +: string configname="./"+s_programname+".conf";
```
```
修改socket创建目录：（注：该目录必须有效,这里仅简单指定为当前运行目录，仅作参考） 
  pdns/dynlistener.cc:184-201 源代码如下：
  string socketname = ::arg()["socket-dir"];
    if (::arg()["socket-dir"].empty()) {
      if (::arg()["chroot"].empty())
        socketname = LOCALSTATEDIR;
    DynListener::DynListener(const string &progname)
    else if(errno!=EEXIST) {
      L<<Logger::Critical<<"Unable to create socket directory ("<< \
      socketname<<") and it does not exist yet"<<endl;
      exit(1);
    }
    替换为：string socketname("./");
```
## 编译带MySQL后端的pdns
```
$ ./bootstrap
$ ./configure --with-modules="mysql" --without-lua
$ make
$ make install
```
## 在MySQL配置
   详情见PowerDns官网: https://doc.powerdns.com/authoritative/backends/generic-mysql.html
* 创建 powerdns 数据库
```
  CREATE DATABASE powerdns;
```
* 创建相关表
```
  USE powerdns;
  CREATE TABLE domains (
  id                    INT AUTO_INCREMENT,
  name                  VARCHAR(255) NOT NULL,
  master                VARCHAR(128) DEFAULT NULL,
  last_check            INT DEFAULT NULL,
  type                  VARCHAR(6) NOT NULL,
  notified_serial       INT UNSIGNED DEFAULT NULL,
  account               VARCHAR(40) CHARACTER SET 'utf8' DEFAULT NULL,
  PRIMARY KEY (id)
) Engine=InnoDB CHARACTER SET 'latin1';
CREATE UNIQUE INDEX name_index ON domains(name);

CREATE TABLE records (
  id                    BIGINT AUTO_INCREMENT,
  domain_id             INT DEFAULT NULL,
  name                  VARCHAR(255) DEFAULT NULL,
  type                  VARCHAR(10) DEFAULT NULL,
  content               VARCHAR(64000) DEFAULT NULL,
  ttl                   INT DEFAULT NULL,
  prio                  INT DEFAULT NULL,
  change_date           INT DEFAULT NULL,
  disabled              TINYINT(1) DEFAULT 0,
  ordername             VARCHAR(255) BINARY DEFAULT NULL,
  auth                  TINYINT(1) DEFAULT 1,
  PRIMARY KEY (id)
) Engine=InnoDB CHARACTER SET 'latin1';
CREATE INDEX nametype_index ON records(name,type);
CREATE INDEX domain_id ON records(domain_id);
CREATE INDEX ordername ON records (ordername);

CREATE TABLE supermasters (
  ip                    VARCHAR(64) NOT NULL,
  nameserver            VARCHAR(255) NOT NULL,
  account               VARCHAR(40) CHARACTER SET 'utf8' NOT NULL,
  PRIMARY KEY (ip, nameserver)
) Engine=InnoDB CHARACTER SET 'latin1';

CREATE TABLE comments (
  id                    INT AUTO_INCREMENT,
  domain_id             INT NOT NULL,
  name                  VARCHAR(255) NOT NULL,
  type                  VARCHAR(10) NOT NULL,
  modified_at           INT NOT NULL,
  account               VARCHAR(40) CHARACTER SET 'utf8' DEFAULT NULL,
  comment               TEXT CHARACTER SET 'utf8' NOT NULL,
  PRIMARY KEY (id)
) Engine=InnoDB CHARACTER SET 'latin1';
CREATE INDEX comments_name_type_idx ON comments (name, type);
CREATE INDEX comments_order_idx ON comments (domain_id, modified_at);

CREATE TABLE domainmetadata (
  id                    INT AUTO_INCREMENT,
  domain_id             INT NOT NULL,
  kind                  VARCHAR(32),
  content               TEXT,
  PRIMARY KEY (id)
) Engine=InnoDB CHARACTER SET 'latin1';
CREATE INDEX domainmetadata_idx ON domainmetadata (domain_id, kind);

CREATE TABLE cryptokeys (
  id                    INT AUTO_INCREMENT,
  domain_id             INT NOT NULL,
  flags                 INT NOT NULL,
  active                BOOL,
  content               TEXT,
  PRIMARY KEY(id)
) Engine=InnoDB CHARACTER SET 'latin1';
CREATE INDEX domainidindex ON cryptokeys(domain_id);

CREATE TABLE tsigkeys (
  id                    INT AUTO_INCREMENT,
  name                  VARCHAR(255),
  algorithm             VARCHAR(50),
  secret                VARCHAR(255),
  PRIMARY KEY (id)
) Engine=InnoDB CHARACTER SET 'latin1';
CREATE UNIQUE INDEX namealgoindex ON tsigkeys(name, algorithm);
```
## pdns配置
在/usr/local/etc(或其他上文所述自定义) 目录下创建配置文件pdns.conf。配置详情见PowerDns官网 https://doc.powerdns.com/authoritative/index.html
示例如下：
```
  local-address=127.0.0.1
  slave=yes
  launch=gmysql
  gmysql-host=127.0.0.1
  gmysql-port=3306
  gmysql-dbname=powerdns
  gmysql-user=root
  gmysql-password=123456
  allow-dnsupdate-from=192.168.0.0/16,::1
  dnsupdate=yes
  api=yes
  api-key=123456
  webserver=yes
  webserver-address=192.168.0.114
  webserver-port=8081
  webserver-allow-from=0.0.0.0/0,::0
```
### 附：打包为独立可执行程序所需Cygwin相关转换库
Cygwin动态库目录：cygwin64\bin （cygwin64为Cygwin根目录）
Cygwin版本不同，所需dll可能不同，以下列出项，仅作参考：
* cygcrypto-1.0.0.dll，
* cyggcc_s-seh-1.dll，
* cygmysqlclient-18.dll，
* cygssl-1.0.0.dll，
* cygstdc++-6.dll，
* cygwin1.dll，
* cygz.dll
