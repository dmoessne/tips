
### setup network
```
nmcli con mod 'eth0' ipv4.method 'manual' ipv4.address '10.66.xxx.xxx/24' ipv4.gateway '10.66.xxx.xxx' ipv4.dns '127.0.0.1 10.xx.xx.xx' ipv4.dns-search 'cluster-0001.rhsacn.org'
nmcli con down 'eth0' && nmcli con up 'eth0'

hostnamectl set-hostname helper.cluster-0001.rhsacn.org

sed -i '/^10.66.xxx.xxx helper.cluster-0001.rhsacn.org*/d' /etc/hosts

cat >> /etc/hosts << 'EOF'
10.66.xxx.xxx helper.cluster-0001.rhsacn.org
EOF
```

### setup repo
```
mkdir -p /etc/yum.repos.d/backup
mv /etc/yum.repos.d/*.repo /etc/yum.repos.d/backup

cat > /etc/yum.repos.d/w.repo << 'EOF'
[rhel-7-server-rpms]
name=rhel-7-server-rpms
baseurl=http://10.66.xxx.xxx/rhel7osp/rhel-7-server-rpms/
enabled=1
gpgcheck=0

[rhel-7-server-extras-rpms]
name=rhel-7-server-extras-rpms
baseurl=http://10.66.xxx.xxx/rhel7osp/rhel-7-server-extras-rpms/
enabled=1
gpgcheck=0


[rhel-7-server-ansible-2.9-rpms]
name=rhel-7-server-ansible-2.9-rpms
baseurl=http://10.66.xxx.xxx/rhel9osp/rhel-7-server-ansible-2.9-rpms/
enabled=1
gpgcheck=0

EOF
```

### update system and reboot
```
yum -y update 
reboot
```

### setup time service
```
cat > /etc/chrony.conf << 'EOF'
server 127.127.1.0 iburst
allow all
local stratum 4
EOF

systemctl enable chronyd && systemctl start chronyd 

chronyc -n sources
chronyc -n tracking

systemctl enable firewalld && systemctl start firewalld

firewall-cmd --permanent --add-service ntp
firewall-cmd --reload
```

### setup helper node
```
yum -y install ansible git
git clone https://github.com/RedHatOfficial/ocp4-helpernode
cd ocp4-helpernode
```

#### generate vars.yml
```
cat > vars.yml << EOF
---
staticips: true
helper:
  name: "helper"
  ipaddr: "10.66.208.138"
  networkifacename: "eth0"
dns:
  domain: "rhsacn.org"
  clusterid: "cluster-0001"
  forwarder1: "10.64.63.6"
bootstrap:
  name: "bootstrap"
  ipaddr: "10.66.208.139"
masters:
  - name: "master0"
    ipaddr: "10.66.208.140"
  - name: "master1"
    ipaddr: "10.66.208.141"  
  - name: "master2"
    ipaddr: "10.66.208.142"  
workers:
  - name: "worker0"
    ipaddr: "10.66.208.143"
  - name: "worker1"
    ipaddr: "10.66.208.144"
  - name: "worker2"
    ipaddr: "10.66.208.145"    
EOF

ansible-playbook -e @vars.yml tasks/main.yml
```

#### disconnected env change ignore_errors to yes (optional)
```
cat tasks/main.yml | sed '/^- hosts: all/, /vars_files/ {/^- hosts: all/!{/vars_files/!d;};}' | sed '/^- hosts: all/a  \ \ ignore_errors: yes' | tee tasks/mail.yml.new

mv -f tasks/mail.yml.new tasks/main.yml
```

#### check status
```
helpernodecheck dns-masters
helpernodecheck dns-workers
helpernodecheck dns-etcd
helpernodecheck install-info
helpernodecheck haproxy
helpernodecheck services
helpernodecheck nfs-info
```

### create helper node registry
```
yum -y install podman httpd httpd-tools wget jq

mkdir -p /opt/registry/{auth,certs,data}

cd /opt/registry/certs

openssl req -newkey rsa:4096 -nodes -sha256 -keyout domain.key -x509 -days 3650 -out domain.crt  -subj "/C=CN/ST=GD/L=SZ/O=Global Security/OU=IT Department/CN=*.cluster-0001.rhsacn.org"

cp /opt/registry/certs/domain.crt /etc/pki/ca-trust/source/anchors/
update-ca-trust extract

htpasswd -bBc /opt/registry/auth/htpasswd dummy dummy

firewall-cmd --add-port=5000/tcp --zone=internal --permanent
firewall-cmd --add-port=5000/tcp --zone=public   --permanent
firewall-cmd --add-service=http  --permanent
firewall-cmd --reload

cat > /usr/local/bin/localregistry.sh << 'EOF'
#!/bin/bash
podman run --name poc-registry -d -p 5000:5000 \
-v /opt/registry/data:/var/lib/registry:z \
-v /opt/registry/auth:/auth:z \
-e "REGISTRY_AUTH=htpasswd" \
-e "REGISTRY_AUTH_HTPASSWD_REALM=Registry" \
-e "REGISTRY_HTTP_SECRET=ALongRandomSecretForRegistry" \
-e REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd \
-v /opt/registry/certs:/certs:z \
-e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/domain.crt \
-e REGISTRY_HTTP_TLS_KEY=/certs/domain.key \
docker.io/library/registry:2 
EOF

chmod +x /usr/local/bin/localregistry.sh

/usr/local/bin/localregistry.sh

curl -u dummy:dummy -k https://helper.cluster-0001.rhsacn.org:5000/v2/_catalog

REPO_URL=helper.cluster-0001.rhsacn.org:5000
curl -u dummy:dummy -s -X GET https://$REPO_URL/v2/_catalog \
 | jq '.repositories[]' \
 | sort \
 | xargs -I _ curl -u dummy:dummy -s -X GET https://$REPO_URL/v2/_/tags/list

```

### prepare artifacts
```
MAJORBUILDNUMBER=4.5
EXTRABUILDNUMBER=4.5.6

wget https://mirror.openshift.com/pub/openshift-v4/clients/ocp/${EXTRABUILDNUMBER}/openshift-client-linux-${EXTRABUILDNUMBER}.tar.gz -P /var/www/html/
wget https://mirror.openshift.com/pub/openshift-v4/clients/ocp/${EXTRABUILDNUMBER}/openshift-install-linux-${EXTRABUILDNUMBER}.tar.gz -P /var/www/html/

tar -xzf /var/www/html/openshift-client-linux-${EXTRABUILDNUMBER}.tar.gz -C /usr/local/bin/
tar -xzf /var/www/html/openshift-install-linux-${EXTRABUILDNUMBER}.tar.gz -C /usr/local/bin/

# download bios and iso
MAJORBUILDNUMBER=4.5
EXTRABUILDNUMBER=4.5.6
wget https://mirror.openshift.com/pub/openshift-v4/dependencies/rhcos/${MAJORBUILDNUMBER}/${EXTRABUILDNUMBER}/rhcos-${EXTRABUILDNUMBER}-x86_64-installer.x86_64.iso -P /var/www/html/
wget https://mirror.openshift.com/pub/openshift-v4/dependencies/rhcos/${MAJORBUILDNUMBER}/${EXTRABUILDNUMBER}/rhcos-${EXTRABUILDNUMBER}-x86_64-metal.x86_64.raw.gz -P /var/www/html/

# Get pull secret
wget http://10.66.208.115/rhel9osp/pull-secret.json -P /root
jq '.auths += {"helper.cluster-0001.rhsacn.org:5000": {"auth": "ZHVtbXk6ZHVtbXk=","email": "noemail@localhost"}}' < /root/pull-secret.json > /root/pull-secret-2.json

# login registries
podman login -u wang.jun.1974 -p ****** registry.redhat.io
podman login -u wang.jun.1974 -p ****** registry.access.redhat.com
podman login -u wang.jun.1974 -p ****** registry.connect.redhat.com

# setup env and record imageContentSources section from output
# see: https://docs.openshift.com/container-platform/4.5/installing/install_config/installing-restricted-networks-preparations.html
export OCP_RELEASE="4.5.6"
export LOCAL_REGISTRY='helper.cluster-0001.rhsacn.org:5000'
export LOCAL_REPOSITORY='ocp4/openshift4'
export PRODUCT_REPO='openshift-release-dev'
export LOCAL_SECRET_JSON="${HOME}/pull-secret-2.json"
export RELEASE_NAME='ocp-release'
export ARCHITECTURE="x86_64"
export REMOVABLE_MEDIA_PATH='/opt/registry'

oc adm -a ${LOCAL_SECRET_JSON} release mirror \
--from=quay.io/${PRODUCT_REPO}/${RELEASE_NAME}:${OCP_RELEASE}-${ARCHITECTURE} \
--to=${LOCAL_REGISTRY}/${LOCAL_REPOSITORY} \
--to-release-image=${LOCAL_REGISTRY}/${LOCAL_REPOSITORY}:${OCP_RELEASE}-${ARCHITECTURE} --dry-run

# mirror to local registry
oc adm -a ${LOCAL_SECRET_JSON} release mirror \
--from=quay.io/${PRODUCT_REPO}/${RELEASE_NAME}:${OCP_RELEASE}-${ARCHITECTURE} \
--to=${LOCAL_REGISTRY}/${LOCAL_REPOSITORY} \
--to-release-image=${LOCAL_REGISTRY}/${LOCAL_REPOSITORY}:${OCP_RELEASE}-${ARCHITECTURE} 

# mirror to local directory (optional)
oc adm release mirror -a ${LOCAL_SECRET_JSON} --to-dir=${REMOVABLE_MEDIA_PATH}/mirror quay.io/${PRODUCT_REPO}/${RELEASE_NAME}:${OCP_RELEASE}-${ARCHITECTURE}

...
sha256:3e9704e62bb8ebaba3e9cda8176fa53de7b4e7e63b067eb94522bf6e5e93d4ea file://openshift/release:4.5.13-cluster-network-operator
info: Mirroring completed in 20ms (0B/s)

Success
Update image:  openshift/release:4.5.13

To upload local images to a registry, run:

    oc image mirror --from-dir=/opt/registry/mirror 'file://openshift/release:4.5.13*' REGISTRY/REPOSITORY

Configmap signature file /opt/registry/mirror/config/signature-sha256-8d104847fc2371a9.yaml created

# get content works with install-iso - i guess 4.5.0 iso only works with 4.5.0 OCP_RELEASE
export OCP_RELEASE="4.5.2"
export LOCAL_REGISTRY='helper.cluster-0001.rhsacn.org:5000'
export LOCAL_REPOSITORY='ocp4/openshift4'
export PRODUCT_REPO='openshift-release-dev'
export LOCAL_SECRET_JSON="${HOME}/pull-secret-2.json"
export RELEASE_NAME='ocp-release'
export ARCHITECTURE="x86_64"
export REMOVABLE_MEDIA_PATH='/opt/registry'

# mirror to local registry
oc adm -a ${LOCAL_SECRET_JSON} release mirror \
--from=quay.io/${PRODUCT_REPO}/${RELEASE_NAME}:${OCP_RELEASE}-${ARCHITECTURE} \
--to=${LOCAL_REGISTRY}/${LOCAL_REPOSITORY} \
--to-release-image=${LOCAL_REGISTRY}/${LOCAL_REPOSITORY}:${OCP_RELEASE}-${ARCHITECTURE}

# Take the media to the restricted network environment and upload the images to the local container registry.
oc image mirror -a ${LOCAL_SECRET_JSON} --from-dir=${REMOVABLE_MEDIA_PATH}/mirror "file://openshift/release:${OCP_RELEASE}*" ${LOCAL_REGISTRY}/${LOCAL_REPOSITORY}

# catalog build need use 
# targetfile='./redhat-operators-manifests/mapping.tag.txt'
# cat $targetfile | while read line ;do echo ${line%=*};skopeo copy --format v2s2 --all docker://${line%=*} docker://${line#*=}; done


OPERATOR_OCP_RELEASE="4.5"
oc adm catalog build \
  --appregistry-org redhat-operators \
  --from=registry.redhat.io/openshift4/ose-operator-registry:v${OPERATOR_OCP_RELEASE} \
  --filter-by-os='linux/amd64' \
  -a ${LOCAL_SECRET_JSON} \
  --to=${LOCAL_REGISTRY}/olm/operator-catalog:redhat-${OPERATOR_OCP_RELEASE}-$(date -I)

oc adm catalog mirror \
  ${LOCAL_REGISTRY}/olm/operator-catalog:redhat-${OPERATOR_OCP_RELEASE}-$(date -I) \
  ${LOCAL_REGISTRY} \
  --filter-by-os='linux/amd64' \
  -a ${LOCAL_SECRET_JSON}

oc adm catalog build \
  --appregistry-org certified-operators \
  --from=registry.redhat.io/openshift4/ose-operator-registry:v${OPERATOR_OCP_RELEASE} \
  --filter-by-os='linux/amd64' \
  -a ${LOCAL_SECRET_JSON} \
  --to=${LOCAL_REGISTRY}/olm/operator-catalog:certified-${OPERATOR_OCP_RELEASE}-$(date -I)

oc adm catalog mirror \
  ${LOCAL_REGISTRY}/olm/operator-catalog:certified-${OPERATOR_OCP_RELEASE}-$(date -I) \
  ${LOCAL_REGISTRY} \
  --filter-by-os='linux/amd64' \
  -a ${LOCAL_SECRET_JSON}

oc adm catalog build \
  --appregistry-org community-operators \
  --from=registry.redhat.io/openshift4/ose-operator-registry:v${OPERATOR_OCP_RELEASE} \
  --filter-by-os='linux/amd64' \
  -a ${LOCAL_SECRET_JSON} \
  --to=${LOCAL_REGISTRY}/olm/operator-catalog:community-${OPERATOR_OCP_RELEASE}-$(date -I)

oc adm catalog mirror \
  ${LOCAL_REGISTRY}/olm/operator-catalog:community-${OPERATOR_OCP_RELEASE}-$(date -I) \
  ${LOCAL_REGISTRY} \
  --filter-by-os='linux/amd64' \
  -a ${LOCAL_SECRET_JSON}

# ToDo: I could not go through this process ... (optional)
# copy catalog relate content into disconnect env
# 1. $oc adm catalog build --appregistry-org redhat-operators --from=registry.redhat.io/openshift4/ose-operator-registry:vXX  --dir=<YOUR_DIR> --to=file://offline/redhat-operators:vXX
# 2. $oc adm catalog mirror --manifests-only=true --from-dir=<YOUR_DIR> file://offline/redhat-operators:vXX localhost
# 3. $sed 's/=/=file:\/\//g' redhat-operators-manifests/mapping.txt > mapping-new.txt
# 4. $oc image mirror  -f mappings-new.txt --dir=<YOUR_DIR>
export OPERATOR_OCP_RELEASE="4.5"
oc adm catalog build \
  --appregistry-org redhat-operators \
  --from=registry.redhat.io/openshift4/ose-operator-registry:v${OPERATOR_OCP_RELEASE} \
  --filter-by-os='linux/amd64' \
  -a ${LOCAL_SECRET_JSON} \
  --dir=${REMOVABLE_MEDIA_PATH} \
  --to-db=file:///offline/redhat-operators:v1
 
oc adm catalog mirror \
  --manifests-only=true \
  --from-dir=${REMOVABLE_MEDIA_PATH} file://offline/redhat-operators:v${OPERATOR_OCP_RELEASE} \
  ${LOCAL_REGISTRY} \
  -a ${LOCAL_SECRET_JSON} \
  --filter-by-os='linux/amd64' 

# install install directory
rm -rf /root/ocp4
mkdir -p /root/ocp4
cd /root/ocp4

ssh-keygen -t rsa -f ~/.ssh/id_rsa -N '' 

cat > install-config.yaml.orig << EOF
apiVersion: v1
baseDomain: rhsacn.org
compute:
- hyperthreading: Enabled
  name: worker
  replicas: 2
controlPlane:
  hyperthreading: Enabled
  name: master
  replicas: 3
metadata:
  name: cluster-0001
networking:
  clusterNetworks:
  - cidr: 10.254.0.0/16
    hostPrefix: 24
  networkType: OpenShiftSDN
  serviceNetwork:
  - 172.30.0.0/16
platform:
  none: {}
pullSecret: '{"auths":{"helper.cluster-0001.rhsacn.org:5000": {"auth": "ZHVtbXk6ZHVtbXk=","email": "noemail@localhost"}}}'
sshKey: |
$( cat /root/.ssh/id_rsa.pub | sed 's/^/  /g' )
additionalTrustBundle: |
$( cat /etc/pki/ca-trust/source/anchors/domain.crt | sed 's/^/  /g' )
imageContentSources:
- mirrors:
  - helper.cluster-0001.rhsacn.org:5000/ocp4/openshift4
  source: quay.io/openshift-release-dev/ocp-release
- mirrors:
  - helper.cluster-0001.rhsacn.org:5000/ocp4/openshift4
  source: quay.io/openshift-release-dev/ocp-v4.0-art-dev
EOF

cp install-config.yaml.orig /var/www/html
cp /var/www/html/install-config.yaml.orig install-config.yaml

# create ignition file
rm -f *.ign
#/bin/rm -rf *.ign .openshift_install_state.json auth bootstrap master-0 master-1 master-2 worker-0 worker-1 worker-2
openshift-install create ignition-configs --dir=/root/ocp4

# generate ignition file
rm -f /var/www/html/ignition/*
/bin/cp -f bootstrap.ign /var/www/html/ignition/bootstrap-static.ign
/bin/cp -f master.ign /var/www/html/ignition/master-0.ign
/bin/cp -f master.ign /var/www/html/ignition/master-1.ign
/bin/cp -f master.ign /var/www/html/ignition/master-2.ign
/bin/cp -f worker.ign /var/www/html/ignition/worker-0.ign
/bin/cp -f worker.ign /var/www/html/ignition/worker-1.ign
/bin/cp -f worker.ign /var/www/html/ignition/worker-2.ign
chmod 644 /var/www/html/ignition/*

yum install -y libguestfs libguestfs-tools genisoimage
systemctl start libvirtd

cd /root
export NGINX_DIRECTORY=/var/www/html
export RHCOSVERSION=4.5.2
export VOLID=$(isoinfo -d -i ${NGINX_DIRECTORY}/rhcos-${RHCOSVERSION}-x86_64-installer.x86_64.iso | awk '/Volume id/ { print $3 }')
TEMPDIR=$(mktemp -d)
echo $VOLID
echo $TEMPDIR

cd ${TEMPDIR}
# Extract the ISO content using guestfish (to avoid sudo mount)
guestfish -a ${NGINX_DIRECTORY}/rhcos-${RHCOSVERSION}-x86_64-installer.x86_64.iso \
  -m /dev/sda tar-out / - | tar xvf -

# Helper function to modify the config files
modify_cfg(){
  for file in "EFI/redhat/grub.cfg" "isolinux/isolinux.cfg"; do
    # Append the proper image and ignition urls
    sed -e '/coreos.inst=yes/s|$| coreos.inst.install_dev=vda coreos.inst.image_url='"${URL}"'\/install\/'"${BIOSMODE}"'.raw.gz coreos.inst.ignition_url='"${URL}"'\/ignition\/'"${NODE}"'.ign ip='"${IP}"'::'"${GATEWAY}"':'"${NETMASK}"':'"${FQDN}"':'"${NET_INTERFACE}"':none:'"${DNS}"' nameserver='"${DNS}"'|' ${file} > $(pwd)/${NODE}_${file##*/}
    # Boot directly in the installation
    sed -i -e 's/default vesamenu.c32/default linux/g' -e 's/timeout 600/timeout 10/g' $(pwd)/${NODE}_${file##*/}
  done
}

# Helper function to modify the config files - using sda as hard disk device (optional)
modify_cfg(){
  for file in "EFI/redhat/grub.cfg" "isolinux/isolinux.cfg"; do
    # Append the proper image and ignition urls
    sed -e '/coreos.inst=yes/s|$| coreos.inst.install_dev=sda coreos.inst.image_url='"${URL}"'\/install\/'"${BIOSMODE}"'.raw.gz coreos.inst.ignition_url='"${URL}"'\/ignition\/'"${NODE}"'.ign ip='"${IP}"'::'"${GATEWAY}"':'"${NETMASK}"':'"${FQDN}"':'"${NET_INTERFACE}"':none:'"${DNS}"' nameserver='"${DNS}"'|' ${file} > $(pwd)/${NODE}_${file##*/}
    # Boot directly in the installation
    sed -i -e 's/default vesamenu.c32/default linux/g' -e 's/timeout 600/timeout 10/g' $(pwd)/${NODE}_${file##*/}
  done
}

URL="http://10.66.208.138:8080/"
GATEWAY="10.66.208.254"
NETMASK="255.255.255.0"
DNS="10.66.208.138"

# BOOTSTRAP
# TYPE="bootstrap"
NODE="bootstrap-static"
IP="10.66.208.139"
FQDN="bootstrap.cluster-0001.rhsacn.org"
BIOSMODE="bios"
NET_INTERFACE="ens3"
modify_cfg

# MASTERS
# TYPE="master"
# MASTER-0
NODE="master-0"
IP="10.66.208.140"
FQDN="master0.cluster-0001.rhsacn.org"
BIOSMODE="bios"
NET_INTERFACE="ens3"
modify_cfg

# MASTER-1
NODE="master-1"
IP="10.66.208.141"
FQDN="master1.cluster-0001.rhsacn.org"
BIOSMODE="bios"
NET_INTERFACE="ens3"
modify_cfg

# MASTER-2
NODE="master-2"
IP="10.66.208.142"
FQDN="master2.cluster-0001.rhsacn.org"
BIOSMODE="bios"
NET_INTERFACE="ens3"
modify_cfg

# WORKERS
NODE="worker-0"
IP="10.66.208.143"
FQDN="worker0.cluster-0001.rhsacn.org"
BIOSMODE="bios"
NET_INTERFACE="ens3"
modify_cfg

NODE="worker-1"
IP="10.66.208.144"
FQDN="worker1.cluster-0001.rhsacn.org"
BIOSMODE="bios"
NET_INTERFACE="ens3"
modify_cfg
#
NODE="worker-2"
IP="10.66.208.145"
FQDN="worker2.cluster-0001.rhsacn.org"
BIOSMODE="bios"
NET_INTERFACE="ens3"
modify_cfg
# Generate the images, one per node as the IP configuration is different...
# https://github.com/coreos/coreos-assembler/blob/master/src/cmd-buildextend-installer#L97-L103

for node in master-0 master-1 master-2 worker-0 worker-1 worker-2 bootstrap-static; do
  # Overwrite the grub.cfg and isolinux.cfg files for each node type
  for file in "EFI/redhat/grub.cfg" "isolinux/isolinux.cfg"; do
    /bin/cp -f $(pwd)/${node}_${file##*/} ${file}
  done
  # As regular user!
  genisoimage -verbose -rock -J -joliet-long -volset ${VOLID} \
    -eltorito-boot isolinux/isolinux.bin -eltorito-catalog isolinux/boot.cat \
    -no-emul-boot -boot-load-size 4 -boot-info-table \
    -eltorito-alt-boot -efi-boot images/efiboot.img -no-emul-boot \
    -o ${NGINX_DIRECTORY}/${node}.iso .
done

# Optionally, clean up
cd
rm -Rf ${TEMPDIR}

cd /var/www/html

# on upload machine first download iso from helper node
curl http://10.66.208.138:8080/bootstrap-static.iso -o bootstrap-static.iso
curl http://10.66.208.138:8080/master-0.iso -o master-0.iso
curl http://10.66.208.138:8080/master-1.iso -o master-1.iso
curl http://10.66.208.138:8080/master-2.iso -o master-2.iso
curl http://10.66.208.138:8080/worker-0.iso -o worker-0.iso
curl http://10.66.208.138:8080/worker-1.iso -o worker-1.iso
curl http://10.66.208.138:8080/worker-2.iso -o worker-2.iso

# upload iso to iso domain
yum install -y expect
prog=/usr/bin/engine-iso-uploader
mypass="<password>"

args="-i ISO11 upload bootstrap-static.iso --force"
/usr/bin/expect <<EOF
set timeout -1
spawn "$prog" $args
expect "Please provide the REST API password for the admin@internal oVirt Engine user (CTRL+D to abort): "
send "$mypass\r"
expect eof
exit
EOF

args="-i ISO11 upload master-0.iso --force"
/usr/bin/expect <<EOF
set timeout -1
spawn "$prog" $args
expect "Please provide the REST API password for the admin@internal oVirt Engine user (CTRL+D to abort): "
send "$mypass\r"
expect eof
exit
EOF

args="-i ISO11 upload master-1.iso --force"
/usr/bin/expect <<EOF
set timeout -1
spawn "$prog" $args
expect "Please provide the REST API password for the admin@internal oVirt Engine user (CTRL+D to abort): "
send "$mypass\r"
expect eof
exit
EOF

args="-i ISO11 upload master-2.iso --force"
/usr/bin/expect <<EOF
set timeout -1
spawn "$prog" $args
expect "Please provide the REST API password for the admin@internal oVirt Engine user (CTRL+D to abort): "
send "$mypass\r"
expect eof
exit
EOF

args="-i ISO11 upload worker-0.iso --force"
/usr/bin/expect <<EOF
set timeout -1
spawn "$prog" $args
expect "Please provide the REST API password for the admin@internal oVirt Engine user (CTRL+D to abort): "
send "$mypass\r"
expect eof
exit
EOF

args="-i ISO11 upload worker-1.iso --force"
/usr/bin/expect <<EOF
set timeout -1
spawn "$prog" $args
expect "Please provide the REST API password for the admin@internal oVirt Engine user (CTRL+D to abort): "
send "$mypass\r"
expect eof
exit
EOF

args="-i ISO11 upload worker-2.iso --force"
/usr/bin/expect <<EOF
set timeout -1
spawn "$prog" $args
expect "Please provide the REST API password for the admin@internal oVirt Engine user (CTRL+D to abort): "
send "$mypass\r"
expect eof
exit
EOF

# (PoC) before install delete old files
# (PoC) no need run every time 
# (PoC) only run when cleanup and do a new poc
cd /root/ocp4
#/bin/rm -rf *.ign .openshift_install_state.json auth bootstrap master-0 master-1 master-2 worker-0 worker-1 worker-2

# start openshift-install bootstrap
openshift-install --dir=/root/ocp4 wait-for bootstrap-complete --log-level debug

# check time sync on bootstrap/masters if there is x509 relate error
date
openssl s_client -connect api.crc.testing:6443 | openssl x509 -noout -dates

# remove bootstrap from openshift-api-server and machine-config-server
sed -ie 's|^    server bootstrap|    #server bootstrap|g' /etc/haproxy/haproxy.cfg
systemctl restart haproxy

# add bootstrap to openshift-api-server and machine-config-server
sed -ie 's|^    #server bootstrap|    server bootstrap|g' /etc/haproxy/haproxy.cfg
systemctl restart haproxy

openshift-install --dir=/root/ocp4 wait-for install-complete --log-level debug

# approve worker csr
oc get nodes
export KUBECONFIG=/root/ocp4/auth/kubeconfig
echo "export KUBECONFIG=/root/ocp4/auth/kubeconfig" >> ~/.bashrc
oc completion bash | sudo tee /etc/bash_completion.d/openshift > /dev/null

/usr/local/bin/oc get csr --no-headers | /usr/bin/awk '{print $1}' | xargs /usr/local/bin/oc adm certificate approve
watch oc get nodes

# see: https://github.com/wangzheng422/docker_env/blob/master/redhat/ocp4/4.5/4.5.disconnect.operator.md

# oc get nodes 
#   all nodes ready
# oc get clusteroperator image-registry
#   image-registry available

# label worker as infra node and patch ingresscontroller (optional with questions)
oc label node worker0.cluster-0001.rhsacn.org node-role.kubernetes.io/infra=""
oc label node worker1.cluster-0001.rhsacn.org node-role.kubernetes.io/infra=""
oc label node worker2.cluster-0001.rhsacn.org node-role.kubernetes.io/infra=""
oc patch ingresscontroller default -n openshift-ingress-operator --type=merge --patch='{"spec":{"nodePlacement":{"nodeSelector": {"matchLabels":{"node-role.kubernetes.io/infra":""}}}}}'
oc patch ingresscontroller default -n openshift-ingress-operator --type=merge --patch='{"spec":{"replicas":3}}'

# setup nfs privioner from helper node
bash /root/ocp4-helpernode/files/nfs-provisioner-setup.sh
oc patch configs.imageregistry.operator.openshift.io cluster -p '{"spec":{"managementState": "Managed","storage":{"pvc":{"claim":""}}}}' --type=merge
oc get clusteroperator image-registry
oc get configs.imageregistry.operator.openshift.io cluster -o yaml

# stop imagepruner
# https://bugzilla.redhat.com/show_bug.cgi?id=1852501#c24
# oc patch imagepruner.imageregistry/cluster --patch '{"spec":{"suspend":true}}' --type=merge
# oc -n openshift-image-registry delete jobs --all

# operator hub disconnected
# see: https://github.com/wangzheng422/docker_env/blob/master/redhat/ocp4/4.5/4.5.disconnect.operator.md
# add additionalTrustedCA to image.config.openshift.io/cluster (optioanl)
oc patch image.config.openshift.io/cluster -p '{"spec":{"additionalTrustedCA":{"name":"user-ca-bundle"}}}'  --type=merge
oc get image.config.openshift.io/cluster -o yaml

oc patch OperatorHub cluster --type json \
    -p '[{"op": "add", "path": "/spec/disableAllDefaultSources", "value": true}]'
oc get OperatorHub cluster -o yaml

# add local pull secret into openshift-marketplace namespace. 这步非常重要
oc create secret docker-registry local-pull-secret \
    --namespace openshift-marketplace \
    --docker-server=helper.cluster-0001.rhsacn.org:5000 \
    --docker-username=dummy \
    --docker-password=dummy
oc patch sa default -n openshift-marketplace --type='json' -p='[{"op":"add","path":"/imagePullSecrets/-", "value":{"name":"local-pull-secret"}}]'

# pause node reboot for machineconfig
oc patch machineconfigpools.machineconfiguration.openshift.io/master -p '{"spec":{"paused":true}}' --type=merge
oc patch machineconfigpools.machineconfiguration.openshift.io/worker -p '{"spec":{"paused":true}}' --type=merge

# change registry.conf file (optional)
# see: https://github.com/wangzheng422/docker_env/blob/master/redhat/ocp4/4.5/scripts/image.registries.conf.sh
cat > image.registries.conf << 'EOF'
unqualified-search-registries = ["registry.access.redhat.com", "docker.io"]

[[registry]]
  prefix = ""
  location = "quay.io/openshift-release-dev/ocp-release"
  mirror-by-digest-only = true

  [[registry.mirror]]
    location = "helper.cluster-0001.rhsacn.org:5000/ocp4/openshift4"

[[registry]]
  prefix = ""
  location = "quay.io/openshift-release-dev/ocp-v4.0-art-dev"
  mirror-by-digest-only = true

  [[registry.mirror]]
    location = "helper.cluster-0001.rhsacn.org:5000/ocp4/openshift4"

[[registry]]
  location = "helper.cluster-0001.rhsacn.org:5000"
  insecure = true
  blocked = false
  mirror-by-digest-only = false
  prefix = ""
EOF

config_source=$(cat ./image.registries.conf | python3 -c "import sys, urllib.parse; print(urllib.parse.quote(''.join(sys.stdin.readlines())))"  )

cat <<EOF > 99-worker-zzz-container-registries.yaml
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfig
metadata:
  labels:
    machineconfiguration.openshift.io/role: worker
  name: 99-worker-zzz-container-registries
spec:
  config:
    ignition:
      version: 2.2.0
    storage:
      files:
      - contents:
          source: data:text/plain,${config_source}
          verification: {}
        filesystem: root
        mode: 420
        path: /etc/containers/registries.conf
EOF

cat <<EOF > 99-master-zzz-container-registries.yaml
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfig
metadata:
  labels:
    machineconfiguration.openshift.io/role: master
  name: 99-master-zzz-container-registries
spec:
  config:
    ignition:
      version: 2.2.0
    storage:
      files:
      - contents:
          source: data:text/plain,${config_source}
          verification: {}
        filesystem: root
        mode: 420
        path: /etc/containers/registries.conf
EOF

oc apply -f ./99-worker-zzz-container-registries.yaml -n openshift-config
oc apply -f ./99-master-zzz-container-registries.yaml -n openshift-config

# time sync configuration
cat > time.sync.conf << EOF
server helper.cluster-0001.rhsacn.org iburst
driftfile /var/lib/chrony/drift
makestep 1.0 10
rtcsync
logdir /var/log/chrony
EOF

config_source=$(cat ./time.sync.conf | base64 -w 0 )

# see: https://openshift.tips/machine-config/
cat << EOF > ./99-master-zzz-chrony-configuration.yaml
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfig
metadata:
  labels:
    machineconfiguration.openshift.io/role: master
  name: masters-chrony-configuration
spec:
  config:
    ignition:
      config: {}
      security:
        tls: {}
      timeouts: {}
      version: 2.2.0
    networkd: {}
    passwd: {}
    storage:
      files:
      - contents:
          source: data:text/plain;charset=utf-8;base64,${config_source}
          verification: {}
        filesystem: root
        mode: 420
        path: /etc/chrony.conf
  osImageURL: ""
EOF

cat << EOF > ./99-worker-zzz-chrony-configuration.yaml
apiVersion: machineconfiguration.openshift.io/v1
kind: MachineConfig
metadata:
  labels:
    machineconfiguration.openshift.io/role: worker
  name: workers-chrony-configuration
spec:
  config:
    ignition:
      config: {}
      security:
        tls: {}
      timeouts: {}
      version: 2.2.0
    networkd: {}
    passwd: {}
    storage:
      files:
      - contents:
          source: data:text/plain;charset=utf-8;base64,${config_source}
          verification: {}
        filesystem: root
        mode: 420
        path: /etc/chrony.conf
  osImageURL: ""
EOF

oc apply -f ./99-master-zzz-chrony-configuration.yaml
oc apply -f ./99-worker-zzz-chrony-configuration.yaml

oc patch machineconfigpools.machineconfiguration.openshift.io/master -p '{"spec":{"paused":false}}' --type=merge
oc patch machineconfigpools.machineconfiguration.openshift.io/worker -p '{"spec":{"paused":false}}' --type=merge

# 检查时间同步情况
for i in $(seq 140 145); do echo 10.66.208.${i} ; ssh core@10.66.208.${i} date -u ; date -u ; echo ; done

# patch samples operator samplesRegistry (optional)
# 当本地有 imagestream 镜像时执行此命令
oc patch configs.samples.operator.openshift.io cluster --type merge \
  --patch '{"spec":{"samplesRegistry": "helper.cluster-0001.rhsacn.org:5000", "managementState": "Managed"}}'
# 如果错误执行上述命令，openshift-samples operator 可能处于 DEGRADED 状态
# 对于 connected cluster，可执行以下命令将状态修正成正常状态
oc patch configs.samples.operator.openshift.io cluster --type merge \
  --patch '{"spec":{"samplesRegistry": "", "managementState": "Managed"}}'

# 对于 disconnected cluster，可参考 https://access.redhat.com/solutions/5067531，同步所需镜像
# 并做相关设置

# test is works here
oc get nodes
oc debug node/<worker0>

# label nodes used for ocs
oc label nodes worker0.cluster-0001.rhsacn.org cluster.ocs.openshift.io/openshift-storage=''
oc label nodes worker1.cluster-0001.rhsacn.org cluster.ocs.openshift.io/openshift-storage=''
oc label nodes worker2.cluster-0001.rhsacn.org cluster.ocs.openshift.io/openshift-storage=''

# see: https://access.redhat.com/documentation/en-us/red_hat_openshift_container_storage/4.5/html-single/deploying_openshift_container_storage_on_vmware_vsphere/index

# see: https://github.com/wangjun1974/tips/blob/master/ocp/operatorhub-disconnected.md
export LOCAL_REGISTRY='helper.cluster-0001.rhsacn.org:5000'

cat > catalogsource.yaml << EOF
apiVersion: operators.coreos.com/v1alpha1
kind: CatalogSource
metadata:
  name: my-operator-catalog
  namespace: openshift-marketplace
spec:
  sourceType: grpc
  image: ${LOCAL_REGISTRY}/olm/redhat-operators:v1
  displayName: My Operator Catalog
  publisher: grpc
EOF

oc apply -f catalogsource.yaml

# 创建 /tmp/ImageContentSourcePolicy.yaml 文件
cat <<EOF > /tmp/ImageContentSourcePolicy.yaml
apiVersion: operator.openshift.io/v1alpha1
kind: ImageContentSourcePolicy
metadata:
  name: redhat-operators
spec:
  repositoryDigestMirrors:
$(cat /tmp/image-policy.txt)
EOF

oc apply -f /tmp/ImageContentSourcePolicy.yaml
# 等待 machineconfigpool 更新完成
watch oc get mcp

# create local-storage namespace and install local storage operator

# install ocs operators

# create local-storage-block.yaml and change dev/by-id
cat > local-storage-block.yaml << EOF
apiVersion: local.storage.openshift.io/v1
kind: LocalVolume
metadata:
  name: local-block
  namespace: local-storage
  labels:
    app: ocs-storagecluster
spec:
  nodeSelector:
    nodeSelectorTerms:
    - matchExpressions:
        - key: cluster.ocs.openshift.io/openshift-storage
          operator: In
          values:
          - ""
  storageClassDevices:
    - storageClassName: localblock
      volumeMode: Block
      devicePaths:
        - /dev/disk/by-id/$(ssh core@10.66.208.143 'ls -l /dev/disk/by-id ' 2>&1 | grep sdb | tail -1 | awk '{print $9}')
        - /dev/disk/by-id/$(ssh core@10.66.208.144 'ls -l /dev/disk/by-id ' 2>&1 | grep sdb | tail -1 | awk '{print $9}')
        - /dev/disk/by-id/$(ssh core@10.66.208.145 'ls -l /dev/disk/by-id ' 2>&1 | grep sdb | tail -1 | awk '{print $9}')
EOF

# create local volume
oc create -f local-storage-block.yaml 

# get pod in local-storage namespace
oc -n local-storage get pods

# get storage class
oc get sc | grep localblock
localblock                          kubernetes.io/no-provisioner   Delete          WaitForFirstConsumer   false                  70s

# get pv
oc get pv
NAME                CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   REASON   AGE
local-pv-8ebcac77   80Gi       RWO            Delete           Available           localblock              33s
local-pv-a07ed3e8   80Gi       RWO            Delete           Available           localblock              41s
local-pv-a4482bcd   80Gi       RWO            Delete           Available           localblock              45s

# get csv in namespace openshift-storage
oc get csv -n openshift-storage
NAME                  DISPLAY                       VERSION   REPLACES   PHASE
ocs-operator.v4.5.0   OpenShift Container Storage   4.5.0                Succeeded

...
# add rook toolbox
oc patch OCSInitialization ocsinit -n openshift-storage --type json --patch  '[{ "op": "replace", "path": "/spec/enableCephTools", "value": true }]'

TOOLS_POD=$(oc get pods -n openshift-storage -l app=rook-ceph-tools -o name)
oc rsh -n openshift-storage $TOOLS_POD

# https://github.com/red-hat-storage/ocs-ci/pull/862
# lower ocs requirement
# 以下链接有 Internal OCS 的资源需求，最小要求 30 CPU
# https://access.redhat.com/documentation/en-us/red_hat_openshift_container_storage/4.5/html-single/planning_your_deployment/index
# https://rook.io/docs/rook/v1.4/ceph-cluster-crd.html
# 尝试通过设置以下参数降低 ocs 4 对 cpu 的需求
oc patch StorageCluster ocs-storagecluster -n openshift-storage --type=merge --patch='{"spec":{"resources":{"mds": {"Limit": {"cpu": "500m"}}}}}'
oc patch StorageCluster ocs-storagecluster -n openshift-storage --type=merge --patch='{"spec":{"resources":{"mds": {"Request": {"cpu": "500m"}}}}}'

oc patch StorageCluster ocs-storagecluster -n openshift-storage --type=merge --patch='{"spec":{"resources":{"mgr": {"Limit": {"cpu": "500m"}}}}}'
oc patch StorageCluster ocs-storagecluster -n openshift-storage --type=merge --patch='{"spec":{"resources":{"mgr": {"Request": {"cpu": "500m"}}}}}'

oc patch StorageCluster ocs-storagecluster -n openshift-storage --type=merge --patch='{"spec":{"resources":{"mon": {"Limit": {"cpu": "500m"}}}}}'
oc patch StorageCluster ocs-storagecluster -n openshift-storage --type=merge --patch='{"spec":{"resources":{"mon": {"Request": {"cpu": "500m"}}}}}'

oc patch StorageCluster ocs-storagecluster -n openshift-storage --type=merge --patch='{"spec":{"resources":{"osd": {"Limit": {"cpu": "500m"}}}}}'
oc patch StorageCluster ocs-storagecluster -n openshift-storage --type=merge --patch='{"spec":{"resources":{"osd": {"Request": {"cpu": "500m"}}}}}'

oc patch StorageCluster ocs-storagecluster -n openshift-storage --type=merge --patch='{"spec":{"resources":{"noobaa-core": {"Limit": {"cpu": "500m"}}}}}'
oc patch StorageCluster ocs-storagecluster -n openshift-storage --type=merge --patch='{"spec":{"resources":{"noobaa-core": {"Request": {"cpu": "500m"}}}}}'

oc patch StorageCluster ocs-storagecluster -n openshift-storage --type=merge --patch='{"spec":{"resources":{"noobaa-db": {"Limit": {"cpu": "500m"}}}}}'
oc patch StorageCluster ocs-storagecluster -n openshift-storage --type=merge --patch='{"spec":{"resources":{"noobaa-db": {"Request": {"cpu": "500m"}}}}}'

oc patch StorageCluster ocs-storagecluster -n openshift-storage --type=merge --patch='{"spec":{"resources":{"rgw": {"Limit": {"cpu": "500m"}}}}}'
oc patch StorageCluster ocs-storagecluster -n openshift-storage --type=merge --patch='{"spec":{"resources":{"rgw": {"Request": {"cpu": "500m"}}}}}'

oc patch StorageCluster ocs-storagecluster -n openshift-storage --type=merge --patch='{"spec":{"resources":{"prepareosd": {"Limit": {"cpu": "500m"}}}}}'
oc patch StorageCluster ocs-storagecluster -n openshift-storage --type=merge --patch='{"spec":{"resources":{"prepareosd": {"Request": {"cpu": "500m"}}}}}'

oc patch StorageCluster ocs-storagecluster -n openshift-storage --type=merge --patch='{"spec":{"resources":{"crashcollector": {"Limit": {"cpu": "500m"}}}}}'
oc patch StorageCluster ocs-storagecluster -n openshift-storage --type=merge --patch='{"spec":{"resources":{"crashcollector": {"Request": {"cpu": "500m"}}}}}'

oc patch StorageCluster ocs-storagecluster -n openshift-storage --type=merge --patch='{"spec":{"resources":{"cleanup": {"Limit": {"cpu": "500m"}}}}}'
oc patch StorageCluster ocs-storagecluster -n openshift-storage --type=merge --patch='{"spec":{"resources":{"cleanup": {"Request": {"cpu": "500m"}}}}}'

# memory
oc patch StorageCluster ocs-storagecluster -n openshift-storage --type=merge --patch='{"spec":{"resources":{"mds": {"Limit": {"memory": "512Mi"}}}}}'
oc patch StorageCluster ocs-storagecluster -n openshift-storage --type=merge --patch='{"spec":{"resources":{"mds": {"Request": {"memory": "512Mi"}}}}}'

oc patch StorageCluster ocs-storagecluster -n openshift-storage --type=merge --patch='{"spec":{"resources":{"mgr": {"Limit": {"memory": "512Mi"}}}}}'
oc patch StorageCluster ocs-storagecluster -n openshift-storage --type=merge --patch='{"spec":{"resources":{"mgr": {"Request": {"memory": "512Mi"}}}}}'

oc patch StorageCluster ocs-storagecluster -n openshift-storage --type=merge --patch='{"spec":{"resources":{"mon": {"Limit": {"memory": "512Mi"}}}}}'
oc patch StorageCluster ocs-storagecluster -n openshift-storage --type=merge --patch='{"spec":{"resources":{"mon": {"Request": {"memory": "512Mi"}}}}}'

oc patch StorageCluster ocs-storagecluster -n openshift-storage --type=merge --patch='{"spec":{"resources":{"osd": {"Limit": {"memory": "512Mi"}}}}}'
oc patch StorageCluster ocs-storagecluster -n openshift-storage --type=merge --patch='{"spec":{"resources":{"osd": {"Request": {"memory": "512Mi"}}}}}'

oc patch StorageCluster ocs-storagecluster -n openshift-storage --type=merge --patch='{"spec":{"resources":{"rgw": {"Limit": {"memory": "512Mi"}}}}}'
oc patch StorageCluster ocs-storagecluster -n openshift-storage --type=merge --patch='{"spec":{"resources":{"rgw": {"Request": {"memory": "512Mi"}}}}}'

oc patch StorageCluster ocs-storagecluster -n openshift-storage --type=merge --patch='{"spec":{"resources":{"noobaa-db": {"Limit": {"memory": "512Mi"}}}}}'
oc patch StorageCluster ocs-storagecluster -n openshift-storage --type=merge --patch='{"spec":{"resources":{"noobaa-db": {"Request": {"memory": "512Mi"}}}}}'

oc patch StorageCluster ocs-storagecluster -n openshift-storage --type=merge --patch='{"spec":{"resources":{"prepareosd": {"Limit": {"memory": "512Mi"}}}}}'
oc patch StorageCluster ocs-storagecluster -n openshift-storage --type=merge --patch='{"spec":{"resources":{"prepareosd": {"Request": {"memory": "512Mi"}}}}}'

oc patch StorageCluster ocs-storagecluster -n openshift-storage --type=merge --patch='{"spec":{"resources":{"crashcollector": {"Limit": {"memory": "512Mi"}}}}}'  
oc patch StorageCluster ocs-storagecluster -n openshift-storage --type=merge --patch='{"spec":{"resources":{"crashcollector": {"Request": {"memory": "512Mi"}}}}}'

oc patch StorageCluster ocs-storagecluster -n openshift-storage --type=merge --patch='{"spec":{"resources":{"cleanup": {"Limit": {"memory": "512Mi"}}}}}'
oc patch StorageCluster ocs-storagecluster -n openshift-storage --type=merge --patch='{"spec":{"resources":{"cleanup": {"Request": {"memory": "512Mi"}}}}}'

# when delete namespace and namespace could not delete 
# Get all resource in this namespace
oc api-resources | grep true | awk '{print $1}'  | while read i ; do echo oc get $i -n openshift-storage ; oc get $i -n openshift-storage ; echo ; done | tee /tmp/err

# check resource 
cat /tmp/err | grep -Ev "No resources found|My Operator Catalog" | more

# patch resource finalizers to []
oc patch CephBlockPool ocs-storagecluster-cephblockpool -n openshift-storage -p '{"metadata":{"finalizers":[]}}' --type=merge

oc patch cephfilesystems ocs-storagecluster-cephfilesystem  -n openshift-storage -p '{"metadata":{"finalizers":[]}}' --type=merge

oc patch cephobjectstores ocs-storagecluster-cephobjectstore -n openshift-storage -p '{"metadata":{"finalizers":[]}}' --type=merge

oc patch cephclusters ocs-storagecluster-cephcluster -n openshift-storage -p '{"metadata":{"finalizers":[]}}' --type=merge

oc patch backingstores noobaa-default-backing-store -n openshift-storage -p '{"metadata":{"finalizers":[]}}' --type=merge

oc patch bucketclasses noobaa-default-bucket-class -n openshift-storage -p '{"metadata":{"finalizers":[]}}' --type=merge

oc patch noobaas noobaa -n openshift-storage -p '{"metadata":{"finalizers":[]}}' --type=merge

oc patch storageclusters ocs-storagecluster -n openshift-storage -p '{"metadata":{"finalizers":[]}}' --type=merge

oc patch cephobjectstoreusers noobaa-ceph-objectstore-user -n openshift-storage -p '{"metadata":{"finalizers":[]}}' --type=merge

oc patch cephobjectstoreusers ocs-storagecluster-cephobjectstoreuser -n openshift-storage -p '{"metadata":{"finalizers":[]}}' --type=merge


# 检查 openshift-storage 有哪些 pods  
[root@helper ocp4]# oc get pods -n openshift-storage
NAME                                                              READY   STATUS      RESTARTS   AGE
csi-cephfsplugin-48lnf                                            3/3     Running     0          144m
csi-cephfsplugin-ccxrx                                            3/3     Running     0          144m
csi-cephfsplugin-provisioner-5f8b66cc96-hll2d                     5/5     Running     2          144m
csi-cephfsplugin-provisioner-5f8b66cc96-ln64m                     5/5     Running     0          78m
csi-cephfsplugin-trc6b                                            3/3     Running     0          144m
csi-rbdplugin-provisioner-66f66699c8-fpswm                        5/5     Running     0          78m
csi-rbdplugin-provisioner-66f66699c8-mkkf5                        5/5     Running     0          144m
csi-rbdplugin-qvzmr                                               3/3     Running     0          144m
csi-rbdplugin-tqzmd                                               3/3     Running     0          144m
csi-rbdplugin-xq9kz                                               3/3     Running     0          144m
noobaa-core-0                                                     1/1     Running     0          143m
noobaa-db-0                                                       1/1     Running     0          143m
noobaa-endpoint-b78d759f-49l6k                                    1/1     Running     0          141m
noobaa-operator-65cb6dcf6c-mfg6v                                  1/1     Running     0          78m
ocs-operator-f95cdd554-lnc5x                                      1/1     Running     0          78m
rook-ceph-crashcollector-worker0.cluster-0001.rhsacn.org-55hz7z   1/1     Running     0          143m
rook-ceph-crashcollector-worker1.cluster-0001.rhsacn.org-8x7xql   1/1     Running     0          78m
rook-ceph-crashcollector-worker2.cluster-0001.rhsacn.org-6xx4hb   1/1     Running     0          147m
rook-ceph-drain-canary-worker0.cluster-0001.rhsacn.org-7787pq9q   1/1     Running     0          143m
rook-ceph-drain-canary-worker1.cluster-0001.rhsacn.org-56d5g8gm   1/1     Running     0          78m
rook-ceph-drain-canary-worker2.cluster-0001.rhsacn.org-5fc99n7z   1/1     Running     0          143m
rook-ceph-mds-ocs-storagecluster-cephfilesystem-a-5b5ddc68z8lnf   1/1     Running     0          78m
rook-ceph-mds-ocs-storagecluster-cephfilesystem-b-55cbcb76xts4g   1/1     Running     0          143m
rook-ceph-mgr-a-695895dbb9-5pgh8                                  1/1     Running     0          143m
rook-ceph-mon-a-7dd968f9bc-c6llt                                  1/1     Running     0          144m
rook-ceph-mon-b-6d49cdfc9d-kjddv                                  1/1     Running     0          78m
rook-ceph-mon-c-5b48599579-rv8zd                                  1/1     Running     0          144m
rook-ceph-mon-d-canary-6bdc7df6b7-92rkg                           0/1     Pending     0          86s
rook-ceph-operator-7b469677f6-t7c5c                               1/1     Running     1          144m
rook-ceph-osd-0-cbbfff44c-z4476                                   1/1     Running     0          143m
rook-ceph-osd-1-7fb75ffb49-htrz5                                  1/1     Running     0          78m
rook-ceph-osd-2-d7f669475-nd5lz                                   1/1     Running     0          143m
rook-ceph-osd-prepare-ocs-deviceset-1-data-0-gmkrk-cjt6m          0/1     Completed   0          143m
rook-ceph-osd-prepare-ocs-deviceset-2-data-0-z2pxh-jzmhm          0/1     Completed   0          143m
rook-ceph-rgw-ocs-storagecluster-cephobjectstore-a-78b8999wgmt2   1/1     Running     0          142m
rook-ceph-rgw-ocs-storagecluster-cephobjectstore-b-6b759b89jwsz   1/1     Running     0          78m
rook-ceph-tools-7fcff79f44-djknd                                  1/1     Running     0          59s

# 查询包含 ocs label 的节点
oc get nodes --show-labels | grep ocs |cut -d' ' -f1

# 查询 storagecluster 状态 
oc get storagecluster -n openshift-storage
NAME                 AGE    PHASE   EXTERNAL   CREATED AT             VERSION
ocs-storagecluster   152m   Ready              2020-10-26T05:30:33Z   4.5.0

# 检查 storagecluster 状态是否为 Ready
oc get storagecluster -n openshift-storage ocs-storagecluster -o jsonpath='{.status.phase}{"\n"}'

# 查询 storageclass 
oc -n openshift-storage get sc
NAME                                PROVISIONER                             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
localblock                          kubernetes.io/no-provisioner            Delete          WaitForFirstConsumer   false                  4h40m
nfs-storage-provisioner (default)   nfs-storage                             Delete          Immediate              false                  5h19m
ocs-storagecluster-ceph-rbd         openshift-storage.rbd.csi.ceph.com      Delete          Immediate              true                   153m
ocs-storagecluster-ceph-rgw         openshift-storage.ceph.rook.io/bucket   Delete          Immediate              false                  153m
ocs-storagecluster-cephfs           openshift-storage.cephfs.csi.ceph.com   Delete          Immediate              true                   153m
openshift-storage.noobaa.io         openshift-storage.noobaa.io/obc         Delete          Immediate              false                  3h23m

# noobaa-db 是 storageclass ocs-storagecluster-ceph-rbd 的用户
# storageclass 为 ocs-storagecluster-ceph-rbd
# pvc 为 openshift-storage/db-noobaa-db-0
# pv 为 pvc-9b2e86b4-0b00-4202-aa8e-07e32f7c74c5
oc get pv
...
pvc-9b2e86b4-0b00-4202-aa8e-07e32f7c74c5   50Gi       RWO            Delete           Bound    openshift-storage/db-noobaa-db-0                  ocs-storagecluster-ceph-rbd            149m

# 下载应用
# 参考：https://red-hat-storage.github.io/ocs-training/training/ocs4/ocs.html
# 创建项目
oc new-project my-shared-storage

# 新建应用，基于 https://github.com/christianh814/openshift-php-upload-demo git repo，采用 php:7.2 builder
oc new-app openshift/php:7.2~https://github.com/christianh814/openshift-php-upload-demo --name=file-uploader

# 查看 s2i build 镜像日志
oc logs -f bc/file-uploader -n my-shared-storage
...
Writing manifest to image destination
Storing signatures
Successfully pushed image-registry.openshift-image-registry.svc:5000/my-shared-storage/file-uploader@sha256:74029bb63e4b7cb33602eb037d45d3d27245ffbfc105fd2a4587037c6b063183
Push successful

# 生成 svc/file-uploader 对应的 route
oc expose svc/file-uploader -n my-shared-storage

# 将 deployment/file-uploader 扩展为 3 副本
oc scale --replicas=3 deployment/file-uploader -n my-shared-storage

# 确认 3 个 pod 都处于 Running 状态
oc get pods -n my-shared-storage

# 为 deployment/file-uploader 添加 claim-mode 为 RWX 的 pvc
# pvc 的名字是 my-shared-storage
# pvc 对应的 storage class 是 ocs-storagecluster-cephfs
# 加载位置为 /opt/app-root/src/uploaded
oc set volume deployment/file-uploader --add --name=my-shared-storage \
-t pvc --claim-mode=ReadWriteMany --claim-size=1Gi \
--claim-name=my-shared-storage --claim-class=ocs-storagecluster-cephfs \
--mount-path=/opt/app-root/src/uploaded \
-n my-shared-storage

# 获取 route
oc get route file-uploader -n my-shared-storage -o jsonpath --template="{.spec.host}"

# 访问 route http://file-uploader-my-shared-storage.apps.cluster-0001.rhsacn.org
# 上传文件

# 扩展 pvc my-shared-storage
oc patch pvc my-shared-storage -n my-shared-storage --type json --patch  '[{ "op": "replace", "path": "/spec/resources/requests/storage", "value": "5Gi" }]'

# 确认 pvc 已扩展，cephfs pvc 扩展对应于增大 quota，ocs 4.5 不支持减小 pvc
echo $(oc get pvc my-shared-storage -n my-shared-storage -o jsonpath='{.status.capacity.storage}')

# 安装 noobaa client
curl -sLO https://github.com/noobaa/noobaa-operator/releases/download/v2.3.0/noobaa-linux-v2.3.0 ; chmod +x noobaa-linux-v2.3.0 ; sudo mv noobaa-linux-v2.3.0 /usr/bin/noobaa

# 检查 noobaa version
noobaa version
INFO[0000] CLI version: 2.3.0                           
INFO[0000] noobaa-image: noobaa/noobaa-core:5.5.0       
INFO[0000] operator-image: noobaa/noobaa-operator:2.3.0 

# 检查 Multi Cloud Gateway 状态
noobaa status -n openshift-storage

# create storage cluster
# see: https://red-hat-storage.github.io/ocs-training/training/ocs4/ocs4-install-no-ui.html#_create_cluster
cat > storagecluster.yaml << 'EOF'
apiVersion: ocs.openshift.io/v1
kind: StorageCluster
metadata:
  name: ocs-storagecluster
  namespace: openshift-storage
spec:
  externalStorage: {}
  monDataDirHostPath: /var/lib/rook
  resources:
    mds:
      limits:
        cpu: "500m"
        memory: "512Mi"
      requests:
        cpu: "500m"
        memory: "512Mi"
    mon:
      limits:
        cpu: "500m"
        memory: "512Mi"
      requests:
        cpu: "500m"
        memory: "512Mi"
    mgr:
      limits:
        cpu: "500m"
        memory: "512Mi"
      requests:
        cpu: "500m"
        memory: "512Mi"
    osd:
      limits:
        cpu: "500m"
        memory: "512Mi"
      requests:
        cpu: "500m"
        memory: "512Mi"
    rgw:
      limits:
        cpu: "500m"
        memory: "512Mi"
      requests:
        cpu: "500m"
        memory: "512Mi"
    noobaa-db:
      limits:
        cpu: "500m"
        memory: "512Mi"
      requests:
        cpu: "500m"
        memory: "512Mi"
    noobaa-core:
      limits:
        cpu: "500m"
        memory: "512Mi"
      requests:
        cpu: "500m"
        memory: "512Mi"
    prepareosd:
      limits:
        cpu: "500m"
        memory: "50Mi"
      requests:
        cpu: "500m"
        memory: "50Mi"
    crashcollector:
      limits:
        cpu: "500m"
        memory: "60Mi"
      requests:
        cpu: "500m"
        memory: "60Mi"
    cleanup:
      limits:
        cpu: "500m"
        memory: "60Mi"
      requests:
        cpu: "500m"
        memory: "60Mi"       
  storageDeviceSets:
  - config: {}
    count: 1   
    dataPVCTemplate:
      spec:
        accessModes:
        - ReadWriteOnce
        resources:
          requests:
            storage: "1"
        storageClassName: localblock
        volumeMode: Block
    name: ocs-deviceset
    placement: {}
    replica: 3
EOF

oc create -f storagecluster.yaml

# 当集群状态为降级时，可以登录 TOOLBOX，执行以下命令检查具体错误
ceph health detail 

# 对于 health: HEALTH_WARN application not enabled on 1 pool(s) 的报错
HEALTH_WARN application not enabled on 1 pool(s)
POOL_APP_NOT_ENABLED application not enabled on 1 pool(s)
    application not enabled on pool 'ocs-storagecluster-cephobjectstore.rgw.control'

ceph osd pool application enable ocs-storagecluster-cephobjectstore.rgw.control rgw


# 配置 openshift-monitoring 使用 ocs 存储
# 在 openshift-monitoring 的 namespace 下生成 configmap cluster-monitoring-config
# 配置内容为 
# prometheusK8s 的 volumeClaimTemplate 使用 storageclass ocs-storagecluster-ceph-rbd 请求 40 Gi 存储
# alertmanagerMain 的 volumeClaimTemplate 使用 storageclass ocs-storagecluster-ceph-rbd 请求 40 Gi 存储

cat > ocslab-cluster-monitoring-noinfra.yaml << EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: cluster-monitoring-config
  namespace: openshift-monitoring
data:
  config.yaml: |
    prometheusK8s:
      volumeClaimTemplate:
        metadata:
          name: prometheusdb
        spec:
          storageClassName: ocs-storagecluster-ceph-rbd
          resources:
            requests:
              storage: 40Gi
    alertmanagerMain:
      volumeClaimTemplate:
        metadata:
          name: alertmanager
        spec:
          storageClassName: ocs-storagecluster-ceph-rbd
          resources:
            requests:
              storage: 40Gi
EOF
oc create -f ocslab-cluster-monitoring-noinfra.yaml

# 如果配置了 infranode 可考虑采用如下的 yaml 文件
# 里面配置了对应的 nodeSelector 选择 infra 节点
cat > ocslab-cluster-monitoring-withinfra.yaml << EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: cluster-monitoring-config
  namespace: openshift-monitoring
data:
  config.yaml: |+
    alertmanagerMain:
      nodeSelector:
        node-role.kubernetes.io/infra: ""
      volumeClaimTemplate:
        metadata:
          name: alertmanager
        spec:
          storageClassName: ocs-storagecluster-ceph-rbd
          resources:
            requests:
              storage: 40Gi
    prometheusK8s:
      nodeSelector:
        node-role.kubernetes.io/infra: ""
      volumeClaimTemplate:
        metadata:
          name: prometheusdb
        spec:
          storageClassName: ocs-storagecluster-ceph-rbd
          resources:
            requests:
              storage: 40Gi
    prometheusOperator:
      nodeSelector:
        node-role.kubernetes.io/infra: ""
    grafana:
      nodeSelector:
        node-role.kubernetes.io/infra: ""
    k8sPrometheusAdapter:
      nodeSelector:
        node-role.kubernetes.io/infra: ""
    kubeStateMetrics:
      nodeSelector:
        node-role.kubernetes.io/infra: ""
    telemeterClient:
      nodeSelector:
        node-role.kubernetes.io/infra: ""
EOF

oc create -f ocslab-cluster-monitoring-withinfra.yaml
```



