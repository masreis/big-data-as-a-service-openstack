# Preparação do sistema operacional
getenforce
sed -i 's/enforcing/disabled/g' /etc/selinux/config
setenforce 0
getenforce
yum update -y
#
systemctl disable firewalld
systemctl stop firewalld
systemctl disable NetworkManager
systemctl stop NetworkManager
systemctl enable network
systemctl start network
# OpenStack Pike
yum install -y centos-release-openstack-pike
yum install -y openstack-packstack
packstack --gen-answer-file=/root/answer_file.txt
yum update -y
# Alterar os seguintes parâmetros no arquivo answer_file.txt
% CONFIG_SERVICE_WORKERS=1
% CONFIG_SAHARA_INSTALL=y
% CONFIG_HEAT_INSTALL=y
% CONFIG_TROVE_INSTALL=y
% CONFIG_PROVISION_DEMO=n
% CONFIG_HEAT_CLOUDWATCH_INSTALL=y
% CONFIG_SWIFT_STORAGES=/dev/sdb1
% CONFIG_SWIFT_STORAGE_FSTYPE=xfs
# Instalação do OpenStack
packstack --answer-file=/root/answer_file.txt
# Configuração da rede
openstack network create external_network \
  --share \
  --project admin \
  --external \
  --provider-network-type flat \
  --provider-physical-network extnet
openstack subnet create external_subnet \
  --allocation-pool start=172.17.74.20,end=172.17.74.40 \
  --no-dhcp \
  --gateway 172.17.72.1 \
  --dns-nameserver 172.16.2.138 \
  --network external_network \
  --subnet-range 172.17.74.0/22
openstack network create net_data_analysis_a
dns=x.x.x.x # Informe o seu DNS
openstack subnet create subnet_data_analysis_a \
  --subnet-range 10.4.4.0/24 \
  --network net_data_analysis_a \
  --dns-nameserver $dns
openstack router create router_data_analysis_a
openstack router add subnet router_data_analysis_a subnet_data_analysis_a
openstack router set \
  --external-gateway external_network router_data_analysis_a
openstack security group create cluster-sec-group --project admin
openstack security group list
sec_group=cluster-sec-group
openstack security group rule create \
  --remote-ip 0.0.0.0/0 \
  --protocol icmp \
  --ingress $sec_group
openstack security group rule create \
  --remote-ip 0.0.0.0/0 \
  --dst-port 22 \
  --protocol tcp \
  --ingress $sec_group
openstack router create external_router
openstack router set \
  --external-gateway external_network external_router
openstack router set \
  --external-gateway external_network router_data_analysis_a
# Configuração do cluster Hadoop
openstack floating ip list
floating_ip_pool=XXX # selecionar o ip pool
openstack flavor create m2.dn \
    --id 100 \
    --ram 4096 \
    --disk 60 \
    --vcpus 1
openstack flavor create m2.small \
    --id 101 \
    --ram 4096 \
    --disk 20 \
    --vcpus 1
openstack flavor create m2.medium \
    --id 102 \
    --ram 8192 \
    --disk 40 \
    --vcpus 2
openstack flavor create m2.large \
    --id 103 \
    --ram 16384 \
    --disk 40 \
    --vcpus 2
openstack dataprocessing node group template create \
	--name cdh-m \
    --plugin cdh \
    --plugin-version 5.11.0 \
    --processes CLOUDERA_MANAGER HDFS_NAMENODE \
    	HDFS_SECONDARYNAMENODE YARN_JOBHISTORY \
    	YARN_RESOURCEMANAGER ZOOKEEPER_SERVER YARN_NODEMANAGER OOZIE_SERVER \
    --flavor m2.large \
    --auto-security-group \
    --autoconfig \
    --floating-ip-pool $floating_ip_pool
openstack dataprocessing node group template create \
    --name cdh-w \
    --plugin cdh \
    --plugin-version 5.11.0 \
    --processes HDFS_DATANODE YARN_NODEMANAGER \
    --flavor m2.dn \
    --auto-security-group \
    --autoconfig \
    --floating-ip-pool $floating_ip_pool
openstack dataprocessing cluster template create \
    --name cdh-3-w \
    --node-groups cdh-m:1 cdh-w:3
openstack image create ubuntu-cdh \
    --disk-format qcow2 \
    --container-format bare \
    --file $discoimagens/sahara/ubuntu_sahara_cloudera_5.11.0.qcow2
openstack dataprocessing image register ubuntu-cdh \
    --username ubuntu
openstack dataprocessing image tags add ubuntu-cdh \
    --tags cdh 5.11.0
\end{verbatim}

\label{anexo-create-cluster}
\small\begin{verbatim}
# Comando para criar um cluster Hadoop com Cloudera
openstack dataprocessing cluster create \
    --name cdh-u \
    --cluster-template cdh-3-w \
    --user-keypair chave-admin \
    --neutron-network net_data_analysis_a \
    --image ubuntu-cdh
