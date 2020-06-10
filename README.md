# Redmine-Starter
레드마인 설치 방법 정리

## 레드마인(Redmine) 설치 정보
설치 OS : Ubuntu 20.04  
사용 DB : mariaDB 10.3.22  
레드마인 : 4.0.6-2  
루비 : ruby 2.7.0p0  

## mariaDB 설치
설치 : `sudo apt-get install -y mariadb-server`  
서비스 추가 : 
  ```
  $ sudo systemctl enable mariadb
  $ sudo mysql -u root -p
  > GRANT ALL PRIVILEGES on *.* to 'root'@'localhost' IDENTIFIED BY '패스워드입력';
  > FLUSH PRIVILEGES;
  > exit
  $ mysql_secure_installation 
  모두 root 계정 패스워드 변경만 n / 나머지 모두 Enter
  sudo vi /etc/mysql/debian.cnf 
  user = root 아래 password = 에 패스워드 넣기(client 및 mysql_upgrade 두개 다)
  ```
### [옵션] mariaDB 데이터 저장 경로 변경하기
* rsync 설치(우분투에 경우 왠만하면 있음) : `sudo apt-get install -y rsync`
* 데이터 디렉터리 위치 확인 : 
```
$ mysql -u root -p
> select @@datadir;
> exit;
```
* db 프로세스 종료 : `sudo systemctl stop mariadb`
* 위에서 확인한 데이터 디렉터리를 신규 경로로 복사
  - 예시) 기존 데이터 디렉터리 : /var/lib/mysql
  - 신규 디렉터리 : /data
  - `rsync -av /var/lib/mysql /data`
* mariaDB 설정 변경
  - `sudo vi /etc/mysql/mariadb.conf.d/50-server.cnf`
  - [mysqld] 아래의 datadir= 부분을 /data/mysql 로 변경 후 :wq 로 저장
* db 프로세스 시작 : `sudo systemctl start mariadb`
* 데이터 디렉터리 위치 재확인:
```
$ mysql -u root -p
> select @@datadir;
> exit;
```
* 참조사이트
  - [데이터 저장 디렉터리 복사하기](https://growingsaja.tistory.com/370)
  - [데이터 저장 경로 변경하기](https://serverfault.com/questions/914164/how-to-change-datadir-for-mariadb)
## 아파치(httpd) 설치
설치 : `sudo apt-get install -y apache2`    
구동 : `sudo systemctl start apache2`    
서비스 등록 : `sudo systemctl enable apache2`    

## Redmine 의존성 라이브러리 설치
`sudo apt-get install -y apt-transport-https gnupg build-essential ruby-dev pkg-config zlib1g-dev libmagickwand-dev`  
`sudo apt-get install -y redmine redmine-mysql`  

### [에러 대처] ERROR 1045 (28000): Access denied for user 'root'@'localhost' (using password: NO).
해당 오류 나올 시에 debian.cnf 변경
```
$ sudo vi /etc/mysql/debian.cnf

# Automatically generated for Debian scripts. DO NOT TOUCH!
[client]
host     = localhost
user     = root
password = 비밀번호 적기.
socket   = /var/run/mysqld/mysqld.sock
[mysql_upgrade]
host     = localhost
user     = root
password = 비밀번호 적기.
socket   = /var/run/mysqld/mysqld.sock
```
* 레드마인 데이터베이스 설정 변경
```
$ cd /usr/share/redmine/config
$ sudo vi database.yml

production:
  adapter: mysql2
  database: redmine_default
  host: localhost
  port: 3306
  username: redmine/instance
  password: 비밀번호 적으면 됨.
  encoding: utf8

:wq
```

## passenger 설치
기존 설치된 아파치 모듈 제거 : `sudo apt-get remove libapache2-mod-passenger`  
bundler 설치 : `sudo gem install bundler`    
passenger 설치 : `sudo gem install passenger`  

## 아파치 연동 모듈 설치
`sudo apt-get install -y libcurl4-openssl-dev libssl-dev zlib1g-dev libapr1-dev libaprutil1-dev apache2-dev`  
`sudo passenger-install-apache2-module`  
(계속 엔터 누르다가 LoadModule passenger_module… 이 나오면 해당 내용 메모장에 복붙 후 터미널 새로 켜서 아래 <b>아파치 설정 변경</b>작업 시작)

## 아파치 설정 변경

### passenger.load 수정
파일 수정: `sudo vi /etc/apache2/mods-available/passenger.load`  
기존의 내용을 다음의 내용으로 대체(예시): `LoadModule passenger_module /var/lib/gems/2.7.0/gems/passenger-6.0.5/buildout/apache2/mod_passenger.so`  

### passenger.conf 수정
파일 수정: `sudo vi /etc/apache2/mods-available/passenger.conf`  
기존의 내용을 다음의 내용으로 대체(예시): 
```
<IfModule mod_passenger.c>
  PassengerRoot /var/lib/gems/2.7.0/gems/passenger-6.0.5
  PassengerDefaultRuby /usr/bin/ruby2.7
  PassengerDefaultUser www-data
</IfModule>
```
* <b>주의</b> : <IfModule mod_passenger.c> 안에 <b>PassengerDefaultUser www-data</b>를 넣어주어야 passenger가 권한 오류없이 redmine 페이지를 보여줄 수 있다. 그러므로 passenger 설정 시에 복사해둔 것에 <b>PassengerDefaultUser www-data</b>가 없었어도 넣어주어야한다.

### 000-default.conf 수정
파일 수정: `sudo vi /etc/apache2/sites-available/000-default.conf`  
<VirtualHost> 아래 다음 항목 추가

```
<Directory /var/www/html/redmine>
     PassengerEnabled on
     RailsBaseURI /redmine
     PassengerAppRoot /usr/share/redmine
</Directory>
```

### 레드마인 연결을 위한 심볼릭 링크 생성
`sudo ln -s /usr/share/redmine/public /var/www/html/redmine`  

### 아파치 재가동
`sudo a2enmod passenger`  
`sudo systemctl restart apache2`  

### apache2 로그 위치 
`/var/log/apache2`

### 참조사이트
* centOS에 레드마인 설치 : https://goddaehee.tistory.com/78
* 우분투 아파치 설치 및 동작 : https://cornswrold.tistory.com/159
* 우분투에 레드마인 설치 : https://techexpert.tips/ko/%EB%A0%88%EB%93%9C-%EB%A7%88%EC%9D%B8/%EB%A0%88%EB%93%9C-%EB%A7%88%EC%9D%B8-%EC%9A%B0%EB%B6%84%ED%88%AC-%EB%A6%AC%EB%88%85%EC%8A%A4%EC%97%90-%EC%84%A4%EC%B9%98/
* 아파치 설정값 변경 오류시 : https://serverfault.com/questions/895581/apache-passenger-resolve-symlinks-stopped-working-invalid-command
