# Vagrant vProfile Multi-Tier Java Application

## Overview
This project sets up a **multi-tier Java application** using **Vagrant** and **VirtualBox**. The architecture consists of multiple virtual machines, each serving a specific role in the deployment. Provisioning is automated using shell scripts for each server.

## Architecture
The multi-tier setup consists of the following servers:

1. **db01 (Database Server)** – Runs **MariaDB/MySQL** for storing application data.
2. **mc01 (Memcached Server)** – Provides caching to optimize performance.
3. **rmq01 (RabbitMQ Server)** – Handles messaging and queueing.
4. **app01 (Tomcat Application Server)** – Hosts the Java-based application.
5. **web01 (Nginx Reverse Proxy Server)** – Acts as a frontend load balancer.

## Prerequisites
Ensure you have the following installed on your machine:
- [Vagrant](https://www.vagrantup.com/downloads)
- [VirtualBox](https://www.virtualbox.org/wiki/Downloads)

## Setup Instructions
1. Clone the repository:
   ```sh
   git clone https://github.com/surendergupta/vprofile
   cd vprofile
   ```
2. Start the environment:
   ```sh
   vagrant up
   ```
3. Verify all machines are running:
   ```sh
   vagrant status
   ```

## Vagrant Configuration
The `Vagrantfile` defines all virtual machines and their configurations:

- **Networking:** Private network with static IPs.
- **Resource Allocation:** Memory and CPU assigned per VM.

## Managing VMs
- To **access a specific VM**:
  ```sh
  vagrant ssh <vm-name>
  ```
- To **restart a VM**:
  ```sh
  vagrant reload <vm-name> --provision
  ```
- To **destroy all VMs**:
  ```sh
  vagrant destroy -f
  ```

## Setup VMs
- To **access a DB01 VM Database Server**:
  ```sh
  vagrant ssh db01
  sudo -i
  dnf update -y
  dnf install epel-release -y  
  dnf install git mariadb-server -y
  systemctl start mariadb
  systemctl enable mariadb
  mysql_secure_installation #Run mysql_secure_installation and select 'Y' for all prompts except 'Disallow root login' (choose 'N'). Set the root password to 'admin123'. If any update is needed, modify it in the Tomcat server.#
  mysql -u root -padmin123 
    mysql> create database accounts;
    mysql> grant all privileges on accounts.* TO 'admin'@'localhost' identified by 'admin123';
    mysql> grant all privileges on accounts.* TO 'admin'@'%' identified by 'admin123';
    mysql> FLUSH PRIVILEGES;
    mysql> exit;
  cd /tmp/
  git clone -b local https://github.com/hkhcoder/vprofile-project.git
  cd vprofile-project
  mysql -u root -padmin123 accounts < src/main/resources/db_backup.sql
  systemctl restart mariadb
  systemctl start firewalld
  systemctl enable firewalld
  firewall-cmd --zone=public --add-port=3306/tcp --permanent
  firewall-cmd --reload
  systemctl restart mariadb

  ```

- To **access a MC01 VM Memcached Server**:
  ```sh
  vagrant ssh mc01
  sudo -i
  dnf update -y
  dnf install epel-release -y
  dnf install memcached -y
  systemctl start memcached
  systemctl enable memcached
  systemctl status memcached
  sed -i 's/127.0.0.1/0.0.0.0/g' /etc/sysconfig/memcached
  systemctl restart memcached
  systemctl start firewalld
  systemctl enable firewalld
  firewall-cmd --add-port=11211/tcp --permanent
  firewall-cmd --add-port=11111/udp --permanent
  memcached -p 11211 -U 11111 -u memcached -d

  ```

- To **access a RMQ01 VM RabbitMQ Server**:
  ```sh
  vagrant ssh rmq01
  sudo -i
  dnf update -y
  dnf install epel-release -y
  dnf install centos-release-rabbitmq-38 -y
  dnf install rabbitmq-server --enablerepo=centos-rabbitmq-38 -y 
  sh -c 'echo "[{rabbit, [{loopback_users, []}]}]." > /etc/rabbitmq/rabbitmq.config'
  rabbitmqctl add_user test test
  rabbitmqctl set_user_tags test administrator
  rabbitmqctl set_permissions -p / test ".*" ".*" ".*"
  systemctl restart rabbitmq-server
  systemctl start firewalld
  systemctl enable firewalld
  firewall-cmd --add-port=5672/tcp
  firewall-cmd --runtime-to-permanent
  systemctl start rabbitmq-server
  systemctl enable --now rabbitmq-server
  systemctl status rabbitmq-server
  ```

- To **access a APP01 VM Tomcat Application Server**:
  ```sh
  vagrant ssh app01
  sudo -i
  dnf update -y
  dnf install epel-release -y
  dnf install java-17-openjdk java-17-openjdk-devel -y
  dnf install git wget unzip -y
  cd /tmp/
  wget https://archive.apache.org/dist/tomcat/tomcat-10/v10.1.26/bin/apache-tomcat-10.1.26.tar.gz
  tar xzvf apache-tomcat-10.1.26.tar.gz
  useradd --home-dir /usr/local/tomcat --shell /sbin/nologin tomcat
  cp -r /tmp/apache-tomcat-10.1.26/* /usr/local/tomcat/
  chown -R tomcat.tomcat /usr/local/tomcat
  cat << EOF | sudo tee /etc/systemd/system/tomcat.service  
    [Unit]
    Description=Tomcat
    After=network.target
    
    [Service]
    User=tomcat
    Group=tomcat
    WorkingDirectory=/usr/local/tomcat
    Environment=JAVA_HOME=/usr/lib/jvm/jre
    Environment=CATALINA_PID=/var/tomcat/%i/run/tomcat.pid
    Environment=CATALINA_HOME=/usr/local/tomcat
    Environment=CATALINA_BASE=/usr/local/tomcat
    ExecStart="/usr/local/tomcat/bin/catalina.sh run"
    ExecStop="/usr/local/tomcat/bin/shutdown.sh"
    RestartSec=10
    Restart=always
    
    [Install]
    WantedBy=multi-user.target
  EOF
  
  systemctl daemon-reload
  systemctl start tomcat
  systemctl enable tomcat
  systemctl status tomcat

  systemctl start firewalld
  systemctl enable firewalld
  firewall-cmd --zone=public --add-port=8080/tcp --permanent
  firewall-cmd --reload

  wget https://archive.apache.org/dist/maven/maven-3/3.9.9/binaries/apache-maven-3.9.9-bin.zip
  unzip apache-maven-3.9.9-bin.zip
  cp -r apache-maven-3.9.9 /usr/local/maven3.9
  export MAVEN_OPTS="-Xmx512m"
  
  git clone -b local https://github.com/hkhcoder/vprofile-project.git
  cd vprofile-project
  vim src/main/resources/application.properties # update password if password not same as admin123
  /usr/local/maven3.9/bin/mvn install
  
  systemctl stop tomcat
  rm -rf /usr/local/tomcat/webapps/ROOT*
  cp target/vprofile-v2.war /usr/local/tomcat/webapps/ROOT.war
  
  systemctl start tomcat
  chown -R tomcat.tomcat /usr/local/tomcat/webapps
  
  systemctl restart tomcat
  ```

- To **access a WEB01 VM Nginx Reverse Proxy Server**:
  ```sh
  vagrant ssh web01
  sudo -i
  apt update -y
  apt upgrade -y
  apt install nginx -y
  vi /etc/nginx/sites-available/vproapp # below content save in this file
    
    upstream vproapp {
        server app01:8080;
    }
    server {
        listen 80;  
        location / {
            proxy_pass http://vproapp;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            proxy_http_version 1.1;
            proxy_set_header Connection "";
        }
    }
  
  rm -rf /etc/nginx/sites-enabled/default
  ln -s /etc/nginx/sites-available/vproapp /etc/nginx/sites-enabled/vproapp
  systemctl restart nginx

  ```

## Testing the Application
1. After provisioning, access the application via:
   ```
   http://<web01_ip> 
   or
   http://192.168.56.11
   ```
   (Replace with the correct IP of `web01` if different.)

## Troubleshooting
- To debug SSH issues:
  ```sh
  vagrant ssh-config
  ```

## Future Enhancements
- Implement in **Vagrant Automation** for on-premises infrastructure.
- Implement in **AWS** for cloud infrastructure.
- Automate deployment with **Ansible**.
- Add **Docker** support.
- Implement **Kubernetes** for container orchestration.
- Integrate **Terraform** for infrastructure as code.
- Implement **CI/CD pipelines** for automated deployments.

## License
This project is licensed under the **MIT License**.

---

🚀 **Developed by Surender Gupta**

