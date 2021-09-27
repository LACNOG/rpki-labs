# RPKI FRR router config (FORT validator)

------

***Created and tested by Nicolas Antoniello*** (*@nantoniello*)

> (2021-09-12)

------



## Initial FRR router configuration

> (this tutorial will get an FRR router up and running in an **Ubuntu Server** fresh install and then configure it to use a FORT RPKI validator server)



### Install FRR router

```
# apt-get update
# apt-get -y --no-install-recommends install \
       autoconf automake build-essential git libjansson-dev \
       libssl-dev pkg-config rsync ca-certificates curl \
       libcurl4-openssl-dev libxml2-dev \
       less vim
```



### Configure FRR router to support BGP and RPKI

```
# cd
# git clone https://github.com/NICMx/FORT-validator.git
# cd FORT-validator
# ./autogen.sh
# ./configure
# make
# make install
# make clean
```



## Configure FRR router to use FORT RPKI validator

### Generate keys for secure connection to FORT validator server



### Add FORT validator server to FRR router



