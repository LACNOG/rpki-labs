# Automação(Utilizando Ansible e BGPq4)

------

***Criado por:***

***Santiago Aggio***,
***Nicolas Antoniello*** (*GitHub: 65007*),
***Guillermo Cicileo,***
***Erika Vega,***
***Silvia Chavez***

> (2025-05-02)

------



## Arquitetura de rede do laboratório
![grpX-routing-network-globalRPKI-map](./RPKI_FRR_Lab_script-pics/grpX-routing-network-globalRPKI-map.png)



> Este laboratório assume que você tem acesso à plataforma do laboratório
```
  EQUIPAMENTO        ENDEREÇOS IPv4           ENDEREÇOS IPv6

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

* **grpX-cli** : cliente  
* **grpX-rtr** : roteador (de borda) correspondente às redes do grupo X

Os seguintes equipamentos também serão utilizados durante a prática, porém os grupos não terão acesso a eles, sendo configurados pelos tutores responsáveis pelo laboratório:

* **iborder-rtr** : roteador de borda de todo o laboratório


## Instalação de módulos do Ansible

Instalaremos o módulo da Cisco, pois é o que utilizaremos neste laboratório:

```
$ ansible-galaxy collection install cisco.ios
```



## Clonar o projeto a partir do repositório do LACNOG no Github
```
$ git clone https://github.com/LACNOG/rpki-labs.git
```



## Estrutura de arquivos

No diretório `ansible` estão os arquivos que vamos utilizar neste laboratório

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



## Descrição do playbook a ser executado, inventário e variáveis

Para executar um playbook no Ansible, utilizamos o comando `ansible-playbook`, indicando o inventário de hosts de destino das tarefas e, como entrada, o arquivo playbook onde são definidos os parâmetros gerais e a lista de tarefas.

```
$ ansible-playbook -i hosts playbook.yml
```

O playbook de automação contém ***"tags"*** para executar cada tarefa individualmente e permitir observar as mudanças no roteador.

```
$ cd ansible
$ cat playbook.yml
---
#
# Laboratório de Automação com Ansible
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

O inventário está definido no arquivo `hosts` e, neste caso, contém apenas a descrição do nosso roteador.

```
$ cat hosts
[grp_rtr]
100.100.X.1
```

É necessário modificar o endereço IP pelo atribuído a cada grupo. Isso pode ser feito com os editores disponíveis (nano, vim) ou recriado utilizando o comando:

```
$ cat <<EOF > hosts
[grp_rtr]
100.100.X.1
EOF
```

Também é necessário alterar algumas variáveis definidas no arquivo `var_ios.yml` para os valores atribuídos a cada grupo.

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
iborder_ipv6: "fd3a:d409::10"
iborder_desc: "iborder-rtr"

rpki_cache_1: "100.64.0.70"
rpki_port_1: "323"
rpki_cache_2: "100.64.0.71"
rpki_port_2: "323"

irr_as_set: AS64135:AS-LabRPKIprivadosTutores
```

As variáveis a serem modificadas com os valores atribuídos a cada grupo são:
> **ansible_password:** 
>
> **my_asn:**
>
> **my_router_id:**



## Mãos à obra!!!!

A seguir, descrevemos cada tarefa definida no playbook e os comandos a serem utilizados para aplicá-las

#### T0: Limpeza da configuração manual aplicada na prática

Neste passo preliminar, retornamos a configuração do roteador ao estado inicial.

Podemos ver a tarefa a ser executada no arquivo `clean_confg.yml`

```
$ cat ios/t0_clean_confg.yml
```

Para executar essa tarefa, usamos o comando:

```
$ ansible-playbook -i hosts -t T0 playbook.yml
```

Verificamos a alteração da configuração no roteador:

```
grpX-rtr# sh run
```

#### T1: Configuramos o validador RPKI
O arquivo `rpki.yml` descreve a tarefa a ser executada:
```
$ cat ios/t1_rpki.yml
```

Para executar essa tarefa, usamos o comando:
```
$ ansible-playbook -i hosts -t T1 playbook.yml
```


#### T2: Criamos as prefix-lists PERMIT-ALL e DENY-ALL

O arquivo `prefix_lists.yml` descreve a tarefa a ser executada:

```
$ cat ios/t2_prefix_lists.yml
```

Para executar essa tarefa, usamos o comando:
```
$ ansible-playbook -i hosts -t T2 playbook.yml
```

#### T3: Criamos os route-maps TODO-IPv4 e TODO-IPv6

O arquivo `route_maps.yml` descreve a tarefa a ser executada:

```
$ cat ios/t3_route_maps_todo.yml
```

Para executar essa tarefa, usamos o comando:
```
$ ansible-playbook -i hosts -t T3 playbook.yml
```

#### T4: Configuramos o roteador BGP AS 6500X

O arquivo `bgp.yml` descreve a tarefa a ser executada:

```
$ cat ios/t4_bgp.yml
```

Para executar essa tarefa, usamos o comando:

```
$ ansible-playbook -i hosts -t T4 playbook.yml
```

#### T5: Configuramos os neighbors BGP a partir do BGP Address Family

O arquivo `bgp_af.yml` descreve a tarefa a ser executada:

```
$ cat ios/t5_bgp_af.yml
```

Para executar essa tarefa, usamos o comando:

```
$ ansible-playbook -i hosts -t T5 playbook.yml
```

#### T6: Geramos as bgp as-path access-lists usando o comando bgpq4 e aplicamos ao route-map PERMIT-SOME-ASN

O arquivo `bgp.yml` descreve a tarefa a ser executada:

```
$ cat ios/t6_bgp_as_path.yml
```

Para executar essa tarefa, usamos o comando:

```
$ ansible-playbook -i hosts -t T6 playbook.yml
```

#### T7: Aplicamos agora o Route-Map PERMIT-SOME-ASN na entrada do neighbor BGP

O arquivo `route_map_some_asn.yml` descreve a tarefa a ser executada:

```
$ cat ios/t7_bgp_route_map_some_asn.yml
```

Para executar essa tarefa, usamos o comando:

```
$ ansible-playbook -i hosts -t T7 playbook.yml
```

#### T8: Adicionamos o match as-path no route-map RPKI (valid, notfound)

O arquivo `route_map_rpki_aspath.yml` descreve a tarefa a ser executada:

```
$ cat ios/t8_route_map_rpki_aspath.yml
```

Para executar essa tarefa, usamos o comando:

```
$ ansible-playbook -i hosts -t T8 playbook.yml
```

#### T9: Aplicamos agora o Route-Map RPKI na entrada do neighbor BGP

O arquivo `bgp_route_map_rpki_aspath.yml` descreve a tarefa a ser executada:

```
$ cat ios/t9_bgp_route_map_rpki_aspath.yml
```

Para executar essa tarefa, usamos o comando:

```
$ ansible-playbook -i hosts -t T9 playbook.yml
```


#### T0-T9: Executar Playbook completo

Para executar o playbook completo, incluindo todas as tarefas e invocando todas as tags, utilizamos ***"all"*** como referência:

```
$ ansible-playbook -i hosts -t all playbook.yml
```

Se quisermos executar apenas algumas tarefas específicas, podemos selecioná-las como uma lista. Por exemplo, para executar as tarefas de T0 a T6:

```
$ ansible-playbooks -i hosts -t T0,T1,T2,T3,T4,T5,T6 playbook.yml
```

