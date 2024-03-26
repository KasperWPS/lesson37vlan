# Домашнее задание № 24 по теме: "Сетевые пакеты. VLAN. LACP ". К курсу Administrator Linux. Professional

## Задание

В Office1 в тестовой подсети появляются сервера с дополнительными интерфейсами и адресами в internal сети testLAN
- testClient1 - 10.10.10.254
- testClient2 - 10.10.10.254
- testServer1- 10.10.10.1
- testServer2- 10.10.10.1

Развести вланами:
- testClient1 <-> testServer1
- testClient2 <-> testServer2

Между centralRouter и inetRouter
- "Пробросить" 2 линка (общая inernal сеть) и объединить их в бонд
- Проверить работу c отключением интерфейсов

### Выполнение

*Структуру сети из методических рекомендаций изменил. Дополнительно vlan{2,5} могут ходить в Интернет через основной шлюз - inetRouter*

![Network topology](https://github.com/KasperWPS/lesson37vlan/blob/main/topology37.svg)

Тестовый стенд разворачивается готовым к тестам

```bash
vagrant ssh office1Router -c 'ip a | grep "inet "'
```
```
inet 127.0.0.1/8 scope host lo
inet 10.0.2.15/24 brd 10.0.2.255 scope global dynamic noprefixroute eth0
inet 192.168.255.10/30 brd 192.168.255.11 scope global noprefixroute eth1
inet 192.168.56.20/24 brd 192.168.56.255 scope global noprefixroute eth3
inet 10.10.2.10/24 brd 10.10.2.255 scope global noprefixroute eth2.2
inet 10.10.5.10/24 brd 10.10.5.255 scope global noprefixroute eth2.5
```

```bash
vagrant ssh office1Router -c 'ip r'
```
```
default via 192.168.255.9 dev eth1 proto static metric 104
10.0.2.0/24 dev eth0 proto kernel scope link src 10.0.2.15 metric 103
10.10.2.0/24 dev eth2.2 proto kernel scope link src 10.10.2.10 metric 400
10.10.5.0/24 dev eth2.5 proto kernel scope link src 10.10.5.10 metric 401
192.168.56.0/24 dev eth3 proto kernel scope link src 192.168.56.20 metric 102
192.168.255.8/30 dev eth1 proto kernel scope link src 192.168.255.10 metric 104
```

- nftables.conf on office1Router:
```bash
vagrant ssh office1Router -c 'sudo nft list ruleset'
```
```
table inet filter {
        chain input {
                type filter hook input priority filter; policy drop;
                ct state established,related accept
                iif "lo" accept
                ct state new tcp dport 22 accept
                icmp type echo-request accept
                udp dport 33434-33524 counter accept comment "for traceroute"
        }

        chain forward {
                type filter hook forward priority filter; policy drop;
                iif { "eth2.2", "eth2.5" } oif "eth1" accept
                ct state established,related accept
        }
}
table ip nat {
        chain postrouting {
                type nat hook postrouting priority srcnat; policy accept;
                ip saddr 10.10.2.0/24 iif "eth2.2" oif "eth1" masquerade
                ip saddr 10.10.5.0/24 iif "eth2.5" oif "eth1" masquerade
        }
}
```

По умолчанию цепочка forward таблицы filter отбрасывает транзитные пакеты, разрешено движение пакетов только в направлении eth1 от vlan интерфейсов ID{2,5} и для уже установленных соединений, тем самым сети 2,5 между собой не маршрутизируются, однако взаимодействие их с внешним миром выполняется (использован маскарадинг). Проверить это можно так:

```bash
vagrant ssh testClient1 -c 'traceroute 10.10.2.254'
```
```
traceroute to 10.10.2.254 (10.10.2.254), 30 hops max, 60 byte packets
 1  _gateway (10.10.5.10)  0.982 ms  0.963 ms  0.858 ms
 2  * * *
 ...
 30 * * *
```

```bash
vagrant ssh testClient1 -c 'traceroute 192.168.255.1'
```
```
traceroute to 192.168.255.1 (192.168.255.1), 30 hops max, 60 byte packets
 1  _gateway (10.10.5.10)  1.229 ms  1.158 ms  1.076 ms
 2  192.168.255.9 (192.168.255.9)  1.009 ms  0.947 ms  0.893 ms
 3  192.168.255.1 (192.168.255.1)  0.732 ms  1.008 ms  0.955 ms
```

### Проверка LACP

- Инициировать обмен icmp пакетами между testServer2 и inetRouter:
```bash
vagrant ssh testServer2 -c 'ping 192.168.255.1'
```

- Положить один из двух объединенных сетевых интерфейсов (eth1 или eth2)

```bash
ip link set down eth1
```

- После проверки поднять
```bash
ip link set up eth1
```

- Тоже можно повторить и с другим сетевым интерфейсом:

```bash
ip link set down eth2
```

```bash
ip link set up eth2
```

**Во время проверки не должен прерываться обмен пакетами между testServer2 и inetRouter**


Конфигурация стенда (config.json):
```json
[
  {
    "name": "inetRouter",
    "cpus": 1,
    "gui": false,
    "box": "centos/8",
    "private_network":
    [
      { "adapter": 2, "auto_config": false, "virtualbox__intnet": "router-net" },
      { "adapter": 3, "auto_config": false, "virtualbox__intnet": "router-net" },
      { "ip": "192.168.56.10", "adapter": 4 }
    ],
    "memory": "640",
    "no_share": true
  },
  {
    "name": "centralRouter",
    "cpus": 1,
    "gui": false,
    "box": "centos/8",
    "private_network":
    [
      { "adapter": 2, "auto_config": false, "virtualbox__intnet": "router-net" },
      { "adapter": 3, "auto_config": false, "virtualbox__intnet": "router-net" },
      { "ip": "192.168.255.9", "adapter": 4, "netmask": "255.255.255.252", "virtualbox__intnet": "office1-central" },
      { "ip": "192.168.56.11", "adapter": 5, "netmask": "255.255.255.0" }
    ],
    "memory": 640,
    "no_share": true
  },
  {
    "name": "office1Router",
    "cpus": 1,
    "gui": false,
    "box": "centos/8",
    "private_network":
    [
      { "ip": "192.168.255.10", "adapter": 2, "netmask": "255.255.255.252", "virtualbox__intnet": "office1-central" },
      { "adapter": 3, "auto_config": false, "virtualbox__intnet": "vlan" },
      { "ip": "192.168.56.20",  "adapter": 4, "netmask": "255.255.255.0" }
    ],
    "memory": 640,
    "no_share": true
  },
  {
    "name": "testClient1",
    "cpus": 1,
    "gui": false,
    "box": "centos/8",
    "private_network":
    [
      { "ip": "192.168.56.21", "adapter": 2, "netmask": "255.255.255.0" },
      { "adapter": 3, "auto_config": false, "virtualbox__intnet": "vlan" }
    ],
    "memory": "640",
    "no_share": true
  },
  {
    "name": "testServer1",
    "cpus": 1,
    "gui": false,
    "box": "centos/8",
    "private_network":
    [
      { "ip": "192.168.56.22", "adapter": 2, "netmask": "255.255.255.0" },
      { "adapter": 3, "auto_config": false, "virtualbox__intnet": "vlan" }
    ],
    "memory": 640,
    "no_share": true
  },
  {
    "name": "testClient2",
    "cpus": 1,
    "gui": false,
    "box": "ubuntu/focal64",
    "private_network":
    [
      { "ip": "192.168.56.31", "adapter": 2, "netmask": "255.255.255.0" },
      { "adapter": 3, "auto_config": false, "virtualbox__intnet": "vlan" }
    ],
    "memory": "640",
    "no_share": true
  },
  {
    "name": "testServer2",
    "cpus": 1,
    "gui": false,
    "box": "ubuntu/focal64",
    "private_network":
    [
      { "ip": "192.168.56.32", "adapter": 2, "netmask": "255.255.255.0" },
      { "adapter": 3, "auto_config": false, "virtualbox__intnet": "vlan" }
    ],
    "memory": "640",
    "no_share": true
  }
]
```

**Playbook**
```yaml
---
- hosts: all
  become: true
  gather_facts: true

  tasks:
  - name: Accept login with password from sshd
    ansible.builtin.lineinfile:
      path: /etc/ssh/sshd_config
      regexp: '^PasswordAuthentication no$'
      line: 'PasswordAuthentication yes'
      state: present
    notify:
      - Restart sshd

  - name: Set timezone
    community.general.timezone:
      name: Europe/Moscow

  - name: List all files in directory /etc/yum.repos.d/*.repo
    find:
      paths: "/etc/yum.repos.d/"
      patterns: "*.repo"
    register: repos
    when: (ansible_os_family == "RedHat")

  - name: Comment mirrorlist /etc/yum.repos.d/CentOS-*
    ansible.builtin.lineinfile:
      backrefs: true
      path: "{{ item.path }}"
      regexp: '^(mirrorlist=.+)'
      line: '#\1'
    with_items: "{{ repos.files }}"
    when: (ansible_os_family == "RedHat")

  - name: Replace baseurl
    ansible.builtin.lineinfile:
      backrefs: true
      path: "{{ item.path }}"
      regexp: '^#baseurl=http:\/\/mirror.centos.org(.+)'
      line: 'baseurl=http://vault.centos.org\1'
    with_items: "{{ repos.files }}"
    when: (ansible_os_family == "RedHat")

  - name: set up forward packages across routers
    sysctl:
      name: net.ipv4.conf.all.forwarding
      value: '1'
      state: present
    when: "'routers' in group_names"

  - name: Install epel-release
    ansible.builtin.yum:
      name: epel-release
      state: present
    when: (ansible_os_family == "RedHat")

  - name: Install soft on RedHat-based OS
    ansible.builtin.yum:
      name:
        - vim
        - tcpdump
        - traceroute
        - net-tools
      state: present
      update_cache: true
    when: (ansible_os_family == "RedHat")

  - name: Install soft on Debian-based OS
    ansible.builtin.apt:
      name:
        - vim
        - tcpdump
        - traceroute
        - net-tools
      state: present
      update_cache: true
    when: (ansible_os_family == "Debian")

  - name: Set up vlan1 on testClent1 & testServer1 (CentOS)
    ansible.builtin.template:
      src: ifcfg-vlan1.j2
      dest: /etc/sysconfig/network-scripts/ifcfg-vlan1
      owner: root
      group: root
      mode: '0644'
    notify: Restart NetworkManager
    when: (ansible_hostname == 'testClient1' or ansible_hostname == 'testServer1')

  - name: Set up vlan2 on testClient2 & testServer2 (Ubuntu)
    ansible.builtin.template:
      src: 50-cloud-init.yaml.j2
      dest: /etc/netplan/50-cloud-init.yaml
      owner: root
      group: root
      mode: '0644'
    notify: Netplan apply
    when: (ansible_hostname == 'testClient2' or ansible_hostname == 'testServer2')

  - name: Configure eth1 and eth2 interfaces on inetRouter & centralRouter
    ansible.builtin.template:
      src: ifcfg-.j2
      dest: /etc/sysconfig/network-scripts/ifcfg-{{ item }}
      owner: root
      group: root
      mode: '0644'
    with_items:
      - eth1
      - eth2
    notify: Restart NetworkManager
    when: (ansible_hostname == 'inetRouter' or ansible_hostname == 'centralRouter')

  - name: Set up ifcfg-bond0
    ansible.builtin.template:
      src: ifcfg-bond0.j2
      dest: /etc/sysconfig/network-scripts/ifcfg-bond0
      owner: root
      group: root
      mode: '0644'
    notify:
      - Reboot the system
      - Gateway configure on centralRouter
      - Add routes on inetRouter
    when: (ansible_hostname == 'inetRouter' or ansible_hostname == 'centralRouter')

  - name: Disable default route on eth0 interface
    community.general.nmcli:
      conn_name: System eth0
      type: ethernet
      ifname: eth0
      gw4_ignore_auto: true
      state: present
    when: (ansible_hostname != "inetRouter" and ansible_os_family == "RedHat")
    notify:
      - Eth0 connect

  - name: Gateway configure on office1Router
    community.general.nmcli:
      conn_name: System eth1
      type: ethernet
      ifname: eth1
      gw4: "192.168.255.9"
      state: present
    notify:
      - Eth1 connect
    when: (ansible_hostname == "office1Router")

  - name: Copy vlan configs on office1Rouer
    ansible.builtin.copy:
      src: files/office1Router/{{ item }}
      dest: /etc/sysconfig/network-scripts/{{ item }}
    notify: Restart NetworkManager
    with_items:
      - ifcfg-eth2.2
      - ifcfg-eth2.5
    when: (ansible_hostname == "office1Router")

  - name: Copy nftables config on office1Router
    ansible.builtin.copy:
      src: files/office1Router/nftables.conf
      dest: /etc/nftables.conf
    notify:
      - Configure nftables
    when: (ansible_hostname == "office1Router")

  - name: Copy nftables config on inetRouter
    ansible.builtin.copy:
      src: files/inetRouter-nftables.conf
      dest: /etc/nftables.conf
    notify:
      - Configure nftables
    when: (ansible_hostname == "inetRouter")

  handlers:

  - name: Restart sshd
    ansible.builtin.service:
      name: sshd
      state: restarted

  - name: Eth0 connect
    command: nmcli connection up 'System eth0'

  - name: Eth1 connect
    command: nmcli connection up 'System eth1'

  - name: Restart NetworkManager
    ansible.builtin.service:
      name: NetworkManager
      state: restarted

  - name: Netplan apply
    ansible.builtin.command: netplan apply

  - name: Reboot the system
    ansible.builtin.reboot:

  - name: Gateway configure on centralRouter
    community.general.nmcli:
      conn_name: bond0
      type: 'bond'
      mode: "active-backup"
      gw4: "192.168.255.1"
      state: present
    notify:
      - Bond0 connect
    when: (ansible_hostname == "centralRouter")

  - name: Add routes on inetRouter
    ansible.builtin.command: nmcli connection modify bond0 +ipv4.routes "192.168.255.8/30 192.168.255.2"
    notify: Bond0 connect
    when: ansible_hostname == 'inetRouter'

  - name: Bond0 connect
    command: nmcli connection up 'bond0'

  - name: Configure nftables
    ansible.builtin.lineinfile:
      path: /etc/sysconfig/nftables.conf
      line: 'include "/etc/nftables.conf"'
      state: present
    notify:
      - Restart nftables service

  - name: Restart nftables service
    ansible.builtin.service:
      name: nftables
      enabled: true
      state: restarted
```

