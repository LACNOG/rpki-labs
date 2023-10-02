# Automatización (Utilizando Ansible y BGPq4)

------

***Creado por:***

***Santiago Aggio***,
***Nicolas Antoniello*** (*GitHub: 65007*),
***Guillermo Cicileo,***
***Erika Vega,***
***Silvia Chavez***

> (2023-10-01)

------



## Arquitectura de red del laboratorio

![grpX-routing-network-globalRPKI-map](./RPKI_FRR_Lab_script-pics/grpX-routing-network-globalRPKI-map.png)



> This Lab assumes you have access to the Lab platform

```
     EQUIPO          DIRECCION IPv4            DIRECCION IPv6

+--------------+-----------------------+-----------------------------+
| grpX-cli     | 100.100.X.2 (eth0)    | fd23:2fe7:X::2 (eth0)        |
+--------------+-----------------------+-----------------------------+
| grpX-rtr     | 100.64.1.X (eth0)     | fd23:2fe7:X::1 (eth1)        |
|              | 100.100.X.65 (eth2)   | fd23:2fe7:X:64::1 (eth2)     |
|              | 100.100.X.193 (eth4)  | fd23:2fe7:X:192::1 (eth4)    |
|              | 100.100.X.129 (eth3)  | fd23:2fe7:X:128::1 (eth3)    |
|              | 100.100.X.1 (eth1)    | fd23:2fe7:0:1::X (eth0)      |
+--------------+-----------------------+-----------------------------+
| iborder-rtr  | 198.18.0.2 (wg0)      | fd23:2fe7::10 (eth0)         |
+--------------+-----------------------+-----------------------------+
| rpki1        | 100.64.0.70 (eth0)    | fd23:2fe7::70 (eth0)         |
+--------------------------------------+-----------------------------+
| rpki2        | 100.64.0.71 (eth0)    | fd23:2fe7::71 (eth0)         |
+--------------+-----------------------+-----------------------------+
```

Donde en esta práctica **solamente** vamos a acceder a los siguientes equipos:

* **grpX-cli** : cliente
* **grpX-rtr** : router (de borde) correspondiente a las redes del equipo X



Los siguientes equipos también serán utilizados durante la práctica, sin embargo los grupos no tendrán acceso a los mismos, siendo estos configurados por los tutores encargados del laboratorio:

* **iborder-rtr** : router de borde para todo el laboratorio



## Instalar Colecciones de Ansible

Instalaremos la colección de Cisco ya que es la que utilizaremos en nuestro caso, para el laboratorio:


```
$ ansible-galaxy collection install cisco.ios
```



## Clonar el proyecto desde Github de LACNOG

```
$ git clone https://github.com/LACNOG/rpki-labs.git
```



## Estructura de archivos

En el directorio ansible se encuentran los archivos  que vamos a utilizar en este laboratorio 

```
$ cd rpki-labs/lab-setup
$ tree ansible/
ansible
├── ansible.cfg
├── hosts
├── ios
│   ├── bgp_af.yml
│   ├── bgp_as_path.yml
│   ├── bgp_route_map_rpki.yml
│   ├── bgp.yml
│   ├── clean_confg.yml
│   ├── prefix_lists.yml
│   ├── route_map_rpki_aspath.yml
│   ├── route_maps.yml
│   └── rpki.yml
├── playbook.yml
└── vars_ios.yml
```



## Descripción del playbook a correr, inventario y variables

Para correr un playbook en Ansible utilizamos el comando ansible-playbook indicando el inventario de hosts destino de las tareas y como entrada el archivo playbook donde se definen los parametros generales y la lista de tareas. 

```
$ ansible-playbook -i hosts playbook.yml
```

El playbook para la automatización contiene  ***"tags"*** para ejecutar cada tarea de manera individual y poder apreciar los cambios en el router.

```
$ cd ansible
$ cat playbook.yml
---
#
# Laboratorio de Automatizacion con Ansible
#
- name: RTR BGP Ansible deployment
  hosts: grp_rtr
  gather_facts: no
  
  vars_files:
  - vars_ios.yml
 
  tasks:
        - include_tasks: ios/t0_clean_confg.yml
          tags: T0

        - include_tasks: ios/t1_rpki.yml
          tags: T1

        - include_tasks: ios/t2_prefix_lists.yml
          tags: T2

        - include_tasks: ios/t3_route_maps_todo.yml
          tags: T3

        - include_tasks: ios/t4_bgp.yml
          tags: T4

        - include_tasks: ios/t5_bgp_af.yml
          tags: T5

        - include_tasks: ios/t6_bgp_as_path.yml
          tags: T6
          
        - include_tasks: ios/t7_bgp_route_map_some_asn.yml
          tags: T7

        - include_tasks: ios/t8_route_map_rpki_aspath.yml
          tags: T8

        - include_tasks: ios/t9_bgp_route_map_rpki.yml
          tags: T9

```

El inventario esta definido en el archivo hosts y en este caso solo contiene la descripción de nuestro router. 

```
$ cat hosts
[grp_rtr]
100.100.X.1
```

Es necesario modificar la dirección IP por la asignada a cada grupo. Pueden hacerlo con los editores disponibles (nano, vim) o recrearlo utilizando el comando:

```
$ cat <<EOF > hosts
[grp_rtr]
100.100.X.1
EOF
```

Es necesario también cambiar algunas variables definidas en el archivo var_ios.yml por los valores asignados a cada grupo. 

```
$ cat vars_ios.yml
ansible_connection: ansible.netcommon.network_cli
ansible_network_os: cisco.ios.ios
ansible_user: rtradm 
ansible_password: 'xxxxxxxx'

ansible_python_interpreter: '/usr/bin/python3'

config_dir: ./ios_config

my_asn: "650XX"
my_router_id: "100.64.1.X"

iborder_asn: "65000"
iborder_ipv4: "100.64.0.10"
iborder_ipv6: "fd23:2fe7::10"
iborder_desc: "iborder-rtr"

rpki_cache_1: "100.64.0.70"
rpki_port_1: "323"
rpki_cache_2: "100.64.0.71"
rpki_port_2: "323"

irr_as_set: AS64135:AS-LabRPKIprivadosTutores
```

Las variables a modificar con los valores asignados a cada grupo son:  

> **ansible_password:** 
>
> **my_asn:**
>
> **my_router_id:**



## Manos a la obra!!!!

A continuación describimos cada tarea definida en el playbook y los comandos a utilizar para aplicarlos

#### T0: Limpieza de configuración manual aplicada en la práctica 

En este paso previo volvemos la configuración del router al estado inicial. 

Podemos ver la tarea a ejecutar contenida en el archivo clean_confg.yml

```
$ cat ios/t0_clean_confg.yml
```

Para correr esta tarea ejecutamos el comando:

```
$ ansible-playbook -i hosts -t T0 playbook.yml
```

Verificamos el cambio de la configuración en el router:

```
grpX-rtr# sh run
```

#### T1: Configuramos el validador RPKI

El archivo rpki.yml describe la tarea a ejecutar:

```
$ cat ios/t1_rpki.yml
```

Para correr esta tarea ejecutamos el comando:

```
$ ansible-playbook -i hosts -t T1 playbook.yml
```

#### T2: Creamos las prefix-List PERMIT-ALL y DENY-ALL

El archivo prefix_lists.yml describe la tarea a ejecutar:

```
$ cat ios/t2_prefix_lists.yml
```

Para correr esta tarea ejecutamos el comando:

```
$ ansible-playbook -i hosts -t T2 playbook.yml
```

#### T3: Creamos los route-maps TODO-IPv4  y TODO IPv6

El archivo route_maps.yml describe la tarea a ejecutar:

```
$ cat ios/t3_route_maps_todo.yml
```

Para correr esta tarea ejecutamos el comando:

```
$ ansible-playbook -i hosts -t T3 playbook.yml
```

#### T4: Configuramos router BGP AS 6500X

El archivo bgp.yml describe la tarea a ejecutar:

```
$ cat ios/t4_bgp.yml
```

Para correr esta tarea ejecutamos el comando:

```
$ ansible-playbook -i hosts -t T4 playbook.yml
```

#### T5: Configuramos los neighbors BGP desde BGP Address Family  

El archivo bgp_af.yml describe la tarea a ejecutar:

```
$ cat ios/t5_bgp_af.yml
```

Para correr esta tarea ejecutamos el comando:

```
$ ansible-playbook -i hosts -t T5 playbook.yml
```

#### T6: Generamos los bgp as-path access-list usando el comando bgpq4 y lo aplicamos al route-map PERMIT-SOME-ASN

El archivo bgp.yml describe la tarea a ejecutar:

```
$ cat ios/t6_bgp_as_path.yml
```

Para correr esta tarea ejecutamos el comando:

```
$ ansible-playbook -i hosts -t T6 playbook.yml
```

#### T7:  Aplicamos ahora el Route-Map PERMIT-SOME-ASN a la entrada del BGP neigboor 

El archivo route_map_some_asn.yml describe la tarea a ejecutar:

```
$ cat ios/t7_bgp_route_map_some_asn.yml
```

Para correr esta tarea ejecutamos el comando:

```
$ ansible-playbook -i hosts -t T7 playbook.yml
```

#### T8: Agregamos match as-path en el route-map RPKI (valid, notfound)

El archivo route_map_rpki_aspath.yml describe la tarea a ejecutar:

```
$ cat ios/t8_route_map_rpki_aspath.yml
```

Para correr esta tarea ejecutamos el comando:

```
$ ansible-playbook -i hosts -t T8 playbook.yml
```

#### T9:  Aplicamos ahora el Route-Map RPKI a la entrada del BGP neigboor 

El archivo bgp_route_map_rpki_aspath.yml describe la tarea a ejecutar:

```
$ cat ios/t9_bgp_route_map_rpki_aspath.yml
```

Para correr esta tarea ejecutamos el comando:

```
$ ansible-playbook -i hosts -t T9 playbook.yml
```

#### T0-T9: Ejecutar Playbook completo

Para ejecutar el playbook completo incluyendo todas las tareas, invocando todos los tags, utilizamos ***"all"*** como referencia: 

```
$ ansible-playbook -i hosts -t all playbook.yml
```

Si queremos ejecutar algunas tareas en particular podemos seleccionarlas a modo de lista. Por ejemplo para ejecutar las tareas T0 a la T6:

```
$ ansible-playbooks -i hosts -t T0,T1,T2,T3,T4,T5,T6 playbook.yml
```

