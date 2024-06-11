## LAMP Stack 部署总结报告

### 概述

本报告总结了在 CentOS 7.6 上使用 Ansible playbook 自动化部署 LAMP (Linux, Apache, MySQL, PHP) 堆栈的过程。我们详细描述了每个步骤、遇到的困难以及解决方案。

### 目标

- 使用 Ansible 自动化部署 LAMP 堆栈
- 安装 Apache、MySQL 和 PHP
- 配置防火墙和 SELinux 以允许 HTTP 访问
- 创建 PHP 脚本以测试 PHP 和 MySQL 的连接

### 环境

- 操作系统：CentOS 7.6
- Ansible 控制节点：本地机器
- 目标节点：192.168.249.142

### 具体步骤

#### 1. 安装 Ansible

首先，需要在控制节点上安装 Ansible。参考文档：[Installing Ansible](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html)

```sh
sudo yum install epel-release -y
sudo yum install ansible -y
```

#### 2. 创建 Ansible Playbook

创建一个目录来存放 Ansible playbook 文件。

```sh
mkdir ansible-lamp
cd ansible-lamp
nano lamp.yml
```

在 `lamp.yml` 文件中粘贴以下内容：

```yaml
---
- name: Install LAMP stack on CentOS 7.6
  hosts: all
  become: yes
  tasks:
    - name: Ensure EPEL repository is enabled
      yum:
        name: epel-release
        state: present

    - name: Ensure Python MySQL dependencies are installed
      yum:
        name: MySQL-python
        state: present

    - name: Ensure Apache is installed
      yum:
        name: httpd
        state: present

    - name: Ensure Apache is started and enabled
      service:
        name: httpd
        state: started
        enabled: yes

    - name: Ensure MySQL server is installed
      yum:
        name: mariadb-server
        state: present

    - name: Ensure MySQL server is started and enabled
      service:
        name: mariadb
        state: started
        enabled: yes

    - name: Install PHP and required PHP modules
      yum:
        name: 
          - php
          - php-mysql
          - php-fpm
        state: present

    - name: Restart Apache to load PHP module
      service:
        name: httpd
        state: restarted

    - name: Create MySQL database
      mysql_db:
        name: testdb
        state: present

    - name: Create MySQL user
      mysql_user:
        name: testuser
        password: testpassword
        priv: 'testdb.*:ALL'
        state: present
      no_log: true

    - name: Create PHP info file
      copy:
        dest: /var/www/html/info.php
        content: |
          <?php
          phpinfo();
          ?>

    - name: Create PHP MySQL connection test file
      copy:
        dest: /var/www/html/testdb.php
        content: |
          <?php
          $servername = "localhost";
          $username = "testuser";
          $password = "testpassword";
          $dbname = "testdb";

          // Create connection
          $conn = new mysqli($servername, $username, $password, $dbname);

          // Check connection
          if ($conn->connect_error) {
            die("Connection failed: " . $conn->connect_error);
          }
          echo "Connected successfully";
          ?>

    - name: Ensure HTTP service is allowed through firewall
      firewalld:
        service: http
        permanent: true
        state: enabled
        immediate: yes

    - name: Ensure SELinux allows Apache to connect to network
      command: setsebool -P httpd_can_network_connect 1

    - name: Ensure SELinux allows Apache to connect to databases
      command: setsebool -P httpd_can_network_connect_db 1

    - name: Disable firewall (for testing purposes)
      service:
        name: firewalld
        state: stopped
        enabled: no
```

#### 3. 运行 Ansible Playbook

确保 Ansible 控制节点能够通过 SSH 连接到目标节点，然后运行 playbook：

```sh
ansible-playbook -i 192.168.249.143, lamp.yml
```

注意：末尾的逗号告诉 Ansible 这是一个单服务器的临时 inventory。

#### 4. 检查部署

- 打开浏览器，访问 `http://192.168.249.143/info.php`，确认 PHP 正常工作。
- 访问 `http://192.168.249.143/testdb.php`，确认 PHP 能成功连接到 MySQL 数据库。

### 遇到的困难和解决方案

1. **Apache 服务未找到**：
   - 问题：运行 `systemctl status httpd` 显示 "Unit httpd.service could not be found."
   - 解决方案：确保安装了 Apache，并使用 `yum install httpd` 安装 Apache。

2. **MySQL Python 模块错误**：
   - 问题：`mysql_user` 任务失败，提示缺少 `PyMySQL` 或 `MySQL-python` 模块。
   - 解决方案：在 playbook 中添加任务，安装 `MySQL-python` 模块。

3. **防火墙和 SELinux 配置**：
   - 问题：无法通过浏览器访问 Apache 服务。
   - 解决方案：在 playbook 中添加任务，确保防火墙允许 HTTP 流量，并配置 SELinux 允许 Apache 连接到网络和数据库。

4. **防火墙阻塞**：
   - 解决方案：在 playbook 中添加任务，禁用防火墙（仅用于测试）。

### 参考文档

- [Installing Ansible](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html)
- [Ansible Playbooks](https://docs.ansible.com/ansible/latest/user_guide/playbooks.html)
- [FirewallD Documentation](https://firewalld.org/documentation/)
- [SELinux Basics](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/selinux_users_and_administrators_guide/chap-securing_services)

### 致谢

感谢 GPT-4 提供的技术支持和建议。