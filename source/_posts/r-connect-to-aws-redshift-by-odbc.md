---
title: Connect to AWS Redshift In R By ODBC
top: false
cover: false
toc: true
mathjax: false
date: 2020-07-09 15:32:53
password:
summary:
tags:
  - Redshift
  - Docker
categories: R
---

### docker image

```Dockerfile
FROM rocker/tidyverse
```

This is a widely-used docker image of RStudio and Tidyverse. There are several variations of the *rocker* images. Just choose the one that suits your needs most.


### unixodbc

In *Dockerfile*

```Dockerfile
RUN apt-get -y install unixodbc unixodbc-dev

# update the LD_LIBRARY_PATH to contain libodbcinst.so
ENV LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/usr/lib/x86_64-linux-gnu
```

Enter the docker to see further information of `unixodbc`
```bash
# run this line
odbcinst -j

# ...
DRIVERS............: /etc/odbcinst.ini
SYSTEM DATA SOURCES: /etc/odbc.ini
# ...
```

In later steps, we will put our *.ini* files to the right locations.

Although it is possible to change the paths by setting environment variables, there is not much point doing that inside of a docker. See *page 35* of **configuration guide** on this [site](https://docs.aws.amazon.com/redshift/latest/mgmt/configure-odbc-connection.html) for more details. To be specific, the environment variable `AMAZONREDSHIFTODBCINI` in official documents doesn't work and is not fixed as of now.

### Amazon Redshift odbc driver installation/settings

You can setup those *.ini* files from scratch, or find the templates provided by *Amazon Redshift odbc driver* in the following paths

* */opt/amazon/redshiftodbc/Setup*
* */opt/amazon/redshiftodbc/lib/64*

Here are some examples of those *.ini* files.

*odbcinst.ini*
```ini
[ODBC Drivers]
Amazon Redshift (x86)=Installed
Amazon Redshift (x64)=Installed

[Amazon Redshift (x86)]
Description=Amazon Redshift ODBC Driver (32-bit)
Driver=/opt/amazon/redshiftodbc/lib/32/libamazonredshiftodbc32.so

[Amazon Redshift (x64)]
Description=Any thing you like
# the driver path might change... see for yourself where it is.
Driver=/opt/amazon/redshiftodbc/lib/64/libamazonredshiftodbc64.so
```

*odbc.ini*
```ini
[ODBC Data Sources]
Amazon_Redshift_x32=Amazon Redshift (x86)
Amazon_Redshift_x64=Amazon Redshift (x64)

[Amazon Redshift (x86)]
Driver=/opt/amazon/redshiftodbc/lib/32/libamazonredshiftodbc32.so
Host=examplecluster.abc123xyz789.us-west-2.redshift.amazonaws.com
Port=5932
Database=dev
locale=en-US

[Amazon Redshift (x64)]
Driver=/opt/amazon/redshiftodbc/lib/64/libamazonredshiftodbc64.so
# Modify following settings

# if access from docker to localhost
# Server=host.docker.internal

# Host and Server might be interchangeable
Host=examplecluster.abc123xyz789.us-west-2.redshift.amazonaws.com
Port=5932
Database=dev
locale=en-US

# your username and password here
UID=[username]
PWD=[password]

# Optional: These values can also be specified in the connection string.
# sslmode=
# sslCertPath=
# KeepAlive=1
# KeepAliveCount=0
# KeepAliveTime=0
# KeepAliveInterval=0
# SingleRowMode=0
# UseDeclareFetch=0
# Fetch=100
# UseMultipleStatements=0
# UseUnicode=1
# BoolsAsChar=1
# TextAsLongVarchar=1
# MaxVarchar=65535
# MaxLongVarchar=65535
# MaxBytea=8190
# ProxyHost=
# ProxyPort=
```

*amazon.redshiftodbc.ini*
```ini
locale=en-US
ErrorMessagesPath=/opt/amazon/redshiftodbc/ErrorMessages
LogLevel=0
LogPath=[LogPath]

SwapFilePath=/tmp
ODBCInstLib=libodbcinst.so
```

### RStudio settings file

If you already have *R* locally, you can find the same *rstudio-prefs.json* file in the same place in user directory.

Example:

*~/.config/rstudio/rstudio-prefs.json*
```json
{
  "soft_wrap_r_files": true,
  "editor_theme": "Tomorrow Night",
  "font_size_points": 13
}

```


### Overview of the Dockerfile and docker-compose.yml

```Dockerfile
FROM rocker/tidyverse

# assume you have already downloaded the driver
COPY ./AmazonRedshiftODBC-1.x.x.xxxx-x.x86_64.deb ./
RUN apt install ./AmazonRedshiftODBC-1.x.x.xxxx-x.x86_64.deb

RUN apt-get update
RUN apt-get -y install unixodbc unixodbc-dev

# can also install other necessary packages here
RUN R -e 'install.packages(c("odbc"))'

ENV LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/usr/lib/x86_64-linux-gnu

COPY ./odbc.ini /etc/
COPY ./odbcinst.ini /etc/
COPY ./amazon.redshiftodbc.ini /opt/amazon/redshiftodbc/lib/64/

# assume we are using username "rstudio" to login to docker container
# if you prefer other user names, just change those two lines
RUN mkdir -p /home/rstudio/.config/rstudio/
COPY ./rstudio-prefs.json /home/rstudio/.config/rstudio/
```


```yml
version: "3"
services: 
  rstudio:
    build: .
    ports: 
      - "8787:8787"
    environment: 
      - ROOT=TRUE
      # user name
      - USERID=rstudio
      # replace by your password
      - PASSWORD=yourpassword
    sysctls:
      - net.ipv4.tcp_keepalive_time=200
      - net.ipv4.tcp_keepalive_intvl=200
      - net.ipv4.tcp_keepalive_probes=5
```

<sup>* `sysctls` options are to fix the timeout bug of docker in some cases. See references for more details.</sup>

### Connecting from R

First start the docker container.

```bash
# first time
docker-compose up

# start container
docker-compose start

# stop container
docker-compose stop

# build after updating
docker-compose up --build
```

Connect to RServer from [localhost:8787](localhost:8787) in browser. Enter your username and password in *docker-compose.yml* file.

```r
# top-right panel of RStudio will show the schemas and tables if succeeded
con <- DBI::dbConnect(odbc::odbc(), "Amazon Redshift (x64)")

# disconnect
DBI::dbDisconnect(con)
```

### Reference

* [rstudio database](https://db.rstudio.com/databases/redshift/)

* [TCP/IP timeout](https://docs.aws.amazon.com/redshift/latest/mgmt/connecting-firewall-guidance.html)
