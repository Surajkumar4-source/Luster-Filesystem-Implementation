# LustreFS  Setup Implementation
















---

## Host Configuration

Add the following entries to the `/etc/hosts` file on all nodes to ensure proper hostname resolution:

```yml
echo "192.168.230.142 node1" | sudo tee -a /etc/hosts
echo "192.168.230.143 node2" | sudo tee -a /etc/hosts
echo "192.168.230.144 node3" | sudo tee -a /etc/hosts
echo "192.168.230.145 client" | sudo tee -a /etc/hosts
```

---

## Server Configuration

### Metadata Server (MDS)

1. Prepare the environment:

```yml

cd /etc/yum.repos.d/
sed -i 's/mirrorlist/#mirrorlist/g' /etc/yum.repos.d/CentOS-*
sed -i 's|#baseurl=http://mirror.centos.org|baseurl=http://vault.centos.org|g' /etc/yum.repos.d/CentOS-*
systemctl stop firewalld
systemctl disable firewalld
yum install -y nano
dnf config-manager --set-enabled powertools
dnf install https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm
dnf install https://zfsonlinux.org/epel/zfs-release-2-3$(rpm --eval "%{dist}").noarch.rpm
nano /etc/selinux/config
dnf install kernel-headers kernel-devel
dnf upgrade kernel
reboot
```

2. Install and configure ZFS:

```yml
dnf install zfs
modprobe -v zfs
```

3. Add Lustre repository and install Lustre packages:

```yml
echo "[lustre-server]
name=lustre-server
baseurl=https://downloads.whamcloud.com/public/lustre/lustre-2.15.4/el8.9/server/
exclude=*debuginfo*
enabled=1
gpgcheck=0" | sudo tee /etc/yum.repos.d/lustre.repo
dnf install -y lustre-dkms lustre-osd-zfs-mount lustre kmod-lustre
modprobe -v lustre

lsmod | grep lustre       # for verification
```

4. Prepare the storage:

```yml
lsblk
parted /dev/nvme0n2 mklabel gpt
parted -a optimal /dev/nvme0n2 mkpart primary 0% 100%
zpool create mds_pool /dev/nvme0n2p1
zfs create mds_pool/mdt0
zfs set atime=off mds_pool/mdt0
zpool list
umount /mds_pool/mdt0
umount /mds_pool
mkfs.lustre --reformat --mdt --fsname=lustrefs --mgs --index=0 --backfstype=zfs mds_pool/mdt0
mkdir /mnt/mdt0/
mount -t lustre mds_pool/mdt0 /mnt/mdt0


lctl dl                                     # for verification
lctl get_param -n health_check              # for verification
```

5. Configure the network:

```yml
nano /etc/modprobe.d/lnet.conf
# Add: options lnet networks="tcp0(ens33)"        # Change with your interface    
modprobe lnet
lsmod | grep lnet
lctl network up
lctl ping <MGS-SERVER-IP>@tcp0      # Change with your server IP
```

### Object Storage Target (OST) Servers

  - *Repeat the following steps for each OST server*:

1. Prepare the environment:

```yml
cd /etc/yum.repos.d/
sed -i 's/mirrorlist/#mirrorlist/g' /etc/yum.repos.d/CentOS-*
sed -i 's|#baseurl=http://mirror.centos.org|baseurl=http://vault.centos.org|g' /etc/yum.repos.d/CentOS-*
systemctl stop firewalld
systemctl disable firewalld
yum install -y nano
dnf config-manager --set-enabled powertools
dnf install https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm
dnf install https://zfsonlinux.org/epel/zfs-release-2-3$(rpm --eval "%{dist}").noarch.rpm
nano /etc/selinux/config
dnf install kernel-headers kernel-devel
dnf upgrade kernel
reboot
```

2. Install and configure ZFS:

```yml
dnf install zfs
modprobe -v zfs
```

3. Add Lustre repository and install Lustre packages:

```yml
echo "[lustre-server]
name=lustre-server
baseurl=https://downloads.whamcloud.com/public/lustre/lustre-2.15.4/el8.9/server/
exclude=*debuginfo*
enabled=1
gpgcheck=0" | sudo tee /etc/yum.repos.d/lustre.repo
dnf install -y lustre-dkms lustre-osd-zfs-mount lustre kmod-lustre
modprobe -v lustre
```

4. Prepare the storage:

```yml
lsblk
parted /dev/nvme0n2 mklabel gpt
parted -a optimal /dev/nvme0n2 mkpart primary 0% 100%
zpool create ost_pool1 /dev/nvme0n2p1
zfs create ost_pool1/ost0
zfs set atime=off ost_pool1/ost0
umount ost_pool1/ost0
umount ost_pool1
mkfs.lustre --ost --fsname=lustrefs --reformat --mgsnode=192.168.230.142@tcp --index=0 --backfstype=zfs ost_pool1/ost0
mkdir -p /mnt/ost0
mount -t lustre ost_pool1/ost0 /mnt/ost0
```

---

## Client Configuration

1. Prepare the environment:

```yml
cd /etc/yum.repos.d/
sed -i 's/mirrorlist/#mirrorlist/g' /etc/yum.repos.d/CentOS-*
sed -i 's|#baseurl=http://mirror.centos.org|baseurl=http://vault.centos.org|g' /etc/yum.repos.d/CentOS-*
systemctl stop firewalld
systemctl disable firewalld
yum install -y nano
dnf config-manager --set-enabled powertools
dnf install https://dl.fedoraproject.org/pub/epel/epel-release-latest-8.noarch.rpm
nano /etc/selinux/config                             # SELINUX=disabled
dnf install kernel-headers kernel-devel
dnf upgrade kernel
reboot
```

2. Add Lustre repository and install Lustre client:

```yml
echo "[lustre-client]
name=lustre-client
baseurl=https://downloads.whamcloud.com/public/lustre/lustre-2.15.4/el8.9/client/
exclude=*debuginfo*
enabled=1
gpgcheck=0" | sudo tee /etc/yum.repos.d/lustre.repo
dnf install -y lustre-client lustre-client-dkms kmod-lustre-client
modprobe lustre
```

3. Configure the network:

```yml
nano /etc/modprobe.d/lnet.conf
# Add: options lnet networks="tcp0(ens33)"
modprobe lnet
lsmod | grep lnet
lctl network up
lctl ping <MGS-SERVER>@tcp0
```

4. Mount the Lustre file system:

```yml
mkdir /mnt/lustre
mount -t lustre <MGS-SERVER>@tcp0:/lustrefs /mnt/lustre
lctl dl
lfs check servers
lfs osts
```

---


<br>
<br>

This implementation ensures every command is clear and detailed. Update the placeholders (like `<MGS-SERVER-IP>` and `<MGS-SERVER>`) with the actual values specific to your setup.
