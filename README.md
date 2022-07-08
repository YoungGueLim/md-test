# 프로젝트 - IaC를 이용해 AWS EC2 인스턴스에 Wordpress CMS 배포하기

## 0. 시나리오 - 목표

- Ansible을 이용해 Wordpress CMS를 설치하는 Playbook을 작성한다.
- Packer로 Ansible Playbook을 실행하는 HCL을 작성하고 AWS AMI 이미지를 생성한다.
- Terraform으로 Packer를 통해 생성한 AWS AMI 이미지를 기반으로 하는 AWS EC2 인스턴스와 관련 리소스를 생성한다.

## 1. Wordpress CMS 설치 및 구성

Ansible Playbook을 작성하기 전 Wordpress CMS를 설치하는 방법을 정리하고, 이를 바탕으로  Ansible Playbook에 사용할 역할 및 작업을 생각해본다.

### 1) 의존성 패키지 설치

Apache, PHP, MySQL 관련 패키지 설치

```console
$ sudo apt update
$ sudo apt install apache2 mysql-server php php-mysql libapache2-mod-php python3-pymysql
```

### 2) Wordpress 설치

Wordpress 설치를 위한 `/srv/www` 디렉토리 생성

```console
$ sudo mkdir -p /srv/www
```

`wordpress.org`에서 Wordpress 소스 다운로드 후 설치

```console
$ wget https://wordpress.org/wordpress-6.0.tar.gz
$ tar xf wordpress-6.0.tar.gz -C /srv/www
```

디렉토리 소유자는 Apache 사용자인 `www-data`로 설정

```console
$ sudo chown -R www-data /srv/www
```

### 3) Wordpress를 위한 Apache 구성

Wordpress 애플리케이션을 위한 Apache 가상 호스트 구성

`/etc/apache2/sites-available/wordpress.conf`

```apache
<VirtualHost *:80>
    DocumentRoot /srv/www/wordpress
    <Directory /srv/www/wordpress>
        Options FollowSymLinks
        AllowOverride Limit Options FileInfo
        DirectoryIndex index.php
        Require all granted
    </Directory>
    <Directory /srv/www/wordpress/wp-content>
        Options FollowSymLinks
        Require all granted
    </Directory>
</VirtualHost>
```

가상호스트 활성화

```console
$ sudo a2ensite wordpress
```

Apache Rewrite 모듈 활성화

```console
$ sudo a2enmod rewrite
```

테스트 페이지 설정 비활성화

```console
$ sudo a2dissite 000-default
```

Apache 서비스 리로드

```console
$ sudo systemctl reload apache2
```

### 4) 데이터베이스 구성

MySQL 데이터베이스 접속

```console
$ sudo mysql -u root
```

`wordpress` 데이터베이스 생성

```console
mysql> create database wordpress;
```

`wordpress` 사용자 및 패스워드 설정

```console
mysql> create user wordpress@localhost identified by 'P@ssw0rd';
```

`wordpress` 사용자에게 `wordpress` 데이터베이스 권한 부여

```console
mysql> grant all on wordpress.* to wordpress@localhost;
```

권한 변경사항 적용

```console
mysql> flush privileges;
```

### 5) Wordpress와 데이터베이스 연결 구성

Wordpress 설정파일 구성

`/srv/www/wordpress/wp-config-sample.php` 파일을 `wp-config-php`로 복사

```console
$ sudo -u www-data cp wp-config-sample.php wp-config.php
```

데이터베이스 이름, 데이터베이스 사용자, 데이터베이스 패스워드 변경

`/srv/www/wordpress/wp-config.php`

```console
$ sudo -u www-data vi wp-config.php 
```
```
define( 'DB_NAME', 'wordpress' );

define( 'DB_USER', 'wordpress' );

define( 'DB_PASSWORD', 'P@ssw0rd' );
```

## 2. Wordpress를 배포하기 위한 Ansible Playbook 작성

### 1) 요구사항

- Main Ansible Playbook
- Ansible 역할
  - wordpress 역할
    - 작업: 패키지 설치, 구성파일 복사, 모듈 및 사이트 활성화, 서비스 시작
    - 핸들러: 모듈 및 사이트 활성화, 서비스 재시작
    - 변수: 데이터베이스명, 데이터베이스 사용자, 데이터베이스 패스워드(암호화) 등
    - 파일: 구성파일
  - mysql 역할
    - 작업: 패키지 설치, 데이터베이스 생성, 사용자 생성, 사용자 권한 부여, 구성파일 복사(위임 또는 wordpress에서 작업 가능), 서비스 시작
    - 핸들러: 서비스 재시작
    - 변수: 데이터베이스명, 데이터베이스 사용자, 데이터베이스 패스워드(암호화) 등
    - 파일: 구성파일

### 2) Ansible Playbook

`ansible-wordpress-deploy/group_vars/all/wordpress.yaml`
```yaml
wordpress_directory: /srv/www
wordpress_version: "6.0"
wordpress_source_file: "wordpress-{{ wordpress_version }}.tar.gz"
wordpress_source_url: "https://wordpress.org/{{ wordpress_source_file }}"
```

`ansible-wordpress-deploy/group_vars/all/database.yaml`
```yaml
database_name: wordpress
database_user: wordpress
database_password: "P@ssw0rd"
database_unix_socket: /run/mysqld/mysqld.sock
```

`ansible-wordpress-deploy/site.yaml`
```yaml
- hosts: default
  gather_facts: no
  become: yes

  roles:
  - wordpress
  - mysql

  post_tasks:
  - name: Access Check
    uri:
      url: "http://{{ inventory_hostname }}"
      method: GET
      status_code: 
      - 200
      - 302
    register: uri_result

  - name: Status Code
    debug:
      var: uri_result.status
```

`ansible-wordpress-deploy/XXX`
```yaml
```

`ansible-wordpress-deploy/XXX`
```yaml
```


## 3. Pakcer와 Ansible Playbook을 위한 AWS AMI 생성

### 1) 요구사항

- Wordpress를 배포하기 위한 Ansible Playbook
- AWS AMI 이미지를 위한 Packer HCL 코드

### 2) Packer HCL

`packer-wordpress-image/wordpress-ubuntu.pkr.hcl`
```terraform
<Paste of Your Code>
```

## 4. Terraform과 Packer로 생성한 AWS AMI 이미지로 AWS EC2 인스턴스 배포

### 1) 요구사항

- Packer로 생성한 AWS AMI 이미지
- AWS EC2 인스턴스 배포를 위한 Terraform HCL 코드
  - VPC, Subnet 등 네트워크 리소스
  - EC2 인스턴스, SSH 키, 보안 그룹 등 리소스

### 2) Terraform HCL

`terraform-wordpress-vm/XXX`
```terraform
<Paste of Your Code> 
```
