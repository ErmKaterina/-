ISP

1. hostnamectl set-hostname isp
2. apt-get update && apt-get install chrony nftables -y
3. vim /etc/net/sysctl.conf 
    1. net.ipv4.ip_forward = 1
4. cd /etc/net/ifaces
5. cp -r enp0s3/ enp0s8
6. cp -r enp0s3/ enp0s9
7. cp -r enp0s3/ enp0s10
8. vim enp0s8/options
    1. BOOTPROTO=static
9. vim enp0s8/ipv4address
    1. 100.100.100.1/28
10. vim enp0s9/options
    1. BOOTPROTO=static
11. vim enp0s9/ipv4address
    1. 150.150.150.1/28
12. vim enp0s10/options
1. BOOTPROTO=static
13. vim enp0s10/ipv4address
    1. 35.35.35.1/28
14. systemctl restart network
15. reboot

--------------------------------------------------------------------
16. ip route add 10.10.10.0/24 via 100.100.100.10 (можно написать в файл /etc/net/ifaces/enp0s8/ipv4route и рестартнуть network)
17. ip route add 20.20.20.0/24 via 150.150.150.10 (можно написать в файл /etc/net/ifaces/enp0s9/ipv4route и рестартнуть network)
18. vim /etc/nftables/nftables.nft
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

    --------------------
22. vim /etc/chrony.conf
    1. в конец пишем:
    server 127.0.0.1
    allow 100.100.100.0/28
    allow 150.150.150.0/28
    allow 35.35.35.0/28
    local stratum 5
23. systemctl restart chronyd

    ----------------------
25. После того как настроили днс на srv и web-r, надо указать в /etc/resolv.conf только nameserver 10.10.10.100, вот так (если инета нет, добавьте еще nameserver 8.8.8.8 и nameserver 94.232.137.104, может помочь):
 ![экз_послжн](https://github.com/ErmKaterina/-/assets/109353253/8521027e-6bfe-4894-950f-65e26788be4f)



CLI
- 
    1. hostnamectl set-hostname cli
    2. “ip a” и “ls /etc/net/ifaces/” проверяем, что для интерфейса ens19 есть директория /etc/net/ifaces/ens19, если нет, то “cp -r /etc/net/ifaces/ens18 /etc/net/ifaces/ens19”
    3. vim /etc/net/ifaces/enp0s3/options
        1. BOOTPROTO=static
    4. vim /etc/net/ifaces/enp0s3/ipv4address
        1. 35.35.35.10/28
    5. vim /etc/net/ifaces/enp0s3/ipv4route
        1. default via 35.35.35.1 (ip адрес isp)
    6. systemctl restart network
    7. reboot
    8. Если “ping 8.8.8.8” идет, а “ping ya.ru” не идет, то в 
    vim /etc/net/ifaces/enp0s3/resolv.conf записываем в начало
    nameserver 94.232.137.104
    9. apt-get update && apt-get install yandex-browser chrony -y
    запустить НЕ от рута с помощью команды:
yandex-browser-stable

запустить от рута с помощью команды:
yandex-browser-stable --no-sandbox
    

    ------------------------
    10. vim /etc/chrony.conf
        1. # “pool pool.ntp.org iburst”
        2. в конец “server 35.35.35.1 iburst”
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
12. apt-get update && apt-get install chrony nftables strongswan -y

-----------------------------------
14. vim /etc/nftables/nftables.nft
    1. в начало:
    flush ruleset
    2. в конец:
    table ip nat {
      chain postrouting {
         type nat hook postrouting priority 0;
         ip saddr 10.10.10.0/24 oifname enp0s3 (интерфейс в сторону интернета) masquerade;
      }
    }
15. systemctl enable --now nftables
16. nft -f /etc/nftables/nftables.nft
17. ip tunnel add tun0 mode gre local 100.100.100.10 (RTR-L) remote 150.150.150.10 (RTR-R)
18. ip addr add 10.5.5.1/30 dev tun0
19. ip link set up tun0
20. ip route add 20.20.20.0/24 (подсеть правого офиса) via 10.5.5.2
21. vim /etc/strongswan/ipsec.conf
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
22. vim /etc/strongswan/ipsec.secrets
    1. 100.100.100.10 150.150.150.10 : PSK “P@ssw0rd”
23. systemctl enable --now ipsec.service
24. -------------------------------
25. vim /etc/chrony.conf
        1. # “pool pool.ntp.org iburst”
        2. в конец server 100.100.100.1 iburst
26. После того как настроили днс на srv и web-r, надо указать в /etc/resolv.conf только nameserver 10.10.10.100, вот так (если инета нет, добавьте еще nameserver 8.8.8.8 и nameserver 94.232.137.104, может помочь):





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
12. apt-get update && apt-get install chrony nftables strongswan -y

13. ------------------------------
14. 13. vim /etc/nftables/nftables.nft
    1. в начало:
    flush ruleset
    2. в конец:
    table ip nat {
      chain postrouting {
         type nat hook postrouting priority 0;
         ip saddr 20.20.20.0/24 oifname enp0s3 (интерфейс в сторону интернета) masquerade;
      }
    }
15. systemctl enable --now nftables
16. nft -f /etc/nftables/nftables.nft
17. ip tunnel add tun0 mode gre local 150.150.150.10 (RTR-R) remote 100.100.100.10 (RTR-L)
18. ip addr add 10.5.5.2/30 dev tun0
19. ip link set up tun0
20. ip route add 10.10.10.0/24 (подсеть левого офиса) via 10.5.5.1
21. vim /etc/strongswan/ipsec.conf
    1. ниже “config setup” пишем:
conn vpn
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
22. vim /etc/strongswan/ipsec.secrets
    1. 100.100.100.10 150.150.150.10 : PSK “P@ssw0rd”
23. systemctl enable --now ipsec.service
24. ---------------------------------------------
25. 12. vim /etc/chrony.conf
        1. .# “pool pool.ntp.org iburst”
        2. в конец server 150.150.150.1 iburst
26. После того как настроили днс на srv и web-r, надо указать в /etc/resolv.conf только nameserver 10.10.10.100, вот так (если инета нет, добавьте еще nameserver 8.8.8.8 и nameserver 94.232.137.104, может помочь):

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
7. apt-get update && apt-get install chrony docker-io -y
   ----------------------------
9. vim /etc/openssh/banner.txt
    1. Authorized access only
10. vim /etc/openssh/sshd_config
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
11. -----------------------
12. vim /etc/chrony.conf
        1. # “pool pool.ntp.org iburst”
        2. в конец server 100.100.100.1 iburst
13. После того как настроили днс на srv и web-r, надо указать в /etc/resolv.conf только nameserver 10.10.10.100, вот так (если инета нет, добавьте еще nameserver 8.8.8.8 и nameserver 94.232.137.104, может помочь):



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
7.  apt-get update && apt-get install chrony bind bind-utils -y
8.  ----------------------------------
9. vim /etc/openssh/banner.txt
    1. Authorized access only
10. vim /etc/openssh/sshd_config
    1. расскоментируем Port 22
    пишем вместо 22 тот порт, который указан в задании (2024)
    2. расскоментируем MaxAuthTries 6
    пишем вместо 6 столько попыток, сколько указано в задании (2)
    3. расскоментируем Banner none
    вместо none пишем путь к banner.txt (/etc/openssh/banner.txt)
    4. добавляем в конец AllowUsers sshuser
sshuser - пользователь указан в задании
11. adduser sshuser
12. systemctl restart sshd
12-----------------------
13. vim /etc/chrony.conf
        1. # “pool pool.ntp.org iburst”
        2. в конец server 150.150.150.1 iburst
14. После того как настроили днс на srv и web-r, надо указать в /etc/resolv.conf только nameserver 127.0.0.1, (если инета нет, добавьте еще nameserver 8.8.8.8 и nameserver 94.232.137.104, может помочь):


SRV-L

1. hostnamectl set-hostname srv-l
2. vim /etc/net/ifaces/enp0s3/options
    1. BOOTPROTO=static
3. vim /etc/net/ifaces/enp0s3/ipv4address
    1. 10.10.10.100/24
4. vim /etc/net/ifaces/enp0s3/ipv4route
    1. default via 10.10.10.1
5. systemctl restart network
6. reboot
7. apt-get update && apt-get install bind bind-utils chrony -y

8. ---------------------------------
9. vim /etc/chrony.conf
    1. .# “pool pool.ntp.org iburst”
    2. в конец  “server 100.100.100.1 iburst”
10. vim /etc/bind/options.conf
    1. добавляем в options:
    listen-on { any; };
    forwarders { 94.232.137.104; };
    dnssec-validation no;
    recursion yes;
    allow-query { any; };
    allow-recursion { any; };
и убираем листен 127.0.0.1 ниже
![1](https://github.com/ErmKaterina/-/assets/109353253/ce8b63b4-4cc6-4587-aa57-b57714f6e384)

11. vim /etc/bind/rfc1912.conf
    1. добавляем в конец:
    zone "au.team" {
                    type master;
                    file "au.team";
                    allow-transfer {20.20.20.100;};
    };
    zone "10.10.10.in-addr.arpa" {
                    type master;
                    file "left.reverse";
                    allow-transfer {20.20.20.100;};
    };
    zone "20.20.20. in-addr.arpa" {
                    type master;
                    file "right. reverse";
                    allow-transfer {20.20.20.100;};
    };
    zone "35.35.35. in-addr.arpa" {
                    type master;
                    file "cli. reverse";
                    allow-transfer {20.20.20.100;};
при настройки вебр меняем на айпи срв

![2](https://github.com/ErmKaterina/-/assets/109353253/c9b0796d-f03b-4f8d-97d5-fc5634cb93ee)


11. cd /etc/bind/zone/
12. cp localhost au.team
13. vim au.team
    1. заменяем localhost. на au.team.  и root.localhost. на root.au.team.
    2. редачим зоны, должны получиться такие зоны:
    @(Tab)IN(Tab)NS(Tab)au.team.
    @(Tab)IN(Tab)NS(Tab)10.10.10.100
    isp(Tab)IN(Tab)A(Tab)100.100.100.1
    rtr-l(Tab)IN(Tab)A(Tab)10.10.10.1
    rtr-r(Tab)IN(Tab)A(Tab)20.20.20.1
    web-l(Tab)IN(Tab)A(Tab)10.10.10.110
    web-r(Tab)IN(Tab)A(Tab)10.10.10.100
    cli(Tab)IN(Tab)A(Tab)35.35.35.10
    dns(Tab)IN(Tab)CNAME(Tab)srv-l
    mediawiki(Tab)IN(Tab)CNAME(Tab)web-r
    3. должно получиться так
   
![3](https://github.com/ErmKaterina/-/assets/109353253/0ff730d6-6c01-45aa-8dad-968fcadaff85)

       
   14. cp localhost right.reverse
15. vim right.reverse
    1. заменяем localhost. на 20.20.20.in-addr.arpa. и root.localhost. на root.20.20.20.in-addr.arpa.
    2. редачим зоны, должны получиться такие зоны:
    @(Tab)IN(Tab)NS(Tab)au.team.
    @(Tab)IN(Tab)NS(Tab)20.20.20.100
    1(Tab)PTR(Tab)rtr-r.au.team.
    100(Tab)PTR(Tab)web-r.au.team.
    3. должно получиться так

![4](https://github.com/ErmKaterina/-/assets/109353253/c05b0add-bdc4-483c-bf61-52a5a4fe789a)

16. cp right.reverse left.reverse
17. vim left.reverse
    1. заменяем 20.20.20.in-addr.arpa. на 10.10.10.in-addr.arpa. и root.localhost. на root.10.10.10.in-addr.arpa.
    2. @(Tab)IN(Tab)NS(Tab)au.team.
    @(Tab)IN(Tab)NS(Tab)10.10.10.100
    1(Tab)PTR(Tab)rtr-l.au.team.
    100(Tab)PTR(Tab)srv-l.au.team.
    110(Tab)PTR(Tab)web-l.au.team.

       ![5](https://github.com/ErmKaterina/-/assets/109353253/b642be96-e626-47d8-b9ff-f4e2b144f57e)

18. cp right.reverse cli.reverse
19. vim left.reverse
    1. заменяем 10.10.10.in-addr.arpa. на 35.35.35.in-addr.arpa. и root.localhost. на root.35.35.35.in-addr.arpa.
   
    ![6](https://github.com/ErmKaterina/-/assets/109353253/c9cf5177-8ac1-46dc-9312-dd24599bd37c)

    
    
    20. chmod 777 au.team
21. chmod 777 right.reverse
22. chmod 777 left.reverse
23. chmod 777 cli.reverse
24. systemctl restart bind
25. После того как настроили днс на srv и web-r, надо указать в /etc/resolv.conf только nameserver 127.0.0.1 (если инета нет, добавьте еще nameserver 8.8.8.8 и nameserver 94.232.137.104, может помочь)

