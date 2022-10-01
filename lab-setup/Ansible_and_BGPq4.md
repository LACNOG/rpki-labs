# Ansible & BGPq4

------

***Created and tested by Santiago Aggio and Nicolas Antoniello*** (*@nantoniello*)

> (2022-10-03)

*Thanks very much **Santiago Aggio** for his help and documentation about how to set up and configure Ansible and BGPq4 for automation purposes*

------



## Instalando el Software

> (this tutorial will get Ansible and BGPq4 up and running in an **Ubuntu Server** fresh install)

- Paquetes de Linux requeridos:

```
# apt install ansible python3-paramiko python3-pip tree
# pip3 install ansible-pylibssh
```

- Colecciones de Ansible (*instalaremos la colección de Cisco ya que es la que utilizaremos en nuestro caso, para el laboratorio*):


```
# ansible-galaxy collection install cisco.ios
```

- El paquete bgpq4 requerido para generar la configuración de los as_path a partir de la consulta a un IRR 


```
# apt install git build-essential autoconf m4 libtool
# cd
# git clone https://github.com/bgp/bgpq4.git
# cd bgpq4
# ./bootstrap
# ./configure
# make
# make install
```



