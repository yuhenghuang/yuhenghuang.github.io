---
title: Setup Environment On Remote Server, Updating
top: false
cover: false
toc: true
mathjax: false
date: 2020-06-13 09:04:14
password:
summary:
tags: Environments
categories: Server
---


This post record necessary documents for setting up environments on a remote server. The example here is **Ubuntu 18**.


### Add user

  [Reference](https://www.cyberciti.biz/faq/create-a-user-account-on-ubuntu-linux/)

  * Add

    ```bash
    sudo useradd -s /bin/bash -d /home/{dirname} -m - G sudo {username}
    sudo passwd {username}
    ```

    <sup>`sudo passwd` can also be used to change password.<sup>

  * Set up "*~/.ssh/*"

    `sudo chmod 0700 /home/{dirname}/.ssh/`

    Then append the contents of your public keys to "*~/.ssh/authorized_keys*"

    `sudo chown -R {username}:{username} /home/{username}/.ssh/`

  * Delete

    `sudo userdel -r {username}`

    <sup>`-r` indicates deleting directory<sup>


### VS code

  Connect to remote server via ssh. [Documents](https://code.visualstudio.com/docs/remote/ssh).

  <sup>hint: set longer `"remote.SSH.connectTimeout"` in case fail to start terminal.<sup>

### Jupyter

  Official instructions can be found [here](https://jupyter-notebook.readthedocs.io/en/latest/public_server.html)

  * Setup password via
    
    `jupyter notebook password`

    The hashed password is in "*~/.jupyter/jupyter_notebook_config.json*"

  * Generate SSL certificate via

    `openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout mykey.key -out mycert.pem`

  * Configure "*~/.jupyter/jupyter_notebook_config.py*"

  ```python
  # Set options for certfile, ip, password, and toggle off
  # browser auto-opening
  c.NotebookApp.certfile = u'/absolute/path/to/your/certificate/mycert.pem'
  c.NotebookApp.keyfile = u'/absolute/path/to/your/certificate/mykey.key'
  # Set ip to '*' to bind on all interfaces (ips) for the public server
  c.NotebookApp.ip = '*'
  c.NotebookApp.password = u'sha1:bcd259ccf...<your hashed password here>'
  c.NotebookApp.open_browser = False

  # It is a good idea to set a known, fixed port for server access
  c.NotebookApp.port = 9999
  ```


### Rstudio

  Instructions from [official site](https://rstudio.com/products/rstudio/download-server/). Additional documents about installing R is found [here](https://www.digitalocean.com/community/tutorials/how-to-install-r-on-ubuntu-18-04).

  * Install `r-base`
    * add GPG key via

      `sudo apt-key adv --keyserver keyserver.ubuntu.com --recv-keys E298A3A825C0D65DFD57CBB651716619E084DAB9`

    * add repository via of R 4.0

      `sudo add-apt-repository 'deb https://cloud.r-project.org/bin/linux/ubuntu bionic-cran40/'`

    * install `r-base`

      `sudo apt update`

      `sudoapt install r-base`

    * test installation
      
      `sudo -i R`

  * Install `rstudio-server`

    ```bash
    sudo apt-get install gdebi-core
    wget https://download2.rstudio.org/server/bionic/amd64/rstudio-server-1.3.959-amd64.deb
    sudo gdebi rstudio-server-1.3.959-amd64.deb
    ```

  * Server status

    `sudo systemctl status rstudio-server.service`

  * Start/Stop `rstudio-server`

    ```bash
    sudo systemctl start rstudio-server
    sudo systemctl stop rstudio-server
    ```

### MySQL
  version: 5.7

  * Install

      `sudo apt update`

      `sudo apt install MySQL-server`

  * Status, start and stop

    `sudo systemctl status mysql.service`

    `sudo systemctl start mysql`
    
    `sudo systemctl stop mysql`

  * Settings for `utf8` characters, [reference](https://stackoverflow.com/questions/10957238/incorrect-string-value-when-trying-to-insert-utf-8-into-mysql-via-jdbc)
    * add `character-set-server=utf8mb4` and `collation-server=utf8mb4_unicode_ci` under `[mysqld]` in "*/etc/mysql/mysql.conf.d/mysqld.cnf*"
    * add `default-character-set=utf8mb4` under `[mysql]` in "*/etc/mysql/conf.d/mysql.cnf*"
    * alter database via

      `ALTER DATABASE $db_name CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;`

    * alter tables via

      `ALTER TABLE $table_name CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;`

      `ALTER TABLE $table_name CONVERT TO CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;`


### clang

* [Official Site](https://releases.llvm.org/download.html)

### Java

* Download JDK 

  `wget --header "Cookie: oraclelicense=accept-securebackup-cookie" https://download.oracle.com/otn-pub/java/jdk/14.0.1+7/664493ef4a6946b186ff29eb326336a2/jdk-14.0.1_linux-x64_bin.tar.gz`

  The link can be found on [oracle.com](https://www.oracle.com/java/technologies/javase-jdk14-downloads.html)

* Instructions

    [official installation instruction](https://docs.oracle.com/javase/10/install/installation-jdk-and-jre-linux-platforms.htm)

