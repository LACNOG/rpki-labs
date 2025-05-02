# Laboratório de RPKI (Utilizando um validador FORT e FRR como roteador via software)

------

***Criado por:***

***Santiago Aggio***,
***Nicolas Antoniello*** (*GitHub: 65007*),
***Guillermo Cicileo,***
***Erika Vega,***
***Silvia Chavez***

> Atualizado: (2025-05-02)

------



## Arquitetura de rede do laboratório
![grpX-routing-network-globalRPKI-map](../_resources/grpX-routing-network-globalRPKI-map-1.png)



> Este laboratório assume que você tem acesso à plataforma do laboratório
> 
```
  EQUIPAMENTO        ENDEREÇOS IPv4          ENDEREÇOS IPv6

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

Nesta prática, vamos acessar **somente** os seguintes equipamentos:

* **grpX-rtr** : roteador (de borda) correspondente às redes do grupo X


Os seguintes equipamentos também serão utilizados durante a prática, porém os grupos não terão acesso a eles, sendo configurados pelos tutores responsáveis pelo laboratório:

* **iborder-rtr** : roteador de borda de todo o laboratório


#### Adicionamos então alguns route-maps e listas de acesso que utilizaremos mais adiante

> Os seguintes serão explicados à medida que os utilizarmos durante a prática
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



#### Configuramos a sessão BGP com o roteador de borda (**iborder-rtr**)
> Observe que nosso sistema autônomo será o 650XX, substituindo o XX pelo número do nosso grupo (01 para o grupo 1, ... 12 para o grupo 12, etc). Assim, o grupo 1 terá o ASN 65001 e o grupo 12 terá o ASN 65012.
> 

Configuramos 2 sessões BGP, uma para IPv4 e outra para IPv6.

Neste ponto, aplicaremos políticas (***route-map***) em ambas as sessões de forma a ***permitir todos*** os prefixos recebidos do roteador de borda e anunciar a ele todos os prefixos que tivermos em nossa tabela BGP (para isso utilizamos alguns dos route-maps criados anteriormente): ***route-map TODO-IPv4*** e ***route-map TODO-IPv6***.

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



#### Visualizamos o estado da tabela BGP (IPv6)
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

