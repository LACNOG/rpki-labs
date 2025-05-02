# Ansible & BGPq4

------

***Criado e testado por Santiago Aggio e Nicolas Antoniello** (*@nantoniello)

> (2022-10-03)

*Agradecimentos especiais ao **Santiago Aggio** por sua ajuda e documentação sobre como configurar e utilizar Ansible e BGPq4 para fins de automação*

------

## Instalando o Software

> (este tutorial colocará Ansible e BGPq4 em funcionamento em uma instalação limpa do **Ubuntu Server**)

- Pacotes de Linux requeridos:


```
# apt install ansible python3-paramiko python3-pip tree
# pip3 install ansible-pylibssh
```

- Módulos do Ansible (*instalaremos o módulo da Cisco, pois é o que utilizaremos neste laboratório*)

```
# ansible-galaxy collection install cisco.ios
```

- O pacote bgpq4 necessário para gerar a configuração dos as_path a partir da consulta a um IRR

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



