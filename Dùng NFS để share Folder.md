# Dùng NFS để share Folder

Dùng cho hệ thống là 2 máy Centos hay Unix nói chung 


## Cấu hình trên Node Server

Bước đầu tiên, chúng tôi sẽ cài đặt các gói này trên máy chủ CentOS với yum:

```
yum install nfs-utils
```

Bây giờ, hãy tạo thư mục sẽ được chia sẻ bởi NFS:

```
mkdir /var/nfsshare
```

Thay đổi quyền của thư mục như sau:

```
chmod -R 755 /var/nfsshare
chown nfsnobody:nfsnobody /var/nfsshare
```

Chúng tôi sử dụng `/var/nfsshare` như một thư mục chia sẻ, nếu chúng tôi sử dụng một ổ đĩa khác như thư mục `/home`, thì việc thay đổi quyền sẽ gây ra sự cố lớn về quyền và làm hỏng toàn bộ hệ thống phân cấp.Vì vậy,trong trường hợp, chúng ta muốn chia sẻ thư mục `/home` thì các quyền không được thay đổi. Tiếp theo, chúng ta cần khởi động các dịch vụ và kích hoạt chúng vào lúc khởi động.

```
systemctl enable rpcbind
systemctl enable nfs-server
systemctl enable nfs-lock
systemctl enable nfs-idmap
systemctl start rpcbind
systemctl start nfs-server
systemctl start nfs-lock
systemctl start nfs-idmap
```

Bây giờ chúng ta sẽ chia sẻ thư mục NFS qua mạng như sau:

```
vim /etc/exports
```

Chúng tôi sẽ tạo hai điểm chia sẻ `/home` và `/var/nfsshare`. Chỉnh sửa tệp xuất như sau:

```
/var/nfsshare    192.168.0.101(rw,sync,no_root_squash,no_all_squash)
/home            192.168.0.101(rw,sync,no_root_squash,no_all_squash)
```

Chú ý:
- Lưu ý 192.168.0.101 là IP của máy khách, nếu bạn muốn bất kỳ máy khách nào khác truy cập vào nó, bạn cần thêm IP của nó 
- Nếu không bạn có thể thêm "*" thay vì IP cho tất cả các truy cập IP. 
- Điều kiện là nó phải ping được ở cả hai đầu.

Cuối cùng, khởi động dịch vụ NFS:

```
systemctl restart nfs-server
```

Một lần nữa, chúng tôi cần thêm ghi đè dịch vụ NFS trong dịch vụ khu vực công cộng tường lửa CentOS 7-cmd như:

```
firewall-cmd --permanent --zone=public --add-service=nfs
firewall-cmd --permanent --zone=public --add-service=mountd
firewall-cmd --permanent --zone=public --add-service=rpc-bind
firewall-cmd --reload
```

## Cấu hình trên Node Client

Trong trường hợp của tôi, tôi có một máy tính để bàn CentOS 7 làm máy khách. Các phiên bản CentOS khác cũng sẽ hoạt động theo cách tương tự. Cài đặt gói nfs-utild như sau:

```
yum install nfs-utils
```

Bây giờ tạo các điểm gắn kết thư mục NFS:

```
mkdir -p /mnt/nfs/home
mkdir -p /mnt/nfs/var/nfsshare
```

Tiếp theo, chúng tôi sẽ gắn kết thư mục chính chia sẻ NFS trong máy khách như hình dưới đây:

```
mount -t nfs 192.168.0.100:/home /mnt/nfs/home/
```

Nó sẽ gắn kết `/home` của máy chủ NFS. Tiếp theo, chúng tôi sẽ gắn kết thư mục `/var/nfsshare`:

```
 mount -t nfs 192.168.0.100:/var/nfsshare /mnt/nfs/var/nfsshare/
 ```
 
 Bây giờ chúng tôi đã kết nối với phần chia sẻ NFS, chúng tôi sẽ kiểm tra chéo nó như sau:
 
 ```
 df -kh
 ```
 ```
 [root@client1 ~]# df -kh
Filesystem                    Size  Used Avail Use% Mounted on
/dev/mapper/centos-root        39G  1.1G   38G   3% /
devtmpfs                      488M     0  488M   0% /dev
tmpfs                         494M     0  494M   0% /dev/shm
tmpfs                         494M  6.7M  487M   2% /run
tmpfs                         494M     0  494M   0% /sys/fs/cgroup
/dev/mapper/centos-home        19G   33M   19G   1% /home
/dev/sda1                     497M  126M  372M  26% /boot
192.168.0.100:/var/nfsshare   39G  980M   38G   3% /mnt/nfs/var/nfsshare
192.168.0.100:/home           19G   33M   19G   1% /mnt/nfs/home
[root@client1 ~]#
```

Vì vậy, chúng tôi được kết nối với phần chia sẻ NFS. 

Bây giờ chúng ta sẽ kiểm tra quyền đọc/ghi trong đường dẫn được chia sẻ. Tại máy khách, nhập lệnh:

```
touch /mnt/nfs/var/nfsshare/test_nfs
```

## Permanent NFS mounting

Chúng tôi phải gắn kết lại phần chia sẻ NFS tại máy khách sau mỗi lần khởi động lại. Dưới đây là các bước để gắn nó vĩnh viễn bằng cách thêm tệp NFS-share trong `/etc/fstab` của máy khách:

```
nano /etc/fstab
```
Thêm các mục như thế này:

```
[...]
192.168.0.100:/home    /mnt/nfs/home   nfs defaults 0 0
192.168.0.100:/var/nfsshare    /mnt/nfs/var/nfsshare   nfs defaults 0 0
```

Điều này sẽ làm cho gắn kết vĩnh viễn của NFS-share. Bây giờ bạn có thể khởi động lại máy và điểm gắn kết sẽ tồn tại vĩnh viễn ngay cả sau khi khởi động lại.
