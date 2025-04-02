# Laboratorio de RPKI (Utilizando un validador FORT y FRR como software router)

------

***Creado por:***

***Santiago Aggio***,
***Nicolas Antoniello*** (*GitHub: 65007*),
***Guillermo Cicileo,***
***Erika Vega,***
***Silvia Chavez***

> Actualizado: (2025-04-01)

------



## Arquitectura de red del laboratorio

![grpX-routing-network-globalRPKI-map](./RPKI_FRR_Lab_script-pics/grpX-routing-network-globalRPKI-map.png)



> This Lab assumes you have access to the Lab platform

```
     EQUIPO          DIRECCION IPv4            DIRECCION IPv6

+--------------+-----------------------+-----------------------------+
| grpX-cli     | 100.100.X.2 (eth0)    | fd3a:d409:X::2 (eth0)        |
+--------------+-----------------------+-----------------------------+
| grpX-rtr     | 100.64.1.X (eth0)     | fd3a:d409:X::1 (eth1)        |
|              | 100.100.X.65 (eth2)   | fd3a:d409:X:64::1 (eth2)     |
|              | 100.100.X.193 (eth4)  | fd3a:d409:X:192::1 (eth4)    |
|              | 100.100.X.129 (eth3)  | fd3a:d409:X:128::1 (eth3)    |
|              | 100.100.X.1 (eth1)    | fd3a:d409:0:1::X (eth0)      |
+--------------+-----------------------+-----------------------------+
| iborder-rtr  | 198.18.0.2 (wg0)      | fd3a:d409::10 (eth0)         |
+--------------+-----------------------+-----------------------------+
| rpki1        | 100.64.0.70 (eth0)    | fd3a:d409::70 (eth0)         |
+--------------------------------------+-----------------------------+
| rpki2        | 100.64.0.71 (eth0)    | fd3a:d409::71 (eth0)         |
+--------------+-----------------------+-----------------------------+
```

Donde en esta práctica **solamente** vamos a acceder a los siguientes equipos:

* **grpX-rtr** : router (de borde) correspondiente a las redes del equipo X



Los siguientes equipos también serán utilizados durante la práctica, sin embargo los grupos no tendrán acceso a los mismos, siendo estos configurados por los tutores encargados del laboratorio:

* **iborder-rtr** : router de borde para todo el laboratorio



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
 neighbor fd3a:d409::10 remote-as 65000
 neighbor fd3a:d409::10 description iborder-rtr
 !
 address-family ipv4 unicast
  neighbor 100.64.0.10 activate
  neighbor 100.64.0.10 soft-reconfiguration inbound
  neighbor 100.64.0.10 route-map TODO-IPv4 in
  neighbor 100.64.0.10 route-map TODO-IPv4 out
 exit-address-family
 !
 address-family ipv6 unicast
  neighbor fd3a:d409::10 activate
  neighbor fd3a:d409::10 soft-reconfiguration inbound
  neighbor fd3a:d409::10 route-map TODO-IPv6 in
  neighbor fd3a:d409::10 route-map TODO-IPv6 out
 exit-address-family
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


