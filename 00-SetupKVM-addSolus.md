# Setup Master Share KVM và Add Cụm Solus
## I. thực hiện check list test phần cứng
### 1. thực hiện test phần cứng theo checklist yêu cầu
Bao gồm các thành phần: 
- CPU
- Memory
- Power Supplies
- Disk
- Network Interface
- iLO,iDRAC
- Card RAID
- Power ON\OFF
- Theo dõi sự ổn định

### 2. Cấu hình cơ bản OS ( CentOS 7)
- Cấu hình Bridge Card mạng IP_Public ( IPv4 và IPv6 )
  - ADD Port mạng `Em1` chạy master card `br0` . Ví dụ:

      <img src="../Solus/images/Add-KVM-Solus/brigde-Public.png">

  - Thống tin cấu hình
```
# Trên Port Em1

TYPE=Ethernet
NAME=em1
DEVICE=em1
ONBOOT=yes
HWADDR=2e:2e:d9:1e:ce:d3  \MAC port Em1
BRIDGE=br0      
IPV6INIT=yes   \ Mở cấu hình IPv6

# Trên br0
DEVICE=br0
TYPE=Bridge
BOOTPROTO=static
IPADDR=103.28.36.11
GATEWAY=103.28.36.1
NETMASK=255.255.255.0
ONBOOT=yes
DNS1=8.8.8.8
IPV6INIT=yes
IPV6_AUTOCONF=no
IPV6_DEFAULTGW=2001:0df1:3200:0000:0000:0000:0000:0001    \\ gateway IPv6
IPV6ADDR_SECONDARIES="2001:df1:3200::19\64"    \\ IPv6

```

- Cấu hình Bridge Card mạng IP_local
  - Cấu hình `br1` tượng tự `br0`. Hình ảnh: 
   <img src="..\Solus\images\Add-KVM-Solus\brigde-local.png">
  - Thông tin cấu hình:

```
#Em3
TYPE=Ethernet
NAME=em3
DEVICE=em3
ONBOOT=yes
HWADDR=b8:ca:3a:5f:7d:76
BRIDGE=br1


#br1  \\ lưu ý không đặt gateway cho IP local
DEVICE=br1
TYPE=Bridge
BOOTPROTO=static
IPADDR=192.168.10.11
NETMASK=255.255.255.0
ONBOOT=yes


```
## II. Cài đặt KVM Solus + Add disk.
### 1. Cài đặt KVM Solus ( CentOS 7)
- Tiến hàng cài đặt theo câu lệnh:
```
curl -o install.sh https:\\files.soluslabs.com\install.sh && sh install.sh

```
chọn `options 2` => `Hypervisor KVM`

<img src="..\Solus\images\Add-KVM-Solus\optionsinstallkvmsolus.png">

- Đường dẫn tham khảo [tại đây](https:\\documentation.solusvm.com\display\BET\Installation+of+KVM+Slave).

### 2. Tạo phân vùng cho DISK.
#### 2.1 Đối với Disk setup VPS.
##### 2.1.1 Tạo LVM cho Disk sẽ setup VPS.
- Bước 1: Sử dụng lệnh sau để tạo phân vùng mới
```
fdisk -l \dev\sdb
```

- Bước 2: Nhập `n` để tạo phân vùng mới sẽ nhắc bạn chỉ định cho một phân vùng chính hoặc phân vùng mở rộng. Nhập `p` cho phân vùng chính hoặc `e` cho phân vùng mở rộng.

```
Command (m for help): n
Partition type:
   p   primary (0 primary, 0 extended, 4 free)
   e   extended
Select (default p): p
```
- Bước 3: Sau đó, bạn sẽ được nhắc nhập số của phân vùng sẽ được tạo. Bạn có thể nhấn Enter để chấp nhận mặc định.
```
Partition number (1-4, default 1): 1
```

Bước 4: Sau đó, bạn sẽ được nhắc nhập kích thước của phân vùng sẽ được tạo ví dụ dưới chúng ta sẽ tạo đĩa với size 5GB. Bạn có thể nhấn `Enter` để sử dụng tất cả không gian có sẵn.

```
First sector (2048-33554431, default 2048): ` gõ Enter`
Last sector, +sectors or +size{K,M,G} (2048-33554431, default 33554431): ` 10002400 ` \\ giá trị disk 
Partition 1 of type Linux and of size 4 MiB is set
```
- Bước 5: Lựa chọn định dạng LVM cho disk
  - chọn `t` để thay đổi định dạng phân vùng disk vừa khởi tạo

  ```
  Command (m for help): t
  selected Partition 1
  ```
  sau đó bấm `L` và gõ `8e` để chọn định dạng `Linux LVM`

- Bước 6: Chạy lệnh  `w` để lưu các thay đổi.
```
Command (m for help): w
The partition table has been altered!

Calling ioctl() to re-read partition table.
Syncing disks.
```

Hình ảnh minh họa:

<img src="..\Solus\images\Add-KVM-Solus\flisk.png">

##### 2.1.2 Tạo Physical Volume.

```
pvcreate \dev\sdb1
```

##### 2.1.3 Tạo Volume Group

```
vgcreate -s 32m vps \dev\sdb1
```

*lưu ý: `VPS` là tên khởi tạo cho Volume Group, với mỗi `\dev\sdxx` sẽ có 1 tên khác nhau. ví dụ: `\dev\sdc1` sẽ là `VPS1`. Trong đó :
- `VPS`: là tên đặt tùy ý, thường sẽ đặt theo quy định hoặc form mẫu có trước.
-  `\dev\sdxx`:  Physical Volume vừa khởi tạo ở bước bên trên.


==> Cấu hình tương tự đối với những `Disk` setup VPS còn lại

### 2.2 Đối với Disk Backup.
#### 2.2.1 Tạo Disk Backup.
- Bước 1: Sử dụng lệnh sau để tạo phân vùng mới
```
fdisk -l \dev\sdf
```

- Bước 2: Nhập `n` để tạo phân vùng mới sẽ nhắc bạn chỉ định cho một phân vùng chính hoặc phân vùng mở rộng. Nhập `p` cho phân vùng chính hoặc `e` cho phân vùng mở rộng.

```
Command (m for help): n
Partition type:
   p   primary (0 primary, 0 extended, 4 free)
   e   extended
Select (default p): p
```
- Bước 3: Sau đó, bạn sẽ được nhắc nhập số của phân vùng sẽ được tạo. Bạn có thể nhấn Enter để chấp nhận mặc định.
```
Partition number (1-4, default 1): 1
```

Bước 4: Sau đó, bạn sẽ được nhắc nhập kích thước của phân vùng sẽ được tạo ví dụ dưới chúng ta sẽ tạo đĩa với size 5GB. Bạn có thể nhấn `Enter` để sử dụng tất cả không gian có sẵn.

```
First sector (2048-33554431, default 2048): ` gõ Enter`
Last sector, +sectors or +size{K,M,G} (2048-33554431, default 33554431): ` 10002400 ` \\ giá trị disk 
Partition 1 of type Linux and of size 4 MiB is set
```

- Bước 5: Chạy lệnh  `w` để lưu các thay đổi.
```
Command (m for help): w
The partition table has been altered!

Calling ioctl() to re-read partition table.
Syncing disks.
```
#### 2.2.2 Cấu hình định dạng ext4 Disk Backup.
- chọn `định dạng` và `mount disk` vừa khởi tạo
```
mkfs -t ext4 \dev\sdf1
cd 
mkdir \backup
mount \dev\sdf1 \backup
```
### 3. Cấu hình gắn phân vùng backup trong \etc\fstab.
- Các phân vùng gắn kết của chúng ta là tạm thời. Nếu hệ điều hành được khởi động lại, các thư mục được gắn kết này sẽ bị mất. Vì vậy, chúng ta cần phải gắn kết vĩnh viễn. Để thực hiện gắn kết vĩnh viễn phải nhập trong tệp `\etc\fstab`. Bạn có thể sử dụng [trình soạn thảo vi](https:\\blogd.net\linux\su-dung-trinh-soan-thao-vi-vim-co-ban\) để nhập vào:

```
[root@localhost ~]# vi \etc\fstab

#
# \etc\fstab
# Created by anaconda on Tue May 18 15:11:49 2021
#
# Accessible filesystems, by reference, are maintained under '\dev\disk'
# See man pages fstab(5), findfs(8), mount(8) and\or blkid(8) for more info
#
UUID=902d0257-47a3-4d3d-b6c9-db86ba153e77 \                       ext4    defaults        1 1
UUID=9485362a-ae97-447a-9c4e-36bde9ebd346 \boot                   ext4    defaults        1 2
UUID=da59ecac-eade-436c-9177-f985ff291ed4 swap                    swap    defaults        0 0
UUID=026c0013-8d39-4725-afcf-2ba5ffbd2c09 \backup                 ext4    defaults        0 0
UUID=50fd3baf-ab10-4979-a2f8-ac5cbee088f5 \backup1                ext4    defaults        0 0

```
- Để lấy UUID của các phân vùng chúng ta thực hiện như sau:
```
[root@kvmshare3611 kvm230]# blkid \dev\sdf1
\dev\sdf1: UUID="026c0013-8d39-4725-afcf-2ba5ffbd2c09" TYPE="ext4"
[root@kvmshare3611 kvm230]# blkid \dev\sdg1
\dev\sdg1: UUID="50fd3baf-ab10-4979-a2f8-ac5cbee088f5" TYPE="ext4"

```

Kiểm disk sau khi khởi tạo bằng lệnh : ` lsblk `

### 4. Mode tunning cho guest
```
tuned-adm profile balanced
echo "net.ipv6.conf.all.disable_ipv6 = 0" >> \etc\sysctl.conf 
echo "net.ipv6.conf.default.disable_ipv6 = 0" >> \etc\sysctl.conf 
echo "net.ipv4.ip_forward = 1" >> \etc\sysctl.conf 
echo "vm.swappiness = 5" >> \etc\sysctl.conf 
echo "vm.vfs_cache_pressure = 50" >> \etc\sysctl.conf 
sysctl -p
```

### 5. Disable SELinux

#### Kiểm tra trạng thái:
```
sestatus

SELinux status:                 enabled
SELinuxfs mount:                \sys\fs\selinux
SELinux root directory:         \etc\selinux
Loaded policy name:             targeted
Current mode:                   enforcing
Mode from config file:          enforcing
Policy MLS status:              enabled
Policy deny_unknown status:     allowed
Max kernel policy version:      31
```
#### Disable SELinux
```
sudo setenforce 0
```
- Open the \etc\selinux\config file and set the SELINUX mod to disabled:
```
\etc\selinux\config
# This file controls the state of SELinux on the system.
# SELINUX= can take one of these three values:
#       enforcing - SELinux security policy is enforced.
#       permissive - SELinux prints warnings instead of enforcing.
#       disabled - No SELinux policy is loaded.
SELINUX=disabled
# SELINUXTYPE= can take one of these two values:
#       targeted - Targeted processes are protected,
#       mls - Multi Level Security protection.
SELINUXTYPE=targeted
```
- Save the file and reboot your CentOS system with:
```
sudo shutdown -r now
```
- Once the system boots up, verify the change with the sestatus command:
```
sestatus

SELinux status:                 disabled
```

## III. Add Master lên control SolusVM.
### 1. Add Solus
- Key và ID lấy trong file \usr\local\solusvm\data\solusvm.conf
```
[root@kvmshare3611 kvm230]# cat \usr\local\solusvm\data\solusvm.conf
2RS7HCppdhqOcLm7qKJQB67Sy2G15tBinbrFnK8Pwwe7km1J4t:pm3S5nuZquaqwtGQiNNpMLW66e3qNjCABph0EkRinv3YUacWyG
```
- Trong đó:
  - 2RS7HCppdhqOcLm7qKJQB67Sy2G15tBinbrFnK8Pwwe7km1J4t: `ID Key`
  - pm3S5nuZquaqwtGQiNNpMLW66e3qNjCABph0EkRinv3YUacWyG: `ID Password`
  
  
<img src="..\Solus\images\Add-KVM-Solus\info.png">

### 2. Add dải IP

<img src="..\Solus\images\Add-KVM-Solus\addvlan.png">

## II. Test VM

tiến hành tạo VM, sau đó test tốc VM
Sử dụng các lệnh: 

1. Hdparm
```
yum update -y 
sudo yum install hdparm
sudo hdparm -Tt \dev\sda
```
2. Tốc độ và đường truyền
```
curl -Lso- tocdo.net | bash

```
Kết quả: 

<img src="..\Solus\images\Add-KVM-Solus\testvm.jpg">



\\ The end