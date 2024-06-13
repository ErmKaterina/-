ISP

1. hostnamectl set-hostname isp
2. apt-get update && apt-get install chrony nano nftables -y
3. nano /etc/net/sysctl.conf 
    1. net.ipv4.ip_forward = 1
4. cd /etc/net/ifaces
5. cp -r enp0s3/ enp0s8
6. cp -r enp0s3/ enp0s9
7. cp -r enp0s3/ enp0s10
8. nano enp0s8/options
    1. BOOTPROTO=static
9. nano enp0s8/ipv4address
    1. 100.100.100.1/28
10. nano enp0s9/options
    1. BOOTPROTO=static
11. nano enp0s9/ipv4address
    1. 150.150.150.1/28
12. nano enp0s10/options
1. BOOTPROTO=static
13. nano enp0s10/ipv4address
    1. 35.35.35.1/28
14. systemctl restart network
15. reboot
16. ip route add 10.10.10.0/24 via 100.100.100.10 (можно написать в файл /etc/net/ifaces/enp0s8/ipv4route и рестартнуть network)
17. ip route add 20.20.20.0/24 via 150.150.150.10 (можно написать в файл /etc/net/ifaces/enp0s9/ipv4route и рестартнуть network)
18. nano /etc/nftables/nftables.nft
    1. в начало:
    flush ruleset
    2. в конец:
    table ip nat {
      chain postrouting {
         type nat hook postrouting priority 0;
         oifname enp0s3 (интерфейс в сторону интернета) masquerade
      }
    }
19. systemctl enable --now nftables
20. nft -f /etc/nftables/nftables.nft
21. nano /etc/chrony.conf
    1. в конец пишем:
    server 127.0.0.1
    allow 100.100.100.0/28
    allow 150.150.150.0/28
    allow 35.35.35.0/28
    local stratum 5
22. systemctl restart chronyd
23. После того как настроили днс на srv и web-r, надо указать в /etc/resolv.conf только nameserver 10.10.10.100, вот так (если инета нет, добавьте еще nameserver 8.8.8.8 и nameserver 94.232.137.104, может помочь):
 ![экз_послжн](https://github.com/ErmKaterina/-/assets/109353253/8521027e-6bfe-4894-950f-65e26788be4f)



CLI
- 
    1. hostnamectl set-hostname cli
    2. Пишем “ip a” и “ls /etc/net/ifaces/” проверяем, что для интерфейса ens19 есть директория /etc/net/ifaces/ens19, если нет, то “cp -r /etc/net/ifaces/ens18 /etc/net/ifaces/ens19”
    3. nano /etc/net/ifaces/enp0s3/options
        1. BOOTPROTO=static
    4. nano /etc/net/ifaces/enp0s3/ipv4address
        1. 35.35.35.10/28
    5. nano /etc/net/ifaces/enp0s3/ipv4route
        1. default via 35.35.35.1 (ip адрес isp)
    6. systemctl restart network
    7. reboot
    8. Если “ping 8.8.8.8” идет, а “ping ya.ru” не идет, то в 
    nano /etc/net/ifaces/enp0s3/resolv.conf записываем в начало
    nameserver 94.232.137.104
    9. apt-get update && apt-get install yandex-browser chrony -y
    10. vim /etc/chrony.conf
        1. комментируем (пишем #) перед “pool pool.ntp.org iburst”
        2. в конец пишем “server 35.35.35.1 iburst”
    11. systemctl restart chronyd.service
    12. После того как настроили днс на srv и web-r, надо указать в /etc/resolv.conf только nameserver 10.10.10.100, вот так (если инета нет, добавьте еще nameserver 8.8.8.8 и nameserver 94.232.137.104, может помочь):



RTR-L
1. hostnamectl set-hostname rtr-l
2. cd /etc/net/ifaces
3. vim /etc/net/sysctl.conf 
    1. net.ipv4.ip_forward = 1
4. vim enp0s3/options
    1. BOOTPROTO=static
5. vim enp0s3/ipv4address
    1. 100.100.100.10/28
6. cp -r enp0s3 enp0s8
7. vim enp0s3/ipv4route
    1. default via 100.100.100.1 (ip адрес isp)
8. vim enp0s8/ipv4address
    1. 10.10.10.1/24
9. systemctl restart network
10. reboot
11. Если “ping 8.8.8.8” идет, а “ping ya.ru” не идет, то в 
vim /etc/net/ifaces/enp0s3/resolv.conf записываем в начало
nameserver 94.232.137.104
12. apt-get update && apt-get install chrony nftables nano strongswan -y
13. nano /etc/nftables/nftables.nft
    1. в начало:
    flush ruleset
    2. в конец:
    table ip nat {
      chain postrouting {
         type nat hook postrouting priority 0;
         ip saddr 10.10.10.0/24 oifname enp0s3 (интерфейс в сторону интернета) masquerade;
      }
    }
14. systemctl enable --now nftables
15. nft -f /etc/nftables/nftables.nft
16. ip tunnel add tun0 mode gre local 100.100.100.10 (RTR-L) remote 150.150.150.10 (RTR-R)
17. ip addr add 10.5.5.1/30 dev tun0
18. ip link set up tun0
19. ip route add 20.20.20.0/24 (подсеть правого офиса) via 10.5.5.2
20. nano /etc/strongswan/ipsec.conf
    1. ниже “config setup” пишем:
    conn vpn
                      auto=start
                      type=tunnel
                      authby=secret
                      left=100.100.100.10 (ip адрес RTR-L)
                      right=150.150.150.10 (ip адрес RTR-R)
                      leftsubnet=0.0.0.0/0
                      rightsubnet=0.0.0.0/0
                      leftprotoport=gre
                      rightprotoport=gre
                      ike=aes128-sha256-modp3072
                      esp=aes128-sha256
21. nano /etc/strongswan/ipsec.secrets
    1. 100.100.100.10 150.150.150.10 : PSK “P@ssw0rd”
22. systemctl enable --now ipsec.service
23. После того как настроили днс на srv и web-r, надо указать в /etc/resolv.conf только nameserver 10.10.10.100, вот так (если инета нет, добавьте еще nameserver 8.8.8.8 и nameserver 94.232.137.104, может помочь):





RTR-R

1. hostnamectl set-hostname rtr-r
2. cd /etc/net/ifaces
3. vim /etc/net/sysctl.conf 
    1. net.ipv4.ip_forward = 1
4. vim enp0s3/options
    1. BOOTPROTO=static
5. vim enp0s3/ipv4address
    1. 150.150.150.10/28
6. cp -r enp0s3 enp0s8
7. vim enp0s3/ipv4route
    1. default via 150.150.150.1 (ip адрес isp)
8. vim enp0s8/ipv4address
    1. 20.20.20.1/24
9. systemctl restart network
10. reboot
11. Если “ping 8.8.8.8” идет, а “ping ya.ru” не идет, то в 
vim /etc/net/ifaces/enp0s3/resolv.conf записываем в начало
nameserver 94.232.137.104
12. apt-get update && apt-get install chrony nftables nano strongswan -y\
13. 13. nano /etc/nftables/nftables.nft
    1. в начало:
    flush ruleset
    2. в конец:
    table ip nat {
      chain postrouting {
         type nat hook postrouting priority 0;
         ip saddr 20.20.20.0/24 oifname enp0s3 (интерфейс в сторону интернета) masquerade;
      }
    }
14. systemctl enable --now nftables
15. nft -f /etc/nftables/nftables.nft
16. ip tunnel add tun0 mode gre local 150.150.150.10 (RTR-R) remote 100.100.100.10 (RTR-L)
17. ip addr add 10.5.5.2/30 dev tun0
18. ip link set up tun0
19. ip route add 10.10.10.0/24 (подсеть левого офиса) via 10.5.5.1
20. nano /etc/strongswan/ipsec.conf
    1. ниже “config setup” пишем:
conn vpn
(следующие строки через tab)
                  auto=start
                  type=tunnel
                  authby=secret
                  left=150.150.150.10 (ip адрес RTR-R)
                  right=100.100.100.10 (ip адрес RTR-L)
                  leftsubnet=0.0.0.0/0
                  rightsubnet=0.0.0.0/0
                  leftprotoport=gre
                  rightprotoport=gre
                  ike=aes128-sha256-modp3072
                  esp=aes128-sha256
21. nano /etc/strongswan/ipsec.secrets
    1. 100.100.100.10 150.150.150.10 : PSK “P@ssw0rd”
22. systemctl enable --now ipsec.service
23. После того как настроили днс на srv и web-r, надо указать в /etc/resolv.conf только nameserver 10.10.10.100, вот так (если инета нет, добавьте еще nameserver 8.8.8.8 и nameserver 94.232.137.104, может помочь):

WEB-L
1. hostnamectl set-hostname web-l
2. vim /etc/net/ifaces/enp0s3/options
    1. BOOTPROTO=static
3. vim /etc/net/ifaces/enp0s3/ipv4address
    1. 10.10.10.110/24
4. vim /etc/net/ifaces/enp0s3/ipv4route
    1. default via 10.10.10.1
5. systemctl restart network
6. reboot
7. vim /etc/openssh/banner.txt
    1. Authorized access only
8. vim /etc/openssh/sshd_config
    1. расскоментируем Port 22
    пишем вместо 22 тот порт, который указан в задании (2024)
    2. расскоментируем MaxAuthTries 6
    пишем вместо 6 столько попыток, сколько указано в задании (2)
    3. расскоментируем Banner none
    вместо none пишем путь к banner.txt (/etc/openssh/banner.txt)
4. добавляем в конец AllowUsers sshuser
sshuser - пользователь указан в задании
9. adduser sshuser
10. systemctl restart sshd
11. apt-get update && apt-get install chrony docker-io -y
12. После того как настроили днс на srv и web-r, надо указать в /etc/resolv.conf только nameserver 10.10.10.100, вот так (если инета нет, добавьте еще nameserver 8.8.8.8 и nameserver 94.232.137.104, может помочь):



WEB-R

1. hostnamectl set-hostname web-r
2. vim /etc/net/ifaces/enp0s3/options
    1. BOOTPROTO=static
3. vim /etc/net/ifaces/enp0s3/ipv4address
    1. 20.20.20.100/24
4. vim /etc/net/ifaces/enp0s3/ipv4route
    1. default via 20.20.20.1
5. systemctl restart network
6. reboot
7. vim /etc/openssh/banner.txt
    1. Authorized access only
8. vim /etc/openssh/sshd_config
    1. расскоментируем Port 22
    пишем вместо 22 тот порт, который указан в задании (2024)
    2. расскоментируем MaxAuthTries 6
    пишем вместо 6 столько попыток, сколько указано в задании (2)
    3. расскоментируем Banner none
    вместо none пишем путь к banner.txt (/etc/openssh/banner.txt)
    4. добавляем в конец AllowUsers sshuser
sshuser - пользователь указан в задании
9. adduser sshuser
10. systemctl restart sshd
11. apt-get update && apt-get install chrony bind bind-utils -y
12. После того как настроили днс на srv и web-r, надо указать в /etc/resolv.conf только nameserver 127.0.0.1, (если инета нет, добавьте еще nameserver 8.8.8.8 и nameserver 94.232.137.104, может помочь):


SRV-L
