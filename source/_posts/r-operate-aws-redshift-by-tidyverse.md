---
title: Operate AWS Redshift In R Syntaxes
top: false
cover: false
toc: true
mathjax: true
date: 2020-07-09 15:32:53
password:
summary:
tags:
  - Tidyverse
  - Redshift
categories: R
---

### docker environment

### unixodbc

```Dockerfile
RUN apt-get -y install unixodbc unixodbc-dev

# update the LD_LIBRARY_PATH to contain libodbcinst.so
ENV LD_LIBRARY_PATH=$LD_LIBRARY_PATH:/usr/lib/x86_64-linux-gnu
```

Enter the docker to see further information of `unixodbc`
```bash
odbcinst -j

# ...
DRIVERS............: /etc/odbcinst.ini
SYSTEM DATA SOURCES: /etc/odbc.ini
# ...
```

In later steps, we will put our *.ini* files to the right locations.

Although it is possible to change the paths by setting environment variables, there is not much point doing that inside of a docker. See *page 35* on this [site](https://s3.amazonaws.com/redshift-downloads/drivers/odbc/1.4.14.1000/Amazon+Redshift+ODBC+Driver+Install+Guide.pdf) for more details. To be specific, the environment variable `AMAZONREDSHIFTODBCINI` in official documents doesn't work and is not fixed as of now.

### Amazon Redshift odbc driver installation/settings



You can setup those *.ini* files from scratch, or find the templates provided by *Amazon Redshift odbc driver* in the following paths

* */opt/amazon/redshiftodbc/Setup*
* */opt/amazon/redshiftodbc/lib/64*

*odbcinst.ini*
```ini
[ODBC Drivers]
Amazon Redshift=Installed

[Amazon Redshift]
Description=Any thing you like
# the driver path might change... see for yourself where it is.
Driver=/opt/amazon/redshiftodbc/lib/64/libamazonredshiftodbc64.so
```

*odbc.ini*
```ini

```

*amazon.redshiftodbc.ini*
```ini

```

### RStudio settings file

Example:

*~/.config/rstudio/rstudio-prefs.json*
```json
{
  "soft_wrap_r_files": true,
  "editor_theme": "Tomorrow Night",
  "font_size_points": 13
}

```