# Laboratorio de RPKI (Utilizando un validador FORT y FRR como software router)

------

***Creado por:***

***Santiago Aggio***,
***Nicolas Antoniello*** (*GitHub: 65007*),
***Guillermo Cicileo,***
***Erika Vega***

> (2021-08-11)

------



## Arquitectura de red del laboratorio

![grpX-routing-network-globalRPKI-map](./RPKI_FRR_Lab_script-pics/grpX-routing-network-globalRPKI-map.png)



> This Lab assumes you have access to the Lab platform

```
     EQUIPO          DIRECCION IPv4            DIRECCION IPv6

+--------------+-----------------------+-----------------------------+
| grpX-cli     | 100.100.X.2 (eth0)    | fd4a:7fe0:X::2 (eth0)       |
+--------------+-----------------------+-----------------------------+
| grpX-rtr     | 100.64.1.X (eth0)     | fd4a:7fe0:X::1 (eth1)       |
|              | 100.100.X.65 (eth2)   | fd4a:7fe0:X:64::1 (eth2)    |
|              | 100.100.X.193 (eth4)  | fd4a:7fe0:X:192::1 (eth4)   |
|              | 100.100.X.129 (eth3)  | fd4a:7fe0:X:128::1 (eth3)   |
|              | 100.100.X.1 (eth1)    | fd4a:7fe0:0:1::X (eth0)     |
+--------------+-----------------------+-----------------------------+
| iborder-rtr  | 198.18.0.2 (wg0)      | fd4a:7fe0::10 (eth0)        |
+--------------+-----------------------+-----------------------------+
| rpki1        | 100.64.0.70 (eth0)    | fd4a:7fe0::70 (eth0)        |
+--------------------------------------+-----------------------------+
| rpki2        | 100.64.0.71 (eth0)    | fd4a:7fe0::71 (eth0)        |
+--------------+-----------------------+-----------------------------+
```

Donde en esta práctica **solamente** vamos a acceder a los siguientes equipos:

* **grpX-cli** : cliente
* **grpX-rtr** : router (de borde) correspondiente a las redes del equipo X



Los siguientes equipos también serán utilizados durante la práctica, sin embargo los grupos no tendrán acceso a los mismos, siendo estos configurados por los tutores encargados del laboratorio:

* **rpki1** : validador RPKI (FORT)
* **rpki2** : validador RPKI (FORT)
* **iborder-rtr** : router de borde para todo el laboratorio



## Verificando la configuración del validador RPKI FORT (*rpki1* y *rpki2*)

Uno de los tutores del Laboratorio mostrará la configuración del validador FORT (contenido del archivo ***/etc/fort/config.json*** en el servidor **rpki1** o **rpki2**):

```
root@rpki1:~# more /etc/fort/config.json 

{
	"tal": "/var/fort/tal",
	"local-repository": "/var/fort/repository",
	"rsync-strategy": "root",
	"shuffle-uris": true,
	"mode": "server",

	"server": {
		"port": "323",
		"backlog": 100,
		"interval": {
	            "validation": 900,
	            "refresh": 900,
	            "retry": 600,
	            "expire": 7200
	        }
	},

	"log": {
		"color-output": true,
		"file-name-format": "file-name"
	},

	"rsync": {
		"program": "rsync",
		"arguments-recursive": [
			"--recursive",
			"--times",
			"$REMOTE",
			"$LOCAL"
		],
		"arguments-flat": [
			"--times",
			"--dirs",
			"$REMOTE",
			"$LOCAL"
		]
	},

	"incidences": [
		{
			"name": "incid-hashalg-has-params",
			"action": "ignore"
		}
	],

	"output": {
		"roa": "/var/fort/fort_roas.csv"
	}
}

```



Los archivos correspondientes a los TAL de cada uno de los 5 RIRs se encuentran en el directorio ***/var/fort/tal/***:

```
root@rpki1:~# ls -larth /var/fort/tal/

total 6.0K
drwxr-xr-x 4 root root   5 Sep 24 00:36 ..
-rw-r--r-- 1 root root 496 Sep 20 15:10 afrinic.tal
-rw-r--r-- 1 root root 466 Sep 20 15:10 apnic.tal
-rw-r--r-- 1 root root 487 Sep 20 15:10 arin.tal
-rw-r--r-- 1 root root 502 Sep 20 15:10 lacnic.tal
-rw-r--r-- 1 root root 482 Sep 20 15:10 ripe-ncc.tal
drwxr-xr-x 2 root root   7 Sep 20 15:10 .
```



Ahora actualizaremos los archivos con los TALs descargándolos nuevamente de los 5 RIRs, utilizando el siguiente comando:

```
root@rpki1:~# fort --init-tals --tal /var/fort/tal
```

```
Sep 27 18:11:44 DBG: HTTP GET: https://rpki.afrinic.net/tal/afrinic.tal
Successfully fetched '/var/fort/tal/afrinic.tal'!

Sep 27 18:11:45 DBG: HTTP GET: https://tal.apnic.net/apnic.tal
Successfully fetched '/var/fort/tal/apnic.tal'!

Attention: ARIN requires you to agree to their Relying Party Agreement (RPA) before you can download and use their TAL.
Please download and read https://www.arin.net/resources/manage/rpki/rpa.pdf
If you agree to the terms, type 'yes' and hit Enter: Sep 27 18:11:47 DBG: HTTP GET: https://www.arin.net/resources/manage/rpki/arin.tal
Successfully fetched '/var/fort/tal/arin.tal'!

Sep 27 18:11:47 DBG: HTTP GET: https://www.lacnic.net/innovaportal/file/4983/1/lacnic.tal
Successfully fetched '/var/fort/tal/lacnic.tal'!

Sep 27 18:11:48 DBG: HTTP GET: https://tal.rpki.ripe.net/ripe-ncc.tal
Successfully fetched '/var/fort/tal/ripe-ncc.tal'!
```



Confirmamos (observando la fecha) que efectivamente disponemos de la versión actual de los TAL (en el directorio ***/var/fort/tal/***):

```
root@rpki1:~# ls -larth /var/fort/tal/

total 6.0K
drwxr-xr-x 4 root root   5 Sep 24 00:36 ..
-rw-r--r-- 1 root root 496 Sep 27 18:11 afrinic.tal
-rw-r--r-- 1 root root 466 Sep 27 18:11 apnic.tal
-rw-r--r-- 1 root root 487 Sep 27 18:11 arin.tal
-rw-r--r-- 1 root root 502 Sep 27 18:11 lacnic.tal
-rw-r--r-- 1 root root 482 Sep 27 18:11 ripe-ncc.tal
drwxr-xr-x 2 root root   7 Sep 27 18:11 .
```



Y reiniciamos el proceso del validador FORT (recordar que el validador fue instalado y ejecutado como demonio del sistema):

```
root@rpki1:~# systemctl restart fortd
```

```
root@rpki1:~# systemctl status fortd

● fortd.service - FORT service (fort) - FORT RPKI validator
     Loaded: loaded (/lib/systemd/system/fortd.service; enabled; vendor preset: enabled)
    Drop-In: /run/systemd/system/service.d
             └─zzz-lxc-service.conf
     Active: active (running) since Mon 2021-09-27 18:15:10 UTC; 7s ago
   Main PID: 1956 (fort)
      Tasks: 37 (limit: 19204)
     Memory: 13.2M
     CGroup: /system.slice/fortd.service
             └─1956 /usr/local/bin/fort --configuration-file=/etc/fort/config.json

Sep 27 18:15:10 rpki1.lac.te-labs.training systemd[1]: Started FORT service (fort) - FORT RPKI validator.
Sep 27 18:15:10 rpki1.lac.te-labs.training fort[1956]: INF: Disabling validation logging on syslog.
Sep 27 18:15:10 rpki1.lac.te-labs.training fort[1956]: INF: Disabling validation logging on standard streams.
Sep 27 18:15:10 rpki1.lac.te-labs.training fort[1956]: INF: Console log output configured; disabling operation logging on syslog.
Sep 27 18:15:10 rpki1.lac.te-labs.training fort[1956]: INF: (Operation Logs will be sent to the standard streams only.)
```



Visualizando el log del demonio **fortd** (***/var/log/fortd.log***) vemos que inició el primer ciclo de validación (luego del reinicio), por lo que deberemos esperar a que este primer ciclo finalize para poder utilizar el validador:

```
root@rpki1:~# tail -f /var/log/fortd.log

/usr/local/bin/fort(+0x3577a)[0x55c4aa92077a]
/usr/local/bin/fort(+0x362a1)[0x55c4aa9212a1]
/usr/local/bin/fort(+0x45fbc)[0x55c4aa930fbc]
/lib/x86_64-linux-gnu/libpthread.so.0(+0x9609)[0x7fc904e47609]
/lib/x86_64-linux-gnu/libc.so.6(clone+0x43)[0x7fc904d6e293]
Sep 27 18:15:10 INF: Disabling validation logging on syslog.
Sep 27 18:15:10 INF: Disabling validation logging on standard streams.
Sep 27 18:15:10 INF: Console log output configured; disabling operation logging on syslog.
Sep 27 18:15:10 INF: (Operation Logs will be sent to the standard streams only.)
Sep 27 18:15:10 WRN: First validation cycle has begun, wait until the next notification to connect your router(s)
```



Una vez que el ciclo finalice, el siguiente mensaje aparecerá en el log (***/var/log/fortd.log***):

```
Sep 27 18:18:13 WRN: First validation cycle successfully ended, now you can connect your router(s)
```

> Notar que le lleva algunos minutos al validador el poder completar el primer ciclo de validación



## Verificando la configuración del router de borde ***iborder-rtr***

Uno de los tutores del Laboratorio mostrará la configuración del router de borde (contenido del archivo ***/etc/frr/frr.conf*** en el equipo **iborder-rtr**):

```
root@iborder-rtr:~# more /etc/frr/frr.conf

frr version 8.0.1
frr defaults traditional
hostname iborder-rtr
service integrated-vtysh-config
!
ip route 0.0.0.0/0 100.64.0.1
ip route 181.199.160.1/32 Null0
ipv6 route ::/0 fd4a:7fe0::1
!
interface eth0
 description "class backbone"
 ip address 100.64.0.10/22
 ipv6 address fd4a:7fe0::10/48
!
interface wg0
 description "tunel contra el ISP (Ariel Weher)"
 ip address 198.18.0.2/29
!
router bgp 65000
 bgp router-id 100.64.0.10
 bgp log-neighbor-changes
 no bgp default ipv4-unicast
 neighbor 100.64.1.1 remote-as 65001
 neighbor 100.64.1.1 description grp1-rtr
 neighbor 100.64.1.2 remote-as 65002
 neighbor 100.64.1.2 description grp2-rtr
 neighbor 100.64.1.3 remote-as 65003
 neighbor 100.64.1.3 description grp3-rtr
 ...
 neighbor 198.18.0.1 remote-as 64512
 neighbor 198.18.0.1 description ISP (Ariel Weher)
 neighbor fd4a:7fe0:0:1::1 remote-as 65001
 neighbor fd4a:7fe0:0:1::1 description grp1-rtr
 neighbor fd4a:7fe0:0:1::2 remote-as 65002
 neighbor fd4a:7fe0:0:1::2 description grp2-rtr
 neighbor fd4a:7fe0:0:1::3 remote-as 65003
 neighbor fd4a:7fe0:0:1::3 description grp3-rtr
 ...
 !
 address-family ipv4 unicast
  neighbor 100.64.1.1 activate
  neighbor 100.64.1.1 soft-reconfiguration inbound
  neighbor 100.64.1.1 route-map TODO-IPv4 in
  neighbor 100.64.1.1 route-map TODO-IPv4 out
  neighbor 100.64.1.2 activate
  neighbor 100.64.1.2 soft-reconfiguration inbound
  neighbor 100.64.1.2 route-map TODO-IPv4 in
  neighbor 100.64.1.2 route-map TODO-IPv4 out
  neighbor 100.64.1.3 activate
  neighbor 100.64.1.3 soft-reconfiguration inbound
  neighbor 100.64.1.3 route-map TODO-IPv4 in
  neighbor 100.64.1.3 route-map TODO-IPv4 out
  ...
  neighbor 198.18.0.1 activate
  neighbor 198.18.0.1 soft-reconfiguration inbound
  neighbor 198.18.0.1 route-map PERMIT-SOME-ASN in
  neighbor 198.18.0.1 route-map NADA-IPv4 out
 exit-address-family
 !
 address-family ipv6 unicast
  neighbor 198.18.0.1 activate
  neighbor 198.18.0.1 soft-reconfiguration inbound
  neighbor 198.18.0.1 route-map PERMIT-SOME-ASN in
  neighbor 198.18.0.1 route-map NADA-IPv6 out
  neighbor fd4a:7fe0:0:1::1 activate
  neighbor fd4a:7fe0:0:1::1 soft-reconfiguration inbound
  neighbor fd4a:7fe0:0:1::1 route-map TODO-IPv6 in
  neighbor fd4a:7fe0:0:1::1 route-map TODO-IPv6 out
  neighbor fd4a:7fe0:0:1::2 activate
  neighbor fd4a:7fe0:0:1::2 soft-reconfiguration inbound
  neighbor fd4a:7fe0:0:1::2 route-map TODO-IPv6 in
  neighbor fd4a:7fe0:0:1::2 route-map TODO-IPv6 out
  neighbor fd4a:7fe0:0:1::3 activate
  neighbor fd4a:7fe0:0:1::3 soft-reconfiguration inbound
  neighbor fd4a:7fe0:0:1::3 route-map TODO-IPv6 in
  neighbor fd4a:7fe0:0:1::3 route-map TODO-IPv6 out
  ...
 exit-address-family
!
ip prefix-list DENY-ALL-IPv4 seq 5 deny any
ip prefix-list PERMIT-ALL-IPv4 seq 5 permit any
!
ipv6 prefix-list DENY-ALL-IPv6 seq 5 deny any
ipv6 prefix-list PERMIT-ALL-IPv6 seq 5 permit any
!
bgp as-path access-list AS-PATH-PERMIT-LIST seq 10 permit _28000_
bgp as-path access-list AS-PATH-PERMIT-LIST seq 20 permit _28001_
bgp as-path access-list AS-PATH-PERMIT-LIST seq 30 permit _12654_
bgp as-path access-list AS-PATH-PERMIT-LIST seq 40 permit _196615_
!
route-map NADA-IPv4 permit 10
 match ip address prefix-list DENY-ALL-IPv4
!
route-map PERMIT-SOME-ASN permit 10
 match as-path AS-PATH-PERMIT-LIST
!
route-map NADA-IPv6 permit 10
 match ipv6 address prefix-list DENY-ALL-IPv6
!
route-map TODO-IPv4 permit 10
 match ip address prefix-list PERMIT-ALL-IPv4
!
route-map TODO-IPv6 permit 10
 match ipv6 address prefix-list PERMIT-ALL-IPv6
!
route-map RPKI permit 10
 match rpki valid
 set local-preference 200
!
route-map RPKI permit 20
 match rpki notfound
 set local-preference 100
!
line vty
!
end
```



Vemos que este router de borde tiene una sesión BGP establecida con un ISP del cual recibe la tabla global IPv4 e IPv6.

También están pre-configuradas en este router de borde, sesiones BGP con cada uno de los router de borde de los grupos, a la espera de que estos configuren las sesiones BGP de su lado.

Para evitar sobrecargar todos los routers de los grupos y el laboratorio en general, se aplicó un filtro que solamente permitirá a BGP poner algunos prefijos (correspondientes a ciertos ASNs) en las tablas BGP (tanto IPv4 como IPv6).



> ***Finalmente notar que este router no posee ningún filtro de rutas basado en información RPKI; reenviando a los routers de borde de los grupos todos los prefijos permitidos en el filtro BGP por ASN mencionado anteriormente*** ***(al mismo tiempo, aceptando todo lo que los equipos envíen)***.



## Verificando la configuración inicial de los routers de borde de los equipos y configurando los mismos para BGP y RPKI

Verificamos la configuración inicial del router de borde correspondiente a nuestro equipo (**grpX-rtr**):

```
grpX-rtr# sh run
Building configuration...

Current configuration:
!
frr version 8.0.1
frr defaults traditional
hostname grpX-rtr
service integrated-vtysh-config
!
ip route 0.0.0.0/0 100.64.0.1
ipv6 route ::/0 fd4a:7fe0::1
!
interface eth0
 description "class backbone"
 ip address 100.64.1.X/22
 ipv6 address fd4a:7fe0:0:1::X/48
!
interface eth1
 description "lan"
 ip address 100.100.X.1/26
 ipv6 address fd4a:7fe0:X::1/64
!
interface eth2
 description "int"
 ip address 100.100.X.65/26
 ipv6 address fd4a:7fe0:X:64::1/64
!
interface eth3
 description "dmz"
 ip address 100.100.X.129/26
 ipv6 address fd4a:7fe0:X:128::1/64
!
interface eth4
 description "extra"
 ip address 100.100.X.193/26
 ipv6 address fd4a:7fe0:X:192::1/64
!
line vty
!
end
```

> En este punto es conveniente revisar nuevamente el diagrama de red correspondiente a cada grupo e identificar las interfaces ya configuradas en el router del equipo.



Ahora, modificamos la configuración del router correspondiente a nuestro equipo, para que resulte la siguiente (*el detalle de los pasos de configuración se encuentra más abajo*):

```
grpX-rtr# sh run
Building configuration...

Current configuration:
!
frr version 8.0.1
frr defaults traditional
hostname grpX-rtr
rpki
 rpki polling_period 300
 rpki cache 100.64.0.70 323 preference 1
 rpki cache 100.64.0.71 323 preference 2
 exit
service integrated-vtysh-config
!
ip route 0.0.0.0/0 100.64.0.1
ipv6 route ::/0 fd4a:7fe0::1
!
interface eth0
 description "class backbone"
 ip address 100.64.1.X/22
 ipv6 address fd4a:7fe0:0:1::X/48
!
interface eth1
 description "lan"
 ip address 100.100.X.1/26
 ipv6 address fd4a:7fe0:X::1/64
!
interface eth2
 description "int"
 ip address 100.100.X.65/26
 ipv6 address fd4a:7fe0:X:64::1/64
!
interface eth3
 description "dmz"
 ip address 100.100.X.129/26
 ipv6 address fd4a:7fe0:X:128::1/64
!
interface eth4
 description "extra"
 ip address 100.100.X.193/26
 ipv6 address fd4a:7fe0:X:192::1/64
!
router bgp 6500X
 bgp router-id 100.64.1.X
 bgp log-neighbor-changes
 no bgp default ipv4-unicast
 neighbor 100.64.0.10 remote-as 65000
 neighbor 100.64.0.10 description iborder-rtr
 neighbor fd4a:7fe0::10 remote-as 65000
 neighbor fd4a:7fe0::10 description iborder-rtr
 !
 address-family ipv4 unicast
  neighbor 100.64.0.10 activate
  neighbor 100.64.0.10 soft-reconfiguration inbound
  neighbor 100.64.0.10 route-map TODO-IPv4 in
  neighbor 100.64.0.10 route-map TODO-IPv4 out
 exit-address-family
 !
 address-family ipv6 unicast
  neighbor fd4a:7fe0::10 activate
  neighbor fd4a:7fe0::10 soft-reconfiguration inbound
  neighbor fd4a:7fe0::10 route-map TODO-IPv6 in
  neighbor fd4a:7fe0::10 route-map TODO-IPv6 out
 exit-address-family
!
ip prefix-list DENY-ALL-IPv4 seq 5 deny any
ip prefix-list PERMIT-ALL-IPv4 seq 5 permit any
!
ipv6 prefix-list DENY-ALL-IPv6 seq 5 deny any
ipv6 prefix-list PERMIT-ALL-IPv6 seq 5 permit any
!
route-map PERMIT-SOME-ASN permit 10
 match as-path AS-PATH-PERMIT-LIST
!
route-map TODO-IPv6 permit 10
 match ipv6 address prefix-list PERMIT-ALL-IPv6
!
route-map NADA-IPv6 permit 10
 match ipv6 address prefix-list DENY-ALL-IPv6
!
route-map TODO-IPv4 permit 10
 match ip address prefix-list PERMIT-ALL-IPv4
!
route-map NADA-IPv4 permit 10
 match ipv6 address prefix-list DENY-ALL-IPv4
!
route-map RPKI permit 10
 match rpki valid
 set local-preference 200
!
route-map RPKI permit 20
 match rpki notfound
 set local-preference 100
!
line vty
!
end
```



#### Agregamos entonces algunos route-map y listas de acceso que utilizaremos más tarde

> Los siguientes los iremos explicando a medida que los utilicemos durante la práctica
>

```
ip prefix-list DENY-ALL-IPv4 seq 5 deny any
ip prefix-list PERMIT-ALL-IPv4 seq 5 permit any
!
ipv6 prefix-list DENY-ALL-IPv6 seq 5 deny any
ipv6 prefix-list PERMIT-ALL-IPv6 seq 5 permit any
!
route-map PERMIT-SOME-ASN permit 10
 match as-path AS-PATH-PERMIT-LIST
!
route-map TODO-IPv6 permit 10
 match ipv6 address prefix-list PERMIT-ALL-IPv6
!
route-map NADA-IPv6 permit 10
 match ipv6 address prefix-list DENY-ALL-IPv6
!
route-map TODO-IPv4 permit 10
 match ip address prefix-list PERMIT-ALL-IPv4
!
route-map NADA-IPv4 permit 10
 match ipv6 address prefix-list DENY-ALL-IPv4
!
route-map RPKI permit 10
 match rpki valid
 set local-preference 200
!
route-map RPKI permit 20
 match rpki notfound
 set local-preference 100
```



#### Configuramos la sesión BGP con el router de borde (**iborder-rtr**)

> Observar que nuestro sistema autónomo será el 650XX cambiando la XX por nuestro número de grupo (01 para el grupo 1, ... 12 para el grupo 12, etc). De forma que el grupo 1 tendrá el ASN 65001 y el grupo 12 tendrá el ASN 65012.
>

Configuramos 2 sesiones BGP, una IPv4 y otra IPv6.

En este punto, aplicaremos políticas (***route-map***) en ambas sesiones de forma de ***permitir todos*** los prefijos recibidos del router de borde y publicar al mismo todos los que tengamos en nuestra tabla BGP (para ello utilizamos algunos de los route-map creados anteriormente): ***route-map TODO-IPv4*** y ***route-map TODO-IPv6***.

```
router bgp 650XX
 bgp router-id 100.64.1.X
 bgp log-neighbor-changes
 no bgp default ipv4-unicast
 neighbor 100.64.0.10 remote-as 65000
 neighbor 100.64.0.10 description iborder-rtr
 neighbor fd4a:7fe0::10 remote-as 65000
 neighbor fd4a:7fe0::10 description iborder-rtr
 !
 address-family ipv4 unicast
  neighbor 100.64.0.10 activate
  neighbor 100.64.0.10 soft-reconfiguration inbound
  neighbor 100.64.0.10 route-map TODO-IPv4 in
  neighbor 100.64.0.10 route-map TODO-IPv4 out
 exit-address-family
 !
 address-family ipv6 unicast
  neighbor fd4a:7fe0::10 activate
  neighbor fd4a:7fe0::10 soft-reconfiguration inbound
  neighbor fd4a:7fe0::10 route-map TODO-IPv6 in
  neighbor fd4a:7fe0::10 route-map TODO-IPv6 out
 exit-address-family
```



#### Agregamos la configuración para conectar el router con nuestro validador RPKI

Para conectar el router al validador RPKI podemos utilizar la dirección IPv4 del validador o el nombre de dominio del mismo... en este caso utilizaremos las direcciones IPv4:

**rpki1**:  *100.64.0.70*
**rpki2**:  *100.64.0.70*

Y el puerto TCP que configuramos en los servidores, en nuestro caso ambos en el puerto 323.

Finalmente también debemos indicar la preferencia de cada servidor de forma que nuestro router intentará primero con el que tenga menor preferencia.

```
rpki
 rpki polling_period 300
 rpki cache 100.64.0.70 323 preference 1
 rpki cache 100.64.0.71 323 preference 2
```



En este punto, ya tenemos completamente configurado nuestro router de borde y podemos comenzar a visualizar algunos detalles y ensayar diferentes escenarios.



## Visualización de la configuración del validador RPKI



