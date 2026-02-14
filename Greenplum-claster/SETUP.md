
--------------------------------------------------------
Как развернуть локально Greenplum в Oracle VirtualBox (Ubuntu)
--------------------------------------------------------

Установить виртуальную машину VirtualBox - https://www.virtualbox.org/wiki/Downloads
Установить образ Ubuntu 22.04 - https://releases.ubuntu.com/22.04/ubuntu-22.04.5-desktop-amd64.iso

#### Последняя open source версия Greenplum 6.26.4 не совместима с убунту версией выше 22

Запустить 4 машины со скаченным образом
На этой версии может не запускаться терминал, нужно поменять в настройках язык на English(United Kingdom) и перезагрузить машину


#### Установить название хоста
--------------------------------------------------------
sudo hostnamectl set-hostname master
sudo hostnamectl set-hostname seg1
sudo hostnamectl set-hostname seg2
sudo hostnamectl set-hostname seg3


#### Добавить пользователя gpadmin на всех машинах
--------------------------------------------------------
sudo adduser gpadmin
sudo usermod -aG sudo gpadmin

#### Добавить права root, перезапускаем машину и жмём shift для вызова меню GRUB.
#### Выбираем Advanced options for Ubuntu → Recovery Mode.
#### В консоле вводим команды

usermod -aG sudo gpadmin
reboot


#### Установить безпарольный доступ на сегментах для мастера
--------------------------------------------------------
sudo visudo
gpadmin ALL=(ALL) NOPASSWD: ALL # Добавить в конец


#### Установить ssh сервер на каждую машину
--------------------------------------------------------
sudo apt update
sudo apt install -y openssh-server
sudo systemctl enable --now ssh


#### Добавить ip машин в файл хостов ->>> sudo nano /etc/hosts

Узнать IP ->>> hostname -I
--------------------------------------------------------
192.168.206.128 master master
192.168.206.131 seg1 seg1
192.168.206.132 seg2 seg2
192.168.206.133 seg3 seg3


#### Сгенерировать ssh ключ на мастере и отправить на каждую машину
--------------------------------------------------------
ssh-keygen -t rsa -N "" -f ~/.ssh/id_rsa
ssh-copy-id gpadmin@seg1 # Потребуется подтверждение пароля
ssh-copy-id gpadmin@seg2
ssh-copy-id gpadmin@seg3


#### Добавить 
->>> sudo nano /etc/sysctl.conf 
--------------------------------------------------------

kernel.shmmax = 5000000000000
kernel.shmmni = 32768
kernel.shmall = 40000000000
kernel.sem = 1000 32768000 1000 32768
kernel.msgmnb = 1048576
kernel.msgmax = 1048576
kernel.msgmni = 32768

net.core.netdev_max_backlog = 80000
net.core.rmem_default = 2097152
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216

vm.overcommit_memory = 2
vm.overcommit_ratio = 95
EOF

sudo sysctl -p


#### Добавить ->>> sudo nano /etc/security/limits.conf 
--------------------------------------------------------
* soft nofile 1048576
* hard nofile 1048576
* soft nproc 1048576
* hard nproc 1048576


#### Скопировать все изменения в файлах на сегменты
--------------------------------------------------------
scp /etc/hosts gpadmin@seg1:~ && ssh gpadmin@seg1 'sudo mv ~/etc/hosts /etc/'
scp /etc/sysctl.conf gpadmin@seg1:~ && ssh gpadmin@seg1 'sudo mv ~/sysctl.conf /etc/'
scp /etc/security/limits.conf gpadmin@seg1:~ && ssh gpadmin@seg1 'sudo mv ~/etc/security/limits.conf /etc/security'

scp /etc/hosts gpadmin@seg2:~ && ssh gpadmin@seg2 'sudo mv ~/etc/hosts /etc/'
scp /etc/sysctl.conf gpadmin@seg2:~ && ssh gpadmin@seg2 'sudo mv ~/sysctl.conf /etc/'
scp /etc/security/limits.conf gpadmin@seg2:~ && ssh gpadmin@seg2 'sudo mv ~/etc/security/limits.conf /etc/security'

scp /etc/hosts gpadmin@seg3:~ && ssh gpadmin@seg3 'sudo mv ~/etc/hosts /etc/'
scp /etc/sysctl.conf gpadmin@seg3:~ && ssh gpadmin@seg3 'sudo mv ~/sysctl.conf /etc/'
scp /etc/security/limits.conf gpadmin@seg3:~ && ssh gpadmin@seg3 'sudo mv ~/etc/security/limits.conf /etc/security'


#### Перезапустить все машины

#### Устанавливаем Greenplum
--------------------------------------------------------
git clone https://github.com/greenplum-db/gpdb-archive.git
cd gpdb-archive
git submodule update --init


########## На мастере следующие команды выполняются в папке /gpconfigs ##########


#### Собрать образ на мастере
--------------------------------------------------------
./configure --prefix=/usr/local/greenplum --disable-orca
make -j$(nproc)
sudo make -j$(nproc) install


#### Скопировать образ на сегменты
--------------------------------------------------------
 Создние временного архива
sudo tar -czf /tmp/greenplum.tar.gz -C /usr/local greenplum # Временный архив
sudo chown gpadmin:gpadmin /tmp/greenplum.tar.gz

scp /tmp/greenplum.tar.gz gpadmin@seg1:/tmp/
scp /tmp/greenplum.tar.gz gpadmin@seg2:/tmp/
scp /tmp/greenplum.tar.gz gpadmin@seg3:/tmp/

 Распаковка архива
ssh gpadmin@seg1 "sudo tar -xzf /tmp/greenplum.tar.gz -C /usr/local && sudo chown -R gpadmin:gpadmin /usr/local/greenplum"
ssh gpadmin@seg2 "sudo tar -xzf /tmp/greenplum.tar.gz -C /usr/local && sudo chown -R gpadmin:gpadmin /usr/local/greenplum"
ssh gpadmin@seg3 "sudo tar -xzf /tmp/greenplum.tar.gz -C /usr/local && sudo chown -R gpadmin:gpadmin /usr/local/greenplum"


#### Установить все необходимые библиотеки на все машины        
--------------------------------------------------------
./libraries.bash
cat ./libraries.bash | ssh gpadmin@seg1 'bash -s'
cat ./libraries.bash | ssh gpadmin@seg2 'bash -s'
cat ./libraries.bash | ssh gpadmin@seg3 'bash -s'


#### Следующие команды выполнить на всех машинах 
--------------------------------------------------------
cp -r /usr/local/greenplum/docs/cli_help/gpconfigs /home/gpadmin
chown -R gpadmin:gpadmin /home/gpadmin*
source /usr/local/greenplum/greenplum_path.sh


#### Открыть файл (на мастере) .bashrc и добавить в него команды
--------------------------------------------------------
nano ~/.bashrc

source /usr/local/greenplum/greenplum_path.sh
export MASTER_DATA_DIRECTORY=/home/gpadmin/data/master/gpseg-1
export PGPORT=5432

#### Скопировать на сегменты

scp ~/.bashrc gpadmin@seg1:~/.bashrc
scp ~/.bashrc gpadmin@seg2:~/.bashrc
scp ~/.bashrc gpadmin@seg3:~/.bashrc

Перезагрузить все машины


--------------------------------------------------------

#### Создать файл 
->>> nano all_hosts
--------------------------------------------------------
master
seg1
seg2
seg3
--------------------------------------------------------

#### Создать файл 
->>> nano seg_hosts
--------------------------------------------------------
seg1
seg2
seg3
--------------------------------------------------------

#### Создать на мастере директорию 
->>> mkdir -p /home/gpadmin/data/master 
#### Создать папки на сегментах (с мастера)
--------------------------------------------------------
gpssh -f seg_hosts -e 'mkdir -p /home/gpadmin/data1/primary'
gpssh -f seg_hosts -e 'mkdir -p /home/gpadmin/data1/mirror'
gpssh -f seg_hosts -e 'mkdir -p /home/gpadmin/data2/primary'
gpssh -f seg_hosts -e 'mkdir -p /home/gpadmin/data2/mirror'



############ Меняем файл конфигурации ############

#### Конфигурационный файл можете найти в репозитории, продублирую его содержимое

->>> nano gpinitsystem_config 

--------------------------------------------------------
# FILE NAME: gpinitsystem_config

# Configuration file needed by the gpinitsystem

################################################
#### REQUIRED PARAMETERS
################################################

#### Naming convention for utility-generated data directories.
SEG_PREFIX=gpseg

#### Base number by which primary segment port numbers 
#### are calculated.
PORT_BASE=6000

#### File system location(s) where primary segment data directories 
#### will be created. The number of locations in the list dictate
#### the number of primary segments that will get created per
#### physical host (if multiple addresses for a host are listed in 
#### the hostfile, the number of segments will be spread evenly across
#### the specified interface addresses).
declare -a DATA_DIRECTORY=(/home/gpadmin/data1/primary /home/gpadmin/data2/primary)

#### OS-configured hostname or IP address of the coordinator host.
COORDINATOR_HOSTNAME=master
##MASTER_HOSTNAME=master

#### File system location where the coordinator data directory 
#### will be created.
COORDINATOR_DIRECTORY=/home/gpadmin/data/master

#### Port number for the coordinator instance. 
COORDINATOR_PORT=5432

#### Shell utility used to connect to remote hosts.
TRUSTED_SHELL=ssh

#### Default server-side character set encoding.
ENCODING=UNICODE

################################################
#### OPTIONAL MIRROR PARAMETERS
################################################

#### Base number by which mirror segment port numbers 
#### are calculated.
MIRROR_PORT_BASE=7000

#### File system location(s) where mirror segment data directories 
#### will be created. The number of mirror locations must equal the
#### number of primary locations as specified in the 
#### DATA_DIRECTORY parameter.
declare -a MIRROR_DATA_DIRECTORY=(/home/gpadmin/data1/mirror /home/gpadmin/data2/mirror)


################################################
#### OTHER OPTIONAL PARAMETERS
################################################

#### Create a database of this name after initialization.
#DATABASE_NAME=name_of_database

#### Specify the location of the host address file here instead of
#### with the -h option of gpinitsystem.
MACHINE_LIST_FILE=/home/gpadmin/gpconfigs/seg_hosts



--------------------------------------------------------

########## Запустить кластер ##########
--------------------------------------------------------
gpinitsystem -c gpinitsystem_config



#### После успешного запуска проверяем работу Postgres

psql postgres
select * from gp_segment_configuration 


#### В случае ошибки очистить папки 

################################################

rm -r -f /home/gpadmin/data/master/*
gpssh -f seg_hosts -e 'rm -r -f /home/gpadmin/data*/primary/*'
gpssh -f seg_hosts -e 'rm -r -f /home/gpadmin/data*/mirror/*'

################################################
