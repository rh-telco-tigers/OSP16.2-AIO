# OSP16.2-AIO
RedHat Openstack Platform 16.2 All-In-One Installation

This document explains how the underlying framework used by the containerized undercloud deployment mechanism can be used to deploy a single node capable of running Openstack services for testing

# System Requirments
TripleO can be used as a standalone environment with all services installed on a single virtual or baremetal machine. The minimum specification for a machine are
- 4 Core CPU
- 8 GB memory
- 60 GB free disk space
- RHEL 8.4 installed

# Deploying a Standalone Openstack Node
Before you can begin deploying the all-in-one environment, you must configure a non-root user and install the necessary packages and dependencies.
```bash
[root@aio]# useradd stack
[root@aio]# passwd stack
[root@aio]# echo "stack ALL=(root) NOPASSWD:ALL" | tee -a /etc/sudoers.d/stack
[root@aio]# chmod 0440 /etc/sudoers.d/stack
```
Also add host entry in /etc/host file. (I was missing this and deployment was failing at step 2)
```bash
[root@aio]# cat /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6
192.168.50.20	aio.osp.home.lab aio  <======= this entry
```
Login as stack and register the machine with Red Hat Subscription Manager
```bash
[stack@aio]# sudo subscription-manager register
[stack@aio]# sudo subscription-manager attach --pool <pool-id>
[stack@aio]# sudo subscription-manager release --set=8.4
[stack@aio]# sudo dnf install -y dnf-utils
[stack@aio]# sudo subscription-manager repos --disable=*
[stack@aio]# sudo subscription-manager repos \
--enable=rhel-8-for-x86_64-baseos-eus-rpms \
--enable=rhel-8-for-x86_64-appstream-eus-rpms \
--enable=rhel-8-for-x86_64-highavailability-eus-rpms \
--enable=ansible-2.9-for-rhel-8-x86_64-rpms \
--enable=openstack-16.2-for-rhel-8-x86_64-rpms \
--enable=fast-datapath-for-rhel-8-x86_64-rpms \
--enable=advanced-virt-for-rhel-8-x86_64-rpms
```
To set the container-tools and virt module versions, enter the follwoing commands
```bash
[stack@aio]# sudo dnf module disable -y container-tools:rhel8
[stack@aio]# sudo dnf module enable -y container-tools:3.0
[stack@aio]# sudo dnf module disable -y virt:rhel
[stack@aio]# sudo dnf module enable -y virt:av
[stack@aio]#
[stack@aio]# sudo dnf update
[stack@aio]# sudo reboot
```
Log back in and install tripleoclient
```bash
[stack@aio]# sudo dnf install -y python3-tripleoclient
```

# Generating YAML files for the all-in-one OpenStack environment

Generate the containers-prepare-parameters.yaml file that contains the default ContainerImagePrepare parameters
```bash
[stack@aio]# openstack tripleo container image prepare default --output-env-file $HOME/containers-prepare-parameters.yaml
```
Edit you containers-prepare-parameters.yaml file to include your Red Hat credentials. At the end you file should look like this
```yaml
# Generated with the following on 2021-09-16T11:33:43.773190
#
#   openstack tripleo container image prepare default --output-env-file /home/stack/containers-prepare-parameters.yaml
#
parameter_defaults:
  ContainerImagePrepare:
  - set:
      ceph_alertmanager_image: ose-prometheus-alertmanager
      ceph_alertmanager_namespace: registry.redhat.io/openshift4
      ceph_alertmanager_tag: 4.1
      ceph_grafana_image: rhceph-4-dashboard-rhel8
      ceph_grafana_namespace: registry.redhat.io/rhceph
      ceph_grafana_tag: 4
      ceph_image: rhceph-4-rhel8
      ceph_namespace: registry.redhat.io/rhceph
      ceph_node_exporter_image: ose-prometheus-node-exporter
      ceph_node_exporter_namespace: registry.redhat.io/openshift4
      ceph_node_exporter_tag: v4.1
      ceph_prometheus_image: ose-prometheus
      ceph_prometheus_namespace: registry.redhat.io/openshift4
      ceph_prometheus_tag: 4.1
      ceph_tag: latest
      name_prefix: openstack-
      name_suffix: ''
      namespace: registry.redhat.io/rhosp-rhel8
      neutron_driver: ovn
      rhel_containers: false
      tag: '16.2'
    tag_from_label: '{version}-{release}'
  ContainerImageRegistryCredentials:
    registry.redhat.io:
      sa-ashish: "PASSWORD"
  ContainerImageRegistryLogin: true
```
Create the $HOME/standalone_parameters.yaml file and configure basic parameters for your all-in-one RHOSP environment, including network configuration and some deployment options
```bash
[stack@aio]# export IP=192.168.50.20
[stack@aio]# export NETMASK=24
[stack@aio]# export INTERFACE=eno1
[stack@aio]# export DNS1=192.168.50.1
[stack@aio]# export DNS2=192.168.1.1
[stack@aio]# export GATEWAY=192.168.50.1

[stack@aio]# cat <<EOF > $HOME/standalone_parameters.yaml
parameter_defaults:
  CloudName: $IP
  CloudDomain: osp.home.lab
  ControlPlaneStaticRoutes:
    - ip_netmask: 0.0.0.0/0
      next_hop: $GATEWAY
      default: true
  Debug: true
  DeploymentUser: $USER
  DnsServers:
    - $DNS1
    - $DNS2
  NeutronPublicInterface: $INTERFACE
  NeutronDnsDomain: localdomain
  NeutronBridgeMappings: datacentre:br-ctlplane
  NeutronPhysicalBridge: br-ctlplane
  StandaloneEnableRoutedNetworks: false
  StandaloneHomeDir: $HOME
  StandaloneLocalMtu: 1500
EOF
```

# Deploying the all-in-one OpenStack environment
 Following are the steps to deploy all-in-one OSP environment
 
 1. Login to registry.redhat.io with your redhat credentails
 ```bash
[stack@aio]# sudo podman login registry.redhat.io 
```
2. Run the deployment command. Ensure that you include all .yaml files relevant to your environment:
```bash
[stack@aio]# sudo openstack tripleo deploy \
  --templates \
  --local-ip=$IP/$NETMASK \
  -e /usr/share/openstack-tripleo-heat-templates/environments/standalone/standalone-tripleo.yaml \
  -r /usr/share/openstack-tripleo-heat-templates/roles/Standalone.yaml \
  -e $HOME/containers-prepare-parameters.yaml \
  -e $HOME/standalone_parameters.yaml \
  --output-dir $HOME \
  --standalone
```
After a successful deployment, you can use the clouds.yaml configuration file in the /home/$USER/.config/openstack directory to query and verify the OpenStack services:
```bash
[stack@aio]# export OS_CLOUD=standalone
[stack@aio]# openstack endpoint list
```

To access the dashboard, go to to http://192.168.50.20/dashboard and use the default username admin and the undercloud_admin_password from the ~/standalone-passwords.conf file
