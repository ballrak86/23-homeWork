# 23-homeWork. BIND and Split-DNS
## Описание файлов в директории
logFileFull.log - полный лог выполнения  

Vagrant_folder - все что понадобится для поднятия ВМ и краткое описание файлов в ней  
Vagrantfile - вагрант файл  
provisioning - директория с файлами провижинга  
```
provisioning/
├── client-motd
├── client-resolv.conf
├── master-named.conf
├── named.ddns.lab
├── named.dns.lab
├── named.dns.lab.client
├── named.dns.lab.rev
├── named.newdns.lab
├── named.zonetransfer.key
├── playbook.yml
├── rndc.conf
├── servers-resolv.conf
├── slave-named.conf
└── zonetransfer.key
```

## Описание как запустить виртуальную машину (кратко)
```
vagrant up
vagrant ssh client
[vagrant@client ~]$ nslookup www.newdns.lab
[vagrant@client ~]$ nslookup web1.dns.lab
[vagrant@client ~]$ nslookup web2.dns.lab

vagrant ssh client2
[vagrant@client2 ~]$ nslookup www.newdns.lab
[vagrant@client2 ~]$ nslookup web1.dns.lab
[vagrant@client2 ~]$ nslookup web2.dns.lab
```

## краткое описание конфигурационных файлов
### provisioning/master-named.conf
Клиент может попасть во view основываясь на адресе источника, адресе назначения или dns TSIG-ключе. Я выбрать отслеживание по адресу источника, и указал в файле 
```
acl client { 192.168.50.15/32; };
acl client2 { 192.168.50.16/32; };
```
Сделал view для client и client2. Для client, в зоне dns.lab, создал новый файл и убрал оттуда имя web2. Для всех остальных создал view default, с включением всех зон.
```
view "client" {
  match-clients { client; };

  zone "dns.lab" {
      type master;
      allow-transfer { key "zonetransfer.key"; };
      file "/etc/named/named.dns.lab.client";
  };

  // lab's newdns zone
  zone "newdns.lab" {
      type master;
      allow-transfer { key "zonetransfer.key"; };
      allow-update { key "zonetransfer.key"; };
      file "/etc/named/named.newdns.lab";
  };

};

view "client2" {
  match-clients { client2; };

  zone "dns.lab" {
      type master;
      allow-transfer { key "zonetransfer.key"; };
      file "/etc/named/named.dns.lab";
  };

  // lab's zone reverse
  zone "50.168.192.in-addr.arpa" {
      type master;
      allow-transfer { key "zonetransfer.key"; };
      file "/etc/named/named.dns.lab.rev";
  };

};
```
### provisioning/slave-named.conf
В этом файле указано все тоже самое только тип зон slave, и указан адрес master
```
  zone "dns.lab" {
      type slave;
      masters { 192.168.50.10; };
      file "/etc/named/named.dns.lab.client";
  };
```

## Логи
### Выполним запросы на client
Ответ на запрос web2.dns.lab мы ничего не получили, так как файле зоны для client его нет. Так же проверяем ответ со slave.
```
[vagrant@client ~]$ nslookup www.newdns.lab
Server:         192.168.50.10
Address:        192.168.50.10#53

Name:   www.newdns.lab
Address: 192.168.50.15
Name:   www.newdns.lab
Address: 192.168.50.16

[vagrant@client ~]$ nslookup web1.dns.lab
Server:         192.168.50.10
Address:        192.168.50.10#53

Name:   web1.dns.lab
Address: 192.168.50.15

[vagrant@client ~]$ nslookup web2.dns.lab
Server:         192.168.50.10
Address:        192.168.50.10#53

** server can't find web2.dns.lab: NXDOMAIN

[vagrant@client ~]$ nslookup www.newdns.lab 192.168.50.11
Server:         192.168.50.11
Address:        192.168.50.11#53

Name:   www.newdns.lab
Address: 192.168.50.16
Name:   www.newdns.lab
Address: 192.168.50.15
```
### Выполним запросы на client2
Ответ www.newdns.lab пуст, зоны для client2 нет
```
[vagrant@client2 ~]$ nslookup www.newdns.lab
Server:         192.168.50.10
Address:        192.168.50.10#53

** server can't find www.newdns.lab: NXDOMAIN

[vagrant@client2 ~]$ nslookup web1.dns.lab
Server:         192.168.50.10
Address:        192.168.50.10#53

Name:   web1.dns.lab
Address: 192.168.50.15

[vagrant@client2 ~]$ nslookup web2.dns.lab
Server:         192.168.50.10
Address:        192.168.50.10#53

Name:   web2.dns.lab
Address: 192.168.50.16
```

📚Домашнее задание/проектная работа разработано(-на) для курса ["Administrator Linux. Professional"](https://otus.ru/lessons/linux-professional/)
