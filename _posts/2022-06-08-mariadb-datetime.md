---
layout: posts
title: MariaDB 10.1.2 하위 버전의 소숫점이 포함된 datetime 형식에 주의하자
tags: [dms, cdc, mariadb]
sitemap :
  priority : 1.0
---

MariaDB 10.1.2 하위 버전에서는 소숫점을 가진 datetime 값을 저장할 때 독특한 형식을 사용한다. 이 독특한 형식이 데이터의 저장이나 조회 등의 일반적인 작업에서는 문제점으로 드러나지 않는다.
하지만 binlog를 사용해야 하는 일부 작업에서는 문제가 발생할 수 있다. 간단한 사례로 어떤 문제점이 있고 해결방안은 있는지 함께 알아보자.

---

## MariaDB의 소숫점이 포함된 datetime 형식 확인 ##
테스트를 위해 아래와 같이 docker compose를 구성한다.

```
.
├── docker-compose.yml
└── custom
    └── config-file.cnf
```

테스트를 위해 10.1.1 버전의 MariaDB를 사용한다.
```yaml
# docker-compose.yml

version: '3.1'

services:

  db:
    image: mariadb:10.1.1
    restart: always
    environment:
      MYSQL_USER: root
      MYSQL_ROOT_PASSWORD: 1234
    volumes:
      - ./custom:/etc/mysql/conf.d
```

binlog를 활성화하고 binlog_format을 row로 설정한다.
```yaml
# custom/config-file.cnf

[mysqld]
log_bin                 = /var/log/mysql/mariadb-bin
log_bin_index           = /var/log/mysql/mariadb-bin.index
default_storage_engine  = InnoDB
binlog_format           = row
```

테스트를 위한 docker 컨테이너를 실행한다.
```shell
$ docker-compose up -d
```

아래와 같이 PS 명령으를 통해 정상 구동 여부를 확인할 수 있다.
```shell
$ docker-compose ps
          Name                     Command           State    Ports
---------------------------------------------------------------------
mariadb-binlog-test_db_1   /docker-entrypoint.sh     Up      3306/tcp
                           mysqld
```

docker 컨테이너에 접근한다. MariaDB 접근 패스워드는 docker-compose.yml 파일에 설정한 1234를 사용한다.
```shell
$ docker exec -it mariadb-binlog-test_db_1 /bin/bash
$ mysql -u root -p
```

이제 테스트를 위한 database와 테이블을 구성하고 샘플 데이터를 저장한다.
```sql
create database test;
use test;

create table test_ad_md_pick (id int auto_increment primary key, created_at datetime(6) null, start_time datetime(6) null);

insert into test_ad_md_pick(start_time, created_at) values('2022-06-08 19:56:00', now());
```

MariaDB 접속을 종료하고 datetime 형식이 binlog에 어떤 형태로 기록되는지 확인한다. /var/log/mysql 경로에 가장 최근 binlog 파일을 사용한다.
```shell
$ ls -al /var/log/mysql
total 892
drwxr-s--- 1 mysql adm    4096 Jun  8 11:02 .
drwxr-xr-x 1 root  root   4096 Nov 25  2014 ..
-rw-rw---- 1 mysql adm   24976 Jun  8 11:02 mariadb-bin.000006
-rw-rw---- 1 mysql adm  858610 Jun  8 11:02 mariadb-bin.000007
-rw-rw---- 1 mysql adm    2030 Jun  8 11:06 mariadb-bin.000008
-rw-rw---- 1 mysql adm     102 Jun  8 11:02 mariadb-bin.index

$ mysqlbinlog -v --base64-output=DECODE-ROW /var/log/mysql/mariadb-bin.000008 > binlog.sql
```

binlog.sql 파일을 열어 INSERT 문을 확인하면 아래와 같이 start_time, created_at 컬럼값에 일반적인 datetime 형식이 사용되지 않는 것을 확인할 수 있다.
```sql
### INSERT INTO `test`.`test_ad_md_pick`
### SET
###   @1=1
###   @2=3297183-15-52 04:96:65
###   @3=5091739-47-54 89:63:85
```

## CDC 작업이 필요한 경우 해당 툴에서 MariaDB datetime 형식을 지원하는가? ##
일반적으로 mysql 계열 DB의 CDC (Change Data Capture) 작업을 위해 binlog를 참고한다. 이러한 경우 해당 툴에서 MariaDB의 datetime 형식을 지원하지 않을 수 있다

하나의 예시로 AWS DMS 서비스(현시점의 최신 버전인 DMS engine 3.4.6 버전)에서 해당 버전의 MariaDB를 소스로 사용하는 경우 CDC 작업에서 문제가 발생한다. 소스의 binlog를 그대로 읽어 DML 문을 만들어내는 DMS 엔진에서 해당 datetime 형식을 변환하지 않고 그대로 사용하기 때문이다. 이 경우 소스 DB의 컬럼 타입에서 소숫점을 제거하지 않는 이상 binlog에 찍히는 datetime 형식이 변하지 않기 때문에 원하는 형태로 사용하는 것이 불가능하다.

## --mysql56-temporal-format 시스템 변수 ##
MariaDB는 10.1.2 버전부터 Mysql5.6의 datetime 형식을 사용할 수 있도록 시스템 변수를 제공한다. 해당 옵션을 활성화할 경우 binlog에 기록되는 datetime 형식을 확인해보자.

기존 사용했던 docker compose에서 MariaDB 버전과 --mysql56-temporal-format 기능 활성화만 부분적으로 변경한다.
```yaml
# docker-compose.yml

version: '3.1'

services:

  db:
    image: mariadb:10.1.2
    restart: always
    environment:
      MYSQL_USER: root
      MYSQL_ROOT_PASSWORD: 1234
    volumes:
      - ./custom:/etc/mysql/conf.d
```

```yaml
# custom/config-file.cnf

[mysqld]
log_bin                 = /var/log/mysql/mariadb-bin
log_bin_index           = /var/log/mysql/mariadb-bin.index
default_storage_engine  = InnoDB
binlog_format           = row
mysql56_temporal_format=ON
```

다시 한 번 mysql에 접근하여 샘플 데이터를 위한 DDL과 DML을 실행한 후 binlog를 조회해본다. 같은 타입의 컬럼이지만 binlog의 결과값이 일반적인 datetime 형식으로 기록되는 것을 확인할 수 있다.
```sql
### INSERT INTO `test`.`test_ad_md_pick`
### SET
###   @1=1
###   @2='2022-06-08 11:42:09.000000'
###   @3='2022-06-08 19:56:00.000000'
```

안타까운 점은 해당 기능을 사용하려면 MariaDB 버전을 올려야한다는 점이다. 업그레이드가 어려운 상황에서는 해당 옵션을 사용할 수 없다. MariaDB 10.1.2 하위 버전의 datetime 형식을 지원하는 툴이나 서비스를 찾아 처리하는 수 밖에 없다.

## 정리 ##
MariaDB 10.1.1 버전은 꽤 오래된 버전이다. 그럼에도 여러가지 사정으로 오래된 버전을 업그레이드하지 못하고 운영할 수 있는 상황을 흔히 마주하게 된다. 버전 업그레이드의 시기를 놓쳐 방치하게 된다면 기능을 확장하려는 시도에 걸림돌이 되며 결국은 해결하기 위해 많은 리소스를 투입하게 된다. 이번 이슈는 시기적절하게 버전 업그레이드를 진행하고 레거시를 제거하는 활동이 반드시 필요하다는 것을 한번 더 느끼며 글을 마무리한다.
