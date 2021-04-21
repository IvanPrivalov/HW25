# OpenVPN

## Задание

Задание

- Между двумя виртуалками поднять vpn в режимах
  * tun
  * tap Прочуствовать разницу.

- Поднять RAS на базе OpenVPN с клиентскими сертификатами, подключиться с локальной машины на виртуалку


## Выполнение ДЗ

Копируем файлы в каталог и запускаем Vagrantfile:

```shell
vagrant up
```

## Между двумя виртуалками поднять vpn в режимах - tun - tap Прочуствовать разницу.

### Проверка:

1. Проверим наличие маршрутов до лупбеков и доступность лупбеков:

```shell
otus@otus-VirtualBox:~/Desktop/HW25$ vagrant ssh ovpn-server
Last login: Wed Apr 21 08:57:09 2021 from 192.168.10.1
[vagrant@ovpn-server ~]$ sudo -i
[root@ovpn-server ~]# ip r
default via 10.0.2.2 dev eth0 proto dhcp metric 100 
10.0.0.2 via 172.16.10.2 dev tap0 
10.0.2.0/24 dev eth0 proto kernel scope link src 10.0.2.15 metric 100 
172.16.10.0/24 dev tap0 proto kernel scope link src 172.16.10.1 
192.168.10.0/24 dev eth1 proto kernel scope link src 192.168.10.10 metric 101 
[root@ovpn-server ~]# ping -I 10.0.0.1 10.0.0.2
PING 10.0.0.2 (10.0.0.2) from 10.0.0.1 : 56(84) bytes of data.
64 bytes from 10.0.0.2: icmp_seq=1 ttl=64 time=5.49 ms
64 bytes from 10.0.0.2: icmp_seq=2 ttl=64 time=1.27 ms
64 bytes from 10.0.0.2: icmp_seq=3 ttl=64 time=0.979 ms
64 bytes from 10.0.0.2: icmp_seq=4 ttl=64 time=0.798 ms
^C
--- 10.0.0.2 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3005ms
rtt min/avg/max/mdev = 0.798/2.136/5.490/1.944 ms
[root@ovpn-server ~]# 
```

```shell
otus@otus-VirtualBox:~/Desktop/HW25$ vagrant ssh ovpn-client
Last login: Wed Apr 21 09:00:53 2021 from 192.168.10.1
[vagrant@ovpn-client ~]$ sudo -i
[root@ovpn-client ~]# ip r
default via 10.0.2.2 dev eth0 proto dhcp metric 100 
10.0.0.1 via 172.16.10.1 dev tap0 
10.0.2.0/24 dev eth0 proto kernel scope link src 10.0.2.15 metric 100 
172.16.10.0/24 dev tap0 proto kernel scope link src 172.16.10.2 
192.168.10.0/24 dev eth1 proto kernel scope link src 192.168.10.20 metric 101 
[root@ovpn-client ~]# ping -I 10.0.0.2 10.0.0.1
PING 10.0.0.1 (10.0.0.1) from 10.0.0.2 : 56(84) bytes of data.
64 bytes from 10.0.0.1: icmp_seq=1 ttl=64 time=1.55 ms
64 bytes from 10.0.0.1: icmp_seq=2 ttl=64 time=1.36 ms
64 bytes from 10.0.0.1: icmp_seq=3 ttl=64 time=0.807 ms
64 bytes from 10.0.0.1: icmp_seq=4 ttl=64 time=1.03 ms
^C
--- 10.0.0.1 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3012ms
rtt min/avg/max/mdev = 0.807/1.191/1.558/0.291 ms
[root@ovpn-client ~]# 
```

2. Измерим скорость в туннеле. 

Для этого запускаем iperf3 на сервере в режиме сервера:

```shell
[root@ovpn-server ~]# iperf3 -s
```

На клиенте в режиме клиента запускаем тест:

```shell
[root@ovpn-client ~]# iperf3 -c 172.16.10.1 -t 40 -i 5
```

Получаем результат:

```shell
[ ID] Interval           Transfer     Bandwidth       Retr
[  4]   0.00-40.01  sec  1.10 GBytes   236 Mbits/sec  749             sender
[  4]   0.00-40.01  sec  1.10 GBytes   235 Mbits/sec                  receiver
```

Изменим в конфигах /etc/openvpn/server.conf на сервере и клиенте режим с tap на tun:

Для изменения режима open-vpn необходимо в ./inventories/host_vars/ovpn-server.yml и в ./inventories/host_vars/ovpn-client.yml изменить значение переменной stsdev на 'tun', после чего выполнить плейбук site-to-site.yml.

Рестартанем сервис openvpn и измерим скорость снова. Получим следующие результаты:

```shell
[ ID] Interval           Transfer     Bandwidth       Retr
[  4]   0.00-40.00  sec  1.13 GBytes   243 Mbits/sec  615             sender
[  4]   0.00-40.00  sec  1.13 GBytes   242 Mbits/sec                  receiver
```

В лабораторной среде результаты в режиме tun лучше. Поэтому режим tap рекомендуется использовать в специфических задачах, например, если в архитектуре сети необходимо достичь связности по L2, в остальных же случаях нужно использовать режим tun.

## Поднять RAS на базе OpenVPN с клиентскими сертификатами.

Для изменения конфигурации стенда на remote-access необходимо выполнить плейбук remote-access.yml, после чего на хосте ovpn-server поменяется server.conf, openvpn будет слушать на нестандартном порту 1195, а хост ovpn-client подключится к серверу с помощью конфига client.conf.

Конфигурационный файл сервера:

```shell
[root@ovpn-server ~]# cat /etc/openvpn/server.conf
port 1195
proto udp
dev tun
ca /etc/openvpn/pki/ca.crt
cert /etc/openvpn/pki/issued/server.crt
key /etc/openvpn/pki/private/server.key
dh /etc/openvpn/pki/dh.pem
server 172.16.10.0 255.255.255.0
route 10.0.0.2 255.255.255.255
push "route 10.0.0.1 255.255.255.255"
client-to-client
client-config-dir /etc/openvpn/client
keepalive 10 120
compress lzo
persist-key
persist-tun
status /var/log/openvpn-status.log
log /var/log/openvpn.log
verb 3
```

Конфигурационный файл клиента:

```shell
[root@ovpn-client ~]# cat /etc/openvpn/client.conf
dev tun
proto udp
remote 192.168.10.10 1195
client
resolv-retry infinite
ca ./ca.crt
cert ./client.crt
key ./client.key
compress lzo
persist-key
persist-tun
status /var/log/openvpn-client-status.log
log-append /var/log/openvpn-client.log
verb 3
```

Подключаемся к серверу с клиента:

```shell
[root@ovpn-client ~]# openvpn --config /etc/openvpn/client.conf
```

Проверим доступность лупбэка на машине ovpn-server:

```shell
[root@ovpn-client ~]# ping 10.0.0.1
PING 10.0.0.1 (10.0.0.1) 56(84) bytes of data.
64 bytes from 10.0.0.1: icmp_seq=1 ttl=64 time=1.72 ms
64 bytes from 10.0.0.1: icmp_seq=2 ttl=64 time=1.20 ms
64 bytes from 10.0.0.1: icmp_seq=3 ttl=64 time=1.31 ms
64 bytes from 10.0.0.1: icmp_seq=4 ttl=64 time=1.09 ms
```


