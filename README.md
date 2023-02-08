# Introdução

A computação em nuvem é um modelo que permite acesso a recursos como aplicativos, armazenamento e processamento, via internet. Em vez de ter esses recursos localmente 
em um dispositivo individual, os usuários podem acessá-los a partir de uma infraestrutura remota mantida por provedores de nuvem. Isso permite que os usuários 
economizem tempo e recursos, já que não precisam gerenciar e manter as infraestruturas de tecnologia de forma independente.

Nos últimos anos a computação em nuvem tem se tornado cada vez mais popular devido às suas vantagens em relação à flexibilidade, escalabilidade e redução de custos, 
permitindo que as organizações ajustem rapidamente sua demanda por recursos de tecnologia de acordo com suas necessidades e como os provedores de nuvem geralmente 
mantêm as infraestruturas de tecnologia, as organizações podem se concentrar em seus negócios, sem se preocupar com questões de TI.

Neste trabalho serão aplicados conceitos da computação em nuvem para a implementação de uma nuvem privada, abordando a escolhas das tecnologias, suas configurações e 
os desafios enfrentados durante o processo. Esta nuvem deve rodar uma aplicação de modo que possa ser acessada por outras máquinas sem que seja necessário a conexão 
com a internet.

A nuvem privada é um modelo de computação em nuvem em que as infraestruturas de computação, armazenamento e rede são provisionadas e gerenciadas para uso exclusivo 
por uma organização, geralmente dentro de sua própria rede corporativa. Ela é diferente de nuvens públicas, onde os recursos são compartilhados entre vários clientes, 
e de nuvens híbridas, que combinam recursos de nuvem privada e pública. Além disso, oferece aos usuários mais controle sobre seus dados e aplicativos e maior 
flexibilidade em termos de configuração e gerenciamento. No entanto a implementação e manutenção de uma nuvem privada podem ser mais caras e exigir mais recursos do 
que os outros modelos.

Na próxima seção vão ser apresentados com detalhes as escolhas feitas durante a implementação do trabalho prático, como ambiente de desenvolvimento, 
sistema operacional e a plataforma escolhida, e também as configurações feitas para execução da nuvem.

# Implementação

Iniciando o desenvolvimento do trabalho a primeira a escolha a ser feita foi a de qual sistema operacional utilizar. Pela flexibilidade, segurança e estabilidade foi 
escolhido o Linux, visto também que a implementação da nuvem em sistemas como o Windows seria muito complexo.

A segunda escolha realizada no trabalho foi de qual plataforma implementar e por possuir uma boa documentação e conteúdos interessantes disponíveis na internet foi 
escolhida a OpenStack. Para criar um ambiente completo de forma rápida e atualizada foi empregado o DevStack, uma série de scripts usados interativamente como um 
ambiente de desenvolvimento e como base para grande parte dos testes funcionais do projeto OpenStack.

Após isto chegou a hora de começar de fato o desenvolvimento da nuvem privada, mas como meu notebook não possuia o sistema operacional Linux optei então por criar 
uma máquina virtual que o simulasse utilizando o Oracle VM VirtualBox. Esta MV foi criada levando em conta os requisitos mínimos apresentados na documentação e o que 
eu achei necessário para que ela apresentasse um bom desempenho. A versão do Linux escolhida foi o Ubuntu 22.04.1 LTS.

Ao concluir com êxito a instalação do Ubuntu executei os seguintes comandos para atualizar o sistema e instalar os programas que vão ser aplicados na implementação da 
nuvem privada.

~~~
    > sudo apt update
    > sudo apt upgrade
    > sudo apt install git
    > sudo apt install net-tools
    > sudo apt install python3-pip
    > sudo apt-get install openssh-server
~~~

Com todos os programas devidamente instalados e atualizados comecei a parte da configuração da rede, onde usei como suporte a documentação do DevStack [2] e o vídeo 
"OpenStack : Bringing up DevStack on Ubuntu 20.04" disponibilizado no Youtube no canal BitsPlease [1], que apesar de ser de uma versão antiga do Ubuntu abordava de 
forma bem explicativa a mesma situação proposta pelo trabalho.

A primeira etapa da configuração foi tornar o IP da máquina virtual um host, para isso ele foi obtido através do comando:

~~~
    > sudo ifconfig
~~~

E para tornar definitivamente o IP em um host foi executado o comando abaixo, levando em consideração o usuário da máquina virtual e o IP obtido na resposta do comando
executado anteriormente.

~~~
    > ssh <user>@<IP>
~~~

A próxima etapa foi criar um novo usuário stack através do root e liberar todas as permissões para que ele possa criar e configurar a nuvem, para isto foram utilizados
os seguintes comandos:

~~~
    > sudo -i
    > adduser stack
    > echo "stack ALL=(ALL) NOPASSWD: ALL" >> /etc/sudoers
~~~

Antes de realizar a troca para o novo usuário é necessário começar a configuração da rede acessando o arquivo referente ao gerenciamento da rede.

~~~
    > cd /etc/netplan
    > nano 01-network-manager-all.yaml
~~~

Logo após acessar o arquivo podem ser configurados o host, o gateway e o DNS.

~~~
network:
	ethernets:
		enp0s3:
			addresses:
				- 192.168.2.5/24
			routes:
				- to: default
				  via: 192.168.2.1
			nameservers:
				search: [home]
				addresses: [192.168.2.1, 8.8.8.8]
	version: 2				  	
~~~

Dando continuidade a configuração da rede o arquivo resolved.conf também tem que ser alterado, informando o DNS para ele.

~~~
    > cd /etc/systemd
    > nano resolved.conf
~~~

Abaixo pode ser vista a configuração do DNS.

~~~
DNS=192.168.2.1 8.8.8.8
~~~

Após isto as configurações iniciais da rede estão feitas, faltando somente ativá-las. Com este propósito foram efetuados os comandos a seguir.

~~~
    > systemctl start systemd-networkd
    > neplan apply
    > systemctl restart systemd-resolved
~~~

Também foi alterado o fuso horário do serviço.

~~~
    > timedatectl set-timezone America/Sao_Paulo
~~~

Com o ambiente adequadamente configurado finalmente foi feita a troca para o usuário stack e acessada sua pasta raiz.

~~~
    > su stack
    > cd ~
~~~

Na pasta raiz então realizei a clonagem do repositório do DevStack e então alterei para a versão Zed, que é a que conta atualmente com suporte.

~~~
    > git clone https://opendev.org/openstack/devstack
    > cd devstack
    > git checkout stable/zed
~~~

Antes de iniciar a execução dos scripts é preciso criar e configurar de acordo com a sua rede o arquivo local.conf.

~~~
    > nano local.conf
~~~

Em seguida estão exibidas as configurações realizadas no arquivo local.conf.

~~~
[[local|localrc]]

###########################################################################
### Passwords

ADMIN_PASSWORD=pass
DATABASE_PASSWORD=$ADMIN_PASSWORD
RABBIT_PASSWORD=$ADMIN_PASSWORD
SERVICE_PASSWORD=$ADMIN_PASSWORD

###########################################################################
### Logging

LOG_COLOR=False
LOGFILE=$DEST/logs/stack.sh.log

###########################################################################
### Networking

IP_VERSION=4
HOST_IP=192.168.2.5

# Enable internal DNS resolution in ml2_conf.ini

Q_ML2_PLUGIN_EXT_DRIVERS=port_security,dns_domain_ports

# The remaining network settings connect the cloud to the external network.

# START and END are the first and last IP address of a range that can be used
# for allocating floating IPs. I.e. this range is not used in your network.
Q_FLOATING_ALLOCATION_POOL=start=192.168.2.245,end=192.168.2.254
FLOATING_RANGE=192.168.2.0/24

# The router in your home or lab network
PUBLIC_NETWORK_GATEWAY=192.168.2.1

# The NIC that connects your machine to the home or lab network
PUBLIC_INTERFACE=enp0s3

Q_ASSIGN_GATEWAY_TO_PUBLIC_BRIDGE=FALSE

# Open vSwitch provider networking configuration
# These are probably default values
Q_USE_PROVIDERNET_FOR_PUBLIC=True
OVS_PHYSICAL_BRIDGE=br-ex
PUBLIC_BRIDGE=br-ex
OVS_BRIDGE_MAPPINGS=public:br-ex

###########################################################################
### Software sources

# Ensure the latest software is installed
PIP_UPGRADE=True

###########################################################################
### Glance

DOWNLOAD_DEFAULT_IMAGES=False
IMAGE_URLS="http://download.cirros-cloud.net/0.4.0/cirros-0.4.0-x86_64-disk.img"

###########################################################################
### Swift

# by default, Devstack interactively asks for a hash value. Avoid this.
SWIFT_HASH=123

enable_service swift

###########################################################################
### Heat

enable_plugin heat https://git.openstack.org/openstack/heat stable/ussuri
enable_service h-eng h-api h-api-cfn h-api-cw

enable_plugin heat-dashboard https://git.openstack.org/openstack/heat-dashboard stable/ussuri
enable_service heat-dashboard

###########################################################################
### Cinder: Enable backup

enable_service c-bak

###########################################################################

###########################################################################
### Post config for Neutron (DNS)

[[post-config|$NEUTRON_CONF]]
[DEFAULT]
dns_domain = devstack.org.
~~~

Para concluir a instalação da nuvem privada só falta a execução dos scripts, que pode ser realizada pelo seguinte comando.

~~~
    > ./stack.sh
~~~

Em seguida, caso não ocorra nenhum erro durante a execução dos scripts, já é possível acessar a nuvem privada digitando o IP da sua máquina na barra de pesquisa 
navegador, sendo requisitado o usuário e senha previamente cadastrados para efetuar o login.

# Conclusão

Chegando ao final do trabalho, fiquei com a sensação de um dever que não foi cumprido, isto por que enfrentei diversos problemas durante a implementação da nuvem, 
principalmente com versionamento dos componentes utilizados e também da configuração da rede em si, que após ler e procurar por soluções em vários sites e vídeos 
acabei conseguindo resolver e subir a nuvem privada.

Estava planejando então para realizar os testes a um dia da entrega do trabalho e ai vieram mais problemas com a conexão de internet na máquina virtual e então tive 
que reinstalar ela e realizar todo processo novamente, mas durante a execução dos scripts a minha cidade estava sendo afetada por uma forte chuva que gerou várias 
quedas de energia até uma cessão completa dela por mais de três horas. Quando a energia retornou tentei acessar a máquina virtual mas a mesma estava com mais problemas
e eu teria que realizar todo processo de novo e por já estar chegando perto do prazo de entrega optei por não realizar e entregar somente o relatório com tudo que fiz,
mesmo que faltando a parte dos testes e a aplicação rodando na minha nuvem privada e por isso o sentimento de dever não cumprido.

Porém, fica o aprendizado que obtive durante estas mais de duas semanas tentando desenvolver a minha nuvem privada.

# Referências

[1] BitsPlease. OpenStack : Bringing up DevStack on Ubuntu 20.04. https://www.youtube.com/watch?v=1uyQUU3gXZoab_channel=BitsPlease, 2021.

[2] OpenStack. DevStack. https://docs.openstack.org/devstack/latest/, January 2021.
