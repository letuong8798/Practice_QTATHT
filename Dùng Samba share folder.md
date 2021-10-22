# Triển khai Samba trên CentOS không cần chứng thực

Chia sẻ giữa CentOS và Windows 7 

## Trên Windows 7

To make the Windows machine reachable in Windows proceed like this. In the run terminal & add  the entry of your server IP address:

```
notepad C:\Windows\System32\drivers\etc\hosts
```
In my case it was like this, just save the values.

```
[...]
192.168.0.100 	server1.example.com	centos
```

## Tạo Folder trên máy Centos 

First I will explain the methodology to install Samba with an anonymous share. To install the Samba software, run:

```
yum install samba samba-client samba-common
```

Now to configure samba, edit the file /etc/samba/smb.conf. Before making changes, I will make the backup of original file as  /etc/samba/smb.conf.bak

```
cp -pf /etc/samba/smb.conf /etc/samba/smb.conf.bak
```

As I want to start with an empty file, I'll use the cat command to empty smb.conf. That's faster than deleting all the lines in vi.

```
cat /dev/null > /etc/samba/smb.conf
```

Further, give the entries like this

```
vi /etc/samba/smb.conf
[global]
workgroup = WORKGROUP
server string = Samba Server %v
netbios name = centos
security = user
map to guest = bad user
dns proxy = no


#============================ Share Definitions ============================== 
[Giao_trinh_KMA]
path = /samba/Giao_trinh_KMA
browsable =yes
writable = yes
guest ok = yes
read only = no


[Bai_tap_KMA]
path = /samba/Bai_tap_KMA
browsable =yes
writable = yes
guest ok = yes
read only = no
```


```
mkdir -p /samba/Giao_trinh_KMA
mkdir -p /samba/Bai_tap_KMA
systemctl enable smb.service
systemctl enable nmb.service
systemctl restart smb.service
systemctl restart nmb.service
```

Further CentOS 7 Firewall-cmd will block the samba access, to get rid of that we will run:

```
firewall-cmd --permanent --zone=public --add-service=samba
```

Finally, reload the firewall to apply the changes.

```
firewall-cmd --reload
```


Tiếp theo, ta có:

```
cd /samba
chmod -R 0755 Bai_tap_KMA/
chmod -R 0755 Giao_trinh_KMA/
chown -R nobody:nobody Bai_tap_KMA/
chown -R nobody:nobody Giao_trinh_KMA/
```

Và cuối cùng là lệnh sau đây:

```
chcon -t samba_share_t Bai_tap_KMA/
chcon -t samba_share_t Giao_trinh_KMA/
```
