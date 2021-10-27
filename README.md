
Для настройки роутеров потребуется Ansible коллекция ansible.posix

```
ansible-galaxy collection install ansible.posix
```

## 1. Настройка роутеров

`Vagrant up` создаст три виртуалки

    ospf1
    ospf2
    ospf3

### * Роутеры объединены разными vlan-ами

#### **ospf1**

|Iface|Link|Address|
|:---:|:---:|---|
|eth1|ospf2 - eth1|192.168.10.1/30|
|eth2|ospf3 - eth2|192.168.30.2/30|

#### **ospf2**

|Iface|Link|Address|
|:---:|:---:|---|
|eth1|ospf1 - eth1|192.168.10.2/30|
|eth2|ospf3 - eth2|192.168.20.1/30|

#### **ospf3**

|Iface|Link|Address|
|:---:|:---:|---|
|eth1|ospf2 - eth1|192.168.20.2/30|
|eth2|ospf1 - eth2|192.168.30.1/30|

### * Настройка ospf между роутерами на базе Quagga;

Установка и настройка ospf роутинга на базе Quagga осуществляется с помощью Ansible

Для корректной работы роутинга, в опциях ядра разрешается форвардинг, и отключается rp_filter на всех интерфейсах:

```
net.ipv4.ip_forward=1
net.ipv4.conf.all.forwarding=1
net.ipv4.conf.all.rp_filter=0
```

После успешной настройки, на роутерах появятся следующие таблицы маршрутизации

<details>
  <summary>ospf1</summary>

    ospf1# sh ip ospf ro
    ============ OSPF network routing table ============
    N    192.168.10.0/30       [10] area: 0.0.0.0
                              directly attached to eth1
    N    192.168.20.0/30       [20] area: 0.0.0.0
                              via 192.168.10.2, eth1
                              via 192.168.30.1, eth2
    N    192.168.30.0/30       [10] area: 0.0.0.0
                              directly attached to eth2

    ============ OSPF router routing table =============
    R    10.0.20.0             [10] area: 0.0.0.0, ASBR
                              via 192.168.10.2, eth1
    R    10.0.30.0             [10] area: 0.0.0.0, ASBR
                              via 192.168.30.1, eth2

    ============ OSPF external routing table ===========
    N E2 0.0.0.0/0             [10/1] tag: 0
                              via 192.168.10.2, eth1
                              via 192.168.30.1, eth2
</details>


<details>
  <summary>ospf2</summary>

    ospf2# sh ip ospf ro
    ============ OSPF network routing table ============
    N    192.168.10.0/30       [10] area: 0.0.0.0
                              directly attached to eth1
    N    192.168.20.0/30       [10] area: 0.0.0.0
                              directly attached to eth2
    N    192.168.30.0/30       [20] area: 0.0.0.0
                              via 192.168.10.1, eth1
                              via 192.168.20.2, eth2

    ============ OSPF router routing table =============
    R    10.0.10.0             [10] area: 0.0.0.0, ASBR
                              via 192.168.10.1, eth1
    R    10.0.30.0             [10] area: 0.0.0.0, ASBR
                              via 192.168.20.2, eth2

    ============ OSPF external routing table ===========
    N E2 0.0.0.0/0             [10/1] tag: 0
                              via 192.168.10.1, eth1
                              via 192.168.20.2, eth2
</details>


<details>
  <summary>ospf3</summary>

    ospf3# sh ip ospf ro
    ============ OSPF network routing table ============
    N    192.168.10.0/30       [20] area: 0.0.0.0
                              via 192.168.20.1, eth1
                              via 192.168.30.2, eth2
    N    192.168.20.0/30       [10] area: 0.0.0.0
                              directly attached to eth1
    N    192.168.30.0/30       [10] area: 0.0.0.0
                              directly attached to eth2

    ============ OSPF router routing table =============
    R    10.0.10.0             [10] area: 0.0.0.0, ASBR
                              via 192.168.30.2, eth2
    R    10.0.20.0             [10] area: 0.0.0.0, ASBR
                              via 192.168.20.1, eth1

    ============ OSPF external routing table ===========
    N E2 0.0.0.0/0             [10/1] tag: 0
                              via 192.168.30.2, eth2
                              via 192.168.20.1, eth1
</details>

## 2. Изобразить ассиметричный роутинг

На роутере ospf2 поднимем "стоимость" линка на интерфейсе eth1

```
[root@ospf2 ~]# vtysh
Hello, this is Quagga (version 0.99.22.4).
Copyright 1996-2005 Kunihiro Ishiguro, et al.
ospf2# conf t
ospf2(config)# int eth1
ospf2(config-if)# ip ospf cost  1000
```

<details>
<summary>В результате маршрут до сети 192.168.10.0/30 был перенаправлен на интерфейс eth2, и будет проходить через роутер ospf3</summary>

    ospf2# sh ip ospf ro
    ============ OSPF network routing table ============
    N    192.168.10.0/30       [30] area: 0.0.0.0
                              via 192.168.20.2, eth2
    N    192.168.20.0/30       [10] area: 0.0.0.0
                              directly attached to eth2
    N    192.168.30.0/30       [20] area: 0.0.0.0
                              via 192.168.20.2, eth2

    ============ OSPF router routing table =============
    R    10.0.10.0             [20] area: 0.0.0.0, ASBR
                              via 192.168.20.2, eth2
    R    10.0.30.0             [10] area: 0.0.0.0, ASBR
                              via 192.168.20.2, eth2

    ============ OSPF external routing table ===========
    N E2 0.0.0.0/0             [10/1] tag: 0
                              via 192.168.20.2, eth2
</details>

## 3. Сделать один из линков "дорогим", но что бы при этом роутинг был симметричным

В дополнение к предыдущему пункту, сделаем "дорогим" линк eth1 на роутере ospf1

```
[root@ospf1 ~]# vtysh
Hello, this is Quagga (version 0.99.22.4).
Copyright 1996-2005 Kunihiro Ishiguro, et al.
ospf1# conf t
ospf1(config)# int eth1
ospf1(config-if)# ip ospf cost  1000
```

И снова проверим таблицы маршрутизации на роутерах ospf1 и ospf2

<details>
<summary>ospf1</summary>

    ospf1# sh ip ospf ro
    ============ OSPF network routing table ============
    N    192.168.10.0/30       [1000] area: 0.0.0.0
                              directly attached to eth1
    N    192.168.20.0/30       [20] area: 0.0.0.0
                              via 192.168.30.1, eth2
    N    192.168.30.0/30       [10] area: 0.0.0.0
                              directly attached to eth2

    ============ OSPF router routing table =============
    R    10.0.20.0             [20] area: 0.0.0.0, ASBR
                              via 192.168.30.1, eth2
    R    10.0.30.0             [10] area: 0.0.0.0, ASBR
                              via 192.168.30.1, eth2

    ============ OSPF external routing table ===========
    N E2 0.0.0.0/0             [10/1] tag: 0
                              via 192.168.30.1, eth2
</details>

<details>
<summary>ospf2</summary>

    ospf2# sh ip ospf ro
    ============ OSPF network routing table ============
    N    192.168.10.0/30       [1000] area: 0.0.0.0
                              directly attached to eth1
    N    192.168.20.0/30       [10] area: 0.0.0.0
                              directly attached to eth2
    N    192.168.30.0/30       [20] area: 0.0.0.0
                              via 192.168.20.2, eth2

    ============ OSPF router routing table =============
    R    10.0.10.0             [20] area: 0.0.0.0, ASBR
                              via 192.168.20.2, eth2
    R    10.0.30.0             [10] area: 0.0.0.0, ASBR
                              via 192.168.20.2, eth2

    ============ OSPF external routing table ===========
    N E2 0.0.0.0/0             [10/1] tag: 0
                              via 192.168.20.2, eth2
</details>

Как видно, маршрутизация снова стала симметричной, пакеты от ospf2 к ospf1(и наоборот) пойдут напрямую, через залинкованные между роутерами интерфейсы.