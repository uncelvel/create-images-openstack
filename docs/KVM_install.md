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
sed -i 's/SELINUX=enforcing/SELINUX=disabled/g' /etc/selinux/config
sed -i 's/SELINUX=permissive/SELINUX=disabled/g' /etc/selinux/config
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

Cấu hình libvirt cho phép TCP connection kết nối 
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
