# Chuẩn bị Server KVM

Kiểm tra vmx enable trên KVM host
```sh
cat /proc/cpuinfo| egrep -c "vmx|svm"
```

> Nếu OUTPUT câu lệnh trên >0 thì đã enable vmx OK 

- Cài đặt epel-release & Update 
```
yum install epel-release -y
yum update -y
```

- Stop firewalld Disable Selinux
``` sh
systemctl disable firewalld
systemctl stop firewalld
sudo systemctl disable NetworkManager
sudo systemctl stop NetworkManager
sudo systemctl enable network
sudo systemctl start network
sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/sysconfig/selinux
sed -i 's/SELINUX=permissive/SELINUX=disabled/g' /etc/sysconfig/selinux
```

- Disable IPv6
```sh
echo "net.ipv6.conf.all.disable_ipv6 = 1" >> /etc/sysctl.conf
echo "net.ipv6.conf.default.disable_ipv6 = 1" >> /etc/sysctl.conf
sysctl -p
```

- Option ssh ipv4
```sh
sed -i 's/#ListenAddress 0.0.0.0/ListenAddress 0.0.0.0/g' /etc/ssh/sshd_config 
systemctl restart sshd 
```

- Cài đặt CMDlog
```sh 
curl -Lso- https://raw.githubusercontent.com/nhanhoadocs/ghichep-cmdlog/master/cmdlog.sh | bash
```

- Cài đặt Chronyd 
```sh
yum install chrony -y
#sed -i 's|server 1.centos.pool.ntp.org iburst|server x.x.x.x iburst|g' /etc/chrony.conf
systemctl enable --now chronyd 
hwclock --systohc
```

# Cài đặt KVM Node (để đóng Images)
```
yum install -y qemu-kvm qemu-img virt-manager libvirt libvirt-python libvirt-client \
virt-install virt-viewer bridge-utils  "@X Window System" xorg-x11-xauth xorg-x11-fonts-* \
xorg-x11-utils mesa-libGLU*.i686 mesa-libGLU*.x86_64 dejavu-lgc-sans-fonts
touch /root/.Xauthority
```

Start libvirt
```sh 
systemctl start --now libvirtd
```

Disable ipv6
```sh
echo "net.ipv6.conf.all.disable_ipv6 = 1" >> /etc/sysctl.conf
echo "net.ipv6.conf.default.disable_ipv6 = 1" >> /etc/sysctl.conf
sysctl -p
```

Enable `X11Forwarding yes` trong `/etc/ssh/sshd_config`
```sh
X11Forwarding yes
```

Thêm cấu hình `/etc/ssh/sshd_config` để sử dụng X11Forward khi disable IPv6
```sh
X11Forwarding yes
AddressFamily inet
```

Restart SSH
```sh
systemctl restart sshd
```

Tạo folder channel cho các target của VM 
```
mkdir -p /var/lib/libvirt/qemu/channel/target
chown -R qemu:kvm /var/lib/libvirt/qemu/channel
```

Restart libvirt 
```sh
systemctl restart libvirtd
```

Cài libguestfs-tools để xử lý file `.qcow2` thành file `.img` sau khi cài đặt cấu hình xong VM.
```sh
yum install libguestfs-tools -y
```

Bật tính năng `nestes` cho phép ảo hóa trên VM 
```sh 
touch /etc/modprobe.d/kvm.conf
echo "options kvm_intel nested=1" > /etc/modprobe.d/kvm.conf
init 6
```

Copy Images vào VM => Tiến hành đóng Images


# WebvirtCloud

Cấu hình libvirt trên các node KVM cho phép TCP connection kết nối 
```sh 
cp /etc/libvirt/libvirtd.conf /etc/libvirt/libvirtd.conf.orig
sed -i 's|#listen_tls = 0|listen_tls = 0|'g /etc/libvirt/libvirtd.conf
sed -i 's|#listen_tcp = 1|listen_tcp = 1|'g /etc/libvirt/libvirtd.conf
sed -i 's|#tcp_port = "16509"|tcp_port = "16509"|'g /etc/libvirt/libvirtd.conf
sed -i 's|#auth_tcp = "sasl"|auth_tcp = "none"|'g /etc/libvirt/libvirtd.conf
cp /etc/sysconfig/libvirtd /etc/sysconfig/libvirtd.orig 
sed -i 's|#LIBVIRTD_ARGS="--listen"|LIBVIRTD_ARGS="--listen"|'g /etc/sysconfig/libvirtd
```

Restart dịch vụ
```sh 
systemctl restart libvirtd
systemctl restart openstack-nova-compute.service
```

https://news.cloud365.vn/kvm-huong-dan-cai-dat-webvirtcloud-quan-li-ha-tang-kvm/

## Cấu hình network 
Mô hình 

Tạo folder backup 
```sh 
mkdir -p /opt/backup-interface/
cp ifcfg-* /opt/backup-interface/
```

### Đối với tạo Bridge bình thường 

Tạo Bridge `public172`
```sh 
cat <<EOF >> /etc/sysconfig/network-scripts/br-public172
DEVICE="public172"
BOOTPROTO="static"
IPADDR="172.16.4.124"
NETMASK="255.255.240.0"
GATEWAY="172.16.10.1"
DNS1=8.8.8.8
ONBOOT="yes"
TYPE="Bridge"
NM_CONTROLLED="no"
EOF
```

Cắm `em1` vào Bridge vừa tạo 
```sh 
rm -f /etc/sysconfig/network-scripts/ifcfg-em1
cat <<EOF >> /etc/sysconfig/network-scripts/br-em1
DEVICE=em1
TYPE=Ethernet
BOOTPROTO=none
ONBOOT=yes
NM_CONTROLLED=no
BRIDGE=public172
EOF
```

### Đối với tạo các Bridge VLAN

Điều chỉnh lại interface `em2`
```sh 
rm -f /etc/sysconfig/network-scripts/ifcfg-em2
cat <<EOF >> /etc/sysconfig/network-scripts/ifcfg-em2
DEVICE=em2
TYPE=Ethernet
BOOTPROTO=none
ONBOOT=yes
NM_CONTROLLED=no
EOF
```

Tạo interface Vlan10 gắn vào interface em2
```sh 
cat <<EOF >> /etc/sysconfig/network-scripts/ifcfg-em2.10
DEVICE=em2.10
BOOTPROTO=none
ONBOOT=yes
VLAN=yes
BRIDGE=vlan10
TYPE=Ethernet
NM_CONTROLLED=no
EOF
```

Taọ Bridge Vlan10 cho Interface Vlan10 
```sh 
cat <<EOF >> /etc/sysconfig/network-scripts/ifcfg-vlan10 
DEVICE=vlan10
TYPE=Bridge
BOOTPROTO=none
ONBOOT=yes
NM_CONTROLLED=no
EOF
```

### Nâng cao hơn tý: Đối với tạo các Bridge VLAN (Có bridge trunk)

Tạo Bridge trunk 
```sh 
cat <<EOF >> /etc/sysconfig/network-scripts/br-trunk
DEVICE=brtrunk
TYPE=Bridge
BOOTPROTO=none
ONBOOT=yes
NM_CONTROLLED=no
EOF 
```

Cắm `em2` vào Bridge vừa tạo 
```sh 
rm -f /etc/sysconfig/network-scripts/ifcfg-em2
cat <<EOF >> /etc/sysconfig/network-scripts/ifcfg-em2
TYPE=Ethernet
BOOTPROTO=none
NAME=em2
DEVICE=em2
ONBOOT=yes
BRIDGE=brtrunk
NM_CONTROLLED=no
EOF
```

Tạo interface Vlan10 gắn vào Bridge trunk 
```sh 
cat <<EOF >> /etc/sysconfig/network-scripts/ifcfg-brtrunk.10
DEVICE=brtrunk.10
BOOTPROTO=none
ONBOOT=yes
VLAN=yes
BRIDGE=vlan10
TYPE=Ethernet
NM_CONTROLLED=no
EOF
```

Taọ Bridge Vlan10 cho Interface Vlan10 
```sh 
cat <<EOF >> /etc/sysconfig/network-scripts/ifcfg-vlan10 
DEVICE=vlan10
TYPE=Bridge
BOOTPROTO=none
ONBOOT=yes
NM_CONTROLLED=no
EOF
```


