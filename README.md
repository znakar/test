1. Настройка имен устройств
На каждом устройстве устанавливаем имя и доменное имя:

Для Alt Linux (ISP, HQ-SRV, BR-SRV, HQ-CLI)

hostnamectl set-hostname <имя_устройства>.nexus.star
echo "<имя_устройства>.nexus.star" > /etc/hostname
Пример для HQ-SRV:


hostnamectl set-hostname hq-srv.nexus.star
echo "hq-srv.nexus.star" > /etc/hostname
Для EcoRouter (HQ-RTR, BR-RTR)

configure terminal
hostname <имя_устройства>
exit
write memory
Пример для HQ-RTR:


configure terminal
hostname HQ-RTR
exit
write memory
2. Настройка IPv4 адресов
Используем адреса из RFC1918:

SRV-Net (HQ-SRV, VLAN100) → 192.168.1.0/28

CLI-Net (HQ-CLI, VLAN200) → 192.168.2.0/27

BR-Net (BR-SRV) → 192.168.3.0/26

Management VLAN (VLAN999) → 192.168.10.0/25

HQ-RTR ↔ BR-RTR → GRE/IP-in-IP туннель

Настройка IP для Alt Linux (HQ-SRV, BR-SRV, HQ-CLI)
Редактируем /etc/network/interfaces (или используем nmcli):
Пример для HQ-SRV:


ip addr add 192.168.1.2/28 dev eth0
ip route add default via 192.168.1.1
Для BR-SRV:


ip addr add 192.168.3.2/26 dev eth0
ip route add default via 192.168.3.1
Для HQ-CLI:


ip addr add 192.168.2.2/27 dev eth0
ip route add default via 192.168.2.1
Настройка IP для EcoRouter (HQ-RTR, BR-RTR)

configure terminal
interface GigabitEthernet0/0
ip address 192.168.1.1 255.255.255.240
no shutdown
exit
write memory
3. Настройка NAT на ISP
Добавляем NAT на Alt Linux:

iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
На Cisco (EcoRouter):


configure terminal
interface GigabitEthernet0/1
ip nat inside
exit
interface GigabitEthernet0/0
ip nat outside
exit
access-list 1 permit 192.168.0.0 0.0.255.255
ip nat inside source list 1 interface GigabitEthernet0/0 overload
write memory
4. Создание пользователей
На Alt Linux (HQ-SRV, BR-SRV)

useradd -u 1036 -m matrixmacaw
echo "matrixmacaw:P@ssw0rd" | chpasswd
echo "matrixmacaw ALL=(ALL) NOPASSWD:ALL" >> /etc/sudoers
На EcoRouter (HQ-RTR, BR-RTR)

configure terminal
username vpnpenguin privilege 15 password P@$$word
write memory
5. Настройка SSH
Редактируем /etc/ssh/sshd_config:


Port 2055
PermitRootLogin no
AllowUsers matrixmacaw
MaxAuthTries 2
Banner /etc/issue.net
Добавляем баннер:


echo "Authorized access only" > /etc/issue.net
systemctl restart sshd
6. Настройка GRE туннеля между HQ и BR
На HQ-RTR:

configure terminal
interface Tunnel0
ip address 10.10.10.1 255.255.255.252
tunnel source GigabitEthernet0/1
tunnel destination <BR-RTR_IP>
exit
write memory
На BR-RTR:

configure terminal
interface Tunnel0
ip address 10.10.10.2 255.255.255.252
tunnel source GigabitEthernet0/1
tunnel destination <HQ-RTR_IP>
exit
write memory
7. Настройка динамической маршрутизации (OSPF)

configure terminal
router ospf 1
network 10.10.10.0 0.0.0.3 area 0
network 192.168.1.0 0.0.0.15 area 0
exit
write memory
8. Настройка DHCP на HQ-RTR
bash

configure terminal
ip dhcp excluded-address 192.168.2.1
ip dhcp pool HQ-CLI
 network 192.168.2.0 255.255.255.224
 default-router 192.168.2.1
 dns-server 192.168.1.2
 domain-name nexus.star
exit
write memory
9. Настройка DNS
На HQ-SRV редактируем /etc/named.conf:


zone "nexus.star" IN {
    type master;
    file "/etc/bind/db.nexus.star";
};
Файл /etc/bind/db.nexus.star:


$TTL 86400
@   IN  SOA hq-srv.nexus.star. root.nexus.star. (
        2024033001 ; Serial
        3600       ; Refresh
        1800       ; Retry
        604800     ; Expire
        86400 )    ; Minimum TTL

@       IN  NS  hq-srv.nexus.star.
hq-rtr  IN  A   192.168.1.1
hq-srv  IN  A   192.168.1.2
hq-cli  IN  A   192.168.2.2
br-srv  IN  A   192.168.3.2
moodle  IN  CNAME hq-rtr
wiki    IN  CNAME hq-rtr
Запускаем:


systemctl restart named
10. Настройка часового пояса
bash
Копировать
Редактировать
timedatectl set-timezone Europe/Moscow
