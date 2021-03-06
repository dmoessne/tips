## 查看接口状态

```
ip -s link show dev ens3
2: ens3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP mode DEFAULT group default qlen 1000                                    
    link/ether 00:1a:4a:16:01:88 brd ff:ff:ff:ff:ff:ff
    RX: bytes  packets  errors  dropped overrun mcast
    33400443957 20920850 0       38209   0       0
    TX: bytes  packets  errors  dropped carrier collsns
    720256020  10566989 0       0       0       0
```

## 安装配置vsftp 
参考：https://www.unixmen.com/install-vsftp-with-virtual-users-on-centos-rhel-scientific-linux-6-4/

```
yum install vsftpd -y 

cat > /tmp/virtual_users.txt << 'EOF'
jwang
redhat123
EOF

db_load -T -t hash -f /tmp/virtual_users.txt /etc/vsftpd/virtual_users.db

cat > /etc/pam.d/vsftpd_virtual << 'EOF'
#%PAM-1.0
auth    required        pam_userdb.so   db=/etc/vsftpd/virtual_users
account required        pam_userdb.so   db=/etc/vsftpd/virtual_users
session required        pam_loginuid.so
EOF

yes | cp -f /etc/vsftpd/vsftpd.conf /etc/vsftpd/vsftpd.conf.sav

cat > /etc/vsftpd/vsftpd.conf << 'EOF'
# Allow anonymous FTP? (Beware - allowed by default if you comment this out).
anonymous_enable=NO

# Uncomment this to allow local users to log in.
local_enable=YES

## Enable virtual users
guest_enable=YES

## Virtual users will use the same permissions as anonymous
virtual_use_local_privs=YES

#
# Uncomment this to enable any form of FTP write command.
write_enable=YES

## PAM file name
pam_service_name=vsftpd_virtual

## Home Directory for virtual users
user_sub_token=$USER
local_root=/ftp/virtual/$USER

# You may specify an explicit list of local users to chroot() to their home
# directory. If chroot_local_user is YES, then this list becomes a list of
# users to NOT chroot().
chroot_local_user=YES

## Hide ids from user
hide_ids=YES

# Allow writable undercloud chroot mode
allow_writeable_chroot=YES
EOF

mkdir -p /ftp/virtual/jwang
chown -R ftp:ftp /ftp
# chcon -R --reference /var/ftp/pub /ftp  
# semanage fcontext -a -t public_content_rw_t "/ftp(/.*)?"
chcon -R -t public_content_rw_t /ftp/virtual/jwang
# setsebool -P ftpd_full_access 1

firewall-cmd --add-service=ftp
firewall-cmd --add-service=ftp --permanent
firewall-cmd --reload

systemctl enable vsftpd && systemctl start vsftpd
```

注意⚠️：对于rhel7来说，安装时看看是否需要（待尝试）；经尝不需要以下命令
```
yum install db4-utils db4 -y
```

注意⚠️：遇到报错，500 OOPS: vsftpd: refusing to run with writable root inside chroot()
参考：https://www.cnblogs.com/wi100sh/p/4542819.html
```
ansible server01 -m lineinfile -a 'path=/etc/vsftpd/vsftpd.conf regexp="allow_writeable_chroot" line="allow_writeable_chroot=YES"'

ansible server01 -m service -a 'name=vsftpd state=restarted'
```

## 解决Homebrew慢的方法
因为访问github慢，所以尝试解决方法为使用镜像站点，是否有效待验证
参考：
* https://www.zhihu.com/question/31360766
* https://lug.ustc.edu.cn/wiki/mirrors/help/homebrew-bottles
* https://nwafu-mirrors-help.readthedocs.io/zh_cn/homebrew-cask-versions.git.html

替换git repo
```
cd "$(brew --repo)"
git remote set-url origin https://mirrors.aliyun.com/homebrew/brew.git

cd "$(brew --repo)/Library/Taps/homebrew/homebrew-core"
git remote set-url origin https://mirrors.aliyun.com/homebrew/homebrew-core.git

echo 'export HOMEBREW_BOTTLE_DOMAIN=https://mirrors.aliyun.com/homebrew/homebrew-bottles' >> ~/.bashrc
source ~/.bashrc

cd "$(brew --repo)"/Library/Taps/homebrew/homebrew-cask
git remote set-url origin https://mirrors.ustc.edu.cn/homebrew-cask.git

cd "$(brew --repo)"/Library/Taps/homebrew/homebrew-cask-versions
git remote set-url origin https://mirrors.nwafu.edu.cn/homebrew-cask-versions.git
```

替换homebrew bottles镜像站点
```
echo 'export HOMEBREW_BOTTLE_DOMAIN=https://mirrors.ustc.edu.cn/homebrew-bottles' >> ~/.bash_profile
source ~/.bash_profile
```

另外一种思路是使用socks代理，参见：https://www.zhihu.com/question/31360766

## 检查哪个软件包提供了命令 yum provides audit2allow
```
yum provides audit2allow
```

## 设置selinux bool变量
```
setsebool -P ftpd_full_access 1
```
## 配置ansible控制节点

生成inventory
```
cat >> /etc/ansible/hosts << 'EOF'
[group1]
server01 ansible_host=10.66.208.158 ansible_user=root
EOF
```

生成ssh public key，生成ssh config
```
ssh-keygen -t rsa -f /root/.ssh/id_rsa -N ''

cat > ~/.ssh/config << 'EOF'
Host *
   StrictHostKeyChecking no
   UserKnownHostsFile=/dev/null

EOF
```

构建控制节点到服务器间的信任关系
```
ansible all -m authorized_key -a 'user=root state=present key="{{ lookup(\"file\",\"/root/.ssh/id_rsa.pub\") }}"' -k
```

配置bbr
```
ansible server01 -m lineinfile -a 'path=/etc/sysctl.conf regexp="net.ipv4.tcp_congestion_control" line="net.ipv4.tcp_congestion_control=bbr"'
```

另外一种方式配置bbr
```
ansible server01 -m sysctl -a 'name=net.ipv4.tcp_congestion_control value=bbr sysctl_set=yes state=present reload=yes'
```

## lftp 命令
参考：https://unix.stackexchange.com/questions/93587/lftp-login-put-file-in-remote-dir-and-exit-in-a-single-command-proper-quoting

一条命令完成认证及上传
```
$ lftp -c "open -u user,pass ftpsite.com; put -O remote/dir/ /local/file.txt" 
$ lftp -c "open -u user,pass ftpsite.com; get remote/dir/file.txt" 
$ lftp -c "open -u user,pass ftpsite.com; pget -n 2 remote/dir/file.txt -o local/dir" 
```

## 配置时间服务器(RHEL8)

### chrony 服务器
```
yum install -y chrony

cat > /etc/chrony.conf << EOF
server 127.127.1.0 iburst
allow all
local stratum 4
EOF

firewall-cmd --add-service=ntp
firewall-cmd --add-service=ntp --permanent
firewall-cmd --reload

systemctl enable chronyd && systemctl start chronyd

chronyc -n sources
chronyc -n tracking
```

### chrony 客户端
```
NTPSERVER="clock.corp.redhat.com"
LOCALSUBNET=$(ip r s | grep ens3 | grep -v default | awk '{print $1}')

yum install -y chrony

cat > /etc/chrony.conf << EOF
server ${NTPSERVER} iburst
stratumweight 0
driftfile /var/lib/chrony/drift
rtcsync
makestep 10 3
bindcmdaddress 127.0.0.1
bindcmdaddress ::1
cmdallow 127.0.0.1
allow ${LOCALSUBNET}
keyfile /etc/chrony.keys
commandkey 1
generatecommandkey
noclientlog
logchange 0.5
logdir /var/log/chrony
EOF

firewall-cmd --add-service=ntp
firewall-cmd --add-service=ntp --permanent
firewall-cmd --reload

systemctl enable chronyd && systemctl start chronyd

chronyc -n sources
chronyc -n tracking
```

## osp16 undercloud registry
OSP16的undercloud上可以运行一个本地registry，用于为undercloud和overcloud提供image

* undercloud registry的监听端口是8787
* undercloud registry的服务是httpd
* undercloud registry的存放位置是/var/lib/image-serve

### OSP13 baremetal node power
```
openstack baremetal node power on <uuid>
openstack baremetal node power off <uuid>

# check node power status
ipmitool -H 172.168.5.5 -v -I lanplus -U ADMIN -P ADMIN power status
```

### restart auditd
```
service auditd restart
```

### dnsmasq infinite lease
在dnsmasq.conf文件里包含描述mac地址，ip地址，过期时间的内容
```
dhcp-range=192.168.0.0,static
dhcp-host=aa:bb:cc:dd:ee:ff,192.168.0.199,infinite
```

### 检查ip地址范围
可以访问以下网址，例子里通过检查172.23.101.4/255.255.252.0的地址查看地址范围
http://jodies.de/ipcalc?host=172.23.101.4&mask1=255.255.252.0&mask2=

### 定期重新生成网卡配置的脚本

```
cat > /usr/local/sbin/network-con-recreate.sh << 'EOF'
#!/bin/bash

# delete all connection 
nmcli -g uuid con | while read i ; do nmcli c delete uuid ${i} ; done 

# re-create primary connection 
nmcli con add type ethernet \
    con-name eth0 \
    ifname eth0 \
    ipv4.method 'manual' \
    ipv4.address '192.168.208.137/24' \
    ipv4.gateway '192.168.208.254' \
    ipv4.dns '192.168.208.254'

# restart interface
nmcli con down eth0 && nmcli con up eth0

exit 0
EOF

chmod +x /usr/local/sbin/network-con-recreate.sh

cat > ~/cron-network-con-recreate << EOF
* */1 * * * /bin/bash /usr/local/sbin/network-con-recreate.sh
EOF

crontab ~/cron-network-con-recreate
```

### 全局忽略.DS_Store
```
echo '.DS_Store' > ~/.gitignore_global

git config --global core.excludesfile ~/.gitignore_global

find . -name .DS_Store -print0 | xargs -0 git rm --ignore-unmatch

git commit -m "Remove .DS_Store from everywhere"

git push origin master
```

### Tunnelblick配置文件
```
```

### RHCOS定制文件系统和分区
参考：Red Hat Enterprise Linux CoreOS Eval Guide
```
Separate /var/log

  "storage": {
    "disks": [
      {
        "device": "/dev/sdb",
        "wipeTable": true,
        "partitions": [
          {
            "label": "LOG",
            "sizeMiB": 0,
            "startMiB": 0,
            "wipePartitionEntry": true
          }
        ]
        }
      }
    ],
    "filesystems": [
      {
        "mount": {
          "device": "/dev/sdb1",
          "format": "xfs",
          "label": "LOG",
          "wipeFilesystem": true
        }
      }
    ]
  },
    "systemd": {
     "units": [
       {
        "name": "var-log.mount",
        "enabled": true,
        "contents": "[Mount]\nWhat=/dev/sdb1\nWhere=/var/log\nType=xfs\nOptions=defaults\n[Install]\nWantedBy=local-fs.target"
       }
     ]
   }
}
```

### Mac查看监听端口
```
lsof -nP -iTCP -sTCP:LISTEN
```

### Homebrew and Homebrew Cask
参考：https://www.jianshu.com/p/fdfa9b8e29f8?utm_campaign=hugo&utm_medium=reader_share&utm_content=note

Homebrew
```
安装软件：brew install 软件名，例：brew install wget
搜索软件：brew search 软件名，例：brew search wget
卸载软件：brew uninstall 软件名，例：brew uninstall wget
更新所有软件：brew update
更新具体软件：brew upgrade 软件名 ，例：brew upgrade git
显示已安装软件：brew list
查看软件信息：brew info／home 软件名 ，例：brew info git ／ brew home git
显示包依赖：brew reps
显示安装的服务：brew services list
安装服务启动、停止、重启：brew services start/stop/restart serverName
```

Homebrew cask
```
brew cask install alfred
```

### Calico相关资料
Calico FAQ
https://docs.projectcalico.org/v3.5/usage/troubleshooting/faq

Kubernetes Networking with Calico
https://www.tigera.io/blog/kubernetes-networking-with-calico/

External connectivity
https://docs.projectcalico.org/networking/external-connectivity#outbound-connectivity

Calico over Ethernet fabrics
https://docs.projectcalico.org/reference/architecture/design/l2-interconnect-fabric

BGP Client, BGP Route Reflector and Centralized route distribution
https://docs.projectcalico.org/reference/architecture/overview#bgp-client-bird

### 解决虚拟机spice客户端显示blank screen的问题
参考：https://bugzilla.redhat.com/show_bug.cgi?id=1611625
```
TERM=linux setterm -blank 0 -powerdown 0  -powersave off >/dev/tty0 </dev/tty0
```

### 使用lvm snapshot 备份及恢复系统

新添加的磁盘/dev/sdb创建label，创建分区，设置分区类型为lvm
```
parted -s /dev/sdb mklabel msdos 
parted -s /dev/sdb unit mib mkpart primary 1 100%
parted -s /dev/sdb set 1 lvm on
```

创建pv
```
pvcreate /dev/sdb1
```

扩展vg
```
vgextend rhel /dev/sdb1
```

为逻辑卷/dev/rhel/root，创建lvm snapshot root_snap1
```
lvcreate --size 75G --snapshot --name root_snap1 /dev/rhel/root
```

更新系统...

如果需要回滚变更，可执行以下命令，然后重启系统恢复快照
```
lvconvert --merge /dev/rhel/root_snap1
yes | cp $(ls -1F -tr /boot/grub2/grub.cfg.*.rpmsave | head -1) /boot/grub2/grub.cfg
reboot 
```

### 从RHEL 7.6升级到RHEL8
```
subscription-manager register
subscription-manager attach --pool=XXXXXX
subscription-manager list --installed

subscription-manager repos --disable rhel-7-server-rpms --enable rhel-7-server-eus-rpms
subscription-manager repos --enable rhel-7-server-extras-rpms
subscription-manager release --set 7.6

yum update -y
reboot

yum install -y leapp
curl http://10.66.208.115/rhel7osp/leapp-data6.tar.gz -o /root/leapp-data6.tar.gz
tar -xzf leapp-data6.tar.gz -C /etc/leapp/files

# 升级前，事前分析
leapp preupgrade --debug 2>&1 | tee /tmp/leapp-preupgrade.log

# 根据分析报告内容/var/log/leapp/leapp-report.txt，执行以下命令解决升级冲突
sed -ie 's|^#PermitRootLogin yes|PermitRootLogin yes|' /etc/ssh/sshd_config

# 执行命令
leapp upgrade --debug 2>&1 | tee /tmp/leapp.log
```

### 使用本地软件仓库和leapp从RHEL7.5升级到RHEL8
```
mkdir -p /etc/yum.repos.d/backup
mv -f /etc/yum.repos.d/*.repo /etc/yum.repos.d/backup

cat > /etc/yum.repos.d/w.repo << 'EOF'
[rhel-7-server-rpms]
name=rhel-7-server-rpms
baseurl=http://10.66.208.115/rhel7osp/rhel-7-server-rpms/
enabled=1
gpgcheck=0

[rhel-7-server-extras-rpms]
name=rhel-7-server-extras-rpms
baseurl=http://10.66.208.115/rhel7osp/rhel-7-server-extras-rpms/
enabled=1
gpgcheck=0

EOF

mkdir -p /leapp
yum install --downloadonly --downloaddir=/leapp leapp

yum upgrade -y audit audit-libs libselinux libselinux-python libselinux-utils  libsemanage libsepol policycoreutils

yum install -y audit-libs-python bzip2 checkpolicy dnf dnf-data json-c json-glib leapp-deps libcgroup libcomps libdnf libmodulemd librepo libreport-filesystem librhsm libsemanage-python libsmartcols libsolv libyaml pciutils policycoreutils-python python-IPy python-babel python-backports python-backports-ssl_match_hostname python-enum34 python-ipaddress python-jinja2 python-markupsafe python-setuptools python2-dnf python2-futures python2-hawkey python2-leapp python2-libcomps python2-libdnf setools-libs 

pushd /leapp
rpm -ivh leapp-0.9.0-1.el7.noarch.rpm leapp-repository-0.9.0-4.el7.noarch.rpm leapp-repository-deps-0.9.0-4.el7.noarch.rpm leapp-repository-sos-plugin-0.9.0-4.el7.noarch.rpm sos-3.8-6.el7.noarch.rpm --force

popd
curl http://10.66.208.115/rhel7osp/leapp-data6.tar.gz -o /root/leapp-data6.tar.gz
tar -xzf leapp-data6.tar.gz -C /etc/leapp/files

cat >> /etc/yum.repos.d/w.repo << 'EOF'
[BASEOS]
name=BASEOS
baseurl=http://10.66.208.158/rhel8osp/rhel-8-for-x86_64-baseos-rpms/
enabled=0
gpgcheck=0

[APPSTREAM]
name=APPSTREAM
baseurl=http://10.66.208.158/rhel8osp/rhel-8-for-x86_64-appstream-rpms/
enabled=0
gpgcheck=0

EOF

export LEAPP_UNSUPPORTED=1
export LEAPP_DEVEL_SKIP_RHSM=1

yum install -y snactor
cd /usr/share/leapp-repository/repositories/system_upgrade/el7toel8/actors
snactor new-actor --produces CustomTargetRepository --tag IPUWorkflowTag --tag FactsPhaseTag CustomRepoActor

cp /usr/share/leapp-repository/repositories/system_upgrade/el7toel8/actors/customrepoactor/actor.py /usr/share/leapp-repository/repositories/system_upgrade/el7toel8/actors/customrepoactor/actor.py.sav

sed -ie "/(self):/a \ \ \ \ \ \ \ \ self.produce(CustomTargetRepository(repoid='APPSTREAM', name='APPSTREAM', baseurl='http://10.66.208.158/rhel8osp/rhel-8-for-x86_64-appstream-rpms/'))" /usr/share/leapp-repository/repositories/system_upgrade/el7toel8/actors/customrepoactor/actor.py

sed -ie "/(self):/a \ \ \ \ \ \ \ \ self.produce(CustomTargetRepository(repoid='BASEOS', name='BASEOS', baseurl='http://10.66.208.158/rhel8osp/rhel-8-for-x86_64-baseos-rpms/'))" /usr/share/leapp-repository/repositories/system_upgrade/el7toel8/actors/customrepoactor/actor.py

# 参考https://docs.google.com/document/d/1MuyO9PN9sFU0r2r01t6-OXi6RK0l1pqTFfi_UkIeGtc/edit里的步骤添加
# 编辑/usr/share/leapp-repository/repositories/system_upgrade/el7toel8/actors/customrepoactor/actor.py
#    def process(self):
#        self.produce(CustomTargetRepository(repoid='BASEOS', name='BASEOS', baseurl='http://10.66.208.158/rhel8osp/rhel-8-for-x86_64-baseos-rpms/'))
#        self.produce(CustomTargetRepository(repoid='APPSTREAM', name='APPSTREAM', baseurl='http://10.66.208.158/rhel8osp/rhel-8-for-x86_64-appstream-rpms/'))
#        pass

# 升级前，事前分析
LEAPP_UNSUPPORTED=1 LEAPP_DEVEL_SKIP_RHSM=1 leapp preupgrade --debug 2>&1 | tee /tmp/leapp-preupgrade.log

# 根据分析报告内容/var/log/leapp/leapp-report.txt，执行以下命令解决升级冲突
sed -ie 's|^#PermitRootLogin yes|PermitRootLogin yes|' /etc/ssh/sshd_config

# 卸载btrfs驱动
rmmod btrfs

# mv efibootorderfix actors to /tmp
mv /usr/share/leapp-repository/repositories/system_upgrade/el7toel8/actors/efibootorderfix/ /tmp/

# 升级
LEAPP_DEVEL_SKIP_RHSM=1 leapp upgrade --debug 2>&1 | tee /tmp/leapp-upgrade.log
```

### convert2rhel workthrough
参考链接：https://access.redhat.com/articles/2360841

```
nmcli con mod 'eth0' ipv4.method 'manual' ipv4.address 'X.X.X.X/24' ipv4.gateway 'Y.Y.Y.Y' ipv4.dns 'Z.Z.Z.Z'

mkdir -p /isos
pushd isos
curl http://pek-iso.usersys.redhat.com/iso_centos/CentOS-7-x86_64-DVD-1810.iso -o CentOS-7-x86_64-DVD-1810.iso
curl http://pek-iso.usersys.redhat.com/iso_rhel/rhel-server-7.7-x86_64-dvd.iso -o rhel-server-7.7-x86_64-dvd.iso

mkdir -p /mnt/{centos,rhel}
mount -o loop /isos/CentOS-7-x86_64-DVD-1810.iso /mnt/centos
mount -o loop /isos/rhel-server-7.7-x86_64-dvd.iso /mnt/rhel

mkdir -p /etc/yum.repos.d/backup
yes | mv -f /etc/yum.repos.d/*.repo /etc/yum.repos.d/backup

cat > /etc/yum.repos.d/centos.repo << 'EOF'
[centos-7-server-rpms-6]
name=centos-7-server-rpms-6
baseurl=file:///mnt/centos
enabled=1
gpgcheck=0
EOF

cat > /etc/yum.repos.d/rhel.repo << 'EOF'
[rhel-7-server-rpms-7]
name=rhel-7-server-rpms-7
baseurl=file:///mnt/rhel
enabled=0
gpgcheck=0
EOF

yum install -y https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm

yum install -y convert2rhel

yum-config-manager --disable epel

yum update -y

reboot

mount -o loop /isos/CentOS-7-x86_64-DVD-1810.iso /mnt/centos
mount -o loop /isos/rhel-server-7.7-x86_64-dvd.iso /mnt/rhel
convert2rhel --disable-submgr --disablerepo "*" --enablerepo rhel-7-server-rpms-7 -v Server -y --debug 2>&1 | tee /tmp/convert2rhel.log

[04/21/2020 13:50:35] TASK - [Prepare: End user license agreement] ******************************
[04/21/2020 13:50:35] TASK - [Prepare: Gather system information] *******************************
[04/21/2020 13:51:08] TASK - [Prepare: Determine RHEL variant] **********************************
[04/21/2020 13:51:08] TASK - [Prepare: Backup System] *******************************************
[04/21/2020 13:51:08] TASK - [Convert: Remove blacklisted packages] *****************************
[04/21/2020 13:51:14] TASK - [Convert: Install Red Hat release package] *************************
[04/21/2020 13:51:14] TASK - [Convert: Patch yum configuration file] ****************************
[04/21/2020 13:51:14] TASK - [Convert: Package analysis] ****************************************
[04/21/2020 13:51:14] TASK - [Convert: Check required repos] ************************************
[04/21/2020 13:51:16] TASK - [Convert: Prepare kernel] ******************************************
[04/21/2020 13:52:32] TASK - [Convert: Replace packages] ****************************************
[04/21/2020 13:55:11] TASK - [Convert: List remaining non-Red Hat packages] *********************
[04/21/2020 13:55:11] TASK - [Final: Non-interactive mode] **************************************

[root@unused ~]# rpm -qa --qf '%{NAME} %{VENDOR}\n' | grep -Ev 'Red Hat' 
yum-plugin-fastestmirror CentOS
gpg-pubkey (none)
convert2rhel Fedora Project
epel-release Fedora Project

reboot
yum remove gpg-pubkey
yum remove yum-plugin-fastestmirror

```

### 安装测试虚拟机
虚拟机使用raw format磁盘，总线为scsi，驱动对应virtio-scsi，通过字符界面安装
```
virt-install --name="jwang-testvm" --vcpus=2 --ram=4096 \
--disk path=/var/lib/libvirt/images/jwang-test-01.img,format=raw,bus=scsi,size=20 \
--os-variant rhel7.0 \
--boot menu=on \
--location /var/lib/libvirt/images/isos/rhel-server-7.6-x86_64-dvd.iso \
--graphics none \
--console pty,target_type=serial --extra-args='console=ttyS0'
```

如果在安装时希望使用kickstart文件，可以参考下面的例子
```
virt-install --name="jwang-testvm" --vcpus=2 --ram=4096 \
--disk path=/var/lib/libvirt/images/jwang-test-01.img,format=raw,bus=scsi,size=20 \
--os-variant rhel7.0 \
--boot menu=on \
--location /var/lib/libvirt/images/isos/rhel-server-7.6-x86_64-dvd.iso \
--graphics none \
--network bridge:virbr0 \
--console pty,target_type=serial \
--initrd-inject /tmp/ks.cfg \
--extra-args='ks=file:/ks.cfg console=ttyS0'
```

### Minimal rhel7 kickstart example file
参考：https://gist.github.com/devynspencer/99cbcf0b09245e285ee4
```
cat > /tmp/ks.cfg << 'EOF'
#version=RHEL7
ignoredisk --only-use=sda

# Partition clearing information
clearpart --all

# Use text install
text

# System language
lang en_US.UTF-8

# Keyboard layouts
keyboard us

# Network information
network --bootproto=static --device=eth0 --ip=192.168.122.122 --netmask 255.255.255.0 --gateway=192.168.122.1 --hostname=jwangtest.example.com

# Root password
rootpw --iscrypted $6$q9HNZaOm8rO91oRR$eSWRwR7Hc9FBRlcEm83EiJx8MeFUQJXd.33YVDjzXkgCTiY3gcMHOvDtI6wh35Zw9.7Ql6rAo9tEZpo3y7Uy6/

# Run the Setup Agent on first boot
firstboot --enable

# Do not configure the X Window System
skipx

# System timezone
timezone Asia/Shanghai --isUtc

# Agree EULA
eula --agreed

# System services
services --enabled=NetworkManager,sshd

# Reboot after installation completes
reboot

# Disk partitioning information
autopart --type=lvm --fstype=xfs

%packages --nobase --ignoremissing --excludedocs
@core
%end

%addon com_redhat_kdump --enable --reserve-mb='auto'
%end
EOF
```

### blktrace用法
参见：https://tunnelix.com/debugging-disk-issues-with-blktrace-blkparse-btrace-and-btt-in-linux-environment/
https://www.ibm.com/support/knowledgecenter/linuxonibm/com.ibm.linux.z.ldsg/ldsg_t_iodata_remote.html
https://www.hwchiu.com/blktrace-example.html
http://linuxperf.com/?p=161


服务器端
```
blktrace -l
```

客户端
```
blktrace -h <server_ip> -d <device>
e.g.
blktrace -h 10.72.32.50 -d /dev/sda
```

客户端执行i/o操作，然后服务器端退出blktrace服务端

在服务端查看blktrace产生的事物日志
```
blkparse -D 10.72.32.49-2020-05-06-12:42:33/ sda -d events.bin > events.txt
```


### Mac下直播软件OBS的设置
https://www.jianshu.com/p/ecfaac6ee7ab<br>
https://blog.csdn.net/lk142500/article/details/91491299<br>

### 什么是Systemtap，如何使用Systemtap
https://access.redhat.com/articles/882463

systemtap运行需要哪些软件包
```
systemtap, systemtap-runtime
gcc
kernel-devel, kernel-debuginfo, kernel-debuginfo-common
```

### 在Mac上启动 Web Server
```
python -m SimpleHTTPServer 8000
```

### 使用dropwatch分析drop包
参考：https://access.redhat.com/solutions/206223
``` 
1. 安装dropwatch和安装客户环境一致的内核包
# yum install -y kernel-3.10.0-957.el7
# yum install dropwatch -y
 
离线安装
# yum localinstall -y dropwatch-1.4-9.el7.x86_64.rpm
 
2. 重启系统
# systemctl reboot
 
3. 安装 debuginfo 包
# yum install -y kernel-debuginfo-3.10.0-957.el7 kernel-debuginfo-common-3.10.0-957.el7
 
离线安装
# yum localinstall -y kernel-debuginfo-3.10.0-957.el7.x86_64.rpm kernel-debuginfo-common-x86_64-3.10.0-957.el7.x86_64.rpm
 
4. 查看网卡信息：
[root@rhel7u5 ~]# ifconfig
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 10.72.37.29  netmask 255.255.254.0  broadcast 10.72.37.255
        inet6 fe80::21a:4aff:fe16:730  prefixlen 64  scopeid 0x20<link>
        inet6 2620:52:0:4824:21a:4aff:fe16:730  prefixlen 64  scopeid 0x0<global>
        ether 00:1a:4a:16:07:30  txqueuelen 1000  (Ethernet)
        RX packets 2618  bytes 560244 (547.1 KiB)
        RX errors 0  dropped 5  overruns 0  frame 0
        TX packets 721  bytes 106800 (104.2 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
 
lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        inet6 ::1  prefixlen 128  scopeid 0x10<host>
        loop  txqueuelen 1000  (Local Loopback)
        RX packets 48  bytes 3600 (3.5 KiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 48  bytes 3600 (3.5 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
 
ip -s link show 
 
5. 抓取 dropwatch 信息
[root@rhel7u5 ~]# dropwatch -l kas
Initalizing kallsyms db
dropwatch> start
Enabling monitoring...
Kernel monitoring activated.
Issue Ctrl-C to stop monitoring
1 drops at skb_queue_purge+18 (0xffffffff88a235d8)
1 drops at __brk_limit+36c27a68 (0xffffffffc06a1a68)
1 drops at __brk_limit+36c27a68 (0xffffffffc06a1a68)
1 drops at __brk_limit+36c27a68 (0xffffffffc06a1a68)
1 drops at __brk_limit+36c27a68 (0xffffffffc06a1a68)
1 drops at skb_queue_purge+18 (0xffffffff88a235d8)
1 drops at __brk_limit+36c27a68 (0xffffffffc06a1a68)
1 drops at nf_hook_slow+f3 (0xffffffff88a785e3)
1 drops at __brk_limit+36c27a68 (0xffffffffc06a1a68)
1 drops at skb_queue_purge+18 (0xffffffff88a235d8)
1 drops at ip_rcv_finish+1d4 (0xffffffff88a82604)
1 drops at __brk_limit+36c27a68 (0xffffffffc06a1a68)
2 drops at skb_queue_purge+18 (0xffffffff88a235d8)
1 drops at __brk_limit+36c27a68 (0xffffffffc06a1a68)
1 drops at ip_rcv_finish+1d4 (0xffffffff88a82604)
1 drops at skb_queue_purge+18 (0xffffffff88a235d8)
1 drops at __brk_limit+36c27a68 (0xffffffffc06a1a68)
1 drops at __brk_limit+36c27a68 (0xffffffffc06a1a68)
1 drops at __brk_limit+36c27a68 (0xffffffffc06a1a68)
1 drops at __brk_limit+36c27a68 (0xffffffffc06a1a68)
^CGot a stop message
dropwatch> exit
Shutting down ...
[root@rhel7u5 ~]#
 
 
6. 把 dropwatch 的输出放入 dropwatch.txt 文件：
[root@rhel7u5 ~]# vim dropwatch.txt
[root@rhel7u5 ~]# awk '/drops at/ {t[$4" "$5]+=$1} END {for (n in t) print t[n], n}' dropwatch.txt | sort -rn
12 __brk_limit+36c27a68 (0xffffffffc06a1a68)
6 skb_queue_purge+18 (0xffffffff88a235d8)
2 ip_rcv_finish+1d4 (0xffffffff88a82604)
1 nf_hook_slow+f3 (0xffffffff88a785e3)
 
 
7. 用 eu-addr2line 查看地址对应的函数：
 
[root@rhel7u5 ~]# yum install elfutils -y
 
[root@rhel7u5 ~]# eu-addr2line -f -k 0xffffffff88a235d8
skb_queue_purge
net/core/skbuff.c:2550
[root@rhel7u5 ~]# printf "%x\n" $((0xffffffff88a235d8+0x18))
ffffffff88a235f0
 
[root@rhel7u5 ~]# eu-addr2line -f -k 0xffffffff88a235f0
skb_complete_wifi_ack
net/core/skbuff.c:3919
 
8. 查看当前的网卡信息:
 
[root@rhel7u5 ~]# ifconfig
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 10.72.37.29  netmask 255.255.254.0  broadcast 10.72.37.255
        inet6 fe80::21a:4aff:fe16:730  prefixlen 64  scopeid 0x20<link>
        inet6 2620:52:0:4824:21a:4aff:fe16:730  prefixlen 64  scopeid 0x0<global>
        ether 00:1a:4a:16:07:30  txqueuelen 1000  (Ethernet)
        RX packets 319659  bytes 569255807 (542.8 MiB)
        RX errors 0  dropped 5  overruns 0  frame 0
        TX packets 224358  bytes 19487629 (18.5 MiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
 
lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
        inet 127.0.0.1  netmask 255.0.0.0
        inet6 ::1  prefixlen 128  scopeid 0x10<host>
        loop  txqueuelen 1000  (Local Loopback)
        RX packets 48  bytes 3600 (3.5 KiB)
        RX errors 0  dropped 0  overruns 0  frame 0
        TX packets 48  bytes 3600 (3.5 KiB)
        TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

```

### wireshark过滤规则
过滤掉 nfs, arp, http, vrrp 和 tcp 的包可以把过滤器定义成
```
!nfs and !arp and !http and !vrrp and !tcp
```

只保留LACP包
```
lacp
```

### 分析丢包问题的方法
参考：<br>
https://access.redhat.com/solutions/2194511<br>
https://access.redhat.com/articles/1311173<br>
https://access.redhat.com/solutions/3684651

```
1. 安装如下rpm包：

    kernel-devel - 和当前的内核相同版本
    kernel-debuginfo - 和当前的内核相同版本
    kernel-debuginfo-common - 和当前的内核相同版本
    gcc
    systemtap


# yum install kernel-debuginfo-3.10.0-957.el7 kernel-devel-3.10.0-957.el7 kernel-debuginfo-common-x86_64-3.10.0-957.el7 gcc

[root@rhel7u5 ~]# rpm -qa | egrep "kernel-debuginfo|kernel-devel"
kernel-debuginfo-3.10.0-957.el7.x86_64
kernel-devel-3.10.0-957.el7.x86_64
kernel-debuginfo-common-x86_64-3.10.0-957.el7.x86_64

[root@rhel7u5 ~]# uname -r
3.10.0-957.el7.x86_64

2. 执行 stap-prep：
[root@rhel7u5 ~]# stap-prep
[root@rhel7u5 ~]#

3. 创建dropwatch2.stp文件：https://access.redhat.com/solutions/2194511
   创建monitor.sh文件：https://access.redhat.com/articles/1311173
https://access.redhat.com/solutions/3684651

4. 同时在另外的两个terminal执行 monitor.sh 和 tcpdump:
[terminal 1]
./monitor.sh -d 2

[terminal 2]
# tcpdump -i ens4 -w /tmp/vm_$(hostname)-$(date +"%Y-%m-%d-%H-%M-%S").pcap

5. 执行netstat -in查看当前的RX-DRP状态

6. 执行 stap 脚本：
[root@rhel7u5 ~]# stap --all-modules -o /tmp/dropwatch2.log dropwatch2.stp

7.执行netstat -in查看测试后的RX-DRP状态

8.收集信息：
monitor.sh:
# tar cvzf net_stats_$(hostname)-$(date +"%Y-%m-%d-%H-%M-%S").tgz *network_stats_*

tcpdump:
/tmp/vm_*

dropwatch2:
/tmp/dropwatch2.log
```

### tcpdump抓包来定位rxdrop问题时如果rxdrop计数器仍然增长的处理方法
运行dropwatch2.stap的同时执行tcpdump，确认netstat -in输出里的rxdrop不再增长

### 使用cgroup限制blkio
https://andrestc.com/post/cgroups-io/

### 使用virsh设置块设备IO限流
https://fedoraproject.org/wiki/QA:Testcase_Virtualization_IO_Throttling
```
virsh blkdeviotune f18 vda
virsh blkdeviotune f18 vda --write_bytes_sec $(expr 1024 \* 1024 \* 10)
virsh blkdeviotune f18 vda
```

### 使用cgroup和systemctl设置虚拟机cpu
为某个虚拟机设置CPUQuota
```
tmpfile=$(mktemp /tmp/tmpXXXXXX)
systemd-cgls | grep machine-qemu | grep test | sed -e 's#| |-##' -e 's#| `-##' | while read i 
do
  echo systemctl set-property --runtime "'"$i"'" CPUQuota=10%
done | tee $tmpfile
sed -ie 's|x2d|\\x2d|g' $tmpfile

sh -x $tmpfile
rm -f $tmpfile
```

### 在ovirt node上执行virsh blkdeviotune
查看virsh blkdeviotune
```
prog=/usr/bin/virsh
myuser="vdsm@ovirt"
mypass="shibboleth"

args=" blkdeviotune jwang-zyjk-01 sda"
/usr/bin/expect <<EOF
set timeout -1
spawn "$prog" $args
expect "Please enter your authentication name: "
send "$myuser\r"
expect "Please enter your password:"
send "$mypass\r"
expect eof
exit
EOF
```

设置 virsh blkdeviotune 虚拟机 jwang-zyjk-01 的硬盘 sda 每秒写入字节为 10M `
```
prog=/usr/bin/virsh
myuser="vdsm@ovirt"
mypass="shibboleth"

args=" blkdeviotune jwang-zyjk-01 sda --write_bytes_sec $(expr 1024 \* 1024 \* 10)"
/usr/bin/expect <<EOF
set timeout -1
spawn "$prog" $args
expect "Please enter your authentication name: "
send "$myuser\r"
expect "Please enter your password:"
send "$mypass\r"
expect eof
exit
EOF
```

### 用tshark查看包的内容
```
# 查看有哪些包
tshark -r /tmp/pcap.ip6 

# 查看包的详情
tshark -r /tmp/pcap.ip6  -x -Y "frame.number==658"  -V
```

### quipudocs 的使用
https://quipucords.github.io/quipudocs/user.html#con-what-is-prod-common_assembly-about-common-ctxt

```
# 生成 repo 配置
cat > /etc/yum.repos.d/public.repo << 'EOF'
[rhel-7-server-rpms]
name=rhel-7-server-rpms
baseurl=http://10.66.208.115/rhel7osp/rhel-7-server-rpms
gpgcheck=0
enabled=1

[rhel-7-server-extras-rpms]
name=rhel-7-server-extras-rpms
baseurl=http://10.66.208.115/rhel7osp/rhel-7-server-extras-rpms
gpgcheck=0
enabled=1

EOF

# 在线安装 qpc-tools
yum install https://github.com/quipucords/qpc-tools/releases/latest/download/qpc-tools.el7.noarch.rpm

# qpc server install
qpc-tools server install

# qpc client install
qpc-tools cli install

# config server
qpc server config --host 10.66.208.160
# Server config /root/.config/qpc/server.config was not found.
# Server connectivity was successfully configured. The server will be contacted via "https" at host "10.66.208.160" with port "9443".

# login into server
qpc server login

# add cred
qpc cred add --type network --name qpcnetworksource --username qpctester --password --become-method su --become-user root --become-password
qpc cred add --type network --name cred_rhvhost --username root --password

# add source
qpc source add --type network --name registry --hosts 10.66.208.115 --cred qpcnetworksource
qpc source add --type network --name source_rhvhost --hosts 10.66.208.[51:53] --cred cred_rhvhost

# add scan
qpc scan add --name scan_registry1 --sources registry
qpc scan add --name scan_rhvhost1 --sources source_rhvhost

# run scan
qpc scan start --name scan_registry1
qpc scan start --name scan_rhvhost1

# view scan
qpc scan job --id 1
qpc scan list
qpc scan job --name scan_registry1

qpc scan job --id 2
qpc scan list
qpc scan job --name scan_rhvhost1

# download scan report
qpc report download --scan-job 1 --output-file=~/scan_output.tar.gz
qpc report download --scan-job 2 --output-file=~/scan_output_rhvhost.tar.gz
```

### Prometheus查询
参考：https://redhat-developer-demos.github.io/istio-tutorial/istio-tutorial/1.4.x/3monitoring-tracing.html

```
istio_requests_total{destination_service="recommendation.tutorial.svc.cluster.local"}
container_memory_rss{namespace="tutorial",container=~"customer|preference|recommendation"}
```

### 常见的pipeline包含哪些步骤
```
Build
Unit Test
Static Code Scan/Security Scan
Packaging/Publishing of Artifacts
Deploying
End to End Tests
Performance Tests
```

### Istio w/ jwt-authz
http://lab-ossm-labguides.6923.rh-us-east-1.openshiftapps.com/workshop/authentication<br>
https://istio.io/latest/docs/tasks/security/authorization/authz-jwt/

### Kiali's Graph
http://lab-ossm-labguides.6923.rh-us-east-1.openshiftapps.com/workshop/authentication

```
Within the Kiali UI select the Graph option from the left hand navigation and then choose

Namespace: -tutorial
Versioned app graph
Requests percentage
Last 1m
Every 10s
```

### Service Mesh Role Based Access Control (RBAC)
http://lab-ossm-labguides.6923.rh-us-east-1.openshiftapps.com/workshop/authentication

RbacConfig objects are used to enable and configure Authorization in the service mesh. Take a look at the following YAML:

```
apiVersion: "rbac.istio.io/v1alpha1"
kind: RbacConfig
metadata:
  name: default
spec:
  mode: 'ON_WITH_INCLUSION'
  inclusion:
    namespaces: ["-tutorial"]
```

### [转发]开放分布式追踪（OpenTracing）入门与 Jaeger 实现
https://yq.aliyun.com/articles/514488

https://www.jianshu.com/p/bd11294cf83e


### 如何降低lvm thinpool meta占用率
```
lvs -a -o +devices
df -h 
fstrim -v <device_mount_point> e.g.
fstrim -v /
```

### 检查nfs的io状态
利用nfsstat和nfsiostat工具检查nfs延时
https://www.redhat.com/sysadmin/using-nfsstat-nfsiostat

### 重置crio存储

```
-------------------------------------------------------
crio storage reset
-------------------------------------------------------

oc adm cordon $NODENAME

oc adm drain $NODENAME --ignore-daemonsets=true --delete-local-data=true


--on the worker node----

systemctl stop kubelet.service 

crictl stop $(crictl ps -q)

systemctl stop crio.service
systemctl stop crio-*


rm -rf /var/lib/containers/storage/*


systemctl start crio.service
systemctl start kubelet.service

--/------------------

oc adm uncordon $NODENAME

With the above, I was able to clear and repopulate all the images and containers.
```

### 如何针对OSP做CI/CD
https://mojo.redhat.com/community/communities-at-red-hat/infrastructure/cloud-platforms-community-of-practice/blog/2019/08/19/deploy-different-a-customer-tale-of-cicd-on-osp13

### virt-install的例子
https://raymii.org/s/articles/virt-install_introduction_and_copy_paste_distro_install_commands.html

### 启用windows远程桌面
https://support.microsoft.com/en-us/help/4028379/windows-10-how-to-use-remote-desktop

### 安装virtualbox extension pack
https://www.nakivo.com/blog/how-to-install-virtualbox-extension-pack/

### rhel8 安装 virsh
dnf install libvirt-client

### rhel8 virsh 禁用网络自动启动
virsh net-autostart --network default --disable

### cinder filters and weighters
openstack cinder 可以通过 filters 和 weighters 来选择后端存储，以下内容来自 Advanced Storage for OpenStack Storage 培训

```
crudini --set /var/lib/config-data/puppet-generated/cinder/etc/cinder/cinder.conf DEFAULT  scheduler_default_filters AvailabilityZoneFilter,CapacityFilter,CapabilitiesFilter,DriverFilter
crudini --set /var/lib/config-data/puppet-generated/cinder/etc/cinder/cinder.conf tripleo_ceph filter_function "volume.size >= 5"
sed -i 's/volume.size >= 5/"volume.size >= 5"/g' /var/lib/config-data/puppet-generated/cinder/etc/cinder/cinder.conf
crudini --set /var/lib/config-data/puppet-generated/cinder/etc/cinder/cinder.conf tripleo_nfs filter_function "volume.size < 5"
sed -i 's/volume.size < 5/"volume.size < 5"/g' /var/lib/config-data/puppet-generated/cinder/etc/cinder/cinder.conf
crudini --del  /var/lib/config-data/puppet-generated/cinder/etc/cinder/cinder.conf tripleo_ceph goodness_function
crudini --del  /var/lib/config-data/puppet-generated/cinder/etc/cinder/cinder.conf tripleo_nfs goodness_function

cat /var/lib/config-data/puppet-generated/cinder/etc/cinder/cinder.conf | grep -Ev "^$|^#" 
cat /var/lib/config-data/puppet-generated/cinder/etc/cinder/cinder.conf | grep scheduler_default_filters

# restart cinder services on the controller
systemctl restart tripleo_cinder_scheduler.service tripleo_cinder_api_cron.service tripleo_cinder_api.service

podman ps | grep cinder

# restart cinder volume service on the controller
pcs resource restart openstack-cinder-volume

# remove filter funtion and add goodness funtion
crudini --set /var/lib/config-data/puppet-generated/cinder/etc/cinder/cinder.conf DEFAULT  scheduler_default_weighers GoodnessWeigher
crudini --set /var/lib/config-data/puppet-generated/cinder/etc/cinder/cinder.conf DEFAULT  scheduler_weight_handler cinder.scheduler.weights.OrderedHostWeightHandler
crudini --set /var/lib/config-data/puppet-generated/cinder/etc/cinder/cinder.conf tripleo_ceph goodness_function 50
sed -i 's/goodness_function = 50/goodness_function = "50"/g' /var/lib/config-data/puppet-generated/cinder/etc/cinder/cinder.conf
crudini --set /var/lib/config-data/puppet-generated/cinder/etc/cinder/cinder.conf tripleo_nfs goodness_function "(volume.size > 5) ? 100 : 40"
sed -i 's/(volume.size > 5) ? 100 : 40/"(volume.size > 5) ? 100 : 40"/g' /var/lib/config-data/puppet-generated/cinder/etc/cinder/cinder.conf
crudini --del  /var/lib/config-data/puppet-generated/cinder/etc/cinder/cinder.conf tripleo_ceph filter_function
crudini --del  /var/lib/config-data/puppet-generated/cinder/etc/cinder/cinder.conf tripleo_nfs filter_function

cat /var/lib/config-data/puppet-generated/cinder/etc/cinder/cinder.conf | grep -Ev "^$|^#" 
cat /var/lib/config-data/puppet-generated/cinder/etc/cinder/cinder.conf | grep scheduler_default_weighers
cat /var/lib/config-data/puppet-generated/cinder/etc/cinder/cinder.conf | grep scheduler_default_filters


# restart cinder services on the controller
systemctl restart tripleo_cinder_scheduler.service tripleo_cinder_api_cron.service tripleo_cinder_api.service

podman ps | grep cinder

# restart cinder volume service on the controller
pcs resource restart openstack-cinder-volume

# delete all volumes in test env
openstack volume list --status available -c ID -f value | xargs openstack volume delete
```

### How do I configure OpenStack with Ceph?
https://access.redhat.com/solutions/2625991

### add/remove openstack server floating ip from server
```
openstack server add floating ip <server> <ip>
openstack server remove floating ip <server> <ip>
```

### 为glance image进行签名
```
## Generate a openssl key to be used for signing
# Generate private key
openssl genrsa -out private_key.pem 1024

# Generate public key from private key
openssl rsa -pubout -in private_key.pem -out public_key.pem

# Create a certification request (CSR)
openssl req -new -key private_key.pem -out cert_request.csr -subj "/C=CN/ST=GD/L=SZ/O=Global Security/OU=IT/CN=openstack.example.com"

# Generate certification from the CSR
openssl x509 -req -days 14 -in cert_request.csr -signkey private_key.pem -out x509_signing_cert.crt

## Create a new secret inside Barbican using the generated SSL cert.
openstack secret store --name signing-cert --algorithm RSA --secret-type certificate --payload-content-type "application/octet-stream" --payload-content-encoding base64  --payload "$(base64 x509_signing_cert.crt)" -c 'Secret href' -f value
# http://10.0.0.150:9311/v1/secrets/ef0a16ba-ed44-4b59-939c-42e053c29ac6

## Use private_key.pem to sign the image and generate the .signature file. Copy the private_key.pem and x509_signing_cert.crt from controller01 to undercloud
# wget https://download.cirros-cloud.net/0.4.0/cirros-0.4.0-x86_64-disk.img
openssl dgst -sha256 -sign private_key.pem -sigopt rsa_padding_mode:pss -out cirros-0.4.0.signature cirros-0.4.0-x86_64-disk.img

## Convert signature to Base64 and store the information in one variable.
base64 -w 0 cirros-0.4.0.signature  > cirros-0.4.0.signature.b64
cirros_signature_b64=$(cat cirros-0.4.0.signature.b64)

## Upload the image with the proper parameters
openstack image create --container-format bare --disk-format qcow2 --property img_signature="$cirros_signature_b64" --property img_signature_certificate_uuid='ef0a16ba-ed44-4b59-939c-42e053c29ac6' --property img_signature_hash_method='SHA-256' --property img_signature_key_type='RSA-PSS' cirros_0_4_0_signed < cirros-0.4.0-x86_64-disk.img
```

### rbd mirror practice
https://github.com/MiracleMa/Blog/issues/2

### 创建6个3gb的卷，让filters和weighters帮助选择合适的volume backend
```
for i in `seq 1 6` ; do openstack volume delete 3gb-vol-0$i ; done 
for i in `seq 1 6` ; do openstack volume create --size 3 3gb-vol-0$i ; done
for i in `seq 1 6` ; do openstack volume show 3gb-vol-0$i | grep os-vol-host-attr ; done 
```

### 什么是Group-Version和Kind-Resource
https://book.kubebuilder.io/cronjob-tutorial/gvks.html

### Ceph 和 SDD HDD Pool
Ceph: mix SATA and SSD within the same box<br>
这个时候主要还是通过人工编辑crush完成的<br>
https://www.sebastien-han.fr/blog/2014/08/25/ceph-mix-sata-and-ssd-within-the-same-box/<br>

通过把 crush rule 和 device class 关联，pool 和 crush rule 关联实现 pool 到 device class 的映射<br>
https://www.bookstack.cn/read/zxj_ceph/crush-class 

### What's new in RHEL8
What’s new in RHEL 8 file systems and storage<br>
https://www.redhat.com/en/blog/whats-new-rhel-8-file-systems-and-storage

What’s new in RHEL 8.1: Kernel patching, more Insights, and right on time<br>
https://www.redhat.com/en/blog/whats-new-rhel-81-kernel-patching-more-insights-and-right-time

Red Hat expands coverage of CVE fixes<br>
https://www.redhat.com/en/blog/red-hat-expands-coverage-cve-fixes

Generate SELinux policies for containers with Udica<br>
https://www.redhat.com/en/blog/generate-selinux-policies-containers-with-udica

Using the rootless containers Tech Preview in RHEL 8.0<br>
https://www.redhat.com/en/blog/using-rootless-containers-tech-preview-rhel-80<br>
https://www.redhat.com/en/blog/preview-running-containers-without-root-rhel-76<br>

### 配置 kvm usb passthrough
https://blog.csdn.net/weixin_33716557/article/details/85104312

### cinderlib 的演示
https://asciinema.org/a/TcTR7Lu7jI0pEsd9ThEn01l7n?autoplay=1

### 检查 corosync 状态
```
corosync-cfgtool -s
```

### 调整 totem token timeout
https://access.redhat.com/solutions/221263

查看corosync心跳链路状态
```
corosync-cfgtool -s
```

查看corosync日志
```
cat /var/log/cluster/corosync.log
```

调整corosync totem token timeout，编辑/etc/corosync/corosync.conf文件
```
totem {
    version: 2
    cluster_name: tripleo_cluster
    transport: knet
    crypto_cipher: aes256
    crypto_hash: sha256
    token: 30000                          # add this line
}
```

让上面的调整生效，在每个节点上编辑corosync.conf文件，添加token，然后执行
```
pcs cluster reload corosync
```

检查修改结果
```
corosync-cmapctl | grep totem | grep token 
```

确认节点心跳状态
```
corosync-cfgtool -s
```

https://www.thegeekdiary.com/how-to-change-pacemaker-cluster-heartbeat-timeout-in-centos-rhel-7/

### OpenShift Virtualization 支持 UEFI
OpenShift Virtualization and UEFI 支持
 
需要让 virt-handler 和 virt-launcher pod 里有 UEFI 所需的 firmware
 
So just for testing I did this...
```
$ oc exec -it virt-handler-hxdvs bash
# rpm -ivh http://mirror.centos.org/centos/7/os/x86_64/Packages/OVMF-20180508-6.gitee3198e672e2.el7.noarch.rpm
# ln -sf /usr/share/OVMF/OVMF_CODE.secboot.fd /usr/share/OVMF/OVMF_CODE.fd
```

Then started a VM with-
```
firmware:
         bootloader:
           efi: {}
```

It'll fail, because virt-launcher pod won't have the symlink (but will have the RPM), so I...
```
$  oc exec -it virt-launcher-example-qv5nw bash
# ln -sf /usr/share/OVMF/OVMF_CODE.secboot.fd /usr/share/OVMF/OVMF_CODE.fd
# virsh start openshift-cnv_example
```

So what we really need is the OVMF ROM's inside of the virt-launcher/virt-handler pods as shipped then it'll just work out of the box

### 如何用好 Bash 的变量替换
How To Use Bash Parameter Substitution Like A Pro<br>
https://www.cyberciti.biz/tips/bash-shell-parameter-substitution-2.html

### 修改 ssh config 不再进行严格的 HostKey 检查
```
cat > ~/.ssh/config << 'EOF'
Host *
  StrictHostKeyChecking no
  UserKnownHostsFile=/dev/null  
EOF
``` 

### 在 rhel7 上启用 software collection 软件频道并且安装 maven 3.0 
https://www.softwarecollections.org/en/scls/rhscl/maven30/

```
# On RHEL, enable RHSCL repository for you system:
sudo yum-config-manager --enable rhel-server-rhscl-7-rpms

# 2. Install the collection:
$ sudo yum install maven30

# 3. Start using the software collection:
$ scl enable maven30 bash

# At this time you should be able to use maven as a normal application. Some available command examples follow:
$ mvn --version
$ mvn package
$ mvn clean dependency:copy-dependencies package
$ mvn site
```

### 遇到不支持 STP 的设备，记得关闭 bridge.stp
在实验室里的设备不支持 STP，默认 bridge.stp 是启用的，因此创建的 bridge 跟外面无法通信。
https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/networking_guide/sec-network_bridging_using_the_networkmanager_command_line_tool_nmcli
```
nmcli con modify bridge-br0 bridge.stp no
```

### OpenShift Cheatsheet
http://www.mastertheboss.com/soa-cloud/openshift/openshift-cheatsheet

### ssh client config 可以让 ssh 访问快起来
根据需要配置
```
cat > ~/.ssh/config << 'EOF'
Host *
  StrictHostKeyChecking no
  CheckHostIP no
  VerifyHostKeyDNS no
  GSSAPIAuthentication no
EOF
``` 

### 在 ovirt hypervisor 上执行 virsh 命令
可指定认证文件 authfile=/etc/ovirt-hosted-engine/virsh_auth.conf 
```
virsh -c qemu:///system?authfile=/etc/ovirt-hosted-engine/virsh_auth.conf list
```

### 为 ovirt 虚拟机设置磁盘 QoS 
```
tmpfile=$(mktemp /tmp/tmpXXXXXX)

for domain in $(virsh -c qemu:///system?authfile=/etc/ovirt-hosted-engine/virsh_auth.conf list | grep testvm | awk '{print $2}')
do
    if [ x$domain == 'x' ]; then
        exit 0
    else    
        for device in $(virsh -c qemu:///system?authfile=/etc/ovirt-hosted-engine/virsh_auth.conf domblklist ${domain} | awk '{print $1}')
        do
            echo virsh -c qemu:///system?authfile=/etc/ovirt-hosted-engine/virsh_auth.conf blkdeviotune ${domain} ${device} --write_bytes_sec $(expr 1024 \* 1024 \* 1)
        done
    fi
done | tee $tmpfile
sh -x $tmpfile
rm -f $tmpfile
```

### 为 ovirt 虚拟机设置网络 QoS
http://manpages.ubuntu.com/manpages/trusty/man1/virsh.1.html
```
tmpfile=$(mktemp /tmp/tmpXXXXXX)

for domain in $(virsh -c qemu:///system?authfile=/etc/ovirt-hosted-engine/virsh_auth.conf list | grep testvm | awk '{print $2}')
do
    if [ x$domain == 'x' ]; then
        exit 0
    else
       for device in $(virsh -c qemu:///system?authfile=/etc/ovirt-hosted-engine/virsh_auth.conf domiflist ${domain} | grep -E "^vnet" | awk '{print $1}')
       do
            SPEED=$(expr 1024 \* 10)
            echo virsh -c qemu:///system?authfile=/etc/ovirt-hosted-engine/virsh_auth.conf domiftune ${domain} ${device} --live --inbound ${SPEED},${SPEED},${SPEED} --outbound ${SPEED},${SPEED},${SPEED}
       done
    fi
done | tee $tmpfile
sh -x $tmpfile
rm -f $tmpfile
```

### 实验环境 AMQ Broker 和 AMQ Broker Maven
https://www.opentlc.com/labs/amq/amq-broker-7.6.0-bin.zip<br>
https://www.opentlc.com/labs/amq/amq-broker-7.6.0-maven-repository.zip<br>

### 更新 ovirt 管理员密码
https://access.redhat.com/solutions/63677<br>
```
ovirt-aaa-jdbc-tool user password-reset admin --password-valid-to="2055-08-01 12:00:00Z"
```

### 设置 irssi 登陆时自动加入 channel
https://irssi.org/documentation/startup/#server-and-channel-automation<br>
```
/CHANNEL ADD -auto #secret IRCnet password
```

### 离线安装 qpc 
```
# upload QPC package
scp ~/Downloads/QPC\ Dependencies_76.zip root@<qpcserverip>:~
scp ~/Downloads/quipucords_server_image.tar.gz root@<qpcserverip>:~
scp ~/Downloads/postgres.9.6.10.tar root@<qpcserverip>:~

# log into qpcserver
ssh root@<qpcserverip>

# define repo
cat > /etc/yum.repos.d/w.repo << 'EOF'
[rhel-7-server-rpms]
name=rhel-7-server-rpms
baseurl=http://10.66.208.115/rhel7osp/rhel-7-server-rpms/
enabled=1
gpgcheck=0

[rhel-7-server-extras-rpms]
name=rhel-7-server-extras-rpms
baseurl=http://10.66.208.115/rhel7osp/rhel-7-server-extras-rpms/
enabled=1
gpgcheck=0

EOF

# prepare qpc content
mkdir /root/qpc
cd /root/qpc
mv /root/quipucords_server_image.tar.gz .
mv /root/postgres.9.6.10.tar .

# install unzip and unzip QPC\ Dependencies_76.zip
yum install -y unzip
unzip /root/QPC\ Dependencies_76.zip

# install dependency
yum install -y ansible policycoreutils-python selinux-policy selinux-policy-base selinux-policy-targeted libseccomp libtirpc

# install offline rpm packages
rpm -Uvh *.rpm --force

# install qpc server - offline
qpc-tools server install --offline-files=/root/qpc --version=0.9.2

# install qpc client - offline
qpc-tools cli install --offline-files=/root/qpc

# config qpc server, replace 10.66.208.160 with ip address in your env
qpc server config --host 10.66.208.160

# login qpc server as 'admin' user
qpc server login

# add cred
qpc cred add --type network --name cred_rhel --username root --password

# add source
qpc source add --type network --name source_rhel --hosts 10.66.208.[51:53] --cred cred_rhel

# add scan
qpc scan add --name scan_rhel --sources source_rhel

# exec scan
qpc scan start --name scan_rhel

# check if scan is complete
qpc scan list
qpc scan job --name scan_rhel | grep status 

# download scan report, scan job id is from output of command 'qpc scan list'
qpc report download --scan-job 1 --output-file=~/scan_output_rhel.tar.gz
```

### OpenShift 下如何实际路由分片
OpenShift Router Sharding for Production and Development Traffic<br>
https://www.openshift.com/blog/openshift-router-sharding-for-production-and-development-traffic

### OpenShift 下 Custom 类型的 Deployment Strategy 的例子
https://github.com/openshift/origin/issues/11963

### Github webhook 与 Jenkins 的集成
https://dzone.com/articles/adding-a-github-webhook-in-your-jenkins-pipeline

### Kata Container 的架构介绍
https://www.cnblogs.com/xiaochina/p/12812158.html<br>
https://blog.csdn.net/feeltouch/article/details/88631193<br>

### CRI-O 1.0 的介绍
https://www.redhat.com/en/blog/introducing-cri-o-10

### DEFCON 28 Safe Mode Presentations
https://www.youtube.com/playlist?list=PL9fPq3eQfaaBk9DFnyJRpxPi8Lz1n7cFv

### OSP 16.1 - new layout of the network_yaml file
https://github.com/broskos/dell-lab/blob/16.1-ExternalCeph/templates/network_data.yaml

### OpenShift 4 如何改变 MTU
https://access.redhat.com/solutions/5307301

### AMQ JMS Client 启用分布式跟踪
https://access.redhat.com/documentation/en-us/red_hat_amq/7.7/html-single/using_the_amq_jms_client/index#enabling_distributed_tracing

### OpenShift 4.5.3 在 Openstack 上的 UPI 离线安装
https://github.com/davidsajare/david-share/blob/master/OpenShift/OpenShift%204.5.3%E5%9C%A8Openstack%E4%B8%8A%E7%9A%84UPI%E7%A6%BB%E7%BA%BF%E5%AE%89%E8%A3%85.txt

### Mac OS iostat and top
```
iostat -d -K 1
top -o cpu
```

### 如何理解 Kafka 与传统消息队列解决方案
https://www.confluent.io/blog/apache-kafka-vs-enterprise-service-bus-esb-friends-enemies-or-frenemies/

### Red Hat Network Adapter Fast Datapath Feature Support Matrix
https://access.redhat.com/articles/3538141

### OCP 3.11 config map 覆盖 AMQ 配置
```
You can mount the /opt/amq/conf folder as a configmap.Then, you can edit the activemq.xml directly from the configmap.

For instance:

mkdir conf & cd conf
oc rsync amq63-amq-1-7jc97:/opt/amq/conf .
oc create configmap amq63-conf --from-file=. 
#configmap/broker-etc created
oc set volume dc/amq63-amq --overwrite --add --name=amq63-conf --type=configmap --configmap-name=amq63-conf --mount-path=/opt/amq/conf
```

### OCP4 安装后需要完成的一些任务
https://post-install-config--ocpdocs.netlify.app/openshift-enterprise/latest/post_installation_configuration/cluster-tasks.html

### 在 OpenShift 下如何为容器指定可定制化的 Satellite 软件仓库
https://github.com/atgreen/howto-change-container-yum-source

### 清理 Satellite 镜像仓库不完整镜像内容的方法
https://access.redhat.com/solutions/3363761

清理脚本
```
#!/bin/bash

docker_v2_dir="/var/lib/pulp/published/docker/v2/master/"
docker_repo=$1
#blobs missing on disk
tmp_mblobs=$(mktemp /tmp/mblob.XXXX)
#blobs with checksum errors
tmp_cblobs=$(mktemp /tmp/cblob.XXXX)

rm -f /tmp/cblob
rm -f /tmp/mblob


function Usage {

echo "Usage: $0 $organization-$docker_repo_name"
echo "  Example: $0 AcmeCorp-openshift3_metrics-hawkular"
echo ""
        exit 1
}

if [ $# -eq 0 ]    
then
        Usage   
fi

txtbld=$(tput bold)             # bold
bldred=${txtbld}$(tput setaf 1) # red
bldylw=${txtbld}$(tput setaf 3) # yellow
bldgrn=${txtbld}$(tput setaf 2) # green
txtrst=$(tput sgr0)             # reset

function safe {
        pass=${bldgrn}$1${txtrst}
        echo -e "####################\n${pass}  \n####################\n\n"
}

function crit {
        crit=${bldred}$1${txtrst}
        echo -e "####################\n${crit}  \n#################\n\n"
}

function warn {
        warn=${bldylw}$1${txtrst}
        echo -e "####################\n${warn}  \n#################\n\n"
}



safe "Checking Docker repo $docker_repo for invalid blobs, that might take time !!!"
#Follow symlinks
for blob in `find $docker_v2_dir/$docker_repo/*/blobs -type l -exec readlink -m {} ';'`;do
  #File doesn't exist
  if [ ! -f $blob ] ; then
      crit "File $blob is missing on disk"
      echo $blob >> $tmp_mblobs
  #File exists
  elif [ -f $blob ] ; then
      #checksum is correct?
      ondisk_chksum=`sha256sum $blob | awk '{print$1}'`
      #is this the best way?
      fname_chksum=`echo $blob| cut -d: -f2`
      if [ "$ondisk_chksum" != "$fname_chksum" ];then
        crit "File $blob sha256sum is wrong"
        echo $blob >> $tmp_cblobs
      fi
  fi
done


if [ -s $tmp_cblobs ] || [ -s $tmp_mblobs ] ; then
    sort $tmp_cblobs|uniq > /tmp/cblob
    sort $tmp_mblobs|uniq > /tmp/mblob

    for blob in `cat /tmp/cblob`;do
    warn "Removing $blob entry from mongoDB"
    mongo pulp_database --eval "db.units_docker_blob.remove({_storage_path: \"$blob\"})"
    done

    for blob in `cat /tmp/mblob`;do
    warn "Removing $blob entry from mongoDB"
    mongo pulp_database --eval "db.units_docker_blob.remove({_storage_path: \"$blob\"})"
    done

    #cleanup
    rm -f /tmp/mblob
    rm -f /tmp/cblob
    rm -f $tmp_cblobs
    rm -f $tmp_mblobs
    safe "Please proceed to satellite WebUI and sync docker repo $docker_repo"
else
    safe "Repo $docker_repo looks fine!!"
fi
```

### 关于 imagebuilder 的视频
https://www.youtube.com/watch?v=UopGqYs0PKA&t=7s<br>
![imagerbuilder user examples](https://github.com/wangjun1974/tips/blob/master/os/pics/imagebuild-adduser-example1.png)

### Github Pull Request(PR) 的含义
https://hackernoon.com/how-to-git-pr-from-the-command-line-a5b204a57ab1

### 很不错的关于 Github/BitBucket/Gitlab 的 Pull Request/Merge Request 工作流程
https://medium.com/@paulrohan/workflow-of-pull-request-or-merge-request-to-github-bitbucket-gitlab-b0942ec5d56e

### OCP4 fluentd 多行日志 RFE
https://docs.openshift.com/container-platform/4.5/architecture/architecture-installation.html#unmanaged-operators_architecture-installation<br>
https://issues.redhat.com/browse/RFE-706<br>

### Istio 如何实现 Deny All 策略
https://www.stackrox.com/post/2019/08/istio-security-basics-running-microservices-on-zero-trust-networks/

几个重要的讨论过程
```
You can set authz policy to deny all (https://istio.io/latest/docs/reference/config/security/authorization-policy/) for a specific workload...

yep, that's what I'm saying. mTLS + authz. We cover pretty much all of it in the DO328 course that we recently wrote. Take a look at ch08, security. Inside the course we also specify how to restrict egress, though I'm not sure which chapter that is.

'Authorization Policy scope (target) is determined by “metadata/namespace” and an optional “selector”. - “metadata/namespace” tells which namespace the policy applies. If set to root namespace, the policy applies to all namespaces in a mesh. - workload “selector” can be used to further restrict where a policy applies.'

Q: What is a _root_ ns? Is that a namespace literally called `root`, being a magic value for SM? Or are you talking about a future feature, e.g. https://kubernetes.io/blog/2020/08/14/introducing-hierarchical-namespaces/ ?
A: it's the control plane namespace

Q: Authz is ns-bound unless it's in the control plane, in which case it influences the whole SM? If that's the case, is it documented somewhere? Because this sounds like an unexpected behavior to me.
A: Set .global.configRootNamespace to define it, by default it is the control plane namespace

Q: Could you point me to the docs please? I don't see it in https://access.redhat.com/documentation/en-us/openshift_container_platform/4.5/html-single/service_mesh/index
A: See https://archive.istio.io/v1.4/docs/reference/config/security/authorization-policy/

Q: Which also doesn't mention `configRootNamespace` though..?
A: That's the AuthorizationPolicy section which describes the behaviour to be expected with policies

Q: Yep, that I understand. But I just don't know where we missed the magic behavior of root namespace, because that's important and I can't find it anywhere.
A: https://archive.istio.io/v1.4/docs/reference/config/istio.mesh.v1alpha1/
There's a table there which describes rootNamespace
but that's general and leaves the behaviour to the other sections, like the Authorization Policy
```

### OpenShift Active Directory LDAP 例子
https://examples.openshift.pub/authentication/activedirectory-ldap/

### OpenShift 4 运维
在 Google Chat OpenShift 4 里找 “OCP 4.x Day 2 Operations”

### RHEL CoreOS 相关资料
https://docs.google.com/presentation/d/1lxE7ArDTv-O3cK2QTUKBYis4tVolqQ863dgVSRg_Qbg/edit#slide=id.g518d9a1131_5_0<br>
https://docs.google.com/presentation/d/1LMqYeXoltwjeU8G8kx9DDch8JF8ysf-SJV6UMNefnxY/edit<br>

### RHEL7 升级后 vnc 报错的处理
如果报如下错误，可尝试更新系统，然后重启系统
```
 vncext:      VNC extension running!
 vncext:      Listening for VNC connections on all interface(s), port 5903
 vncext:      created VNC server for screen 0
/root/.vnc/xstartup: line 5:  7863 Terminated              /etc/X11/xinit/xinitrc
Killing Xvnc process ID 7854
```

### RHEL7 如何设置默认 Java
https://access.redhat.com/documentation/en-us/jboss_enterprise_application_platform/5/html/installation_guide/sect-use_alternatives_to_set_default_jdk
```
/usr/sbin/alternatives --config java
There are 2 programs which provide 'java'.

  Selection    Command
-----------------------------------------------
*+ 1           java-1.8.0-openjdk.x86_64 (/usr/lib/jvm/java-1.8.0-openjdk-1.8.0.262.b10-0.el7_8.x86_64/jre/bin/java)
   2           java-11-openjdk.x86_64 (/usr/lib/jvm/java-11-openjdk-11.0.8.10-0.el7_8.x86_64/bin/java)

Enter to keep the current selection[+], or type selection number: 2
```

### 查看 openstack stack 失效资源
```
openstack stack resource -n 5 list overcloud | grep -v COMPLETE
openstack stack failures list overcloud
```

### OpenShift 4.5 文档链接
https://docs.openshift.com/container-platform/4.5/welcome/index.html<br>
https://access.redhat.com/documentation/en-us/openshift_container_platform/4.5/

### RHEL 是如何做到 ABI 兼容性的
https://access.redhat.com/articles/rhel-abi-compatibility<br>
https://mojo.redhat.com/docs/DOC-1080350

### 添加用户
```
# gen encrypt passwd
python -c "import crypt, getpass; print(crypt.crypt(getpass.getpass(), crypt.METHOD_SHA512))"

# gen chpasswd format file in form <user>:<pass>
cat > /tmp/pass << 'EOF'
tester:$6$WPuEqIRfSyKjwY6e$3eLkZXDNI6Ysn9412nF4tCgvRMMhHr0Mfx19Hw82tJdC/yOpS3c4WKk1r4c0aVY4qO.35s0101Uxme9hrHv6Q1
EOF

# Apply password by run command chpasswd
cat /tmp/pass | chpasswd -e
```

```
useradd -m tester
passwd tester

echo "tester ALL=(root) NOPASSWD:ALL" | sudo tee -a /etc/sudoers.d/tester
sudo chmod 0440 /etc/sudoers.d/tester

su - tester

python -c "import crypt, getpass; print(crypt.crypt(getpass.getpass(), crypt.METHOD_SHA512))"
```

### 在 KubeVirt 下如何导入虚拟机磁盘模版并且从虚拟机磁盘模版创建虚拟机
https://kubevirt.io/2020/KubeVirt-VM-Image-Usage-Patterns.html

### 开源 IPAM 
一个开源的IP地址管理系统
https://phpipam.net/

### 在创建 OCS4 StorageCluster 之前需要满足的条件
以下命令输出的状态必须是 'Successed'
```
oc get csv -n openshift-storage -o json | jq '.items[0].status.phase'```
```

### rpm-ostree相关信息
https://rpm-ostree.readthedocs.io/en/latest/<br>
What is rpm-ostree?<br>
rpm-ostree is a hybrid image/package system. It uses libOSTree as a base image format, and accepts RPM on both the client and server side, sharing code with the dnf project; specifically libdnf.

https://github.com/coreos/rpm-ostree<br>
rpm-ostree combines libostree (an image system), with libdnf (a package system), bringing many of the benefits of both together.

```
                         +-----------------------------------------+
                         |                                         |
                         |       rpm-ostree (daemon + CLI)         |
                  +------>                                         <---------+
                  |      |     status, upgrade, rollback,          |         |
                  |      |     pkg layering, initramfs --enable    |         |
                  |      |                                         |         |
                  |      +-----------------------------------------+         |
                  |                                                          |
                  |                                                          |
                  |                                                          |
+-----------------|-------------------------+        +-----------------------|-----------------+
|                                           |        |                                         |
|         libostree (image system)          |        |            libdnf (pkg system)          |
|                                           |        |                                         |
|   C API, hardlink fs trees, system repo,  |        |    ties together libsolv (SAT solver)   |
|   commits, atomic bootloader swap         |        |    with librepo (RPM repo downloads)    |
|                                           |        |                                         |
+-------------------------------------------+        +------------------------------
```

### 关于 DCO - Developer Certificate of Origin
https://github.com/apps/dco<br>
This App enforces the Developer Certificate of Origin (DCO) on Pull Requests. It requires all commit messages to contain the Signed-off-by line with an email address that matches the commit author.<br>

### 关于 OpenShift Infra Node 的视频
https://www.youtube.com/watch?v=9VNjDh1vPXI&feature=youtu.be&t=262

### reposync and modules on RHEL8
https://www.reddit.com/r/redhat/comments/d5e1w6/reposync_and_modules_on_rhel8/
```
reposync --download-path="$localPath" --download-metadata --repoid=$i  --setopt=repo_id.module_hotfixes=1
```

### RHEL8 上同步离线仓库的方法
https://redhatnordicssa.github.io/rhel8-really-fast
```
#!/bin/bash

localPath="/repos/rhel8osp/"
fileConn="/getPackage/"

## RHEL 8 OSP
# rhel-8-for-x86_64-baseos-rpms
# rhel-8-for-x86_64-appstream-rpms

for i in rhel-8-for-x86_64-baseos-rpms rhel-8-for-x86_64-appstream-rpms 
do

  rm -rf "$localPath"$i/repodata
  echo "sync channel $i..."
  reposync -n --delete --download-path="$localPath" --repoid $i --downloadcomps --download-metadata

  echo "create repo $i..."
  #createrepo "$localPath"$i
  time createrepo -g $(ls "$localPath"$i/repodata/*comps.xml) --update --skip-stat --cachedir /tmp/empty-cache-dir "$localPath"$i

done

exit 0
```

### RHEL8 reposync 的一个问题解决
--newest-only does not download the latest package<br>
https://bugzilla.redhat.com/show_bug.cgi?id=1833074
```
2. After using "reposync -n --download-metadata --repo=<repo-id>", when a package is identified, you can download the latest and put it in the repository with no further changes. The repodata already thinks the package is there. Using rhsnd as my example.

    # yumdownloader rhnsd
    # mv rhnsd-5.0.35-3.module+el8+2754+6a08e8f4.x86_64.rpm <repo-id>/Packages/r/
```

### DNF module 相关命令
https://docs.fedoraproject.org/en-US/modularity/installing-modules/
```
  <modular command>     disable: disable a module with all its streams
                        enable: enable a module stream
                        info: print detailed information about a module
                        install: install a module profile including its packages
                        list: list all module streams, profiles and states
                        provides: list modular packages
                        remove: remove installed module profiles and their packages
                        repoquery: list packages belonging to a module
                        reset: reset a module
                        update: update packages associated with an active stream

dnf module list
...
Hint: [d]efault, [e]nabled, [x]disabled, [i]nstalled

dnf module list | grep nginx
nginx                1.14 [d]     common [d]                               nginx webserver                                                             
nginx                1.16         common [d]                               nginx webserver                                                             
dnf module list | grep nginx
...
nginx                1.14 [d]     common [d]                               nginx webserver                                                             
nginx                1.16 [e]     common [d]                               nginx webserver     

dnf module install nginx:1.16
...

dnf module list | grep nginx
...
nginx                1.14 [d]     common [d]                               nginx webserver                                                             
nginx                1.16 [e]     common [d] [i]                           nginx webserver      
```

### OSP13 with OVS/DPDK templates
https://gitlab.consulting.redhat.com/bajmera/osp13-hci-nfvi/-/blob/master/templates/nic-configs/computeovsdpdksriov.yaml

### Ansible Tower 3.7.2 如何在 RHEL 8 上安装
```
# 从红帽官网下载 rsyslog rsyslog-8.1911 rhel8 软件包
https://drive.google.com/file/d/1hGybyRfL5fFZbgXNVThspbLjs6MlpcjV/view?usp=sharing

# 从以下网址下载 ansible-tower-3.7.2 bundle rhel8 离线 repo 
https://drive.google.com/file/d/155jVxypKKPg_0zw2Rs3A0Aq7pQzwik9D/view?usp=sharing

# 在目标主机上准备 rhel8 yum repo rhel-8-for-x86_64-baseos-rpms, rhel-8-for-x86_64-appstream-rpms 和 ansible-2.8-for-rhel-8-x86_64-rpms
cat > /etc/yum.repos.d/public.repo << 'EOF'
[rhel-8-for-x86_64-baseos-rpms]
name=rhel-8-for-x86_64-baseos-rpms
baseurl=http://10.66.208.158/rhel8osp/rhel-8-for-x86_64-baseos-rpms/
gpgcheck=0
enabled=1

[rhel-8-for-x86_64-appstream-rpms]
name=rhel-8-for-x86_64-appstream-rpms
baseurl=http://10.66.208.158/rhel8osp/rhel-8-for-x86_64-appstream-rpms/
gpgcheck=0
enabled=1

[ansible-2.8-for-rhel-8-x86_64-rpms]
name=ansible-2.8-for-rhel-8-x86_64-rpms
baseurl=http://10.66.208.158/rhel8osp/ansible-2.8-for-rhel-8-x86_64-rpms/
gpgcheck=0
enabled=1
EOF

# 在目标主机上准备本地文件，用于覆盖 ansible-tower 安装程序生成的 repo 文件
cat > /root/ansible-tower.repo << 'EOF'
[ansible-tower]
name=Ansible Tower Repository - $releasever $basearch
baseurl=file:///repos/ansible-tower
enabled=0
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release

[ansible-tower-dependencies]
name=Ansible Tower Dependencies Repository - $releasever $basearch
baseurl=file:///repos/ansible-tower-dependencies
enabled=0
gpgcheck=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release
EOF

# 在目标主机上生成后台刷新脚本
cat /root/test.sh << 'EOF'
for ((;;))
do
  sleep .1
  echo yes | cp -rf /root/ansible-tower.repo /etc/yum.repos.d
done
EOF

# 在目标主机上的1个终端里运行此脚本
/bin/bash -x /root/test.sh

# 在安装主机上解压缩 3.7.2 对应的 ansible-tower-setup-latest.tar.gz 压缩包
# 进入解压后的目录
# 执行 setup 脚本
./setup 
```
### RHEL7 使用 dnf 的例子
https://blog.remirepo.net/post/2019/12/03/Install-PHP-7.4-on-CentOS-RHEL-or-Fedora<br>
https://www.vultr.com/docs/use-dnf-to-manage-software-packages-on-centos-7<br>

### 在 Ansible Tower 里使用 dynamic inventory 如何处理 group_vars
在 plugin 里创建 keyed_groups，然后在 group_vars 里创建对应的 infra_type.yml 和 environment.yml
```
I do this from an AWS inventory, but the idea should be the same. Use the Inventory plugin to pull the inventory hosts, then create groups. Those groups would then correspond to `group_vars/<group_name>.yml`


keyed_groups:
  - key: tags['infra_type']
  - key: tags['environment']

In my AWS plugin file. Then group vars that are named by infra_type tags and environment tags. According to the docs, the Azure plugin can do the same
```

### ServiceNow 与 Ansible Tower 集成
https://cloudautomation.pharriso.co.uk/post/snow-call-tower/

### Mac 上 如何查看 RHV 虚拟机的 Console
https://github.com/heinlein/ovirt-console/blob/master/ovirt-console

保存以上内容到~/bin/ovirt-console

```
mkdir -p ~/bin
pushd ~/bin
curl -O https://raw.githubusercontent.com/heinlein/ovirt-console/master/ovirt-console
chmod +x ~/bin/ovirt-console

sed -ie '/ovirt-console/d' ~/.bashrc

cat >> ~/.bashrc << 'EOF'
alias ovc='~/bin/ovirt-console ~/Downloads/console.vv'
EOF


```

### 如何在 Mac OS X 上降低 Google Chrome 的 CPU 占用
```
# make chrome be nice to other process
for f in $(pgrep 'Chrome'); do renice +20 -p $f; done

# make chrome normal
for f in $(pgrep 'Chrome'); do renice 0 -p $f; done

```
 
### 如何在 url 里将特殊字符进行编码
https://www.w3schools.com/tags/ref_urlencode.ASP

### 如何在 kubevirt 下通过 cloudinit 为虚拟机指定静态 IP 地址
```
---
apiVersion: kubevirt.io/v1alpha3
kind: VirtualMachineInstance
metadata:
  labels:
    special: vmi1
  name: vmi1
spec:
  domain:
    devices:
      disks:
      - disk:
          bus: virtio
        name: containerdisk
      - disk:
          bus: virtio
        name: cloudinitdisk
      interfaces:
      - masquerade: {}
        name: default
      - bridge: {}
        name: br10
    machine:
      type: ""
    resources:
      requests:
        memory: 1024M
  networks:
  - name: default
    pod: {}
  - name: br10
    multus:
      networkName: br10
  volumes:
  - containerDisk:
      image: kubevirt/fedora-cloud-container-disk-demo
    name: containerdisk
  - cloudInitNoCloud:
      networkData: |
        version: 2
        ethernets:
          eth1:
            addresses: [ 192.168.111.10/24 ]
      userData: |-
        #!/bin/bash
        echo "fedora" |passwd fedora --stdin
    name: cloudinitdisk
```

### Using Operator Lifecycle Manager on restricted networks
https://docs.openshift.com/container-platform/4.5/operators/olm-restricted-networks.html<br>

同时请参考王征写的离线 Operator Hub 和离线 Operator 的相关内容<br>
https://github.com/wangzheng422/docker_env/blob/master/redhat/ocp4/4.5/4.5.disconnect.operator.md<br>

### 在 AWS 上建立 FTP 服务
https://medium.com/tensult/configure-ftp-on-aws-ec2-85b5b56b9c94

### 更新 RHV admin@internal 用户口令
https://access.redhat.com/solutions/63677

### Open Data Hub 白皮书
https://gitlab.com/opendatahub/opendatahub.io/blob/master/pages/arch.md

### OpenShift 3.4 配置 External IP 的步骤
配置 master 和 node 的路由，以及服务的 External IP，实现：外部 -> external ip -> 主机转发 -> service iptables -> Pod 的过程
https://docs.openshift.com/container-platform/3.4/dev_guide/expose_service/expose_internal_ip_service.html

### OpenShift 4.6 复制离线 operator hub 的步骤
```
The process is simpler now, but the docs are not updated yet.
https://issues.redhat.com/browse/OSDOCS-1349 tracks that work.

A quick summary of the changes:

- You no longer need to run `oc adm catalog build` at all
- You can reference the default catalogs directly when mirroring, i.e.
`oc adm catalog mirror
registry.redhat.io/redhat/redhat-operator-index:v4.6`
- You'll need to `oc image mirror` the index image itself (i.e.
`registry.redhat.io/redhat/redhat-operator-index`), at least until `oc
adm catalog mirror` is updated to include that image.
- If you configure the ICSP for the index images to use the default
image names, there is no need to disable default catalogs via the
OperatorHub config.
```

### 同步 OperatorHub 内容到离线环境
```
- On an internet connected hosts:

1) Stand up podman registry
2) oc adm catalog build to create the catalog image for redhat-operators
3) oc adm catalog mirror to the podman registry on the connected node

This will result in all of the images being pulled into your local podman registry and a mapping.txt file. The mapping.txt file will contain lines like:

registry.redhat.io/#####   localhost:5000/######

Drag the data directory backing the podman registry and the mapping file over to the disconnected environment
Stand up a temporary podman registry with that data directory
update the mapping.txt file such that the line I referred to above now says

localhost:5000/####   registry.private.permanent:5000/#####

run oc image mirror --file mapping.txt

This will not result in any reachback to the internet on the disconnected cluster.
```

### 关于 OpenShift Lifecycle Manager 在离线环境下更新 CatalogSource version 版本的问题
```
IHAC wants to upgrade the Red Hat operator registry image in disconnected mode. Mirroring the new operator registry image and changing CatalogSource image version does the job.

2- is there any map of supported Red Hat operator registry image versions vs OpenShift versions ?
Q1. I found no documentation about this procedure in doc, is this supported ? Otherwise, what's the official procedure ?
A1. You can use the docs here for updating https://docs.openshift.com/container-platform/4.5/operators/olm-restricted-networks.html#olm-updating-operator-catalog-image_olm-restricted-networks. As you mention just update the version number of the catalogue image. If you only need a catalogue with a small list of operators, you can use the scripts here . https://github.com/arvin-a/openshift-disconnected-operators

Q2. is there any map of supported Red Hat operator registry image versions vs OpenShift versions ?
A2. There is no matrix, but on the operator level its usually n + or - 1 from the OCP version the operator version was released on. For example OCS 4.5 will be supported for OCP 4.4, 4.5, 4.6. To be on the safe side I would check with the individual engineering team for now.
```

### GitHub Merge a pull request
https://docs.github.com/en/github/collaborating-with-issues-and-pull-requests/merging-a-pull-request

### How to install the Operator on OCP4 using only CLI ?
https://access.redhat.com/solutions/5240251<br>
```
apiVersion: operators.coreos.com/v1alpha1
kind: Subscription
metadata:
  name: jaeger-product
  namespace: openshift-operators          <--- Set the target namespace, default "openshift-opeartors" supported the all namespaces Install mode.
spec:
  channel: stable                         <--- Set specific channel
  name: jager-product
  installPlanApproval: Automatic          <--- Set install plan, Automatic or Manual
  source: redhat-operators                <--- Set catalog source name
  sourceNamespace: openshift-marketplace  <--- Set catalog source namespace name

// Create new project
$ oc new-project myjaeger

// Verify the CSV is created in the new project
$ oc get csv
NAME                      DISPLAY                    VERSION   REPLACES   PHASE
jaeger-operator.v1.17.4   Red Hat OpenShift Jaeger   1.17.4

// Check "alm-examples" in the Annotations section, usually the sample CRD format is provided here.               
$ oc describe csv jaeger-operator.v1.17.4
Name:         jaeger-operator.v1.17.4
Namespace:    myjaeger
Labels:       olm.api.c9f771e815ec55e=provided
              olm.copiedFrom=myoperators
Annotations:  alm-examples:
                [
                  {
                    "apiVersion": "jaegertracing.io/v1",
                    "kind": "Jaeger",
                    "metadata": {
                      "name": "jaeger-all-in-one-inmemory"
                    }
                  }
                ]
:

// Create CRD for launching pods
$ oc create -n myjaeger -f - <<EOF
  {
    "apiVersion": "jaegertracing.io/v1",
    "kind": "Jaeger",
    "metadata": {
      "name": "jaeger-all-in-one-inmemory"
    }
  }
EOF

// Check the lauched pod though the CRD
$ oc get pod -n myjaeger
NAME                                         READY   STATUS    RESTARTS   AGE
jaeger-all-in-one-inmemory-f95749dc6-vr9vw   2/2     Running   0          21s
```

### 为 OpenShift Project 设置合理的 LimitRange
设置合适的默认 LimitRange 可以让集群资源被更合理的使用，这项设置对生产集群尤其重要
```
apiVersion: v1
kind: LimitRange
metadata:
  name: ${PROJECT_NAME}-core-resource-limits
  namespace: ${PROJECT_NAME}
spec:
   limits:
   - type: Container
     max:
       cpu: 2
       memory: 6Gi
     default:
       cpu: 500m[e]
       memory: 1.5Gi
     defaultRequest:
       cpu: 50m
       memory: 256Mi
   - type: Pod
     max:
       cpu: 2
       memory: 12Gi

# modify default limitrange for template/project-request 
oc edit template project-request -n openshift-config
```

### 在 RHEL/CentOS 7.x 上使用 ISCSI 存储或者 FC 存储
ISCSI 存储需安装 sg3_utils, device_mapper_multipath 和 iscsi-initiator-utils 软件包
```
yum install -y sg3_utils iscsi-initiator-utils device-mapper-multipath
```

FC 存储需安装 sg3_utils 和 device_mapper_multipath 软件包
```
yum install -y sg3_utils device-mapper-multipath
```

|软件包|说明|
|---|---|
| sg3_utils | 处理 SCSI 指令 |
| device-mapper-multipath | 配置服务器与存储间的多路径 |
| iscsi-initiator-utils | 运行 iscsi initiator 所需程序 |

RHEL7 与 IBM FlashSystem V7200 的 /etc/multipath.conf 配置例子
```
devices {
    device {
        vendor “IBM”
        product “2145”
        path_grouping_policy “group_by_prio”
        path_selector “service-time 0”
        prio “alua”
        path_checker “tur”
        failback “immediate”
        no_path_retry 5
        rr_weight uniform
        rr_min_io_rq “1”
        dev_loss_tmo 120
    }
}
```

### OpenShift Pipeline 相关信息
Tekton Trigers<br>
https://github.com/tektoncd/triggers

当 repository 发生预先定义的事件，通过 Webhook 的方式将事件相关信息发送给 EventListener，EventListener 通过 TriggerBinding 解析接收到的 HTTP JSON body 获得感兴趣的参数，这些参数可被 TriggerTemplate 用来作为参数传递给生成 PipelineResources 和 PipelineRun 

### 什么是 KEDA
https://keda.sh/

KEDA 是 Kubernetes Event Driven Autoscaler 的缩写。是对 Kubernetes 的 Horizontal Pod Autoscaler 的补充，可以基于事件驱动进行水平扩展

### Infiniband
https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/7/html/networking_guide/ch-configure_infiniband_and_rdma_networks<br>
InfiniBand refers to two distinct things. The first is a physical link-layer protocol for InfiniBand networks. The second is a higher level programming API called the InfiniBand Verbs API. The InfiniBand Verbs API is an implementation of a remote direct memory access (RDMA) technology.
RDMA provides direct access from the memory of one computer to the memory of another without involving either computer’s operating system. This technology enables high-throughput, low-latency networking with low CPU utilization, which is especially useful in massively parallel computer clusters.<br>
In a typical IP data transfer, application X on machine A sends some data to application Y on machine B. As part of the transfer, the kernel on machine B must first receive the data, decode the packet headers, determine that the data belongs to application Y, wake up application Y, wait for application Y to perform a read syscall into the kernel, then it must manually copy the data from the kernel's own internal memory space into the buffer provided by application Y. This process means that most network traffic must be copied across the system's main memory bus at least twice (once when the host adapter uses DMA to put the data into the kernel-provided memory buffer, and again when the kernel moves the data to the application's memory buffer) and it also means the computer must execute a number of context switches to switch between kernel context and application Y context. Both of these things impose extremely high CPU loads on the system when network traffic is flowing at very high rates and can make other tasks to slow down.<br>
RDMA communications differ from normal IP communications because they bypass kernel intervention in the communication process, and in the process greatly reduce the CPU overhead normally needed to process network communications. The RDMA protocol allows the host adapter in the machine to know when a packet comes in from the network, which application should receive that packet, and where in the application's memory space it should go. Instead of sending the packet to the kernel to be processed and then copied into the user application's memory, it places the contents of the packet directly in the application's buffer without any further intervention necessary. However, it cannot be accomplished using the standard Berkeley Sockets API that most IP networking applications are built upon, so it must provide its own API, the InfiniBand Verbs API, and applications must be ported to this API before they can use RDMA technology directly.<br>
Red Hat Enterprise Linux 7 supports both the InfiniBand hardware and the InfiniBand Verbs API. In addition, there are two additional supported technologies that allow the InfiniBand Verbs API to be utilized on non-InfiniBand hardware:<br>
The Internet Wide Area RDMA Protocol (iWARP)<br>
iWARP is a computer networking protocol that implements remote direct memory access (RDMA) for efficient data transfer over Internet Protocol (IP) networks.<br>
The RDMA over Converged Ethernet (RoCE) protocol, which later renamed to InfiniBand over Ethernet (IBoE).<br>
RoCE is a network protocol that allows remote direct memory access (RDMA) over an Ethernet network.<br>

关于InfiniBand架构和知识点漫谈<br>
https://zhuanlan.zhihu.com/p/74238082

InfiniBand, RDMA, iWARP, RoCE , CNA, FCoE, TOE, RDMA, iWARP, iSCSI等概念<br>
https://blog.csdn.net/jhzh951753/article/details/78813666

Working with RDMA in RedHat/CentOS 7.*<br>
https://www.rdmamojo.com/2014/10/11/working-rdma-redhatcentos-7/

### Bridge 与 macvlan 的对别
https://hicu.be/bridge-vs-macvlan

### Linux 虚拟网络接口介绍
Introduction to Linux interfaces for virtual networking<br>
https://developers.redhat.com/blog/2018/10/22/introduction-to-linux-interfaces-for-virtual-networking/#macvlan

### 网络性能测试工具
https://medium.com/@duhroach/tools-to-profile-networking-performance-3141870d5233

非常好的一片文章，介绍使用 netperf 和 ping 来评估网络时延，并且设定可对比的时间间隔
https://cloud.google.com/blog/products/networking/using-netperf-and-ping-to-measure-network-latency

iperf<br>
https://iperf.fr/

### How many Packets per Second per port are needed to achieve Wire-Speed?
每个端口每秒需要多少个数据包才能达到线速？<br>
https://kb.juniper.net/InfoCenter/index?page=content&id=kb14737

### RHEL6 添加 iptables 规则
```
iptables -I INPUT 1 -m state --state NEW -m tcp -p tcp --dport 5901 -j ACCEPT
iptables -I INPUT 1 -m state --state NEW -m tcp -p tcp --dport 6001 -j ACCEPT
```

### Tekton and ArgoCD
https://developers.redhat.com/blog/2020/09/03/introduction-to-tekton-and-argo-cd-for-multicluster-development/?sc_cid=7013a0000025vsOAAQ<br>
https://youtu.be/pVZ-3LEIHc8<br>

### OpenShift 4 如何指定使用的 Cipher
Ingress controller TLS profiles<br>
https://docs.openshift.com/container-platform/4.5/networking/ingress-operator.html<br>
Recommended configurations<br>
https://wiki.mozilla.org/Security/Server_Side_TLS#Recommended_configurations<br>

### OCP4 with Azure AD 
https://www.arctiq.ca/our-blog/2020/1/30/ocp4-auth-with-azure-ad/

group sync operator<br>
https://github.com/redhat-cop/group-sync-operator

### Declarative OpenShift
https://github.com/redhat-cop/declarative-openshift<br>
This repository contains sets of example resources to be used with a declarative management strategy. Please familiarize yourself with the terminology in that document before reading on.

The purpose of these examples is twofold:
* To act as supporting content for a GitOps series being written for uncontained.io
* To serve as a starting point for establishing a GitOps practice for cluster management

### OpenShift Node Tuning Operator
https://docs.openshift.com/container-platform/4.1/scalability_and_performance/using-node-tuning-operator.html

### Node feature discovery for Kubernetes
https://github.com/openshift/node-feature-discovery<br>

### Virtio-networking 系列高质量博客
https://www.redhat.com/en/virtio-networking-series

### SCTP
https://en.wikipedia.org/wiki/Stream_Control_Transmission_Protocol

### 基于 Open Data Hub 构建 ML 平台
https://www.openshift.com/blog/building-an-open-ml-platform-with-red-hat-openshift-and-open-data-hub-project

### 如何使用 Custom Resource Definition 扩展 Kubernetes 
https://kubernetes.io/docs/tasks/extend-kubernetes/custom-resources/custom-resource-definitions/

### Writing Your First Kubernetes Operator
非常好的文章介绍如何使用 Operator SDK 帮助生成 Operator 来扩展 Kubernetes<br>
https://medium.com/faun/writing-your-first-kubernetes-operator-8f3df4453234

### 为 undercloud 添加 overcloud 控制节点登陆 alias
```
(undercloud) [stack@undercloud ~]$ 
cat >> ~/.bashrc << 'EOF'

alias c0="source ~/stackrc; ssh heat-admin@$( openstack server list | grep controller-0 | awk -F'|' '{print $5}' | awk -F'=' '{print $2}' )"
alias c1="source ~/stackrc; ssh heat-admin@$( openstack server list | grep controller-1 | awk -F'|' '{print $5}' | awk -F'=' '{print $2}' )"
alias c2="source ~/stackrc; ssh heat-admin@$( openstack server list | grep controller-2 | awk -F'|' '{print $5}' | awk -F'=' '{print $2}' )"
alias cp0="source ~/stackrc; ssh heat-admin@$( openstack server list | grep compute-0 | awk -F'|' '{print $5}' | awk -F'=' '{print $2}' )"
alias cp1="source ~/stackrc; ssh heat-admin@$( openstack server list | grep compute-1 | awk -F'|' '{print $5}' | awk -F'=' '{print $2}' )"
EOF
```

### RHEL8 如何同步软件频道内容
参见：https://access.redhat.com/solutions/23016
```
# yum install yum-utils createrepo
# reposync -p /var/www/html --download-metadata --repo=<repo id>

```

### 寻找占用资源较多的 kvm 虚拟机 
Quick How To: Finding IO Abuser in KVM VM<br>
https://mellowhost.com/blog/quick-how-to-finding-io-abuser-in-kvm-vm.html<br>
这篇博客介绍了 virt-top 工具，使用它可以查看哪个 kvm 虚拟机占用资源较多，尤其是 blkio 角度
```
virt-top -o blockwrrq -3 
```

### Patch code under linux
参考：https://www.cyberciti.biz/faq/appy-patch-file-using-patch-command/
```
(undercloud) [stack@undercloud ~]$ cp ~/templates-custom/roles_data.yaml ~/templates-custom/roles_data.yaml.orig
(undercloud) [stack@undercloud ~]$ 
cat > patch-roles-data-templates-custom << EOF
--- /home/stack/templates-custom/roles_data.yaml.orig   2020-09-24 00:48:18.831700531 -0400
+++ /home/stack/templates-custom/roles_data.yaml        2020-09-24 00:51:06.340984487 -0400
@@ -21,8 +21,6 @@
       subnet: storage_subnet
     StorageMgmt:
       subnet: storage_mgmt_subnet
-    Tenant:
-      subnet: tenant_subnet
   # For systems with both IPv4 and IPv6, you may specify a gateway network for
   # each, such as ['ControlPlane', 'External']
   default_route_networks: ['External']
@@ -181,7 +179,6 @@
     - OS::TripleO::Services::SwiftProxy
     - OS::TripleO::Services::SwiftDispersion
     - OS::TripleO::Services::SwiftRingBuilder
-    - OS::TripleO::Services::SwiftStorage
     - OS::TripleO::Services::Timesync
     - OS::TripleO::Services::Timezone
     - OS::TripleO::Services::TripleoFirewall
@@ -203,6 +200,8 @@
       subnet: tenant_subnet
     Storage:
       subnet: storage_subnet
+    ProviderNetwork:
+      subnet: provider_network_subnet
   HostnameFormatDefault: '%stackname%-novacomputeiha-%index%'
   RoleParametersDefault:
     TunedProfileName: "virtual-host"
@@ -267,6 +266,8 @@
       subnet: internal_api_subnet
     Tenant:
       subnet: tenant_subnet
+    ProviderNetwork:
+      subnet: provider_network_subnet
   tags:
     - external_bridge
   HostnameFormatDefault: '%stackname%-networker-%index%'
EOF

(undercloud) [stack@undercloud ~]$ patch ~/templates-custom/roles_data.yaml < patch-roles-data-templates-custom 
```

### OpenShift/OLM 下，如何通过 yaml 安装 operator 

```
Basically, OperatorGroup is required to specify where the required permission to create for an Operator, and what namespaces are required to watch the CR for an Operator.

In other words, if you want to install ArgoCD Operator to "argocd" namespace, CSV checks the OperatorGroup in target namespace to decide what namespaces need to create RBAC for deploying Operator pod and CRDs. If "InstallMode" is "OwnNamespace", create role and rolebinding at the namespace, if "AllNamespaces", create clusterrole and clusterrolebinding for the Operator. 

After installing the Operator, the Operator watches its custom resources based on the namespaces specified in the OperatorGroup.

Further information is here: Operator Multitenancy with OperatorGroups

For instance, if you create #1, #2, #3 in order, then InstallPlan is created in "argocd" -> the CSV which will create CRD, RBAC, Serviceaccount in the "argocd" targetNamespace belong to "og-for-argocd" OperatorGroup is also created by it. 
Finally, the Operator is created in the same namespace, the Operator watch if CR is created or not at only the target "argocd" namespace specified Operatorgroup. If CR is created, the Operator create required resources for ArgoCD.

#1 Create namespace(project)
apiVersion: v1
 kind: Namespace
 metadata:
   name: argocd
   
#2 Create OperatorGroup which is matched with the InstallMode.
kind: OperatorGroup
apiVersion: operators.coreos.com/v1
metadata:
  name: og-for-argocd
  namespace: argocd
spec:
  targetNamespaces:
  - argocd            <--- CSV is looking at this, then it creates required permission resources for installing the Operator.
 
#3 Create Subscription of the Operator
apiVersion: operators.coreos.com/v1alpha1
 kind: Subscription
 metadata:
   name: argocd-operator
   namespace: argocd
:

```

### OpenShift: 维护模式下把工作负载迁移到其他节点上 
https://www.techbeatly.com/2018/11/openshift-cluster-how-to-drain-or-evacuate-a-node-for-maintenance.html#.X3A3M9MzbOQ

```
oc adm manage-node compute-102 --schedulable=false
oc adm drain compute-102 --delete-local-data --ignore-daemonsets  --force
oc adm manage-node compute-102 --list-pods
# do maintenance work here
oc adm manage-node compute-102 --schedulable=true
```

### 关于 Pod Disruption Budget 的说明
https://jimmysong.io/kubernetes-handbook/concepts/pod-disruption-budget.html

### Velero 面向 Kuberenetes 的备份恢复 API
https://velero.io/<br>
https://github.com/vmware-tanzu/velero<br>

### 删除可能存在问题的文件
```
...

info: Planning completed in 570ms
uploading: helper.cluster-0001.rhsacn.org:5000/ocp4/openshift4 sha256:c9fa7d57b9028d4bd02b51cef3c3039fa7b23a8b2d9d26a6ce66b3428f6e2457 72.71MiB
warning: Layer size mismatch for sha256:c9fa7d57b9028d4bd02b51cef3c3039fa7b23a8b2d9d26a6ce66b3428f6e2457: had 76247035, wrote 78250687

find /opt/registry/mirror -name "sha256:c9fa*" -exec rm {} \;
```

### 获取 Engine 配置，设置 Engine 配置
https://www.ovirt.org/develop/developer-guide/engine/engine-config-options.html
```
# get all engine config 
engine-config -a

# get engine config about StorageDomainFailureTimeoutInMinutes
engine-config --get StorageDomainFailureTimeoutInMinutes

# set engine config about StorageDomainFailureTimeoutInMinutes to 10
engine-config --set StorageDomainFailureTimeoutInMinutes=10
```

### 查看 Ansible Task 的文件情况
```
ansible_task_pid=$(ps awx | grep "/usr/bin/python.*\.ansible/tmp" | grep -Ev "/bin/sh|grep"  | awk '{print $1}')
watch ls -l $(lsof -p $ansible_task_pid | grep ".ansible/tmp" | grep -Ev "grep" | awk '{print $9}')
```

### 通过 ovirt-shell 访问 rhv
https://www.ovirt.org/develop/release-management/features/infra/cli.html
```
cat > ~/.ovirtshellrc << EOF
[ovirt-shell]
username = admin@internal
url = https://rhvm.rhcnsa.org/ovirt-engine/api
#insecure = False
#filter = False
#timeout = -1
password = 321321
EOF

# download ca.pem from rhv manager
curl -k 'https://rhvm.rhcnsa.org/ovirt-engine/services/pki-resource?resource=ca-certificate&format=X509-PEM-CA' -o /tmp/ca.pem

# test connection
ovirt-shell -c -A /tmp/ca.pem

# remove disk
for i in bootstrap master-0 master-1 master-2 worker-0 worker-1 worker-2
do 
cat > ovirt-shell-cmd << EOF
list disks --parent-vm-name jwang-$i 
EOF

diskid=$(ovirt-shell -c -A /tmp/ca.pem -f ovirt-shell-cmd | grep id | head -1 | awk '{print $3}' )

cat > ovirt-shell-cmd << EOF
remove disk $diskid
EOF

ovirt-shell -c -A /tmp/ca.pem -f ovirt-shell-cmd

done

# remove another disk from workers
for i in worker-0 worker-1 worker-2
do 
cat > ovirt-shell-cmd << EOF
list disks --parent-vm-name jwang-$i 
EOF

diskid=$(ovirt-shell -c -A /tmp/ca.pem -f ovirt-shell-cmd | grep id | awk '{print $3}' )

cat > ovirt-shell-cmd << EOF
remove disk $diskid
EOF

ovirt-shell -c -A /tmp/ca.pem -f ovirt-shell-cmd

done

# add disk
for i in bootstrap master-0 master-1 master-2 worker-0 worker-1 worker-2
do 
cat > ovirt-shell-cmd << EOF
add disk --parent-vm-name jwang-$i --provisioned_size 85899345920 --interface virtio_scsi --format cow --storage_domains-storage_domain storage_domain.name=DS31 --bootable true --wipe_after_delete false
EOF

ovirt-shell -c -A /tmp/ca.pem -f ovirt-shell-cmd
done

# add another disk on worker nodes
for i in worker-0 worker-1 worker-2
do 
cat > ovirt-shell-cmd << EOF
add disk --parent-vm-name jwang-$i --name jwang-${i}_Disk2 --provisioned_size 85899345920 --interface virtio_scsi --format cow --storage_domains-storage_domain storage_domain.name=DS31 --wipe_after_delete false
EOF

ovirt-shell -c -A /tmp/ca.pem -f ovirt-shell-cmd
done

# activate Disk2 on workers
for i in worker-0 worker-1 worker-2
do 
cat > ovirt-shell-cmd << EOF
action disk jwang-${i}_Disk2 activate --parent-vm-name jwang-$i 
EOF

ovirt-shell -c -A /tmp/ca.pem -f ovirt-shell-cmd
done

# stop vm 
for i in bootstrap master-0 master-1 master-2 worker-0 worker-1 worker-2
do 
cat > ovirt-shell-cmd << EOF
action vm jwang-$i stop --async true
EOF

ovirt-shell -c -A /tmp/ca.pem -f ovirt-shell-cmd
done

# start vm
for i in master-0 master-1 master-2 worker-0 worker-1 worker-2
do 
cat > ovirt-shell-cmd << EOF
action vm jwang-$i start --async true
EOF

ovirt-shell -c -A /tmp/ca.pem -f ovirt-shell-cmd
done
```

# 检查 OpenShift master 节点启动日志
```
journalctl -u kubelet -u crio
pods from crictl ps -a using crictl logs <id>
```

### 检查目标主机的事件和 ocp 证书过期时间
对于证书过期报错，请检查节点时间和证书过期时间，see also: https://bugzilla.redhat.com/show_bug.cgi?id=1760181
```
date
openssl s_client -connect api.crc.testing:6443 | openssl x509 -noout -dates
```

### 检查 rhcos 的日志， openshift 部署 troubleshooting
https://access.redhat.com/articles/4292081
```
sudo journalctl > /tmp/err
cat /tmp/err | grep -Ev "I1014|kernel:|systemd|ignition|dracut|multipathd|iscsid|rhcos-fips|coreos-cryptfs|ostree-prepare-root|ostree-remount|restorecon|auditd|augenrules|chronyd|NetworkManager|sssd|dbus-daemon|network-manager|rpc.statd|rhcos-growpart|sshd|rpm-ostree|polkitd|dbus-daemon|machine-config-daemon" | more
```

### 输出 RHEL 7 nfs server 和尝试挂载 nfs 文件系统
```
# on nfs server 
systemctl restart nfs-server

# export nfs share on nfs server
exportfs -r 

# check nfs exports
showmount -e <nfs-server>

# mount nfs share on nfs client
mount -t nfs <nfs-server>:<share> /mnt
umount /mnt
```

### Variations on imagestreams in OpenShift 4
https://itnext.io/variations-on-imagestreams-in-openshift-4-f8ee5e8be633

### build custom operator catalog in disconnect ocp4 
# see: https://www.cnblogs.com/ericnie/p/11777384.html
```
mkdir -p ${HOME}/redhat-operators/build/manifests
cd ${HOME}/redhat-operators

curl https://quay.io/cnr/api/v1/packages?namespace=redhat-operators > packages.txt

cat packages.txt | sed -e 's|,"name"|\n"name"|g'  | grep local
"name":"redhat-operators/local-storage-operator","namespace":"redhat-operators","releases":["79.0.0","78.0.0","77.0.0","76.0.0","75.0.0","74.0.0","73.0.0","72.0.0","71.0.0","70.0.0","69.0.0","68.0.0","67.0.0","66.0.0","65.0.0","64.0.0","63.0.0","62.0.0","61.0.0","60.0.0","59.0.0","58.0.0","57.0.0","56.0.0","55.0.0","54.0.0","53.0.0","52.0.0","51.0.0","50.0.0","49.0.0","48.0.0","47.0.0","46.0.0","45.0.0","44.0.0","43.0.0","42.0.0","41.0.0","40.0.0","39.0.0","38.0.0","37.0.0","36.0.0","35.0.0","34.0.0","33.0.0","32.0.0","31.0.0","30.0.0","29.0.0","28.0.0","27.0.0","26.0.0","25.0.0","24.0.0","23.0.0","22.0.0","21.0.0","20.0.0","19.0.0","18.0.0","17.0.0","16.0.0","15.0.0","14.0.0","13.0.0","12.0.0","11.0.0","10.0.0","9.0.0","8.0.0","7.0.0","6.0.0","5.0.0","4.0.0","3.0.0","2.0.0","1.0.0"],"updated_at":"2020-10-05T05:06:33","visibility":"public"},{"channels":null,"created_at":"2019-10-16T20:37:32","default":"79.0.0","manifests":["helm"]

curl https://quay.io/cnr/api/v1/packages/redhat-operators/local-storage-operator/79.0.0

digest=$(curl -s https://quay.io/cnr/api/v1/packages/redhat-operators/local-storage-operator/79.0.0 | jq -r '.[]|.content.digest' )

# 将 operator 的内容保存为1个 tar.gz 的包
curl -XGET https://quay.io/cnr/api/v1/packages/redhat-operators/local-storage-operator/blobs/sha256/${digest} \
    -o local-storage-operator.tar.gz

mkdir -p manifests/ 
tar -xf local-storage-operator.tar.gz -C manifests/

tree manifests/
manifests/
`-- local-storage-operator-12k64npz
    |-- 4.2
    |   |-- local-storage-operator.v4.2.0.clusterserviceversion.yaml
    |   `-- local-volumes.crd.yaml
    |-- 4.2-s390x
    |   |-- local-storage-operator.v4.2.0.clusterserviceversion.yaml
    |   `-- local-volumes.crd.yaml
    |-- 4.3
    |   |-- local-storage-operator.v4.3.0.clusterserviceversion.yaml
    |   `-- local-volumes.crd.yaml
    |-- 4.4
    |   |-- local-storage-operator.v4.4.0.clusterserviceversion.yaml
    |   `-- local-volumes.crd.yaml
    |-- 4.5
    |   |-- local-storage-operator.v4.5.0.clusterserviceversion.yaml
    |   `-- local-volumes.crd.yaml
    `-- local-storage-operator.package.yaml

cat manifests/local-storage-operator.package.yaml
channels:
- currentCSV: local-storage-operator.4.2.36-202006230600.p0
  name: '4.2'
- currentCSV: local-storage-operator.4.2.36-202006230600.p0-s390x
  name: 4.2-s390x
- currentCSV: local-storage-operator.4.3.37-202009151447.p0
  name: '4.3'
- currentCSV: local-storage-operator.4.4.0-202009161309.p0
  name: '4.4'
- currentCSV: local-storage-operator.4.5.0-202009161248.p0
  name: '4.5'
defaultChannel: '4.5'

export LOCAL_REG="helper.cluster-0001.rhsacn.org:5000"
export LOCAL_SECRET_JSON="${HOME}/pull-secret-2.json"
export OCS_OPERATOR_VERSION="4.5"

cat manifests/local-storage-operator-12k64npz/4.5/local-storage-operator.v4.5.0.clusterserviceversion.yaml  | grep registry | sed -e "s|^.*value: ||g" -e "s|^.*image: ||g" -e "s|^.*containerImage: ||g" | sort -u | while read i ; 
do 
  echo oc image mirror --filter-by-os=\'.*\' -a ${LOCAL_SECRET_JSON} ${i} $(echo $i | sed -e "s|registry.redhat.io|${LOCAL_REG}|" -e "s|@sha256.*$|:${OCS_OPERATOR_VERSION}|")
done | tee /tmp/sync-local-storage-operator-images.sh

podman login registry.redhat.io
podman login helper.cluster-0001.rhsacn.org:5000

/bin/bash -x /tmp/sync-local-storage-operator-images.sh

sed -ie 's|registry.redhat.io|helper.cluster-0001.rhsacn.org:5000|g' manifests/local-storage-operator-12k64npz/4.5/local-storage-operator.v4.5.0.clusterserviceversion.yaml
sed -ie 's|@sha256.*$|:4.5|g' manifests/local-storage-operator-12k64npz/4.5/local-storage-operator.v4.5.0.clusterserviceversion.yaml

rm -f manifests/local-storage-operator-12k64npz/4.5/local-storage-operator.v4.5.0.clusterserviceversion.yamle

mv manifests/local-storage-operator-12k64npz /root/redhat-operators/build/manifests/

cd /root/redhat-operators/build

cat > custom-registry.Dockerfile << EOF
FROM registry.redhat.io/openshift4/ose-operator-registry:v4.5.0 AS builder

COPY manifests manifests

RUN /bin/initializer -o ./bundles.db;sleep 20 

EXPOSE 50051

ENTRYPOINT ["/bin/registry-server"]

CMD ["--database", "/registry/bundles.db"]
EOF

oc patch OperatorHub cluster --type json \
    -p '[{"op": "add", "path": "/spec/disableAllDefaultSources", "value": true}]'

podman build -f custom-registry.Dockerfile  -t helper.cluster-0001.rhsacn.org:5000/ocp4/custom-registry 

podman push helper.cluster-0001.rhsacn.org:5000/ocp4/custom-registry

cat > my-operator-catalog.yaml << EOF 
apiVersion: operators.coreos.com/v1alpha1
kind: CatalogSource
metadata:
  name: my-operator-catalog
  namespace: openshift-marketplace
spec:
  displayName: My Operator Catalog
  sourceType: grpc
  image: helper.cluster-0001.rhsacn.org:5000/ocp4/custom-registry:latest
EOF

oc create -f my-operator-catalog.yaml

export OCP_RELEASE="4.5.2"
export LOCAL_REGISTRY='helper.cluster-0001.rhsacn.org:5000'
export LOCAL_REPOSITORY='ocp4/openshift4'
export PRODUCT_REPO='openshift-release-dev'
export LOCAL_SECRET_JSON="${HOME}/pull-secret-2.json"
export RELEASE_NAME='ocp-release'
export ARCHITECTURE="x86_64"
export REMOVABLE_MEDIA_PATH='/opt/registry'
export OPERATOR_OCP_RELEASE="4.5"

oc adm catalog build \
  --appregistry-org redhat-operators \
  --from=registry.redhat.io/openshift4/ose-operator-registry:v${OPERATOR_OCP_RELEASE}  \
  --filter-by-os="linux/amd64" \
  -a ${LOCAL_SECRET_JSON} \
  --to=${LOCAL_REGISTRY}/olm/redhat-operators:v1 2>&1 | tee /tmp/catalog-build.log
...
time="2020-10-20T17:43:47+08:00" level=info msg=directory dir=/tmp/cache-201560791/manifests-044983308 file=web-terminal load=package
time="2020-10-20T17:43:47+08:00" level=info msg=directory dir=/tmp/cache-201560791/manifests-044983308 file=web-terminal-_656cyb8 load=package
time="2020-10-20T17:43:47+08:00" level=info msg=directory dir=/tmp/cache-201560791/manifests-044983308 file=1.0.1 load=package
Uploading ... 11.88MB/s
Uploading 9.6MB ...
Uploading 76.39MB ...
Uploading 1.81kB ...
Uploading 3.516MB ...
Uploading 96.33MB ...
Pushed sha256:178e24cabdb7b8c84d8f8326f0f3412822b2d8510ed5a3a51fdea494213ba1fb to helper.cluster-0001.rhsacn.org:5000/olm/redhat-operators:v1

# 目录 /tmp/cache-201560791/manifests-044983308 包含 operator 的 metadata
cd /tmp/cache-201560791/manifests-044983308
ls 
3scale-operator              aws-ebs-csi-driver-operator        eap                         metering-ocp                     quay-operator
advanced-cluster-management  businessautomation-operator        elasticsearch-operator      mtc-operator                     red-hat-camel-k
amq-broker                   cincinnati-operator                fuse-apicurito              nfd                              rh-service-binding-operator
amq-broker-lts               cluster-kube-descheduler-operator  fuse-console                ocs-operator                     rhsso-operator
amq-broker-rhel8             cluster-logging                    fuse-online                 openshift-pipelines-operator-rh  serverless-operator
amq-online                   clusterresourceoverride            jaeger-product              openshiftansibleservicebroker    service-registry-operator
amq-streams                  codeready-workspaces               kiali-ossm                  openshifttemplateservicebroker   servicemeshoperator
amq7-cert-manager            container-security-operator        kubevirt-hyperconverged     performance-addon-operator       sriov-network-operator
amq7-interconnect-operator   datagrid                           local-storage-operator      ptp-operator                     vertical-pod-autoscaler
apicast-operator             dv-operator                        manila-csi-driver-operator  quay-bridge-operator             web-terminal

# 进入 local-storage-operator 目录
cd local-storage-operator/local-storage-operator-0dvlnkig/4.5

pwd
/tmp/cache-201560791/manifests-044983308/local-storage-operator/local-storage-operator-0dvlnkig/4.5

grep -o 'image:.*' local-storage-operator.v4.5.0.clusterserviceversion.yaml 

image: registry.redhat.io/openshift4/ose-local-storage-diskmaker@sha256:3f8595fee46c37ce68eeeb36a7d925659b3424252cb862dbe79fa8e4cc71903a
image: registry.redhat.io/openshift4/ose-local-storage-operator@sha256:e40685aef7071d3cfd2c42c5b17d9c4309e11b4ad77c2213dc6a0903592789dd
image: registry.redhat.io/openshift4/ose-local-storage-static-provisioner@sha256:3496d6fe089a2a7b3a1cb1fdfb144b91de5a387635c8d6ac3ef1a40c0e7efb3f
image: registry.redhat.io/openshift4/ose-local-storage-operator@sha256:e40685aef7071d3cfd2c42c5b17d9c4309e11b4ad77c2213dc6a0903592789dd

# 生成 /tmp/registry-images.lst 文件
grep -o 'image:.*' local-storage-operator.v4.5.0.clusterserviceversion.yaml | awk '{print $2}'  > /tmp/registry-images.lst

# 进入 ocs-operator 4.5.0 目录
cd /tmp/cache-201560791/manifests-044983308/ocs-operator/ocs-operator-70m9f1nh/4.5.0

# 添加镜像到 /tmp/registry-images.lst 文件中
grep -o 'image:.*' *.clusterserviceversion.yaml | awk '{print $2}'  >> /tmp/registry-images.lst

# 生成 /tmp/mapping.txt 文件
cat /dev/null > /tmp/mapping.txt

  for source in `cat /tmp/registry-images.lst`; do  local=`echo $source|awk -F'@' '{print $1}'|sed 's|registry.redhat.io|helper.cluster-0001.rhsacn.org:5000|g'`   ; echo "$source=$local" >> /tmp/mapping.txt; done

# 生成 /tmp/image-policy.txt 文件
cat /dev/null > /tmp/image-policy.txt

  for source in `cat /tmp/registry-images.lst`; do  local=`echo $source|awk -F'@' '{print $1}'|sed 's/registry.redhat.io/helper.cluster-0001.rhsacn.org:5000/g'` ; mirror=`echo $source|awk -F'@' '{print $1}'`; echo "  - mirrors:" >> /tmp/image-policy.txt; echo "    - $local" >> /tmp/image-policy.txt; echo "    source: $mirror" >> /tmp/image-policy.txt; done

# 使用 skopeo copy --all 拷贝镜像
for source in `cat /tmp/registry-images.lst`; do  local=`echo $source|awk -F'@' '{print $1}'|sed 's/registry.redhat.io/helper.cluster-0001.rhsacn.org:5000/g'`   ; 
echo skopeo copy --format v2s2 --authfile ${LOCAL_SECRET_JSON} --all docker://$source docker://$local; skopeo copy --format v2s2 --authfile ${LOCAL_SECRET_JSON} --all docker://$source docker://$local; echo; done

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


skopeo inspect --authfile /root/pull-secret-2.json docker://registry.redhat.io/openshift4/ose-local-storage-operator@sha256:e40685aef7071d3cfd2c42c5b17d9c4309e11b4ad77c2213dc6a0903592789dd


```

### 如何暂时设置 Operator 对 CR 的管理
https://docs.openshift.com/container-platform/4.5/architecture/architecture-installation.html#unmanaged-operators_architecture-installation

### 处理 Failing to create pod sandbox on OpenShift 3 and 4 的问题
https://access.redhat.com/solutions/4321791

报错：'  Failed create pod sandbox: rpc error code: = Unknown desc = [failed to set up sandbox container.'

1. 首先尝试删除有问题的 pod，问题仍然存在
2. 然后尝试重启机器，看问题是否仍然在

### 替换不健康的 etcd member
https://docs.openshift.com/container-platform/4.4/backup_and_restore/replacing-unhealthy-etcd-member.html

```
检查 etcd 成员健康状态
oc -n openshift-etcd get etcd -o=jsonpath='{range .items[0].status.conditions[?(@.type=="EtcdMembersAvailable")]}{.message}{"\n"}'a
2 of 3 members are available, master1.cluster-0001.rhsacn.org is unhealthy

检查节点是否 stopped，如果节点是 stopped，则执行替换过程
oc get machines -A -ojsonpath='{range .items[*]}{@.status.nodeRef.name}{"\t"}{@.status.providerStatus.instanceState}{"\n"}' | grep -v running

如果节点是 running，则接着检查 node 是否 ready
oc get nodes -o jsonpath='{range .items[*]}{"\n"}{.metadata.name}{"\t"}{range .spec.taints[*]}{.key}{" "}' | grep unreachable

```

```
nodes=$(oc get node --no-headers -o custom-columns=NAME:.metadata.name)

for node in $nodes; do
  echo "Node: $node"
  oc describe node "$node" | awk '/machineconfiguration.openshift.io/'
  echo
done
```

### 如何关闭 OpenShift 4 虚拟机，如何恢复 OpenShift 4 虚拟机
出于安全考虑，bootstrap 证书有效期为 24 小时，之后每 30 天自动更换一次证书。因此在刚部署的 25 小时内不要关闭集群，在此后可关闭集群虚拟机，并且请在 30 天内启动虚拟机。

参考以下链接：
https://www.openshift.com/blog/enabling-openshift-4-clusters-to-stop-and-resume-cluster-vms

### 理解 OpenShift Machine Config Operator
https://www.redhat.com/en/blog/openshift-container-platform-4-how-does-machine-config-pool-work

```
# 查看 rendered-worker 文件数量

oc -n openshift-machine-api get machineconfig | grep render | grep worker | awk '{print $1}' | while read i ; do echo $i ; oc -n openshift-machine-api get machineconfig $i -o json | jq .spec.config.storage.files[].path | wc -l ; done | more  

# 查看 rendered-worker CreateTime

oc -n openshift-machine-api get machineconfig | grep render | grep worker | awk '{print $1}' | while read i ; do echo $i ; oc -n openshift-machine-api get machineconfig $i -o json | jq .metadata.creationTimestamp  ; done

rendered-worker-79175908d2f255d95306f589824c79a6
"2020-10-22T12:14:44Z"
rendered-worker-c75a95deaa58ece09cc7322019f9a0f0
"2020-10-22T11:42:12Z"
rendered-worker-d97677145fe5c3ccd616b684c725e90c
"2020-10-22T11:42:10Z"
rendered-worker-f6e7aa257f819b7f364bf9d1b7c87d20
"2020-10-22T12:30:40Z"

# 最新时间是 2020-10-22T12:30:40Z，看看哪个 machineconfig 的时间与此时间接近

oc -n openshift-machine-api get machineconfig --no-headers | grep -Ev "render" | awk '{print $1}' | while read i ; do echo $i ; oc -n openshift-machine-api get machineconfig $i -o json | jq .metadata.creationTimestamp  ; done

00-master
"2020-10-22T11:42:08Z"
00-worker
"2020-10-22T11:42:09Z"
01-master-container-runtime
"2020-10-22T11:42:09Z"
01-master-kubelet
"2020-10-22T11:42:09Z"
01-worker-container-runtime
"2020-10-22T11:42:09Z"
01-worker-kubelet
"2020-10-22T11:42:09Z"
99-master-f1635907-11d6-42f8-b36f-c9aeff7455bc-registries
"2020-10-22T11:42:09Z"
99-master-ssh
"2020-10-22T11:37:55Z"
99-worker-ea37cbd1-675c-497b-9758-dc64224b8a27-registries
"2020-10-22T11:42:10Z"
99-worker-ssh
"2020-10-22T11:37:55Z"
masters-chrony-configuration
"2020-10-22T12:14:35Z"
workers-chrony-configuration
"2020-10-22T12:14:39Z"

# 没有 2020-10-22T12:30:40Z 的 machineconfig，那为什么生成了新的 rendered 呢?

oc -n openshift-machine-api get machineconfig rendered-worker-79175908d2f255d95306f589824c79a6 -o yaml | tee /tmp/rendered-worker-79175908d2f255d95306f589824c79a6

oc -n openshift-machine-api get machineconfig rendered-worker-f6e7aa257f819b7f364bf9d1b7c87d20 -o yaml | tee /tmp/rendered-worker-f6e7aa257f819b7f364bf9d1b7c87d20

diff -urN /tmp/rendered-worker-79175908d2f255d95306f589824c79a6 /tmp/rendered-worker-f6e7aa257f819b7f364bf9d1b7c87d20
...
@@ -160,7 +160,7 @@
         mode: 420
         path: /etc/kubernetes/kubelet.conf
       - contents:
-          source: data:text/plain,unqualified-search-registries%20%3D%20%5B%22registry.access.redhat.com%22%2C%20%22docker.io%22%5D%0A%0A%5B%5Bregistry%
5D%5D%0A%20%20prefix%20%3D%20%22%22%0A%20%20location%20%3D%20%22quay.io%2Fopenshift-release-dev%2Focp-release%22%0A%20%20mirror-by-digest-only%20%3D%20tr
ue%0A%0A%20%20%5B%5Bregistry.mirror%5D%5D%0A%20%20%20%20location%20%3D%20%22helper.cluster-0001.rhsacn.org%3A5000%2Focp4%2Fopenshift4%22%0A%0A%5B%5Bregis
try%5D%5D%0A%20%20prefix%20%3D%20%22%22%0A%20%20location%20%3D%20%22quay.io%2Fopenshift-release-dev%2Focp-v4.0-art-dev%22%0A%20%20mirror-by-digest-only%2
0%3D%20true%0A%0A%20%20%5B%5Bregistry.mirror%5D%5D%0A%20%20%20%20location%20%3D%20%22helper.cluster-0001.rhsacn.org%3A5000%2Focp4%2Fopenshift4%22%0A
+          source: data:text/plain,unqualified-search-registries%20%3D%20%5B%22registry.access.redhat.com%22%2C%20%22docker.io%22%5D%0A%0A%5B%5Bregistry%
5D%5D%0A%20%20prefix%20%3D%20%22%22%0A%20%20location%20%3D%20%22quay.io%2Fopenshift-release-dev%2Focp-release%22%0A%20%20mirror-by-digest-only%20%3D%20tr
ue%0A%0A%20%20%5B%5Bregistry.mirror%5D%5D%0A%20%20%20%20location%20%3D%20%22helper.cluster-0001.rhsacn.org%3A5000%2Focp4%2Fopenshift4%22%0A%0A%5B%5Bregis
try%5D%5D%0A%20%20prefix%20%3D%20%22%22%0A%20%20location%20%3D%20%22quay.io%2Fopenshift-release-dev%2Focp-v4.0-art-dev%22%0A%20%20mirror-by-digest-only%2
0%3D%20true%0A%0A%20%20%5B%5Bregistry.mirror%5D%5D%0A%20%20%20%20location%20%3D%20%22helper.cluster-0001.rhsacn.org%3A5000%2Focp4%2Fopenshift4%22%0A%0A%5
B%5Bregistry%5D%5D%0A%20%20prefix%20%3D%20%22%22%0A%20%20location%20%3D%20%22registry.redhat.io%2Focs4%2Fcephcsi-rhel8%22%0A%20%20mirror-by-digest-only%2
0%3D%20true%0A%0A%20%20%5B%5Bregistry.mirror%5D%5D%0A%20%20%20%20location%20%3D%20%22helper.cluster-0001.rhsacn.org%3A5000%2F...

# 可以看到 /etc/kubernetes/kubelet.conf 这部分内容变化了
# 增加了新的容器镜像仓库 mirror 内容
# 这部分内容是在执行 oc apply -f oc apply -f /tmp/ImageContentSourcePolicy.yaml 之后产生的
# 谜团解开了
```

### 检查 master worker 时间同步情况 
```
for i in `seq 140 145` ; do echo 10.66.208.${i} ; ssh core@10.66.208.${i} date -u ; date -u ; echo ; done 

10.66.208.140
Warning: Permanently added '10.66.208.140' (ECDSA) to the list of known hosts.
Fri Oct 23 03:16:12 UTC 2020
Fri Oct 23 03:16:17 UTC 2020

10.66.208.141
Warning: Permanently added '10.66.208.141' (ECDSA) to the list of known hosts.
Fri Oct 23 03:16:13 UTC 2020
Fri Oct 23 03:16:19 UTC 2020

10.66.208.142
Warning: Permanently added '10.66.208.142' (ECDSA) to the list of known hosts.
Fri Oct 23 03:16:14 UTC 2020
Fri Oct 23 03:16:20 UTC 2020

10.66.208.143
Warning: Permanently added '10.66.208.143' (ECDSA) to the list of known hosts.
Fri Oct 23 03:16:15 UTC 2020
Fri Oct 23 03:16:21 UTC 2020

10.66.208.144
Warning: Permanently added '10.66.208.144' (ECDSA) to the list of known hosts.
Fri Oct 23 03:16:13 UTC 2020
Fri Oct 23 03:16:21 UTC 2020

10.66.208.145
Warning: Permanently added '10.66.208.145' (ECDSA) to the list of known hosts.
Fri Oct 23 03:16:17 UTC 2020
Fri Oct 23 03:16:22 UTC 2020
```

### 检查 ocs toolbox 无法查看 ceph status 的问题
```
TOOLS_POD=$(oc get pods -n openshift-storage -l app=rook-ceph-tools -o name)
oc rsh -n openshift-storage $TOOLS_POD

# 查看 ceph status 报错
sh-4.4# ceph status
[errno 1] error connecting to the cluster

# 查看 toolbox 的 mon_host
sh-4.4# cat /etc/ceph/ceph.conf | grep mon_host
mon_host = 172.30.171.41:6789,172.30.241.175:6789,172.30.232.250:6789

# 从 svc 的输出看 mon

oc get svc -n openshift-storage --no-headers | grep mon

rook-ceph-mon-a                                    ClusterIP      172.30.241.175   <none>      6789/TCP,3300/TCP                                          13h
rook-ceph-mon-c                                    ClusterIP      172.30.232.250   <none>      6789/TCP,3300/TCP                                          13h
rook-ceph-mon-d                                    ClusterIP      172.30.171.41    <none>      6789/TCP,3300/TCP                                          85m


```

### 在升级或者更新 machineconfig 时删除 ocs 的 pdb
https://bugzilla.redhat.com/show_bug.cgi?id=1861104
```
删除 ocs pdb
oc get pdb -n openshift-storage --no-headers | awk '{print $1}'  | while read i ; do echo oc delete pdb $i -n openshift-storage ; oc delete pdb $i -n openshift-storage ; echo ; done  
```

### RHEL7 如何配置 crash dump 和 分析 crash dump
https://www.thegeekdiary.com/centos-rhel-7-how-to-configure-kdump/<br>
https://sites.google.com/site/syscookbook/rhel/rhel-kdump-rhel7<br>

```
1. 安装如下rpm包：
    kernel-devel - 和当前的内核相同版本
    kernel-debuginfo - 和当前的内核相同版本
    kernel-debuginfo-common - 和当前的内核相同版本
yum localinstall -y kernel-devel-3.10.0-862.el7.x86_64.rpm kernel-debuginfo-3.10.0-862.el7.x86_64.rpm kernel-debuginfo-common-x86_64-3.10.0-862.el7.x86_64.rpm

2. 安装 crash 
yum install -y crash

3. 执行 crash 分析
crash /usr/lib/debug/lib/modules/3.10.0-862.el7.x86_64/vmlinux \
   /var/crash/127.0.0.1-2020-10-15-14\:01\:14/vmcore

4. 参考 https://sites.google.com/site/syscookbook/rhel/rhel-kdump-rhel7 继续分析

看到的报错为：
[ 5457.639640] NMI watchdog: Watchdog detected hard LOCKUP on cpu 9
[ 5457.639691] Modules linked in:
[ 5457.639695]  vhost_net vhost macvtap macvlan tun rpcsec_gss_krb5 nfsv4 dns_resolver nfsv3 nfs fscache ebtable_filter ebtables ip6table_filter ip6_tables iptable_filter devlink bnx2fc cnic uio fcoe libfcoe libfc scsi_transport_fc scsi_tgt intel_powerclamp coretemp intel_rapl iosf_mbi kvm_intel kvm iTCO_wdt irqbypass iTCO_vendor_support dm_service_time mei_me sg pcspkr mei shpchp i2c_i801 lpc_ich ipmi_si ipmi_devintf ipmi_msghandler acpi_pad dm_multipath nfsd bridge auth_rpcgss nfs_acl lockd bonding grace ip_tables ext4 mbcache jbd2 dm_thin_pool dm_persistent_data dm_bio_prison dm_bufio libcrc32c sd_mod crc_t10dif crct10dif_generic 8021q garp mrp stp llc ast drm_kms_helper syscopyarea sysfillrect ahci sysimgblt fb_sys_fops ttm libahci igb crct10dif_pclmul crct10dif_common crc32_pclmul drm libata crc32c_intel
[ 5457.639758]  ghash_clmulni_intel mxm_wmi dca ptp aesni_intel nvme pps_core i2c_algo_bit lrw gf128mul glue_helper ablk_helper cryptd i2c_core nvme_core scsi_transport_iscsi wmi sunrpc dm_mirror dm_region_hash dm_log dm_mod
[ 5457.639777] CPU: 9 PID: 5013 Comm: CPU 0/KVM Kdump: loaded Not tainted 3.10.0-862.el7.x86_64 #1
[ 5457.639779] Hardware name: To Be Filled By O.E.M. To Be Filled By O.E.M./D1541D4I, BIOS P1.20 09/19/2016
[ 5457.639781] Call Trace:
[ 5457.639784]  <NMI>  [<ffffffff9d30d768>] dump_stack+0x19/0x1b
[ 5457.639797]  [<ffffffff9cd3fa55>] watchdog_overflow_callback+0x135/0x140
[ 5457.639802]  [<ffffffff9cd7f517>] __perf_event_overflow+0x57/0x100
[ 5457.639806]  [<ffffffff9cd87f04>] perf_event_overflow+0x14/0x20
[ 5457.639810]  [<ffffffff9cc0a580>] intel_pmu_handle_irq+0x220/0x510
[ 5457.639815]  [<ffffffff9cf4c774>] ? ioremap_page_range+0x2b4/0x450
[ 5457.639819]  [<ffffffff9cdd6944>] ? vunmap_page_range+0x234/0x470
[ 5457.639824]  [<ffffffff9d00a4d6>] ? ghes_copy_tofrom_phys+0x116/0x210
[ 5457.639828]  [<ffffffff9d00a670>] ? ghes_read_estatus+0xa0/0x190
[ 5457.639833]  [<ffffffff9d316031>] perf_event_nmi_handler+0x31/0x50
[ 5457.639837]  [<ffffffff9d31790c>] nmi_handle.isra.0+0x8c/0x150
[ 5457.639840]  [<ffffffff9d317be8>] do_nmi+0x218/0x460
[ 5457.639844]  [<ffffffff9d316d79>] end_repeat_nmi+0x1e/0x7e
[ 5457.639849]  [<ffffffff9cd088ae>] ? native_queued_spin_lock_slowpath+0x1ce/0x200
[ 5457.639853]  [<ffffffff9cd088ae>] ? native_queued_spin_lock_slowpath+0x1ce/0x200
[ 5457.639856]  [<ffffffff9cd088ae>] ? native_queued_spin_lock_slowpath+0x1ce/0x200
[ 5457.639857]  <EOE>  [<ffffffff9d30842a>] queued_spin_lock_slowpath+0xb/0xf
[ 5457.639863]  [<ffffffff9d315707>] _raw_spin_lock_irqsave+0x37/0x40
[ 5457.639868]  [<ffffffff9ccceaa1>] try_to_wake_up+0x31/0x350
[ 5457.639874]  [<ffffffffc12eb39c>] ? vmcs_set_bits+0x1c/0x20 [kvm_intel]
[ 5457.639879]  [<ffffffffc12ebb10>] ? vmx_set_hv_timer+0xb0/0xc0 [kvm_intel]
[ 5457.639883]  [<ffffffff9cccee92>] default_wake_function+0x12/0x20
[ 5457.639887]  [<ffffffff9ce308a3>] pollwake+0x73/0x90
[ 5457.639891]  [<ffffffff9cccee80>] ? wake_up_state+0x20/0x20
[ 5457.639894]  [<ffffffff9ccc4abb>] __wake_up_common+0x5b/0x90
[ 5457.639897]  [<ffffffff9ccc4b28>] __wake_up_locked_key+0x18/0x20
[ 5457.639902]  [<ffffffff9ce6ad29>] eventfd_signal+0x59/0x70
[ 5457.639918]  [<ffffffffc09814fe>] ioeventfd_write+0x7e/0xb0 [kvm]
[ 5457.639928]  [<ffffffffc097af77>] __kvm_io_bus_write+0x87/0xc0 [kvm]
[ 5457.639937]  [<ffffffffc097affd>] kvm_io_bus_write+0x4d/0x70 [kvm]
[ 5457.639943]  [<ffffffffc12f036f>] handle_ept_misconfig+0x2f/0x130 [kvm_intel]
[ 5457.639948]  [<ffffffffc12f72b4>] vmx_handle_exit+0x294/0xc90 [kvm_intel]
[ 5457.639953]  [<ffffffffc12f58bb>] ? vmx_vcpu_run+0x32b/0x8f0 [kvm_intel]
[ 5457.639958]  [<ffffffffc12f58c7>] ? vmx_vcpu_run+0x337/0x8f0 [kvm_intel]
[ 5457.639962]  [<ffffffffc12f58bb>] ? vmx_vcpu_run+0x32b/0x8f0 [kvm_intel]
[ 5457.639980]  [<ffffffffc09b79bf>] ? wait_lapic_expire+0xaf/0x190 [kvm]
[ 5457.639993]  [<ffffffffc099171d>] vcpu_enter_guest+0x64d/0x12c0 [kvm]
[ 5457.640007]  [<ffffffffc09b59d3>] ? apic_has_interrupt_for_ppr+0x83/0xb0 [kvm]
[ 5457.640020]  [<ffffffffc09b8355>] ? kvm_apic_has_interrupt+0x45/0x90 [kvm]
[ 5457.640033]  [<ffffffffc0998e58>] kvm_arch_vcpu_ioctl_run+0x358/0x480 [kvm]
[ 5457.640043]  [<ffffffffc097e441>] kvm_vcpu_ioctl+0x2b1/0x650 [kvm]
[ 5457.640047]  [<ffffffff9ce2fb90>] do_vfs_ioctl+0x350/0x560
[ 5457.640050]  [<ffffffff9d31291c>] ? __schedule+0x41c/0xa20
[ 5457.640053]  [<ffffffff9ce2fe41>] SyS_ioctl+0xa1/0xc0
[ 5457.640057]  [<ffffffff9d31f7d5>] system_call_fastpath+0x1c/0x21
[ 5457.640060] Kernel panic - not syncing: Hard LOCKUP
[ 5457.640095] CPU: 9 PID: 5013 Comm: CPU 0/KVM Kdump: loaded Not tainted 3.10.0-862.el7.x86_64 #1
[ 5457.640153] Hardware name: To Be Filled By O.E.M. To Be Filled By O.E.M./D1541D4I, BIOS P1.20 09/19/2016
[ 5457.640215] Call Trace:
[ 5457.640233]  <NMI>  [<ffffffff9d30d768>] dump_stack+0x19/0x1b
[ 5457.640277]  [<ffffffff9d307a6a>] panic+0xe8/0x21f
[ 5457.640312]  [<ffffffff9cc9142f>] nmi_panic+0x3f/0x40
[ 5457.640349]  [<ffffffff9cd3fa41>] watchdog_overflow_callback+0x121/0x140
[ 5457.640396]  [<ffffffff9cd7f517>] __perf_event_overflow+0x57/0x100
[ 5457.640439]  [<ffffffff9cd87f04>] perf_event_overflow+0x14/0x20
[ 5457.640480]  [<ffffffff9cc0a580>] intel_pmu_handle_irq+0x220/0x510
[ 5457.640523]  [<ffffffff9cf4c774>] ? ioremap_page_range+0x2b4/0x450
[ 5457.640566]  [<ffffffff9cdd6944>] ? vunmap_page_range+0x234/0x470
[ 5457.640609]  [<ffffffff9d00a4d6>] ? ghes_copy_tofrom_phys+0x116/0x210
[ 5457.640654]  [<ffffffff9d00a670>] ? ghes_read_estatus+0xa0/0x190
[ 5457.640696]  [<ffffffff9d316031>] perf_event_nmi_handler+0x31/0x50
[ 5457.640740]  [<ffffffff9d31790c>] nmi_handle.isra.0+0x8c/0x150
[ 5457.640781]  [<ffffffff9d317be8>] do_nmi+0x218/0x460
[ 5457.640816]  [<ffffffff9d316d79>] end_repeat_nmi+0x1e/0x7e
[ 5457.640856]  [<ffffffff9cd088ae>] ? native_queued_spin_lock_slowpath+0x1ce/0x200
[ 5457.640906]  [<ffffffff9cd088ae>] ? native_queued_spin_lock_slowpath+0x1ce/0x200
[ 5457.640957]  [<ffffffff9cd088ae>] ? native_queued_spin_lock_slowpath+0x1ce/0x200
[ 5457.641006]  <EOE>  [<ffffffff9d30842a>] queued_spin_lock_slowpath+0xb/0xf
[ 5457.641057]  [<ffffffff9d315707>] _raw_spin_lock_irqsave+0x37/0x40
[ 5457.641101]  [<ffffffff9ccceaa1>] try_to_wake_up+0x31/0x350
[ 5457.641143]  [<ffffffffc12eb39c>] ? vmcs_set_bits+0x1c/0x20 [kvm_intel]
[ 5457.641190]  [<ffffffffc12ebb10>] ? vmx_set_hv_timer+0xb0/0xc0 [kvm_intel]
[ 5457.641238]  [<ffffffff9cccee92>] default_wake_function+0x12/0x20
[ 5457.641280]  [<ffffffff9ce308a3>] pollwake+0x73/0x90
[ 5457.641316]  [<ffffffff9cccee80>] ? wake_up_state+0x20/0x20
[ 5457.641355]  [<ffffffff9ccc4abb>] __wake_up_common+0x5b/0x90
[ 5457.641394]  [<ffffffff9ccc4b28>] __wake_up_locked_key+0x18/0x20
[ 5457.641436]  [<ffffffff9ce6ad29>] eventfd_signal+0x59/0x70
[ 5457.641482]  [<ffffffffc09814fe>] ioeventfd_write+0x7e/0xb0 [kvm]
[ 5457.641530]  [<ffffffffc097af77>] __kvm_io_bus_write+0x87/0xc0 [kvm]
[ 5457.641580]  [<ffffffffc097affd>] kvm_io_bus_write+0x4d/0x70 [kvm]
[ 5457.641625]  [<ffffffffc12f036f>] handle_ept_misconfig+0x2f/0x130 [kvm_intel]
[ 5457.641676]  [<ffffffffc12f72b4>] vmx_handle_exit+0x294/0xc90 [kvm_intel]
[ 5457.641724]  [<ffffffffc12f58bb>] ? vmx_vcpu_run+0x32b/0x8f0 [kvm_intel]
[ 5457.641772]  [<ffffffffc12f58c7>] ? vmx_vcpu_run+0x337/0x8f0 [kvm_intel]
[ 5457.641819]  [<ffffffffc12f58bb>] ? vmx_vcpu_run+0x32b/0x8f0 [kvm_intel]
[ 5457.641876]  [<ffffffffc09b79bf>] ? wait_lapic_expire+0xaf/0x190 [kvm]
[ 5457.641930]  [<ffffffffc099171d>] vcpu_enter_guest+0x64d/0x12c0 [kvm]
[ 5457.641985]  [<ffffffffc09b59d3>] ? apic_has_interrupt_for_ppr+0x83/0xb0 [kvm]
[ 5457.642044]  [<ffffffffc09b8355>] ? kvm_apic_has_interrupt+0x45/0x90 [kvm]
[ 5457.642101]  [<ffffffffc0998e58>] kvm_arch_vcpu_ioctl_run+0x358/0x480 [kvm]
[ 5457.642155]  [<ffffffffc097e441>] kvm_vcpu_ioctl+0x2b1/0x650 [kvm]
[ 5457.642199]  [<ffffffff9ce2fb90>] do_vfs_ioctl+0x350/0x560
[ 5457.642238]  [<ffffffff9d31291c>] ? __schedule+0x41c/0xa20
[ 5457.642277]  [<ffffffff9ce2fe41>] SyS_ioctl+0xa1/0xc0
[ 5457.642313]  [<ffffffff9d31f7d5>] system_call_fastpath+0x1c/0x21

参考的 article 为 https://access.redhat.com/solutions/3469991
```

### 检查 RHCoreOS 系统日志
```
# 生成文件
sudo journalctl > /tmp/system.log

# 过滤信息
sudo cat /tmp/system.log  | grep -Ev "kernel: | systemd| multipathd| ignition| coreos| dracut| iscsid| ostree| restorecon| augenrules| NetworkManager| rhcos| machine-config-daemon| I1026 | auditd| chronyd| dbus| network-manager| rpc.statd| sssd| polkitd| rpm-ostree| dbus-daemon" | mor

# 删除临时文件
sudo rm -f /tmp/system.log

# 生成 kubelet 日志文件
sudo journalctl -u kubelet > /tmp/kubelet.log

# 检查最后的文件内容获取最新的 kubelet 进程 ID

```

### 重新生成节点 certificate
https://access.redhat.com/solutions/4923031

```
# 在跳板机生成脚本 recover_kubeconfig.sh
cat << 'EOF' > recover_kubeconfig.sh 
#!/bin/bash

set -eou pipefail

# context
intapi=$(oc get infrastructures.config.openshift.io cluster -o "jsonpath={.status.apiServerInternalURI}")
context="$(oc config current-context)"
# cluster
cluster="$(oc config view -o "jsonpath={.contexts[?(@.name==\"$context\")].context.cluster}")"
server="$(oc config view -o "jsonpath={.clusters[?(@.name==\"$cluster\")].cluster.server}")"
# token
ca_crt_data="$(oc get secret -n openshift-machine-config-operator node-bootstrapper-token -o "jsonpath={.data.ca\.crt}" | base64 --decode)"
namespace="$(oc get secret -n openshift-machine-config-operator node-bootstrapper-token  -o "jsonpath={.data.namespace}" | base64 --decode)"
token="$(oc get secret -n openshift-machine-config-operator node-bootstrapper-token -o "jsonpath={.data.token}" | base64 --decode)"

export KUBECONFIG="$(mktemp)"
oc config set-credentials "kubelet" --token="$token" >/dev/null
ca_crt="$(mktemp)"; echo "$ca_crt_data" > $ca_crt
oc config set-cluster $cluster --server="$intapi" --certificate-authority="$ca_crt" --embed-certs >/dev/null
oc config set-context kubelet --cluster="$cluster" --user="kubelet" >/dev/null
oc config use-context kubelet >/dev/null
cat "$KUBECONFIG"
EOF

# 执行脚本
sh -x recover_kubeconfig.sh > kubeconfig-bootstrap

# 拷贝 kubeconfig-bootstrap 到 node
scp kubeconfig-bootstrap core@<nodeip>:/tmp

# ssh 到 node 上备份 old node certificate
systemctl stop kubelet
mkdir -p /root/backup-certs
cp -a /var/lib/kubelet/pki /var/lib/kubelet/kubeconfig /root/backup-certs
rm -rf /var/lib/kubelet/pki /var/lib/kubelet/kubeconfig

# 拷贝生成的 kubeconfig-bootstrap 到节点的 /etc/kubernetes/kubeconfig
cp /tmp/kubeconfig-bootstrap /etc/kubernetes/kubeconfig

# 启动 kubelet
systemctl start kubelet

# 在跳板机检查 certificate request 并且批准
oc get csr
/usr/local/bin/oc get csr --no-headers | /usr/bin/awk '{print $1}' | xargs /usr/local/bin/oc adm certificate approve
```

### 删除 ocs cluster 的步骤
参考：https://access.redhat.com/documentation/en-us/red_hat_openshift_container_storage/4.5/html-single/deploying_openshift_container_storage_on_vmware_vsphere/index#assembly_uninstalling-openshift-container-storage_rhocs

```
1. 获取使用 ocs 的 pvc 和 obc 

# 获取 pvc , storageclass == ocs-storagecluster-ceph-rbd，并且 app 不是 noobaa
oc get pvc -o=jsonpath='{range .items[?(@.spec.storageClassName=="ocs-storagecluster-ceph-rbd")]}{"Name: "}{@.metadata.name}{" Namespace: "}{@.metadata.namespace}{" Labels: "}{@.metadata.labels}{"\n"}{end}' --all-namespaces|awk '! ( /Namespace: openshift-storage/ && /app:noobaa/ )' | grep -v noobaa-default-backing-store-noobaa-pvc

# 获取 pvc , storageclass == ocs-storagecluster-cephfs
oc get pvc -o=jsonpath='{range .items[?(@.spec.storageClassName=="ocs-storagecluster-cephfs")]}{"Name: "}{@.metadata.name}{" Namespace: "}{@.metadata.namespace}{"\n"}{end}' --all-namespaces

# 获取 obc , storageclass == ocs-storagecluster-ceph-rgw
oc get obc -o=jsonpath='{range .items[?(@.spec.storageClassName=="ocs-storagecluster-ceph-rgw")]}{"Name: "}{@.metadata.name}{" Namespace: "}{@.metadata.namespace}{"\n"}{end}' --all-namespaces

# 获取 obc , storageclass == openshift-storage.noobaa.io
oc get obc -o=jsonpath='{range .items[?(@.spec.storageClassName=="openshift-storage.noobaa.io")]}{"Name: "}{@.metadata.name}{" Namespace: "}{@.metadata.namespace}{"\n"}{end}' --all-namespaces

2. 删除找到的 pvc 和 obc 
2.1 如果监控，registry 和 日志使用了OCS，参考相关链接里的步骤进行删除工作
2.2 删除其他 pvc 和 obc

3. 删除自定义 bucketclass 
# 列出自定义 bucketclass
oc get bucketclass -A  | grep -v noobaa-default-bucket-class
# 删除自定义 bucketclass
oc delete bucketclass <bucketclass name> -n <project-name>

4. 删除自定义 backingstore
# 列出自定义 backingstore
for bs in $(oc get backingstore -o name -n openshift-storage | grep -v noobaa-default-backing-store); do echo "Found backingstore $bs"; echo "Its has the following pods running :"; echo "$(oc get pods -o name -n openshift-storage | grep $(echo ${bs} | cut -f2 -d/))"; done

# 删除自定义 backingstore，删除 backingstore 前，删除对应 pod 和 pvc
for bs in $(oc get backingstore -o name -n openshift-storage | grep -v noobaa-default-backing-store); do echo "Deleting Backingstore $bs"; oc delete -n openshift-storage $bs; done

# 列出自定义 noobaa-pod
oc get pods -n openshift-storage | grep noobaa-pod | grep -v noobaa-default-backing-store-noobaa-pod

# 列出自定义 noobaa-pvc
oc get pvc -n openshift-storage --no-headers | grep -v noobaa-db | grep noobaa-pvc | grep -v noobaa-default-backing-store-noobaa-pvc

5. 列出 local volume 对象
for sc in $(oc get storageclass|grep 'kubernetes.io/no-provisioner' |grep -E $(oc get storagecluster -n openshift-storage -o jsonpath='{ .items[*].spec.storageDeviceSets[*].dataPVCTemplate.spec.storageClassName}' | sed 's/ /|/g')| awk '{ print $1 }');
do
    echo -n "StorageClass: $sc ";
    oc get storageclass $sc -o jsonpath=" { 'LocalVolume: ' }{ .metadata.labels['local\.storage\.openshift\.io/owner-name'] } { '\n' }";
done

6. 删除 storagecluster
oc delete -n openshift-storage storagecluster --all --wait=true

7. 删除 namespace 等待删除结束
oc project default
oc delete project openshift-storage --wait=true --timeout=5m
等待以下命令返回
oc get project  openshift-storage
Error from server (NotFound): namespaces "openshift-storage" not found

如果 project openshift-storage 有对象处于 Terminating 状态无法返回
可执行以下命令进行修复
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

8. 删除节点 storage operator artifacts (optional，彻底删除时执行)
for i in $(oc get node -l cluster.ocs.openshift.io/openshift-storage= -o jsonpath='{ .items[*].metadata.name }'); do oc debug node/${i} -- chroot /host rm -rfv /var/lib/rook; done

for i in $(oc get node -l cluster.ocs.openshift.io/openshift-storage= -o jsonpath='{ .items[*].metadata.name }'); do oc debug node/${i} -- chroot /host  ls -l /var/lib/rook; done

9. 删除 localvolume （optional，彻底删除时执行）
LV=local-block
SC=localblock

# 列出待清除的设备
oc get localvolume -n local-storage $LV -o jsonpath='{ .spec.storageClassDevices[*].devicePaths[*] }'

# 删除 localvolume
oc delete localvolume -n local-storage --wait=true $LV

# 删除与 LV 关联的 pv
oc delete pv -l storage.openshift.com/local-volume-owner-name=${LV} --wait --timeout=5m

# 删除 storageclass localblock
oc delete storageclass $SC --wait --timeout=5m

# 删除节点上与 localvolume 有关的 artifacts
[[ ! -z $SC ]] && for i in $(oc get node -l cluster.ocs.openshift.io/openshift-storage= -o jsonpath='{ .items[*].metadata.name }'); do oc debug node/${i} -- chroot /host rm -rfv /mnt/local-storage/${SC}/; done

9. 使用 sgdisk --zap-all 删除磁盘数据 （optional，彻底删除时执行）
# 获取节点列表
oc get nodes -l cluster.ocs.openshift.io/openshift-storage=
# 登录节点
oc debug node/node-xxx
# chroot
sh-4.2# chroot /host
# 设置 DISKS 变量
sh-4.2# DISKS="/dev/disk/by-id/nvme-xxxxxx /dev/disk/by-id/nvme-yyyyyy /dev/disk/by-id/nvme-zzzzzz"
# 执行 sgdisk --zap-all $disk 清除磁盘内容
sh-4.4# for disk in $DISKS; do sgdisk --zap-all $disk;done
# exit
# exit 
# 在其它节点上重复此步骤

10. 删除 openshift-storage.noobaa.io storageclass （optional，彻底删除时执行）
oc delete storageclass  openshift-storage.noobaa.io --wait=true --timeout=5m

11. 取消节点 lable 信息 （optional，彻底删除时执行）
oc label nodes  --all cluster.ocs.openshift.io/openshift-storage-
oc label nodes  --all topology.rook.io/rack-

12. 再次确认所有 ocs 相关 pv 已删除
oc get pv | egrep 'ocs-storagecluster-ceph-rbd|ocs-storagecluster-cephfs'

13. 删除 ocs 相关 crd（optional，彻底删除时执行）
oc delete crd backingstores.noobaa.io bucketclasses.noobaa.io cephblockpools.ceph.rook.io cephclusters.ceph.rook.io cephfilesystems.ceph.rook.io cephnfses.ceph.rook.io cephobjectstores.ceph.rook.io cephobjectstoreusers.ceph.rook.io noobaas.noobaa.io ocsinitializations.ocs.openshift.io  storageclusterinitializations.ocs.openshift.io storageclusters.ocs.openshift.io cephclients.ceph.rook.io --wait=true --timeout=5m
```

### 博客 Deploying OpenShift Container Storage using Local Devices
https://www.openshift.com/blog/deploying-openshift-container-storage-using-local-devices

### 找到 OpenShift4 namespace openshift-monitoring 下无法删除的资源的命令
```
oc api-resources --verbs=list --namespaced -o name | xargs -n 1 oc get --show-kind --ignore-not-found -n openshift-monitoring
```

关于 finalizer 值得详细阅读的博客<br>
这篇博客讲述了：<br>
 * 什么是 finalizer <br>
 * 为什么要有 finalizer <br>
 * 如果不使用 finalizer 可能会有什么问题 <br>
 * 如何找到 namespace 下等待删除的资源 <br>
 * 如果不通过 API，而是通过直接查询 etcd 该怎么做 <br>
https://www.openshift.com/blog/the-hidden-dangers-of-terminating-namespaces

ToDo: 需要仔细阅读一下这篇博客

### OpenShift Smaple Operator
https://github.com/openshift/cluster-samples-operator<br>
用来管理以 RHEL 镜像为基础的 ImageStream 和 Templates<br>

### OpenShift Container Platform 4.x Tested Integrations (for x86_x64)
https://access.redhat.com/articles/4763741<br>

### 重新安装 OCP 4 的 worker
Remove it from the cluster first (cordon -> drain -> oc delete node).  Then reinstall CoreOS however you like (ISO, PXE, whatever) using the same worker ignition

### 博客：在资源超分配情况下 OpenShift 如何处理 worker 节点资源 
How to Handle OpenShift Worker Nodes Resources in Overcommitted State<br>
https://www.openshift.com/blog/how-to-handle-openshift-worker-nodes-resources-in-overcommitted-state

### 生成新的 ignition 文件的步骤
https://docs.openshift.com/container-platform/4.1/installing/installing_bare_metal/installing-bare-metal.html

1. 创建新的空的目录
2. 拷贝备份好的 install-config.yaml 文件到此目录下
3. 执行命令
```
mkdir -p <new_empty_directory>
cp <backup/install-config.yaml> <new_empty_directory>
cd <new_empty_directory>
openshift-install create ignition-configs --dir=`pwd`
```

### 如何创建 serviceaccount 并且授予 anyuid 策略，然后将此 serviceaccount 与 dc/myapp 关联起来
```
# Create a ServiceAccount to let our container startup as a privileged pod
oc create serviceaccount mysa
# As `cluster-admin`, give the ServiceAccount the permission start as an anyuid pod
oc adm policy add-scc-to-user anyuid -z mysa
# Apply the ServiceAccount to the DeploymentConfig
oc patch dc/myapp --patch '{"spec":{"template":{"spec":{"serviceAccountName": "mysa"}}}}'
```

### openshift-install 也是支持 explain 命令
这个命令可以让我们了解 installconfig 在特定平台上的支持能力
```
$ openshift-install explain installconfig.platform.azure.resourceGroupName
KIND:     InstallConfig
VERSION:  v1

RESOURCE: <string>
  ResourceGroupName is the name of an already existing resource group where the cluster should be installed. This resource group should only be used for this specific cluster and the cluster components will assume assume ownership of all resources in the resource group. Destroying the cluster using installer will delete this resource group. This resource group must be empty with no other resources when trying to use it for creating a cluster. If empty, a new resource group will created for the cluster.
```

### OCP 4.6 RHV UPI document
https://github.com/openshift/installer/blob/master/docs/user/ovirt/install_upi.md

### 拷贝镜像的脚本，因为采用了 skopeo copy --all，因此拷贝前和拷贝后 sha256 不变
https://gitlab.cee.redhat.com/ltitov/openshift-disconnected-stuff/-/blob/master/copy-images.sh 
```
cat > copy-images.sh << 'EOF'
#!/bin/bash

REGEX="(.*)=(.*)"
while read p; do

# echo $p

[[ $p =~ $REGEX ]]
echo "Copy ${BASH_REMATCH[1]} to ${BASH_REMATCH[2]}"
$(which skopeo) copy --all docker://${BASH_REMATCH[1]} docker://${BASH_REMATCH[2]}

done
EOF
```

### 查看某节点上的 pod
```
oc get pods -o wide --all-namespaces | grep worker0 
```

### 问题 Error: error creating container storage: layer not known 的处理方式
划重点：这个问题在异常掉电重启时经常会遇到<br>
划重点：这个问题在异常掉电重启时经常会遇到<br>
划重点：这个问题在异常掉电重启时经常会遇到<br>

解决方案参考：
https://bugzilla.redhat.com/show_bug.cgi?id=1857224<br>

https://github.com/openshift/okd/issues/297<br>
After rebooting a node, it sometimes never transitions to the Ready state.  This may happen more frequently under load.  Typical messages are:<br>
Jun 25 14:08:07 worker-2.ostest.test.metalkube.org podman[1424]: Error: error creating container storage: layer not known<br>

```
ssh core@<workerip>

sudo systemctl stop kubelet
sudo systemctl stop crio
sudo rm -rf /var/lib/containers
sudo systemctl start crio
sudo systemctl start kubelet
```

### 介绍 OpenShift Logging 的博客
https://blog.csdn.net/weixin_43902588/article/details/105586460

### 如何在 OpenShift 下实现首次 Deployment 从外部 registry 获取 image，再次则从本地 cache 获取 image
问题： Question regarding the image registry: Suppose i want to deploy an image coming from quay.io/myproject/myapp:latest ? Will this image be cached in the registry so next time i will redeploy it the deployment will be potentially faster? <br>
回答：You can achieve this with ImageStream --reference-policy=local <br>
Your ImageStream should point to the external repo and the DC/Deployment should point to the internal registry url <br>
The image itself is only cached the first time a Pod needs it (it's "pullthrough") <br>


### 使用 fio 测试您的存储是否满足 etcd 的响应及时延要求
https://www.ibm.com/cloud/blog/using-fio-to-tell-whether-your-storage-is-fast-enough-for-etcd


### 24 小时后添加 worker nodes 到 openshift 4 集群
Adding worker nodes to the OCP 4 UPI cluster existing 24+ hours<br>
https://access.redhat.com/solutions/4799921

```
export MCS=api-int.cluster-0001.rhsacn.org:22623

echo "q" | openssl s_client -connect $MCS  -showcerts | awk '/-----BEGIN CERTIFICATE-----/,/-----END CERTIFICATE-----/' | base64 --wrap=0 | tee ./api-int.base64 && \
sed --regexp-extended --in-place=.backup "s%base64,[^,]+%base64,$(cat ./api-int.base64)\"%" ./worker.ign

# upload worker.ign to web server
```

更推荐的方式是，这种方法更不容易出错
```
for master: 
  oc extract -n openshift-machine-api secret/master-user-data --keys=userData --to=-
for worker: 
  oc extract -n openshift-machine-api secret/worker-user-data --keys=userData --to=-
```

### OpenShift 4 如何创建定制化 catalog
https://docs.openshift.com/container-platform/4.5/operators/admin/olm-managing-custom-catalogs.html

### 部署 openshift 相关
部署 openshift 到已经存在的 vpc (aws) <br>
https://www.openshift.com/blog/deploy-openshift-to-existing-vpc-on-aws
