# MySQL 8.x 설치

이 문서는 MySQL 8.x 설치에 대해서 다룹니다. 대상 운영체제는 리눅스(Linux) 입니다.

## 컴파일 설치 vs 바이너리 설치

많은 사람들이 컴파일 설치를 하게되면 좀 더 성능이 좋을 것이라고 생각하지만 바이너리 설치가 훨씬 성능이 좋습니다.
컴파일 설치를 하려면 먼저 어떤 기능만을 위해, 혹은 바어너리 설치로는 활성화가 안되는 기능 때문이라는 이유가 있어야 합니다.
MySQL 에서 공식적으로 배포하는 바이너리가 훨씬 성능이 좋기 때문에 대부분 이것을 사용하는 것이 좋습니다.

## Ubuntu 배포판

배포판 마다 컴파일 설치를 위한 의존성 패키지명이 다릅니다. 또, 같은 배포판이라고 하더라도 버전마다 조금씩 다를 수 있습니다.
Ubuntu 배포판 설치에서는 20.04 LTS 배포판을 대상으로 진행 했습니다.

### 컴파일러 및 의존성 라이브러리 설치

컴파일을 하기 위한 라이브러리와 명령어들을 설치해줘야 합니다. 우분투는 이러한 패키지들을 **base-devel** 패키지로 묶어 놨습니다.

```console

sudo apt update
sudo apt install build-essential

```

gcc 컴파일러를 비롯한 관련 라이브러리들을 한꺼번에 설치를 해줍니다. 하지만 MySQL 8.x 를 컴파일하기 위해서는 추가적으로 몇가지 패키지를 설치해줘야 합니다.

```console
sudo apt install cmake bison flex
```

또, MySQL 8.x 의 기능을 이용하기 위해서 다음과 같이 의존성 패키지를 설치해 줍니다.

```console
sudo apt install libncurses-dev libssl-dev libudev-dev pkg-config libreadline6-dev libossp-uuid-dev uuid libaio-dev libnuma-dev

```

### 설정하기

MySQL 8.x 는 cmake 를 이용하기 때문에 다음과 같이 cmake 명령어를 이용해서 설정을 해줍니다.

```console

cmake -DCMAKE_INSTALL_PREFIX=/opt/mysql-8.0.31 \
-DSYSTEMD_PID_DIR=/opt/mysql-8.0.31/run \
-DDEFAULT_CHARSET=utf8mb4 \
-DDEFAULT_COLLATION=utf8mb4_general_ci \
-DENABLED_LOCAL_INFILE=1 \
-DWITH_EXTRA_CHARSETS=all \
-DWITH_NUMA=ON \
-DWITH_EDITLINE=bundled \
-DWITH_ICU=bundled \
-DWITH_LIBEVENT=bundled \
-DWITH_LZ4=bundled \
-DWITH_PROTOBUF=bundled \
-DWITH_SSL=yes \
-DWITH_ZSTD=bundled \
-DWITH_SYSTEMD=ON \
-WITH_ZLIB=bundled \
-DWITH_BOOST=/usr/src/mysql-8.0.31/boost/boost_1_77_0 \
-DWITH_SSL=system  \
-DFORCE_INSOURCE_BUILD=ON

```

### 컴파일 및 설치

다음과 같이 컴파일과 설치를 해줍니다.

```console
make
make install
```

## 후속 작업

MySQL 을 설치했다면 후속작업이 필요합니다.

설치 디렉토리를 /opt/mysql-8.0.31 으로 했는데, 이를 간결하기 mysql 로 하기 위해서 다음과 같이 심볼릭 링크를 걸어 줍니다.

```console
cd /opt
ln -s mysql-8.0.31 mysql
```

mysql 라이브러리를 인식시키기 위해서 다음과 같이 해줍니다.

```console
vim /etc/ld.so.conf.d/mysql.conf

# mysql-8.0.31 library
/opt/mysql/lib

ldconfig
```

man 페이지 패스를 /etc/profile 에 설정해 줍니다.

```console
vim /etc/profile

export MANPATH="$(manpath -g):/opt/mysql/man"
```

mysql 운영을 위한 디렉토리를 다음과 같이 생성해 줍니다.

```console
mkdir /opt/mysql/run
mkdir /opt/mysql/logs
```

### 계정 만들기

mysql 를 root 로 운영하는 것은 보안상 좋지 않습니다. mysql 를 운영하기 위한 계정을 만들어 줍니다. 대부분 mysql 계정이름으로 만듭니다.

```console
/usr/sbin/groupadd -g 47 -o -r mysql
/usr/sbin/useradd -M -g mysql -o -r -d /opt/mysql/logs -s /usr/sbin/nologin -c "MySQL Server" -u 47 mysql
```

그룹을 반드시 사용자와 동일하게 생성할 필요는 없습니다. 또, 반드시 생성할 필요도 없습니다. 이미 적절한 그룹이 있다면 mysql 계정만 생성하고 그룹에 추가하면 됩니다.

### my.cnf

my.cnf 파일을 /etc 디렉토리에 생성한 다음, 적절한 설정들을 해줍니다.

## MySQL 초기화

MySQL 을 시작하기 전에 초기화 작업을 해줘야 합니다. 초기화 작업은 시스템 데이터베이스, 테이블, 권한등을 만들어주는 과정입니다.

```console
/opt/mysql/bin/mysqld --defaults-file=/etc/my.cnf --initialize --user=mysql
```

실행이 끝나면 mysqld_error.log 파일을 살펴봐야 합니다. 이 파일에 관련 진행 내용이 기록되며 오류가 발생한다면 역시 이 파일에 기록 됩니다.

## Systemd 파일 등록

컴파일 설정에서 systemd 를 사용으로 했기 때문에 systemd 를 사용해 mysql 서비스를 제어해야 합니다.

mysql 관련 된 systemd 파일은 컴파일한 소스 디렉토리에 scripts 디렉토리에 있습니다. 이것을 다음과 같이 디렉토리에 복사해 줍니다.

```console
cp /usr/src/mysql-8.0.31/scripts/mysqld.service /etc/systemd/system
```

복사가 정상적으로 됐다면 이제 다음과 같이 systemctl 명령어로 서비스를 활성화 및 시작해 줍니다.

```console
systemctl enable mysqld.service
systemctl start mysqld
```