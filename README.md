# Installation guid

After installing a Centos-7 minimal server follow the instructions below.

## 1. Network Connection

run `ip a` to see if you have an IP address or not, if not do the following:

1. Edit the network interface __ifcfg-xxx__ and make sure it is in correct format (HWADDR is important)
2. Restart network using `systemctl restart network.service`
3. check ip and internet access using `ip a` and `ping` command

### Sample network interface configuration:

```bash
TYPE=Ethernet
BOOTPROTO=dhcp
DEFROUTE=yes
IPV4_FAILURE_FATAL=yes
IPV6INIT=yes
IPV6_AUTOCONF=yes
IPV6_DEFROUTE=yes
IPV6_FAILURE_FATAL=no
NAME=eno16777736
UUID=72455f58-74ea-4bcc-97b6-6cc2a44f0b00
DEVICE=eno16777736
ONBOOT=yes
HWADDR=00:0C:29:2C:59:05
PEERDNS=yes
PEERROUTES=yes
IPV6_PEERDNS=yes
IPV6_PEERROUTES=yes
IPV6_PRIVACY=no
```

## 2. Access to the server

Use `ssh-copy-id` command to copy your key to the server for further use and then ssh to the server:

```bash
ssh-copy-id user@host
ssh user@host
```

## 3. Install base tools

This step, especially the `yum update` is very important so don't skip it.

```bash
sudo yum update -y
sudo yum install -y vim 
sudo yum install -y policycoreutils-python
sudo yum install -y setools setools-console setroubleshoot*
```

## 4. Secure SSH access

### 4.1. Edit `/etc/ssh/ssd_config` and set the following items accordingly

```bash
Port <new ssh port>
PermitRootLogin no
StrictModes yes
MaxAuthTries 5
MaxSessions 5
PasswordAuthentication no
ChallengeResponseAuthentication no
GSSAPIAuthentication no
X11Forwarding no
UsePAM no
```

### 4.2. Add the \<new-ssh-port\> to ssh ports in SELinux

```bash
sudo semanage port -a -t ssh_port_t -p tcp <new-ssh-port>
```

### 4.3. Restart the ssh daemon

```bash
sudo systemctl restart sshd.service
```

### 4.4. Add the \<new-ssh-port\> to firewall

```bash
sudo firewall-cmd --permanent --zone=public --remove-service=ssh
sudo firewall-cmd --permanent --zone=public --add-port=<new-ssh-port>/tcp
sudo firewall-cmd --reload
```

__Note__: Changing SSH port to your new port in the `/usr/lib/firewalld/servicess/ssh.xml` file is not a good practice, because updating the `yum` repositories causes the file to be restored to it's original state and cause you loose access to your machine after a restart or `firewall-cmd --reload`.

## 5. Install Python

### 5.1. Install dependencies, download and build `Python`

```bash
echo "install compilers and related tools:"
sudo yum groupinstall -y "development tools"
echo "install libraries needed during compilation to enable all features of Python:"
sudo yum install -y zlib-devel bzip2-devel openssl-devel ncurses-devel sqlite-devel readline-devel tk-devel gdbm-devel db4-devel libpcap-devel xz-devel expat-devel libffi-devel
echo "download latest python, build and install it:"
curl -O https://www.python.org/ftp/python/3.7.2/Python-3.7.2.tar.xz 
tar xvf Python-3.7.2.tar.xz
cd Python-3.7.2
./configure --enable-optimizations --prefix=/usr/local --enable-shared LDFLAGS="-Wl,-rpath /usr/local/lib"
make && make test && sudo make altinstall
echo "remove debug symbols from python shared library:"
sudo strip /usr/local/lib/libpython3.7m.so.1.0
```

### 5.2. Give `root` user access to python and pip

1. run `sudo visudo`
2. Add the path `/usr/local/bin` to `secure_path` section in the sudoers file.
3. Run `sudo printenv PATH` and make sure `usr/local/bin` is in the `$PATH` environment variable.

__NOTE__: You can also do this by creating a symbolic link to `python` and `pip` executables in the `/usr/bin` directory. There is some security debates adding the `/usr/local/bin` directory to `root` user's `$PATH` environment variable which I give it over to curious readers.

## 6. install nginx

### 6.1. edit `/etc/yum.repos.d/CentOS-Base.repo` and add `exclude=nginx*` to the `base` section

```bash
...
[base]
exclude=nginx*
name=CentOS-$releasever - Base
mirrorlist=http://mirrorlist.centos.org/?release=$releasever&arch=$basearch&repo=os&infra=$infra
#baseurl=http://mirror.centos.org/centos/$releasever/os/$basearch/
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-7
...
```

### 6.2. Add the file `/etc/yum.repos.d/nginx.repo` with the following content:

```bash
[nginx]
name=nginx repo
baseurl=http://nginx.org/packages/centos/7/$basearch/
gpgcheck=0
enabled=1
```

### 6.3. Install, enable and start nginx service:

```bash
sudo yum install -y nginx
sudo systemctl enable nginx.service
sudo systemctl start nginx.service
```

### 6.4. Open `HTTP` and `HHTPS` ports on firewall

```bash
sudo firewall-cmd --permanent --zone=public --add-service=http
sudo firewall-cmd --permanent --zone=public --add-service=https
sudo firewall-cmd --reload
```

## 7. install postgresql

__Note__: You can go to [Postgresql Linux downloads -Red Hat family](https://www.postgresql.org/download/linux/redhat/) and use the web  form to copy and install the postgresql repo for your platform of choice. It includes all the required instructions step-by-step.

### 7.1. Install the database server and initialize it

```bash
echo "add the yum repository"
sudo yum install -y https://download.postgresql.org/pub/repos/yum/11/redhat/rhel-7-x86_64/pgdg-centos11-11-2.noarch.rpm
echo "install required tools and services"
sudo yum install -y postgresql11 postgresql11-server postgresql11-contrib
echo "initialize the database"
sudo /usr/pgsql-11/bin/postgresql-11-setup initdb
```

### 7.2. Enable passwod login

#### 7.2.1. Open the _Host Based Authentication_ configuration file located at `/var/lib/pgsql/11/data/pg_hba.conf` with your favorite text editor

```bash
sudo vim /var/lib/pgsql/11/data/pg_hba.conf
```

#### 7.2.2. Find the lines that looks like this, near the bottom of the file

|TYPE   |DATABASE   |UER   |ADDRESS   |METHOD   |
|---|---|---|---|---|
|  host|  all|   all|   127.0.0.1/32| ident|
|  host|  all|   all|   ::1/128|   ident|

#### 7.2.3. Replace `ident` with `md5`, so they look like this

|TYPE   |DATABASE   |UER   |ADDRESS   |METHOD   |
|---|---|---|---|---|
|  host|  all|   all|   127.0.0.1/32| md5|
|  host|  all|   all|   ::1/128|   md5|

#### 7.2.4. Postgresql is now configured to allow password authentication. Save and exit

### 7.3. Start and enable postgresql

```bash
sudo systemctl start postgresql-11
sudo systemctl enable postgresql-11
```

## 8. Install redis

### 8.1. Start by enabling the Remi repository by running the following commands in your SSH terminal

```bash
sudo yum install -y epel-release yum-utils
sudo yum install -y http://rpms.remirepo.net/enterprise/remi-release-7.rpm
sudo yum-config-manager --enable remi
```

### 8.2 Install the Redis package by typing

```bash
sudo yum install -y redis
```

### 8.3. Once the installation is completed, start the Redis service and enable it to start automatically on boot with

```bash
sudo systemctl start redis
sudo systemctl enable redis
```

### 8.4. Configure Redis if required

Edit the redis configuration file located at `/etc/redis.conf` and set whatever required.

### To be continued

## Install fail2ban

```bash
sudo yum install -y epel-release
sudo yum install -y fail2ban fail2ban-systemd
yum update -y selinux-policy*
```

### To be continued ...