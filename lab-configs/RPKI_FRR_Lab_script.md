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
| grpX-cli     | 100.100.X.2 (eth0)    | fd98:21e2:X::2 (eth0)        |
+--------------+-----------------------+-----------------------------+
| grpX-rtr     | 100.64.1.X (eth0)     | fd98:21e2:X::1 (eth1)        |
|              | 100.100.X.65 (eth2)   | fd98:21e2:X:64::1 (eth2)     |
|              | 100.100.X.193 (eth4)  | fd98:21e2:X:192::1 (eth4)    |
|              | 100.100.X.129 (eth3)  | fd98:21e2:X:128::1 (eth3)    |
|              | 100.100.X.1 (eth1)    | fd98:21e2:0:1::X (eth0)      |
+--------------+-----------------------+-----------------------------+
| iborder-rtr  | 198.18.0.2 (wg0)      | fd98:21e2::10 (eth0)         |
+--------------+-----------------------+-----------------------------+
| rpki1        | 100.64.0.70 (eth0)    | fd98:21e2::70 (eth0)         |
+--------------------------------------+-----------------------------+
| rpki2        | 100.64.0.71 (eth0)    | fd98:21e2::71 (eth0)         |
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
ipv6 route ::/0 fd98:21e2::1
!
interface eth0
 description "class backbone"
 ip address 100.64.0.10/22
 ipv6 address fd98:21e2::10/48
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
 neighbor fd98:21e2:0:1::1 remote-as 65001
 neighbor fd98:21e2:0:1::1 description grp1-rtr
 neighbor fd98:21e2:0:1::2 remote-as 65002
 neighbor fd98:21e2:0:1::2 description grp2-rtr
 neighbor fd98:21e2:0:1::3 remote-as 65003
 neighbor fd98:21e2:0:1::3 description grp3-rtr
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
  neighbor fd98:21e2:0:1::1 activate
  neighbor fd98:21e2:0:1::1 soft-reconfiguration inbound
  neighbor fd98:21e2:0:1::1 route-map TODO-IPv6 in
  neighbor fd98:21e2:0:1::1 route-map TODO-IPv6 out
  neighbor fd98:21e2:0:1::2 activate
  neighbor fd98:21e2:0:1::2 soft-reconfiguration inbound
  neighbor fd98:21e2:0:1::2 route-map TODO-IPv6 in
  neighbor fd98:21e2:0:1::2 route-map TODO-IPv6 out
  neighbor fd98:21e2:0:1::3 activate
  neighbor fd98:21e2:0:1::3 soft-reconfiguration inbound
  neighbor fd98:21e2:0:1::3 route-map TODO-IPv6 in
  neighbor fd98:21e2:0:1::3 route-map TODO-IPv6 out
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



## Verificando la configuración inicial de los routers de borde de los equipos

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
ipv6 route ::/0 fd98:21e2::1
!
interface eth0
 description "class backbone"
 ip address 100.64.1.X/22
 ipv6 address fd98:21e2:0:1::X/48
!
interface eth1
 description "lan"
 ip address 100.100.X.1/26
 ipv6 address fd98:21e2:X::1/64
!
interface eth2
 description "int"
 ip address 100.100.X.65/26
 ipv6 address fd98:21e2:X:64::1/64
!
interface eth3
 description "dmz"
 ip address 100.100.X.129/26
 ipv6 address fd98:21e2:X:128::1/64
!
interface eth4
 description "extra"
 ip address 100.100.X.193/26
 ipv6 address fd98:21e2:X:192::1/64
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
ipv6 route ::/0 fd98:21e2::1
!
interface eth0
 description "class backbone"
 ip address 100.64.1.X/22
 ipv6 address fd98:21e2:0:1::X/48
!
interface eth1
 description "lan"
 ip address 100.100.X.1/26
 ipv6 address fd98:21e2:X::1/64
!
interface eth2
 description "int"
 ip address 100.100.X.65/26
 ipv6 address fd98:21e2:X:64::1/64
!
interface eth3
 description "dmz"
 ip address 100.100.X.129/26
 ipv6 address fd98:21e2:X:128::1/64
!
interface eth4
 description "extra"
 ip address 100.100.X.193/26
 ipv6 address fd98:21e2:X:192::1/64
!
router bgp 6500X
 bgp router-id 100.64.1.X
 bgp log-neighbor-changes
 no bgp default ipv4-unicast
 neighbor 100.64.0.10 remote-as 65000
 neighbor 100.64.0.10 description iborder-rtr
 neighbor fd98:21e2::10 remote-as 65000
 neighbor fd98:21e2::10 description iborder-rtr
 !
 address-family ipv4 unicast
  neighbor 100.64.0.10 activate
  neighbor 100.64.0.10 soft-reconfiguration inbound
  neighbor 100.64.0.10 route-map TODO-IPv4 in
  neighbor 100.64.0.10 route-map TODO-IPv4 out
 exit-address-family
 !
 address-family ipv6 unicast
  neighbor fd98:21e2::10 activate
  neighbor fd98:21e2::10 soft-reconfiguration inbound
  neighbor fd98:21e2::10 route-map TODO-IPv6 in
  neighbor fd98:21e2::10 route-map TODO-IPv6 out
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
 neighbor fd98:21e2::10 remote-as 65000
 neighbor fd98:21e2::10 description iborder-rtr
 !
 address-family ipv4 unicast
  neighbor 100.64.0.10 activate
  neighbor 100.64.0.10 soft-reconfiguration inbound
  neighbor 100.64.0.10 route-map TODO-IPv4 in
  neighbor 100.64.0.10 route-map TODO-IPv4 out
 exit-address-family
 !
 address-family ipv6 unicast
  neighbor fd98:21e2::10 activate
  neighbor fd98:21e2::10 soft-reconfiguration inbound
  neighbor fd98:21e2::10 route-map TODO-IPv6 in
  neighbor fd98:21e2::10 route-map TODO-IPv6 out
 exit-address-family
```



#### Visualizamos el estado de las sesiones BGP (IPv6)

```
grpX-rtr# sh bgp ipv6 unicast summary 
BGP router identifier 100.64.1.X, local AS number 650XX vrf-id 0
BGP table version 557
RIB entries 88, using 16 KiB of memory
Peers 1, using 723 KiB of memory

Neighbor        V         AS   MsgRcvd   MsgSent   TblVer  InQ OutQ  Up/Down State/PfxRcd   PfxSnt Desc
fd98:21e2::10   4      65000     15619     10555        0    0    0 01w0d01h           46       46 iborder-rtr

Total number of neighbors 1
```



#### Visualizamos el estado de la tabla BGP (IPv6)

```
grpX-rtr# sh bgp ipv6 unicast
BGP table version is 15706, local router ID is 100.64.1.X, vrf id 0
Default local pref 100, local AS 65001
Status codes:  s suppressed, d damped, h history, * valid, > best, = multipath,
               i internal, r RIB-failure, S Stale, R Removed
Nexthop codes: @NNN nexthop's vrf id, < announce-nh-self
Origin codes:  i - IGP, e - EGP, ? - incomplete
RPKI validation codes: V valid, I invalid, N Not found

   Network          Next Hop            Metric LocPrf Weight Path
*> 2001:7fb:fd02::/48
                    fe80::216:3eff:fee0:2b4b
                                                           0 65000 64512 264759 7049 3549 3356 8455 12654 i
*> 2001:7fb:fe00::/48
                    fe80::216:3eff:fee0:2b4b
                                                           0 65000 64512 264759 7049 3549 3356 30781 204092 12654 i
*> 2001:7fb:fe01::/48
                    fe80::216:3eff:fee0:2b4b
                                                           0 65000 64512 264759 7049 3549 3356 9002 12654 i
*> 2001:7fb:fe03::/48
                    fe80::216:3eff:fee0:2b4b
                                                           0 65000 64512 264759 7049 3549 3356 8455 12654 i
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



#### Visualización de la configuración del validador RPKI

```
grpX-rtr# sh rpki cache-connection 
Connected to group 1
rpki tcp cache 100.64.0.70 323 pref 1
```



#### Visualización del estado de validación de un prefijo en particular

```
grpX-rtr# sh rpki prefix 2001:13c7:7002::/48
Prefix                                   Prefix Length  Origin-AS
2001:13c7:7002::                            48 -  48        28001
```



#### Visualizamos el estado de la tabla BGP (IPv6)

```
grpX-rtr# sh bgp ipv6 unicast
BGP table version is 8453, local router ID is 100.64.1.X, vrf id 0
Default local pref 100, local AS 650XX
Status codes:  s suppressed, d damped, h history, * valid, > best, = multipath,
               i internal, r RIB-failure, S Stale, R Removed
Nexthop codes: @NNN nexthop's vrf id, < announce-nh-self
Origin codes:  i - IGP, e - EGP, ? - incomplete
RPKI validation codes: V valid, I invalid, N Not found

   Network          Next Hop            Metric LocPrf Weight Path
V*> 2001:7fb:fd02::/48
                    fe80::216:3eff:fee0:2b4b
                                                           0 65000 64512 264759 7049 3549 3356 8455 12654 i
V*> 2001:7fb:fe00::/48
                    fe80::216:3eff:fee0:2b4b
                                                           0 65000 64512 264759 7049 3549 3356 30781 204092 12654 i
V*> 2001:7fb:fe01::/48
                    fe80::216:3eff:fee0:2b4b
                                                           0 65000 64512 264759 7049 3549 3356 9002 12654 i
V*> 2001:7fb:fe03::/48
                    fe80::216:3eff:fee0:2b4b
                                                           0 65000 64512 264759 7049 3549 3356 8455 12654 i
V*> 2001:7fb:fe04::/48
                    fe80::216:3eff:fee0:2b4b
                                                           0 65000 64512 264759 7049 3549 3356 25091 513 12654 i
```





## Analysis de un prefijo particular a modo de demostración

Visualizamos el prefijo ***2001:7fb:fd02::1*** en la tabla BGP

```
grpX-rtr# sh bgp ipv6 unicast 2001:7fb:fd02::1
BGP routing table entry for 2001:7fb:fd02::/48, version 2960
Paths: (1 available, best #1, table default)
  Advertised to non peer-group peers:
  fd98:21e2::10
  65000 64512 264759 7049 3549 3356 8455 12654
    fd98:21e2::10 from fd98:21e2::10 (100.64.0.10)
    (fe80::216:3eff:fee0:2b4b) (used)
      Origin IGP, valid, external, best (First path received), rpki validation-state: valid
      Last update: Mon Oct  4 21:34:09 2021
```

* ***¿Qué sucede con el estado de validación?***
* ***¿Que ASN origina el prefijo?***



Accedemos al cliente y realizamos un mtr (traceroute) al mismo prefijo (***2001:7fb:fd02::1***) y lo dejamos ejecutando

```
root@cli:~# mtr 2001:7fb:fd02::1

cli.grpX.lac.te-labs.training (fd98:21e2:X::2)                 2021-10-04T22:06:48+0000
Keys:  Help   Display mode   Restart statistics   Order of fields   quit
                                               Packets               Pings
 Host                                        Loss%   Snt   Last   Avg  Best  Wrst StDev
 1. fd98:21e2:1::1                            0.0%    10    0.1   0.1   0.1   0.1   0.0
 2. fd98:21e2::10                             0.0%    10    0.1   0.1   0.1   0.2   0.0
 3. fd98:21e2::1                              0.0%    10    0.1   0.1   0.1   0.1   0.0
 4. (waiting for reply)
 5. (waiting for reply)
 6. (waiting for reply)
 7. 2620:107:4000:cfff::f3ff:1e61             0.0%    10    0.8   0.8   0.7   1.0   0.1
 8. 2620:107:4000:c5e0::f3fd:c02              0.0%    10    0.7   0.7   0.6   0.8   0.1
 9. 2620:107:4000:a710::f000:3013             0.0%    10    0.7   0.8   0.7   0.9   0.0
10. 2620:107:4000:a710::f000:3005             0.0%    10    0.9   0.9   0.8   1.0   0.1
11. 2620:107:4000:cfff::f200:baa1             0.0%    10    0.7   1.5   0.5   5.5   1.6
12. 2620:107:4000:2000::b3                    0.0%    10    0.9   4.0   0.6  15.0   5.4
13. 2620:107:4000:8002::26                    0.0%    10    0.7   1.3   0.6   6.4   1.8
14. 2620:107:4008:575::2                      0.0%    10    0.8   0.9   0.8   1.3   0.1
15. ae27.mpr1.cdg11.fr.zip.zayo.com           0.0%     9   84.4  79.2  78.3  84.4   2.0
16. ae10.tcr2.th2.par.core.as8218.eu          0.0%     9   78.2  81.3  78.2 103.9   8.5
17. 2001:1b48:2:3::24:1                       0.0%     9   78.4  78.8  78.4  80.4   0.8
18. 2001:7fb:fd02::1                          0.0%     9   90.8  90.8  90.7  90.8   0.0
   
```



Ahora aguardamos a que los instructores realicen unas modificaciones... observando que sucede con el MTR.

> Discusión

- ***¿Qué se observa?***
- Intente refrescar el mtr (*presionando la letra "r"*)

```
cli.grpX.lac.te-labs.training (fd98:21e2:X::2)                 2021-10-04T22:25:36+0000
Keys:  Help   Display mode   Restart statistics   Order of fields   quit
                                               Packets               Pings
 Host                                        Loss%   Snt   Last   Avg  Best  Wrst StDev
 1. fd98:21e2:1::1                            0.0%   131    0.2   0.1   0.1   0.2   0.0
 2. fd98:21e2::10                             0.0%   130    0.2   0.1   0.1   0.2   0.1
 3. 2001:7fb:fd02::1                          0.0%   130    0.2   0.1   0.1   0.4   0.1
```



Visualizamos el prefijo en nuestra tabla BGP

```
grpX-rtr# sh bgp ipv6 unicast 2001:7fb:fd02::1
BGP routing table entry for 2001:7fb:fd02::/64, version 8495
Paths: (1 available, best #1, table default)
  Advertised to non peer-group peers:
  fd98:21e2::10
  65000 65002
    fd98:21e2:0:1::2 from fd98:21e2::10 (100.64.0.10)
    (fe80::216:3eff:fee0:2b4b) (used)
      Origin IGP, valid, external, best (First path received), rpki validation-state: invalid
      Last update: Mon Oct  4 22:22:12 2021
```



* ***¿Qué sucede con el estado de validación?***
* ***¿Que ASN origina el prefijo?***
* ***¿Qué sucedió?***
* Entonces, si el estado es Invalido y tengo activado RPKI, ***¿por qué sigo viendo el prefijo en mi tabla BGP?***.



### Configurando filtros BGP basados en el estado de validación

Aplicamos el filtro "RPKI" que solo instalará en la tabla BGP los prefijos con estado "valid" o "not found" (descartando los "invalid")

```
grpX-rtr# conf t
grpX-rtr(config)# router bgp 650XX
grpX-rtr(config-router)# address-family ipv6 unicast
grpX-rtr(config-router-af)# neighbor fd98:21e2::10 route-map RPKI in
```



Visualizamos el prefijo en nuestra tabla BGP

```
grpX-rtr# sh bgp ipv6 unicast 2001:7fb:fd02::1
BGP routing table entry for 2001:7fb:fd02::/48, version 8503
Paths: (1 available, best #1, table default)
  Advertised to non peer-group peers:
  fd98:21e2::10
  65000 64512 264759 7049 3549 3356 8455 12654
    fd98:21e2::10 from fd98:21e2::10 (100.64.0.10)
    (fe80::216:3eff:fee0:2b4b) (used)
      Origin IGP, localpref 200, valid, external, best (First path received), rpki validation-state: valid
      Last update: Mon Oct  4 22:48:00 2021
```



* ***¿Qué sucede con el estado de validación?***
* ***¿Que ASN origina el prefijo?***
* ***¿Qué sucedió?***



En el cliente visualizamos nuevamente el MTR, intente refrescarlo (presionando la letra "r")

```
cli.grpX.lac.te-labs.training (fd98:21e2:X::2)                 2021-10-04T22:50:51+0000
Keys:  Help   Display mode   Restart statistics   Order of fields   quit
                                               Packets               Pings
 Host                                        Loss%   Snt   Last   Avg  Best  Wrst StDev
 1. fd98:21e2:1::1                            0.0%     8    0.1   0.1   0.1   0.2   0.0
 2. fd98:21e2::10                             0.0%     8    0.2   0.1   0.1   0.2   0.0
 3. 2001:7fb:fd02::1                          0.0%     7    0.2   0.2   0.1   0.2   0.0

```

* ***¿Qué sucede? ¿Cómo lo explica?***

  (Tip: recordar la topología de red y quien provee la conectividad a Internet)



### Los tutores procederán a habilitar filtros RPKI similares en el router iborder-rtr (ANS 65000)

Los tutores aplican el filtro RPKI en el router de borde.



Accedemos al cliente y visualizamos lo que sucede. Intente refrescarlo (presionando la letra "r")

```
cli.grpX.lac.te-labs.training (fd98:21e2:X::2)                 2021-10-04T23:09:10+0000
Keys:  Help   Display mode   Restart statistics   Order of fields   quit
                                               Packets               Pings
 Host                                        Loss%   Snt   Last   Avg  Best  Wrst StDev
 1. fd98:21e2:1::1                            0.0%    20    0.1   0.1   0.1   0.1   0.0
 2. fd98:21e2::10                             0.0%    20    0.1   0.1   0.1   0.1   0.0
 3. fd98:21e2::1                              0.0%    20    0.1   0.1   0.1   0.2   0.0
 4. (waiting for reply)
 5. (waiting for reply)
 6. (waiting for reply)
 7. 2620:107:4000:cfff::f3ff:c41              0.0%    20    0.6   0.7   0.6   0.9   0.1
 8. 2620:107:4000:cfff::f200:9850             0.0%    20    0.9   0.8   0.7   0.9   0.1
 9. 2620:107:4000:a5d0::f000:2010             0.0%    20    0.7   0.8   0.7   0.9   0.1
10. 2620:107:4000:a5d0::f000:2006             0.0%    20    0.8  57.6   0.8 261.1  87.4
11. 2620:107:4000:cfff::f200:9b31             0.0%    20    0.6   2.1   0.5  19.9   5.0
12. 2620:107:4000:2000::dd                    0.0%    20    1.0   1.4   0.7   5.2   1.0
13. 2620:107:4000:8002::224                   0.0%    20    0.8   2.6   0.7  19.5   4.5
14. 2620:107:4008:b974::2                     0.0%    20    0.9   0.9   0.8   1.1   0.1
15. ae27.mpr1.cdg11.fr.zip.zayo.com           0.0%    20   78.4  80.5  78.3 102.6   5.3
16. ae10.tcr2.th2.par.core.as8218.eu          0.0%    20   78.3  78.4  78.3  78.8   0.1
17. 2001:1b48:2:3::24:1                       0.0%    19   78.4  78.5  78.3  79.2   0.2
18. 2001:7fb:fd02::1                          0.0%    19   91.1  90.9  90.9  91.2   0.1

```



***¿Conclusiones?***

> Discutir como se resolvió el secuestro de rutas en este caso.



