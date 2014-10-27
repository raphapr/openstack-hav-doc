### Tutorial de Instalação do OpenStack Havana<br/>

**Autor:** Raphael Pereira Ribeiro

A idéia deste tutorial é proporcionar um caminho simplificado para implementação de uma estrutura de computação em nuvem utilizando OpenStack, preocupando-se mais com aspectos técnicos do que conceitos gerais.

Trabalharemos em ambiente Linux, mais especificamente teremos como base o Debian 7.0 Linux e OpenStack versão Havana.

Utilizaremos duas máquinas ( do cluster ) modelo RU, cada máquina conta com:
- 2 processadores Intel(R) Xeon(R) CPU X5570
- 24GB de memória RAM
- 128GB de disco (armazenamento HDD)
- 2 interface infiniband ( rede de interconexão de alto desempenho )
- 4 interfaces Gigabit Ethernet

As máquinas estão ligadas por três redes:
- Uma rede llom ( rede de gerência através da qual obtemos acesso remoto as máquinas )
- Duas redes Gigabit Ethernet
São Necessárias apenas duas interconexões de rede Gigabit Ethernet.

A partir de agora chamaremos as duas máquinas respectivamente de controller e compute.
- nó controller -> máquina que hospedará os serviços de gerência ( Keystone, Glance, Nova e Horizon )
- nó compute -> máquina que hospedará as máquinas virtuais
O serviço Nova é hospedado em ambas as máquinas.

O processo de implementação, neste tutorial, segue basicamente 6 tópicos:

1. Configurações Básicas do Sistema Operacional
2. Serviço de Identificação ( Keystone )
3. Serviço de gerência de Imagens ( Glance )
4. Serviço de Computação ( Nova )
5. Serviço de Orquestração ( Heat )
6. Painel de Controle ( Horizon )


### 1. Configurações Básicas do Sistema Operacional


A arquitetura de rede dos dois nós será da seguinte forma:

![alt text](https://raw.githubusercontent.com/raphapr/openstack-havana-installation/master/img/arch.png "Topologia")
<URL DA IMAGEM>

**Exemplo 1 (controller) /etc/network/interfaces:**

```
# Internal Network
auto eth0
iface eth0 inet static
address 192.168.0.10
netmask 255.255.255.0
# External Network
auto eth1
iface eth1 inet static
address 10.0.0.10
netmask 255.255.255.0

```

Configurar da mesma maneira para o compute node, apenas mudando o endereço para 10.0.0.11 e 192.168.0.11

Depois, restarte o serviço de rede:

```
# service networking restart
```

É importante também mudar o hostname de cada máquina. O nome do controller node de controller e o primeiro compute node de compute1.

```
# hostname controller
```

(Fazer o mesmo pra o compute node)

Edite o seguinte arquivo para que a mudança permaneça após o reboot:

**Exemplo 2: /etc/hostname**

```
controller
```

(Fazer o mesmo pra o compute node)

Por último, para garantir que cada nó alcance o outro através do hostname, é necessário editar o arquivo /etc/hosts para cada sistema:

**Exemplo 3: /etc/hosts**

```
127.0.0.1       localhost
192.168.0.10    controller
192.168.0.11    compute1
```

#### 1.2. NTP


Para sincronizar os serviços do OpenStack em múltiplas máquinas, você precisa installar o NTP:

```
# apt-get install ntp
```

Configure os seguintes servidores NTP no /etc/ntp.conf:

```
server 0.br.pool.ntp.org
server 3.south-america.pool.ntp.org
server 2.south-america.pool.ntp.org
```

- **Problema encontrado**

O problema se deu pois não estava sendo possível sincronizar o controller e o compute1, a solução foi que o controller estava em uma timezone diferente, o que não foi percebido inicialmente.

- **Solução**

Mudar a timezone no Debian Wheezy:

```
# mv /etc/localtime /etc/localtime.old
# cp /usr/share/zoneinfo/America/Maceio /etc/localtime
```


#### 1.3. MySQL



**Controller Node**
```
# apt-get install python-mysqldb mysql-server
```

Edite o /etc/mysql/my.cnf pra configurar o bind-address para o endereço de IP interno do controller:
```
[mysqld]
...
bind-address = 192.168.0.10
```
Reinicie o serviço MySQL:
```
# service mysql restart
```

**Compute Node**

```
# apt-get install python-mysqldb
```

- **Problema encontrado**

Após instalar o MySQL no controller, na tentativa de mudar a senha root pelo comando “mysql -u root -p” acusava o seguinte erro: mySQL ERROR 1045 (28000): Access denied for user 'root'@'localhost.

- **Solução**

http://dev.mysql.com/doc/refman/5.0/en/resetting-permissions.html#resetting-permissions-windows


### 1.4. Pacotes OpenStack


Installe o RabbitMQ apenas no controller node:
```
# apt-get install rabbitmq-server
```
O rabbitmq-server configura o RabbitMQ para startar automaticamente com o usuário guest, que utilizaremos nesse tutorial, com uma senha padrão, recomendável mudar essa senha:
```
# rabbitmqctl change_password guest RABBIT_PASS
```

### 1.5 Python-argparse


Instalar em ambos os nós:
```
# apt-get install python-argparse
```

## 2. Configurando o identity service


- Não houve nenhum problema encontrado na instalação deste serviço.


Instale o OpenStack Identity Service

```
# apt-get install keystone
```

Edite /etc/keystone/keystone.conf e mude a seção [sql].
```
...
[sql]
# The SQLAlchemy connection string used to connect to the database
connection = mysql://keystone:KEYSTONE_DBPASS@controller/keystone
…
```

Crie um banco de dados para o serviço keystone.

```
# mysql -u root -p
mysql> CREATE DATABASE keystone;
mysql> GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'localhost' \
IDENTIFIED BY 'KEYSTONE_DBPASS';
mysql> GRANT ALL PRIVILEGES ON keystone.* TO 'keystone'@'%' \
IDENTIFIED BY 'KEYSTONE_DBPASS';
```

Crie as tabelas de banco de dados:

```
# keystone-manage db_sync
```

Gere um token de autorização  para que o Keystone possa se comunicar com os outros serviços do OpenStack.

```
# openssl rand -hex 10
```

Edite /etc/keystone/keystone.conf e mude a seção [DEFAULT], substituindo ADMIN_TOKEN pelo resultado do comando.

```
[DEFAULT]
# A "shared secret" between keystone and other openstack services
admin_token = ADMIN_TOKEN
…
```

Reinicie o serviço

```
# service keystone restart
```


### 2.2 Definindo users, tenants, and roles


Para ter autorização ao Keystone, você precisa se autenticar à ele da seguinte forma:

```
# export OS_SERVICE_TOKEN=ADMIN_TOKEN
# export OS_SERVICE_ENDPOINT=http://controller:35357/v2.0
```

Onde ADMIN_TOKEN é o TOKEN de autorização criado na seção 2.1

Crie um tenant para o usuário admin e outro para outros serviços OpenStack.

```
# keystone tenant-create --name=admin --description="Admin Tenant"
# keystone tenant-create --name=service --description="Service Tenant"
```

Logo após, crie um usuário admin.

```
# keystone user-create --name=admin --pass=ADMIN_PASS \
   --email=admin@example.com
```

Crie um role para tarefas administrativas.

```
# keystone role-create --name=admin
```

Atribua o role e tenant ao usuário.

```
# keystone user-role-add --user=admin --tenant=admin --role=admin
```

### 2.3 Definindo serviços e API endpoints


Para que o Identity Service possa rastrear outros serviços OpenStack instalados e localizá-los na rede, vocẽ precisa registrar cada serviço no Keystone. Pra registrar um serviço, rode os seguintes comandos:

Crie uma entrada do serviço.

```
# keystone service-create --name=keystone --type=identity \
  --description="Keystone Identity Service"
+-------------+----------------------------------+
|   Property  |              Value               |
+-------------+----------------------------------+
| description | Keystone Identity Service        |
| id          | 15c11a23667e427e91bc31335b45f4bd |
| name        | keystone                         |
| type        | identity                         |
+-------------+----------------------------------+
```

Especifique um API endpoint para o identity Service usando a ID de retorno do comando anterior.

```
# keystone endpoint-create \
  --service-id=the_service_id_above \
  --publicurl=http://controller:5000/v2.0 \
  --internalurl=http://controller:5000/v2.0 \
  --adminurl=http://controller:35357/v2.0
+-------------+-----------------------------------+
|   Property  |             Value                 |
+-------------+-----------------------------------+
| adminurl    | http://controller:35357/v2.0      |
| id          | 11f9c625a3b94a3f8e66bf4e5de2679f  |
| internalurl | http://controller:5000/v2.0       |
| publicurl   | http://controller:5000/v2.0       |
| region      | regionOne                         |
| service_id  | 15c11a23667e427e91bc31335b45f4bd  |
+-------------+-----------------------------------+
```

Ao longo da instalação dos serviços OpenStack, você precisará registrar os serviços no Identify Service (Keystone).


### 2.4 Verificando a instalação do Identify Service


É importante criar um arquivo local com as credenciais do Keystone e carregar suas informações nas variáveis do ambiente:

Crie um arquivo ~/.keystonerc:
```
export OS_USERNAME=admin
export OS_PASSWORD=ADMIN_PASS
export OS_TENANT_NAME=admin
export OS_AUTH_URL=http://controller:35357/v2.0
```

Carregue-o:
```
# source ~./.keystonerc
# echo “ source  ~/.keystonerc” >> ~/.bash_profile
```

Finalmente, para verificar se o Keystone está funcionando juntamente com suas credenciais, insira o comando abaixo.

```
# keystone user-list
+----------------------------------+---------+--------------------+--------+
|                id                | enabled | email              |  name  |
+----------------------------------+---------+--------------------+--------+
| a4c2d43f80a549a19864c89d759bb3fe | True    | admin@example.com  | admin  |
```


## 3. Instalando o Image Service (Glance)


- Não houve nenhum problema encontrado na instalação deste serviço.


Instale o Image Service no controller node.
```
# apt-get install glance python-glanceclient
```

Edite /etc/glance/glance-api.conf e /etc/glance/glance-registry.conf e mude a seção [DEFAULT]

```
...
[DEFAULT]
...
# SQLAlchemy connection string for the reference implementation
# registry server. Any valid SQLAlchemy connection string is fine.
# See: http://www.sqlalchemy.org/docs/05/reference/sqlalchemy/connections.html#sqlalchemy.create_engine
sql_connection = mysql://glance:GLANCE_DBPASS@controller/glance
…
```

Crie um usuário glance no banco de dados MySQL.

```
# mysql -u root -p
mysql> CREATE DATABASE glance;
mysql> GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'localhost' \
IDENTIFIED BY 'GLANCE_DBPASS';
mysql> GRANT ALL PRIVILEGES ON glance.* TO 'glance'@'%' \
IDENTIFIED BY 'GLANCE_DBPASS';
```

Gere as tabelas de banco de dados do Glance.
```
# glance-manage db_sync
```

Crie um usuário e role glance no Identity Service para que possa se autenticar com o Keystone.
```
# keystone user-create --name=glance --pass=GLANCE_PASS \
   --email=glance@example.com
# keystone user-role-add --user=glance --tenant=service --role=admin
```

Configure o Image Service para usar o Identity Service para autenticação.
Edite os arquivos /etc/glance/glance-api.conf e /etc/glance/glance-registry.conf:

```
[keystone_authtoken]
...
auth_uri = http://controller:5000
auth_host = controller
auth_port = 35357
auth_protocol = http
admin_tenant_name = service
admin_user = glance
admin_password = GLANCE_PASS
```

Adicione a linha seguinte na seção [paste_deploy].

```
[paste_deploy]
...
flavor = keystone
```

Adicione as credenciais nos arquivos /etc/glance/glance-api-paste.ini e /etc/glance/glance-registry-paste.ini.
Edite cada arquivo com as seguintes opções na seção [filter:authtoken] e deixe todas as outras opções existentes.

```
[filter:authtoken]
paste.filter_factory=keystoneclient.middleware.auth_token:filter_factory
auth_host=controller
admin_user=glance
admin_tenant_name=service
admin_password=GLANCE_PASS
```

Registre o Image Service no keystone.
```
# keystone service-create --name=glance --type=image \
  --description="Glance Image Service"
```

Use o ID retornado com o comando acima para criar um endpoint.
```
# keystone endpoint-create \
  --service-id=the_service_id_above \
  --publicurl=http://controller:9292 \
  --internalurl=http://controller:9292 \
  --adminurl=http://controller:9292
```

Reinicie o glance
```
# service glance-registry restart
# service glance-api restart
```

### 3.2 Verificando a instalação do Image Service (Glance)

Baixe uma imagem para testes (cirrOS) usando wget.
```
# mkdir images
# cd images/
# wget http://download.cirros-cloud.net/0.3.1/cirros-0.3.1-x86_64-disk.img
```

- Repositório: http://download.cirros-cloud.net/

Faça o upload dessa imagem através do Image Service.
```
# glance image-create --name="CirrOS 0.3.1" --disk-format=qcow2 \
  --container-format=bare --is-public=true < cirros-0.3.1-x86_64-disk.img
+------------------+--------------------------------------+
| Property         | Value                                |
+------------------+--------------------------------------+
| checksum         | d972013792949d0d3ba628fbe8685bce     |
| container_format | bare                                 |
| created_at       | 2013-10-08T18:59:18                  |
| deleted          | False                                |
| deleted_at       | None                                 |
| disk_format      | qcow2                                |
| id               | acafc7c0-40aa-4026-9673-b879898e1fc2 |
| is_public        | True                                 |
| min_disk         | 0                                    |
| min_ram          | 0                                    |
| name             | CirrOS 0.3.1                         |
| owner            | efa984b0a914450e9a47788ad330699d     |
| protected        | False                                |
| size             | 13147648                             |
| status           | active                               |
| updated_at       | 2013-05-08T18:59:18                  |
+------------------+--------------------------------------+
```

Confirme o upload da imagem mostrando seus  atributos através do comando:
```
# glance image-list
+--------------------------------------+-----------------+-------------+------------------+----------+--------+
| ID                                   | Name            | Disk Format | Container Format | Size     | Status |
+--------------------------------------+-----------------+-------------+------------------+----------+--------+
| acafc7c0-40aa-4026-9673-b879898e1fc2 | CirrOS 0.3.1    | qcow2       | bare             | 13147648 | active |
+--------------------------------------+-----------------+-------------+------------------+----------+--------+

```

## 4. Configurando os serviços compute (Nova)

### 4.2. Instalando os serviços compute no controller node

* Não houve nenhum problema encontrado na instalação deste serviço.


Instale esses pacotes OpenStack, que fornece os serviços compute no controller.
```
# apt-get install nova-consoleproxy nova-api \
  nova-cert nova-conductor nova-consoleauth \
  nova-scheduler python-novaclient
```

Configure a localização do banco de dados, rabbitMQ e rede editando o arquivo /etc/nova/nova.conf e deixando semelhante ao arquivo do link abaixo:

- http://pastebin.com/THRr4eCR

Lembrando de alterar NOVA_DBPASS, NOVA_PASS e RABBIT_PASS.

Crie um usuário nova no banco de dados MySQL com a senha NOVA_DBPASS definida no arquivo /etc/nova/nova.conf.
```
# mysql -u root -p
mysql> CREATE DATABASE nova;
mysql> GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'localhost' \
IDENTIFIED BY 'NOVA_DBPASS';
mysql> GRANT ALL PRIVILEGES ON nova.* TO 'nova'@'%' \
IDENTIFIED BY 'NOVA_DBPASS';
```

Crie as tabelas do Compute Service.
```
# nova-manage db sync
```

Crie um usuário e role do Nova no Keystone.
```
# keystone user-create --name=nova --pass=NOVA_PASS --email=nova@example.com
# keystone user-role-add --user=nova --tenant=service --role=admin
```

Adicione as credentiais no arquivo /etc/nova/api-paste.ini, na seção [filter:authtoken]:
```
[filter:authtoken]
paste.filter_factory = keystoneclient.middleware.auth_token:filter_factory
auth_host = controller
auth_port = 35357
auth_protocol = http
auth_uri = http://controller:5000/v2.0
admin_tenant_name = service
admin_user = nova
admin_password = NOVA_PASS
```
Agora você precisa registrar o serviço Nova no Keystone.
```
# keystone service-create --name=nova --type=compute \
  --description="Nova Compute service"
```

Use o ID retornado pelo comando acima para criar um endpoint no Keystone.
```
# keystone endpoint-create \
  --service-id=the_service_id_above \
  --publicurl=http://controller:8774/v2/%\(tenant_id\)s \
  --internalurl=http://controller:8774/v2/%\(tenant_id\)s \
  --adminurl=http://controller:8774/v2/%\(tenant_id\)s
```

Reinicie os serviços do Nova.
```
# service nova-api restart
# service nova-cert restart
# service nova-consoleauth restart
# service nova-scheduler restart
# service nova-conductor restart
# service nova-novncproxy restart
```

Verifique a configuração, listando as imagens disponíveis

```
# nova image-list
+--------------------------------------+-----------------+--------+--------+
| ID                                   | Name            | Status | Server |
+--------------------------------------+-----------------+--------+--------+
| acafc7c0-40aa-4026-9673-b879898e1fc2 | CirrOS 0.3.1    | ACTIVE |        |
+--------------------------------------+-----------------+--------+--------+
```

### 4.3. Instalando os serviços compute no compute node

Certifique-se de ter feito todos os passos da seção “1. Configurações Básicas do Sistema Operacional” no compute node.

Depois de ter configurado seu sistema, rode o seguinte comando para instalar os pacotes apropriados:

```
# apt-get install nova-compute-kvm python-guestfs nova-network nova-api-metadata
```


- Problemas encontrados

Houve muita dificuldade por causa da documentação oficial mal escrita, o problema é que o principal arquivo de configuração do nova, /etc/nova/nova.conf, possui algumas linhas não comentadas que podem atrapalhar na hora da configuração, então o ideal seria comentar todas elas, você deve adicionar as seguintes linhas tendo que ser alterado como se deve as principais flags: DB_PASS, RABBIT_PASS

```
[DEFAULT]
### DATABASE
connection = mysql://nova:NOVA_DBPASS@controller/nova
### APIs
enabled_apis=ec2,osapi_compute,metadata
api_paste_config=api-paste.ini
### SYSTEM
state_path=/var/lib/nova
lock_path=/var/lock/nova
rootwrap_config=/etc/nova/rootwrap.conf
### AUTHENTICATION
auth_strategy=keystone
### RABBITMQ
rpc_backend = nova.rpc.impl_kombu
rabbit_host = controller
rabbit_password = RABBIT_PASS
rabbit_virtual_host=/
### GLANCE
glance_host=controller
### Type of network APIs to use
network_api_class=nova.network.api.API
security_group_api = nova
### NETWORK (linux net)
dhcpbridge_flagfile=/etc/nova/nova.conf
dhcpbridge=/usr/bin/nova-dhcpbridge
network_manager=nova.network.manager.FlatDHCPManager
firewall_driver=nova.virt.libvirt.firewall.IptablesFirewallDriver
network_size=254
allow_same_net_traffic=False
send_arp_for_ha=True
flat_network_bridge=br100
flat_interface=eth1
public_interface=eth1
### NOVNC CONSOLE
vnc_enabled=True
vncserver_listen=0.0.0.0
vncserver_proxyclient_address=192.168.0.11
novncproxy_base_url=http://controller:6080/vnc_auto.html
```

No nova-compute.conf, editar a flag libvirt_vif_driver para libvirt_vif_driver=nova.virt.libvirt.vif.LibvirtGenericVIFDriver

Edite o arquivo /etc/nova/api-paste.ini e adicione as credenciais na seção [filter:authtoken]:
```
[filter:authtoken]
paste.filter_factory = keystoneclient.middleware.auth_token:filter_factory
auth_host = controller
auth_port = 35357
auth_protocol = http
admin_tenant_name = service
admin_user = nova
admin_password = NOVA_PASS
```

Reinicie o compute service e o nova-network.

```
# service nova-compute restart
# service nova-network restart
```

**AVISO:** A partir daqui, em se tratando da rede do OpenStack muitos problemas inesperados podem surgir, sempre olhar os logs /var/log/nova/

Crie a rede das máquinas virtuais.
```
# nova network-create vmnet --fixed-range-v4=10.0.0.0/24 \
  --bridge=br100 --multi-host=T
```

### 4.4 Lançando uma instância

Você deverá criar um par de chaves (pública/privada) para estar apto a lançar instâncias no OpenStack e acessá-las via SSH.

```
$ ssh-keygen
$ cd .ssh
$ nova keypair-add --pub_key id_rsa.pub mykey
```

Cerfique-se de que você adicionou as chaves no Nova.

```
$ nova keypair-list
+--------+-------------------------------------------------+
|  Name  |                   Fingerprint                   |
+--------+-------------------------------------------------+
| mykey  | b0:18:32:fa:4e:d4:3c:1b:c4:6c:dd:cb:53:29:13:82 |
+--------+-------------------------------------------------+
```

Escolha um flavor.
```
$ nova flavor-list
```

Pegue o ID da imagem que você usará na instância.
```
$ nova image-list
+--------------------------------------+--------------+--------+--------+
| ID                                   | Name         | Status | Server |
+--------------------------------------+--------------+--------+--------+
| 9e5c2bee-0373-414c-b4af-b91b0246ad3b | CirrOS 0.3.1 | ACTIVE |        |
+--------------------------------------+--------------+--------+--------+
```

Para poder usar ping e ssh, você deve configurar as regras do seu security group.

```
# nova secgroup-add-rule default tcp 22 22 0.0.0.0/0
# nova secgroup-add-rule default icmp -1 -1 0.0.0.0/0
```

Lance a instância:
```
$ nova boot --flavor flavorType --key_name keypairName --image ID newInstanceName
```

**Exemplo:**
```
$ nova boot --flavor 1 --key_name mykey --image 9e5c2bee-0373-414c-b4af-b91b0246ad3b --security_group default cirrOS
+--------------------------------------+--------------------------------------+
| Property                             | Value                                |
+--------------------------------------+--------------------------------------+
| OS-EXT-STS:task_state                | scheduling                           |
| image                                | CirrOS 0.3.1                         |
| OS-EXT-STS:vm_state                  | building                             |
| OS-EXT-SRV-ATTR:instance_name        | instance-00000001                    |
| OS-SRV-USG:launched_at               | None                                 |
| flavor                               | m1.tiny                              |
| id                                   | 3bdf98a0-c767-4247-bf41-2d147e4aa043 |
| security_groups                      | [{u'name': u'default'}]              |
| user_id                              | 530166901fa24d1face95cda82cfae56     |
| OS-DCF:diskConfig                    | MANUAL                               |
| accessIPv4                           |                                      |
| accessIPv6                           |                                      |
| progress                             | 0                                    |
| OS-EXT-STS:power_state               | 0                                    |
| OS-EXT-AZ:availability_zone          | nova                                 |
| config_drive                         |                                      |
| status                               | BUILD                                |
| updated                              | 2013-10-10T06:47:26Z                 |
| hostId                               |                                      |
| OS-EXT-SRV-ATTR:host                 | None                                 |
| OS-SRV-USG:terminated_at             | None                                 |
| key_name                             | mykey                                |
| OS-EXT-SRV-ATTR:hypervisor_hostname  | None                                 |
| name                                 | cirrOS                               |
| adminPass                            | DWCDW6FnsKNq                         |
| tenant_id                            | e66d97ac1b704897853412fc8450f7b9     |
| created                              | 2013-10-10T06:47:23Z                 |
| os-extended-volumes:volumes_attached | []                                   |
| metadata                             | {}                                   |
+--------------------------------------+--------------------------------------+
```

Depois de lançar a instância, use nova list para ver seu status.

```
$ nova list
```

## 5. Instalando o serviço de Orquestração Heat

- Não houve nenhum problema encontrado na instalação deste serviço.

Instale o módulo de orquestração Heat no controller node:

```
# apt-get install heat-api heat-api-cfn heat-engine

```

No arquivo de configuração, especifique a localização do banco de dados onde o heat irá armazenar os dados:


Edite /etc/heat/heat.conf e mude a seção [DEFAULT]
```
[DEFAULT]
# The SQLAlchemy connection string used to connect to the database
sql_connection = mysql://heat:HEAT_DBPASS@controller/heat
...
```

Crie o banco de dados para o serviço heat:

```
# mysql -u root -p
mysql> CREATE DATABASE heat;
mysql> GRANT ALL PRIVILEGES ON heat.* TO 'heat'@'localhost' \
IDENTIFIED BY 'HEAT_DBPASS';
mysql> GRANT ALL PRIVILEGES ON heat.* TO 'heat'@'%' \
IDENTIFIED BY 'HEAT_DBPASS';
```

Crie as tabelas do serviço heat:

```
# heat-manage db_sync
```

### 5.2 Verificando a instalação do heat

#### 5.2.1 Crie um stack com um arquivo template

Pra criar um stack, baixe um [template exemplo](https://github.com/openstack/heat-templates) e rode o seguinte comando:
```
heat stack-create mystack --template-file=/PATH_TO_HEAT_TEMPLATES/WordPress_Single_Instance.template
     --parameters="InstanceType=m1.large;DBUsername=USERNAME;DBPassword=PASSWORD;KeyName=HEAT_KEY;LinuxDistribution=F17"
```

Os valores *--parameters* que você especifica depende dos definidos no template.

O comando terá como saída:

```
+--------------------------------------+---------------+--------------------+----------------------+
| id           | stack_name    | stack_status       | creation_time        |
+--------------------------------------+---------------+--------------------+----------------------+
| 4c712026-dcd5-4664-90b8-0915494c1332 | mystack       | CREATE_IN_PROGRESS | 2013-04-03T23:22:08Z |
+--------------------------------------+---------------+--------------------+----------------------+
```

Você também pode usar o comando *stack-create* para validar um template sem criar um stack:


```
heat stack-create mystack --template-file=/PATH_TO_HEAT_TEMPLATES/WordPress_Single_Instance.template
```

Se a validação falhar, a resposta irá retornar uma mensagem de erro.

### 5.2.2 Obtenha informações dos stacks

Explore o estado e historico de uma stack específica, com os comandos abaixo.


- Listar os stacks atuais

```
$ heat stack-list
+--------------------------------------+---------------+-----------------+----------------------+
| id           | stack_name    | stack_status    | creation_time        |
+--------------------------------------+---------------+-----------------+----------------------+
| 4c712026-dcd5-4664-90b8-0915494c1332 | mystack       | CREATE_COMPLETE | 2013-04-03T23:22:08Z |
| 7edc7480-bda5-4e1c-9d5d-f567d3b6a050 | my-otherstack | CREATE_FAILED   | 2013-04-03T23:28:20Z |
+--------------------------------------+---------------+-----------------+----------------------+
```

- Mostrar os detalhes de um stack

```
$ heat resource-list mystack
+---------------------+--------------------+-----------------+----------------------+
| logical_resource_id | resource_type      | resource_status | updated_time         |
+---------------------+--------------------+-----------------+----------------------+
| WikiDatabase        | AWS::EC2::Instance | CREATE_COMPLETE | 2013-04-03T23:25:56Z |
+---------------------+--------------------+-----------------+----------------------+
```

- Um stack consiste em uma coleção de recursos, liste seus recursos

```
$ heat resource-show mystack WikiDatabase

```

- Uma série de eventos é gerado durante o ciclo de vida de um stack, mostre seu ciclo de vida

```
$ heat event-list mystack
+---------------------+----+------------------------+-----------------+----------------------+
| logical_resource_id | id | resource_status_reason | resource_status | event_time           |
+---------------------+----+------------------------+-----------------+----------------------+
| WikiDatabase        | 1  | state changed          | IN_PROGRESS     | 2013-04-03T23:22:09Z |
| WikiDatabase        | 2  | state changed          | CREATE_COMPLETE | 2013-04-03T23:25:56Z |
+---------------------+----+------------------------+-----------------+----------------------+
```

- Veja detalhes de um evento particular

```
$ heat event-show WikiDatabase 1
```

- Atualize o template de um stack

```
$ heat stack-update mystack --template-file=/path/to/heat/templates/WordPress_Single_Instance_v2.template
     --parameters="InstanceType=m1.large;DBUsername=wp;DBPassword=verybadpassword;KeyName=heat_key;LinuxDistribution=F17"
     +--------------------------------------+---------------+-----------------+----------------------+
     | id           | stack_name    | stack_status    | creation_time        |
     +--------------------------------------+---------------+-----------------+----------------------+
     | 4c712026-dcd5-4664-90b8-0915494c1332 | mystack       | UPDATE_COMPLETE | 2013-04-03T23:22:08Z |
     | 7edc7480-bda5-4e1c-9d5d-f567d3b6a050 | my-otherstack | CREATE_FAILED   | 2013-04-03T23:28:20Z |
     +--------------------------------------+---------------+-----------------+----------------------+
```


##6. Adicionando o Dashboard (Horizon)


- Não houve nenhum problema encontrado na instalação deste serviço.


Instale os seguintes pacotes no controller node.
```
# apt-get install memcached libapache2-mod-wsgi openstack-dashboard
```

Abra o /etc/openstack-dashboard/local_settings.py e procure por essa linha:
```
CACHES = {
'default': {
'BACKEND' : 'django.core.cache.backends.memcached.MemcachedCache',
'LOCATION' : '127.0.0.1:11211'
}
}
```

O endereço e a porta devem ser o mesmo que definido pelo arquivo /etc/memcached.conf

Atualize o ALLOWED_HOSTS em local_settings.py para incluir os endereços que você deseja que acessem o dashboard:
ALLOWED_HOSTS = ['localhost', 'my-desktop']

Edite também o OPENSTACK_HOST para controller:
```
OPENSTACK_HOST = "controller"
```

Reinicie o Apache Web e memcached.
```
# service apache2 restart
# service memcached restart
```
Agora você pode acessar o dashboard a partir do endereço https://controller/

### 7. Links importantes para Troubleshooting

- http://virtual2privatecloud.com/troubleshooting-openstack-nova/
- http://wiki.libvirt.org/page/VM_lifecycle
- https://access.redhat.com/site/documentation/en-US/Red_Hat_Enterprise_Linux/6/html/Virtualization_Host_Configuration_and_Guest_Installation_Guide/apb.html
- https://ask.openstack.org/en/question/12623/i-cannot-create-vms-libvirt-error/
- https://ask.openstack.org/en/question/8581/no-connection-to-metadata-service-nova-network/?answer=8611#post-id-8611
- https://ask.openstack.org/en/question/5142/nova-network-create-fails-with-401-unauthorized/
- https://ask.openstack.org/en/question/3542/error-installing-nova-compute/

## 8. Referência
- http://docs.openstack.org/havana/install-guide/install/apt/content/
- http://docs.openstack.org/user-guide/content/

<br/>

<a rel="license" href="http://creativecommons.org/licenses/by-nc-sa/4.0/"><img alt="Creative Commons License" style="border-width:0" src="https://i.creativecommons.org/l/by-nc-sa/4.0/88x31.png" /></a><br />This work is licensed under a <a rel="license" href="http://creativecommons.org/licenses/by-nc-sa/4.0/">Creative Commons Attribution-NonCommercial-ShareAlike 4.0 International License</a>.
