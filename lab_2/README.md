# Ход работы

Сначала я склонировал CHR, потом арендовал виртуальный сервер на том же Hostkey. В общем то первый CHR соединить с новой машиной было несложно, вторую пришлось настраивать, как и в первой лабораторной. 

В целом потребовалось всё то же самое: создание интерфейса, пира, настройка фаерволла.

```
/interface wireguard peers remove [find]
/ip address remove [find where interface~"wg"]
/interface wireguard remove [find]
/interface wireguard add name=wg-vps listen-port=51820
/interface wireguard print detail
/ip address add address=10.20.0.2/24 interface=wg-vps
/interface wireguard peers add \
    interface=wg-vps \
    public-key="G1Xr3ClRFILqOW4p3HaHH8VPoTdtjSm3u77DKC3pc0U=" \
    endpoint-address=91.210.106.55 \
    endpoint-port=51820 \
    allowed-address=10.20.0.1/32 \
    persistent-keepalive=25
/ip firewall filter add chain=input action=accept protocol=udp dst-port=51820 comment="allow WG"
```

IP у CHR2 10.20.0.3, также нужно добавить адрес CHR2 в allowed-address, чтобы могли пинговать друг друга:
```
/interface wireguard peers set [find] allowed-address=10.20.0.1/32,10.20.0.3/32
```

В vps:
```
iptables -A FORWARD -i wg0 -j ACCEPT
iptables -A FORWARD -o wg0 -j ACCEPT
```


проверка пинга: 

<img width="862" height="351" alt="image" src="https://github.com/user-attachments/assets/1fc98e05-e0f3-4b7e-9683-7263b88ccc86" />

<img width="868" height="339" alt="image" src="https://github.com/user-attachments/assets/e6d4828c-8f03-4d47-b36e-1ad866c52548" />

<img width="845" height="341" alt="image" src="https://github.com/user-attachments/assets/b62a605e-78ff-4aa0-a742-0925070333c8" />

Установка необходимых пакетов:

```
ansible-galaxy collection install community.routeros
ansible-galaxy collection install ansible.netcommon
```

Инвентори: 
```
chr1 ansible_host=10.20.0.2 router_id=1.1.1.1 ospf_peer_ip=10.20.0.3 ansible_command_timeout=120
chr2 ansible_host=10.20.0.3 router_id=2.2.2.2 ospf_peer_ip=10.20.0.2 ansible_command_timeout=120

[routers]
chr1 ansible_host=10.20.0.2 router_id=1.1.1.1 ospf_peer_ip=10.20.0.3
chr2 ansible_host=10.20.0.3 router_id=2.2.2.2 ospf_peer_ip=10.20.0.2

[routers:vars]
ansible_connection=ansible.netcommon.network_cli
ansible_network_os=community.routeros.routeros
ansible_user=ansible
ansible_password="123"
ansible_ssh_common_args='-o StrictHostKeyChecking=no'
```

Оба устройства объединены в группу routers, к которой применяются общие параметры подключения: Ansible работает по SSH через network_cli, используя драйвер для RouterOS. Также заданы логин и пароль и отключена строгая проверка SSH-ключей для упрощения подключения.

Плейбук: 
```
---
- name: lab2
  hosts: routers
  gather_facts: false

  tasks:

    - name: Create local configs dir
      delegate_to: localhost
      ansible.builtin.file:
        path: ./configs
        state: directory

    - name: Set login, identity and NTP
      community.routeros.command:
        commands:
          - /user add name=ansible group=full password=123
          - /system identity set name={{ inventory_hostname }}
          - /system ntp client set enabled=yes servers=pool.ntp.org

    - name: Configure OVPN client (chr2 only)
      community.routeros.command:
        commands:
          - /interface ovpn-client add name=ovpn-out1 connect-to=10.20.0.1 port=1194 user=ovpnuser password=ovpnpass disabled=no
      when: inventory_hostname == "chr2"

    - name: Configure OSPF (chr2 only)
      community.routeros.command:
        commands:
          - /routing ospf instance add name=ospf1 router-id={{ router_id }} disabled=no
          - /routing ospf area add name=backbone area-id=0.0.0.0 instance=ospf1
          - /routing ospf interface-template add interfaces=ovpn-out1 area=backbone network-type=point-to-point
      when: inventory_hostname == "chr2"

    - name: Gather RouterOS facts
      community.routeros.facts:
      register: ros_facts

    - name: Collect OSPF and system data
      community.routeros.command:
        commands:
          - /system identity print
          - /ip address print
          - /routing ospf neighbor print
          - /routing ospf instance print
          - /routing ospf interface-template print
      register: ros_out
      vars:
        ansible_command_timeout: 180

    - name: Save config
      delegate_to: localhost
      ansible.builtin.copy:
        content: |
          {{ ros_out.stdout | join("\n\n") }}
        dest: "./configs/{{ inventory_hostname }}_config.txt"

    - name: Save OSPF info
      delegate_to: localhost
      ansible.builtin.copy:
        content: |
          NEIGHBORS:
          {{ ros_out.stdout[2] }}

          INSTANCE:
          {{ ros_out.stdout[3] }}

          INTERFACES:
          {{ ros_out.stdout[4] }}
        dest: "./configs/{{ inventory_hostname }}_ospf.txt"

    - name: Save facts
      delegate_to: localhost
      ansible.builtin.copy:
        content: "{{ ros_facts | to_nice_json }}"
        dest: "./configs/{{ inventory_hostname }}_facts.json"
```

Сначала на локальной машине создаётся папка configs, куда будут сохраняться результаты. Затем на каждом роутере создаётся пользователь ansible, задаётся имя устройства (hostname) и включается NTP-синхронизация времени.

Далее идут задачи, которые применяются только к chr2: на нём настраивается OVPN-клиент для подключения к серверу и поднимается OSPF — создаётся инстанс, backbone-зона и шаблон интерфейса для установления соседства через VPN.

После настройки playbook собирает информацию с устройств: общие факты RouterOS, IP-адреса, состояние OSPF (соседи, инстансы, интерфейсы). Эти данные сохраняются на локальной машине в виде текстовых файлов и JSON — отдельно полный конфиг, отдельно информация по OSPF и отдельно факты.

Запуск автоматической настройки и сбора информации:

```
ansible-playbook -i inventory.ini lab.yml
```

<img width="1441" height="1047" alt="image" src="https://github.com/user-attachments/assets/3a2b8467-7999-4cc7-b179-14f7eae441da" />

Проверка Instance:

```
/routing ospf instance print
```

<img width="745" height="81" alt="image" src="https://github.com/user-attachments/assets/3ba0c134-507d-46db-976b-97ed98cc84e8" />

<img width="769" height="130" alt="image" src="https://github.com/user-attachments/assets/a5bf2b56-45b2-4c46-b21d-86e86203d0d3" />

Проверка NTP Client:

```
/system ntp client print
```

<img width="504" height="183" alt="image" src="https://github.com/user-attachments/assets/2ea3428c-6795-4cbc-9933-f671b2eb2a5c" />

<img width="538" height="248" alt="image" src="https://github.com/user-attachments/assets/012e049d-3177-4880-b49f-35734eefe560" />

Созданные конфиги:

<img width="1244" height="95" alt="image" src="https://github.com/user-attachments/assets/d550cc61-8778-4ced-8a9a-6a288886dc77" />


Пример:

<img width="1633" height="641" alt="image" src="https://github.com/user-attachments/assets/3c2ec8c5-c988-49f8-8541-6c0bdce0558d" />

