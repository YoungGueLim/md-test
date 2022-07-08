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

`ansible-wordpress-deploy/roles/mysql/defaults/main.yaml`
```
---
# defaults file for roles/mysql

mysql_packages: mysql-server, mysql-client, python3-pymysql
```

`ansible-wordpress-deploy/roles/mysql/handlers/main.yaml`

```
---
# handlers file for roles/mysql
```

`ansible-wordpress-deploy/roles/mysql/meta/main.yaml`
```
galaxy_info:
  author: your name
  description: your role description
  company: your company (optional)

  # If the issue tracker for your role is not on github, uncomment the
  # next line and provide a value
  # issue_tracker_url: http://example.com/issue/tracker

  # Choose a valid license ID from https://spdx.org - some suggested licenses:
  # - BSD-3-Clause (default)
  # - MIT
  # - GPL-2.0-or-later
  # - GPL-3.0-only
  # - Apache-2.0
  # - CC-BY-4.0
  license: license (GPL-2.0-or-later, MIT, etc)

  min_ansible_version: 2.1

  # If this a Container Enabled role, provide the minimum Ansible Container version.
  # min_ansible_container_version:

  #
  # Provide a list of supported platforms, and for each platform a list of versions.
  # If you don't wish to enumerate all versions for a particular platform, use 'all'.
  # To view available platforms and versions (or releases), visit:
  # https://galaxy.ansible.com/api/v1/platforms/
  #
  # platforms:
  # - name: Fedora
  #   versions:
  #   - all
  #   - 25
  # - name: SomePlatform
  #   versions:
  #   - all
  #   - 1.0
  #   - 7
  #   - 99.99

  galaxy_tags: []
    # List tags for your role here, one per line. A tag is a keyword that describes
    # and categorizes the role. Users find roles by searching for tags. Be sure to
    # remove the '[]' above, if you add tags to this list.
    #
    # NOTE: A tag is limited to a single word comprised of alphanumeric characters.
    #       Maximum 20 tags per role.

dependencies: []
  # List your role dependencies here, one per line. Be sure to remove the '[]' above,
  # if you add dependencies to this list.

```


`ansible-wordpress-deploy/roles/mysql/tasks/main.yaml`
```
---
# tasks file for roles/mysql

- name: Install Packages for MySQL
  apt:
    name: "{{ mysql_packages }}"
    state: latest
    update_cache: yes

- name: Start MySQL Service
  service:
    name: mysql
    enabled: yes
    state: started

- name: Create wordpress Database
  mysql_db:
    name: "{{ database_name }}"
    state: present
    login_unix_socket: "{{ database_unix_socket }}"

- name: Create Database User
  mysql_user:
    name: "{{ database_user }}"
    password: "{{ database_password }}"
    priv: "{{ database_name }}.*:ALL,GRANT"
    state: present
    login_unix_socket: "{{ database_unix_socket }}"

- name: Copy Wordpress Configuration Files for MySQL
  template:
    src: wp-config.php.j2
    dest: "{{ wordpress_directory }}/wordpress"
    owner: www-data
    group: www-data

```



`ansible-wordpress-deploy/roles/mysql/templates/main.yaml`
```
<?php
/**
 * The base configuration for WordPress
 *
 * The wp-config.php creation script uses this file during the installation.
 * You don't have to use the web site, you can copy this file to "wp-config.php"
 * and fill in the values.
 *
 * This file contains the following configurations:
 *
 * * Database settings
 * * Secret keys
 * * Database table prefix
 * * ABSPATH
 *
 * @link https://wordpress.org/support/article/editing-wp-config-php/
 *
 * @package WordPress
 */

// ** Database settings - You can get this info from your web host ** //
/** The name of the database for WordPress */
define( 'DB_NAME', '{{ database_name }}' );

/** Database username */
define( 'DB_USER', '{{ database_user }}' );

/** Database password */
define( 'DB_PASSWORD', '{{ database_password }}' );

/** Database hostname */
define( 'DB_HOST', 'localhost' );

/** Database charset to use in creating database tables. */
define( 'DB_CHARSET', 'utf8' );

/** The database collate type. Don't change this if in doubt. */
define( 'DB_COLLATE', '' );

/**#@+
 * Authentication unique keys and salts.
 *
 * Change these to different unique phrases! You can generate these using
 * the {@link https://api.wordpress.org/secret-key/1.1/salt/ WordPress.org secret-key service}.
 *
 * You can change these at any point in time to invalidate all existing cookies.
 * This will force all users to have to log in again.
 *
 * @since 2.6.0
 */
define( 'AUTH_KEY',         'put your unique phrase here' );
define( 'SECURE_AUTH_KEY',  'put your unique phrase here' );
define( 'LOGGED_IN_KEY',    'put your unique phrase here' );
define( 'NONCE_KEY',        'put your unique phrase here' );
define( 'AUTH_SALT',        'put your unique phrase here' );
define( 'SECURE_AUTH_SALT', 'put your unique phrase here' );
define( 'LOGGED_IN_SALT',   'put your unique phrase here' );
define( 'NONCE_SALT',       'put your unique phrase here' );

/**#@-*/

/**
 * WordPress database table prefix.
 *
 * You can have multiple installations in one database if you give each
 * a unique prefix. Only numbers, letters, and underscores please!
 */
$table_prefix = 'wp_';

/**
 * For developers: WordPress debugging mode.
 *
 * Change this to true to enable the display of notices during development.
 * It is strongly recommended that plugin and theme developers use WP_DEBUG
 * in their development environments.
 *
 * For information on other constants that can be used for debugging,
 * visit the documentation.
 *
 * @link https://wordpress.org/support/article/debugging-in-wordpress/
 */
define( 'WP_DEBUG', false );

/* Add any custom values between this line and the "stop editing" line. */



/* That's all, stop editing! Happy publishing. */

/** Absolute path to the WordPress directory. */
if ( ! defined( 'ABSPATH' ) ) {
	define( 'ABSPATH', __DIR__ . '/' );
}

/** Sets up WordPress vars and included files. */
require_once ABSPATH . 'wp-settings.php';

```


`ansible-wordpress-deploy/roles/mysql/.travis.yml`
```
---
language: python
python: "2.7"

# Use the new container infrastructure
sudo: false

# Install ansible
addons:
  apt:
    packages:
    - python-pip

install:
  # Install ansible
  - pip install ansible

  # Check ansible version
  - ansible --version

  # Create ansible.cfg with correct roles_path
  - printf '[defaults]\nroles_path=../' >ansible.cfg

script:
  # Basic role syntax check
  - ansible-playbook tests/test.yml -i tests/inventory --syntax-check

notifications:
  webhooks: https://galaxy.ansible.com/api/v1/notifications/


```

`ansible-wordpress-deploy/roles/wordpress/defaults/main.yaml`
```
---
# defaults file for roles/wordpress

apache_packages: apache2, php, php-mysql, libapache2-mod-php
```


`ansible-wordpress-deploy/roles/wordpress/handlers/main.yaml`
```
---
# handlers file for roles/wordpress
- name: Restart Apache2 Service
  service:
    name: apache2
    state: restarted
```


`ansible-wordpress-deploy/roles/wordpress/meta/main.yaml`
```
galaxy_info:
  author: your name
  description: your role description
  company: your company (optional)

  # If the issue tracker for your role is not on github, uncomment the
  # next line and provide a value
  # issue_tracker_url: http://example.com/issue/tracker

  # Choose a valid license ID from https://spdx.org - some suggested licenses:
  # - BSD-3-Clause (default)
  # - MIT
  # - GPL-2.0-or-later
  # - GPL-3.0-only
  # - Apache-2.0
  # - CC-BY-4.0
  license: license (GPL-2.0-or-later, MIT, etc)

  min_ansible_version: 2.1

  # If this a Container Enabled role, provide the minimum Ansible Container version.
  # min_ansible_container_version:

  #
  # Provide a list of supported platforms, and for each platform a list of versions.
  # If you don't wish to enumerate all versions for a particular platform, use 'all'.
  # To view available platforms and versions (or releases), visit:
  # https://galaxy.ansible.com/api/v1/platforms/
  #
  # platforms:
  # - name: Fedora
  #   versions:
  #   - all
  #   - 25
  # - name: SomePlatform
  #   versions:
  #   - all
  #   - 1.0
  #   - 7
  #   - 99.99

  galaxy_tags: []
    # List tags for your role here, one per line. A tag is a keyword that describes
    # and categorizes the role. Users find roles by searching for tags. Be sure to
    # remove the '[]' above, if you add tags to this list.
    #
    # NOTE: A tag is limited to a single word comprised of alphanumeric characters.
    #       Maximum 20 tags per role.

dependencies: []
  # List your role dependencies here, one per line. Be sure to remove the '[]' above,
  # if you add dependencies to this list.
```


`ansible-wordpress-deploy/roles/wordpress/tasks/main.yaml`
```
---
# tasks file for roles/wordpress

- name: Install Packages for Apache2
  apt:
    name: "{{ apache_packages }}"
    state: latest
    update_cache: yes

- name: Create Wordpress Directory
  file:
    path: "{{ wordpress_directory }}"
    owner: www-data
    group: www-data
    state: directory

- name: Download Wordpress Source
  get_url:
    url: "{{ wordpress_source_url }}"
    dest: /tmp

- name: Unarchive Wordpress Tar
  unarchive:
    src: "/tmp/{{ wordpress_source_file }}"
    remote_src: yes
    dest: /srv/www
    owner: www-data
    group: www-data

- name: Copy Wordpress Configuration for Apache2
  template:
    src: wordpress.conf.j2
    dest: /etc/apache2/sites-available/wordpress.conf

- name: Enable Wordpress Site for Apache2
  file:
    src: /etc/apache2/sites-available/wordpress.conf
    path: /etc/apache2/sites-enabled/wordpress.conf
    state: link
  notify:
  - Restart Apache2 Service

- name: Disable Default Site for Apache2
  file:
    path: /etc/apache2/sites-enabled/000-default.conf
    state: absent

- name: Enable Rewrite Module for Apache2
  apache2_module:
    name: rewrite
    state: present
  notify:
  - Restart Apache2 Service

- name: Start/Enable Apache2 Service
  service:
    name: apache2
    enabled: yes
    state: started
```


`ansible-wordpress-deploy/roles/wordpress/templates/wordpress.conf.j2`
```
{{ ansible_managed | comment }}

<VirtualHost *:80>
    DocumentRoot "{{ wordpress_directory }}/wordpress"
    <Directory "{{ wordpress_directory }}/wordpress">
        Options FollowSymLinks
        AllowOverride Limit Options FileInfo
        DirectoryIndex index.php
        Require all granted
    </Directory>
    <Directory "{{ wordpress_directory }}/wordpress/wp-content">
        Options FollowSymLinks
        Require all granted
    </Directory>
</VirtualHost>
```


`ansible-wordpress-deploy/roles/wordpress/.travis.yml`
```
---
language: python
python: "2.7"

# Use the new container infrastructure
sudo: false

# Install ansible
addons:
  apt:
    packages:
    - python-pip

install:
  # Install ansible
  - pip install ansible

  # Check ansible version
  - ansible --version

  # Create ansible.cfg with correct roles_path
  - printf '[defaults]\nroles_path=../' >ansible.cfg

script:
  # Basic role syntax check
  - ansible-playbook tests/test.yml -i tests/inventory --syntax-check

notifications:
  webhooks: https://galaxy.ansible.com/api/v1/notifications/
```




## 3. Pakcer와 Ansible Playbook을 위한 AWS AMI 생성

### 1) 요구사항

- Wordpress를 배포하기 위한 Ansible Playbook
- AWS AMI 이미지를 위한 Packer HCL 코드

### 2) Packer HCL

`packer-wordpress-image/wordpress-ubuntu.pkr.hcl`
```
packer {
  required_plugins {
    amazon = {
      version = ">= 0.0.2"
      source  = "github.com/hashicorp/amazon"
    }
  }
}

source "amazon-ebs" "wordpress" {
  region        = "ap-northeast-2"
  profile       = "default"

  ami_name      = "wordpress"
  instance_type = "t2.small"
  source_ami_filter {
    filters = {
      name                = "ubuntu/images/hvm-ssd/ubuntu-focal-20.04-amd64-server-*"
      root-device-type    = "ebs"
      virtualization-type = "hvm"
    }
    most_recent = true
    owners      = ["099720109477"]
  }
  ssh_username = var.ssh_account
  #force_deregister = true
}

build {
  name = "wordpress-build"
  sources = [
    "source.amazon-ebs.wordpress"
  ]

  provisioner "ansible" {
    playbook_file = "../ansible-wordpress-deploy/site.yaml"
    extra_arguments = [
      "--become",
    ]
    ansible_env_vars = [
      "ANSIBLE_HOST_KEY_CHECKING=False",
    ]
  }
}
```

## 4. Terraform과 Packer로 생성한 AWS AMI 이미지로 AWS EC2 인스턴스 배포

### 1) 요구사항

- Packer로 생성한 AWS AMI 이미지
- AWS EC2 인스턴스 배포를 위한 Terraform HCL 코드
  - VPC, Subnet 등 네트워크 리소스
  - EC2 인스턴스, SSH 키, 보안 그룹 등 리소스

### 2) Terraform HCL

`terraform-wordpress-vm/data_source.tf`
```
data "aws_ami" "wordpress" {
  most_recent = true
  owners      = ["self"]

  filter {
    name   = "name"
    values = ["wordpress"]
  }
}

```

`terraform-wordpress-vm/local.tf`
```
locals {
  common_tags = {
    Name        = "${var.project_name}-${var.environment}"
    Project     = var.project_name
    Environment = var.environment
  }
}

```

`terraform-wordpress-vm/main.tf`
```
module "my_vpc" {
  source = "terraform-aws-modules/vpc/aws"

  name = var.vpc_name
  cidr = var.vpc_cidr

  azs             = var.vpc_azs
  public_subnets  = var.vpc_public_subnets

  enable_nat_gateway = var.vpc_enable_nat_gateway

  tags = local.common_tags
}

resource "aws_instance" "my_instance" {
  ami           = data.aws_ami.ubuntu_focal.id
  instance_type = var.instance_type
  vpc_security_group_ids = [aws_security_group.my_sg_web.id]

  tags = local.common_tags
}

resource "aws_eip" "my_eip" {
  vpc      = true
  instance = aws_instance.my_instance.id

  tags = local.common_tags
}

```

`terraform-wordpress-vm/output.tf`
```
output "elastic_ip" {
  description = "Elastic IP of Instance"
  value       = aws_eip.my_eip.public_ip
}

```

`terraform-wordpress-vm/provider.tf`
```
provider "aws" {
  profile = "default"
  region  = "ap-northeast-2"
}

```

`terraform-wordpress-vm/security_group.tf`
```
resource "aws_security_group" "my_sg_web" {
  name = "allow-web"
  vpc_id = module.my_vpc.vpc_id

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

```

`terraform-wordpress-vm/variable.tf`
```
variable "aws_region" {
  description = "AWS region"
  type        = string
  default     = "ap-northeast-2"
}

variable "instance_type" {
  description = "EC2 instance type"
  type        = string
  default     = "t3.small"
}

variable "project_name" {
  description = "Name of the project"
  type        = string
  default     = "wordpress"
}

variable "environment" {
  description = "Name of the environment"
  type        = string
  default     = "dev"
}

variable "vpc_name" {
  description = "Name of VPC"
  type        = string
  default     = "my-vpc"
}

variable "vpc_cidr" {
  description = "CIDR block for VPC"
  type        = string
  default     = "10.0.0.0/16"
}

variable "vpc_azs" {
  description = "Availability zones for VPC"
  type        = list(string)
  default     = ["ap-northeast-2a"]
}

variable "vpc_public_subnets" {
  description = "Public subnets for VPC"
  type        = list(string)
  default     = ["10.0.1.0/24"]
}

variable "vpc_enable_nat_gateway" {
  description = "Enable NAT gateway for VPC"
  type        = bool
  default     = false
}

```
