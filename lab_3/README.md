# Ход работы

Перед началом выполнения были установлены следующие пакеты: 

```
`ansible-galaxy collection install netbox.netbox`
`ansible-galaxy collection install community.routeros`
`ansible-galaxy collection install ansible.netcommon`
```

Далее нужно было поднять веб интерфейс Нетбокс для создания апи токена и не только.

Для того, чтобы его запустить необходимо выполнить следующие команды: 

```
sudo apt update
sudo apt install -y git python3-venv python3-pip redis-server postgresql
git clone https://github.com/netbox-community/netbox.git
sudo mkdir -p /opt
cd /opt
sudo git clone https://github.com/netbox-community/netbox.git
cd netbox
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
cp netbox/netbox/configuration_example.py netbox/netbox/configuration.py
nano netbox/netbox/configuration.py
```

В configuration.py ALLOWED_HOSTS = [''] поменять на ALLOWED_HOSTS = ['*'],

```
DATABASE = {
    'NAME': 'netbox',
    'USER': 'netbox',
    'PASSWORD': '192856',
}
```

а также в постгре создать бд:

```
CREATE DATABASE netbox;
CREATE USER netbox WITH PASSWORD '192856';
ALTER DATABASE netbox OWNER TO netbox;
\q
```

И первый запуск веб интерфейса: 

```
python3 netbox/manage.py migrate
python3 netbox/manage.py createsuperuser
python3 netbox/manage.py runserver 0.0.0.0:8000
```
После спокойно авторизируемся и видим такой экран (у меня уже настроено):

<img width="1911" height="1009" alt="Снимок экрана 2026-05-06 150616" src="https://github.com/user-attachments/assets/c05a609c-3258-443e-abce-fde6d91b221c" />

В веб интерфейсе нужно:

1) Создать Site

2) Создать Device Role

3) Создать Device Type

4) Создать устройства

5) Добавить IP адреса

6) Назначить Primary IP

7) Создать Custom Fields

8) Заполнить Custom Fields

После всего этого можно увидеть это:

<img width="1919" height="1030" alt="Снимок экрана 2026-05-06 151900" src="https://github.com/user-attachments/assets/99859e05-daa1-4733-9f58-e65da8bcd4ab" />

Всё это нужно было для работы как полноценный источник истины для Ansible для генерации конфигов.

Теперь можно прописать непосредственно сами конфигурации:

lab3_export.yml:

```
---
- name: Export NetBox data to file
  hosts: localhost
  gather_facts: false

  vars:
    netbox_api: "http://91.210.106.55:8000"
    netbox_token: "ca5e21fb1b2a4ff1a51f78017e99e1adaaebc067"
    export_file: "./netbox_data.json"

  tasks:
    - name: Read sites from NetBox
      set_fact:
        nb_sites: "{{ query('netbox.netbox.nb_lookup', 'sites', api_endpoint=netbox_api, token=netbox_token) }}"

    - name: Read device roles from NetBox
      set_fact:
        nb_roles: "{{ query('netbox.netbox.nb_lookup', 'device-roles', api_endpoint=netbox_api, token=netbox_token) }}"

    - name: Read devices from NetBox
      set_fact:
        nb_devices: "{{ query('netbox.netbox.nb_lookup', 'devices', api_endpoint=netbox_api, token=netbox_token) }}"

    - name: Read IP addresses from NetBox
      set_fact:
        nb_ips: "{{ query('netbox.netbox.nb_lookup', 'ip-addresses', api_endpoint=netbox_api, token=netbox_token) }}"

    - name: Build export structure
      set_fact:
        netbox_export:
          sites: "{{ nb_sites | map(attribute='value') | list }}"
          device_roles: "{{ nb_roles | map(attribute='value') | list }}"
          devices: "{{ nb_devices | map(attribute='value') | list }}"
          ip_addresses: "{{ nb_ips | map(attribute='value') | list }}"

    - name: Save NetBox data to file
      copy:
        content: "{{ netbox_export | to_nice_json }}"
        dest: "{{ export_file }}"
```

Данная конфигурация используется для автоматического получения данных из NetBox и сохранения их в отдельный JSON-файл. Сценарий подключается к NetBox через API с использованием токена доступа, после чего считывает информацию о площадках, ролях устройств, самих устройствах и IP-адресах. Все полученные данные объединяются в одну структуру и сохраняются в файл netbox_data.json в удобном читаемом формате. Это позволяет быстро сделать резервную копию информации из NetBox или использовать экспортированные данные для дальнейшей автоматизации и настройки сетевых устройств.

lab3_sync.yml:

```
---
- name: Collect serial number and write it to NetBox
  hosts: router
  gather_facts: false

  vars:
    netbox_api: "http://91.210.106.55::8000"
    netbox_token: "ca5e21fb1b2a4ff1a51f78017e99e1adaaebc067"
    ansible_connection: ansible.netcommon.network_cli
    ansible_network_os: community.routeros.routeros
    ansible_user: ansible
    ansible_password: "123"
    ansible_ssh_common_args: "-o StrictHostKeyChecking=no"
    ansible_command_timeout: 180

  tasks:
    - name: Get RouterOS serial number
      community.routeros.command:
        commands:
          - /system routerboard print
      register: routerboard_out

    - name: Extract serial number
      set_fact:
        router_serial: "{{ (routerboard_out.stdout[0] | regex_search('serial-number:.*', multiline=True) | default('serial-number: unknown')) | regex_replace('serial-number:\s*', '') }}"

    - name: Write serial number to NetBox
      delegate_to: localhost
      netbox.netbox.netbox_device:
        netbox_url: "{{ netbox_api }}"
        netbox_token: "{{ netbox_token }}"
        data:
          name: "{{ inventory_hostname }}"
          serial: "{{ router_serial }}"
        state: present

    - name: Save serial locally
      delegate_to: localhost
      copy:
        content: |
          device: {{ inventory_hostname }}
          serial: {{ router_serial }}
        dest: "./configs/{{ inventory_hostname }}_serial.txt"
```

Вот это нужно для автоматического получения серийного номера маршрутизатора MikroTik и сохранения этой информации в NetBox. Сценарий подключается к устройству по SSH с использованием Ansible, выполняет команду RouterOS для получения информации о RouterBOARD и извлекает из вывода серийный номер устройства. После этого полученный серийный номер автоматически записывается в соответствующее устройство в NetBox через API. Дополнительно информация о серийном номере сохраняется локально в отдельный текстовый файл, что позволяет хранить резервную копию данных об оборудовании и использовать их для инвентаризации и учета сетевых устройств.

lab3_configure.yml:

```
---
- name: Configure two CHR from NetBox data
  hosts: router
  gather_facts: false

  vars:
    ansible_connection: ansible.netcommon.network_cli
    ansible_network_os: community.routeros.routeros
    ansible_user: ansible
    ansible_password: "123"
    ansible_ssh_common_args: "-o StrictHostKeyChecking=no"
    ansible_command_timeout: 180

  tasks:
    - name: Set router identity from NetBox inventory hostname
      community.routeros.command:
        commands:
          - /system identity set name={{ inventory_hostname }}

    - name: Ensure NTP client is enabled
      community.routeros.command:
        commands:
          - /system ntp client set enabled=yes servers=pool.ntp.org

    - name: Add management IP from NetBox to WireGuard interface
      community.routeros.command:
        commands:
          - /ip address add address={{ ansible_host }}/24 interface=wg-vps comment="NetBox managed IP"
      when: ansible_host is defined

    - name: Update OSPF router-id from NetBox
      community.routeros.command:
        commands:
          - /routing ospf instance set [find] router-id={{ router_id }} disabled=no
      when: router_id is defined

    - name: Add default user for automation
      community.routeros.command:
        commands:
          - /user add name=ansible group=full password=123
      ignore_errors: true

    - name: Show basic info after configuration
      community.routeros.command:
        commands:
          - /system identity print
          - /ip address print
          - /system ntp client print
      register: router_state

    - name: Save router state locally
      delegate_to: localhost
      copy:
        content: "{{ router_state.stdout | join('\n\n') }}"
        dest: "./configs/{{ inventory_hostname }}_state.txt"
```

Эта для автоматической настройки двух маршрутизаторов CHR на основе данных, полученных из NetBox. Сценарий подключается к устройствам MikroTik по SSH и выполняет базовую настройку сети и системы. В процессе работы автоматически задаётся имя маршрутизатора в соответствии с именем устройства в NetBox, включается NTP-клиент для синхронизации времени, назначается управляющий IP-адрес на интерфейс WireGuard, а также настраивается OSPF router-id, если он указан в данных устройства. Дополнительно создаётся пользователь для автоматизации с полными правами доступа. После завершения настройки сценарий собирает основную информацию о состоянии устройства, включая имя маршрутизатора, IP-адреса и параметры NTP, а затем сохраняет эти данные локально в текстовый файл для последующей проверки и хранения результатов настройки.

и наконец netbox_inventory.yml:

```
plugin: netbox.netbox.nb_inventory
api_endpoint: "http://91.210.106.55::8000"
token: "ca5e21fb1b2a4ff1a51f78017e99e1adaaebc067"
validate_certs: false

group_by:
  - device_roles

compose:
  ansible_host: primary_ip4.address | regex_replace('/.*$', '')
  router_id: custom_fields.router_id
  ospf_peer_ip: custom_fields.ospf_peer_ip
```

Для автоматического формирования инвентаря Ansible на основе данных из NetBox. Система подключается к NetBox через API с использованием токена доступа и получает информацию об устройствах, которые затем автоматически используются в Ansible без необходимости вручную прописывать хосты и параметры. Устройства группируются по их ролям, что упрощает управление и применение различных сценариев автоматизации. Дополнительно из NetBox автоматически извлекается основной IPv4-адрес устройства, который используется для подключения по SSH, а также пользовательские параметры, такие как OSPF router-id и IP-адрес OSPF-соседа. Это позволяет централизованно хранить сетевые данные в NetBox и автоматически использовать их при настройке и управлении маршрутизаторами.

На выходе должно появиться netbox_data.json:

```
{
    "device_roles": [
        {
            "color": "9e9e9e",
            "config_template": null,
            "created": "2026-05-06T11:28:40.173574Z",
            "custom_fields": {},
            "description": "",
            "device_count": 2,
            "display": "router",
            "id": 1,
            "last_updated": "2026-05-06T11:28:40.173610Z",
            "name": "router",
            "slug": "router",
            "tags": [],
            "url": "http://91.210.106.55:8000/api/dcim/device-roles/1/",
            "virtualmachine_count": 0,
            "vm_role": true
        },
        {
            "color": "9e9e9e",
            "config_template": null,
            "created": "2026-05-06T11:29:58.739222Z",
            "custom_fields": {},
            "description": "",
            "device_count": 0,
            "display": "routers",
            "id": 2,
            "last_updated": "2026-05-06T11:29:58.739248Z",
            "name": "routers",
            "slug": "routers",
            "tags": [],
            "url": "http://91.210.106.55:8000/api/dcim/device-roles/2/",
            "virtualmachine_count": 0,
            "vm_role": true
        }
    ],
    "devices": [
        {
            "airflow": null,
            "asset_tag": null,
            "cluster": null,
            "comments": "",
            "config_context": {},
            "config_template": null,
            "console_port_count": 0,
            "console_server_port_count": 0,
            "created": "2026-05-06T11:46:44.449745Z",
            "custom_fields": {
                "ospf_peer_ip": "10.20.0.3",
                "router_id": "1.1.1.1"
            },
            "description": "",
            "device_bay_count": 0,
            "device_role": {
                "display": "router",
                "id": 1,
                "name": "router",
                "slug": "router",
                "url": "http://91.210.106.55:8000/api/dcim/device-roles/1/"
            },
            "device_type": {
                "display": "CHR",
                "id": 1,
                "manufacturer": {
                    "display": "MikroTik",
                    "id": 1,
                    "name": "MikroTik",
                    "slug": "mikroMikroTiktik",
                    "url": "http://91.210.106.55:8000/api/dcim/manufacturers/1/"
                },
                "model": "CHR",
                "slug": "chr",
                "url": "http://91.210.106.55:8000/api/dcim/device-types/1/"
            },
            "display": "chr1",
            "face": null,
            "front_port_count": 0,
            "id": 1,
            "interface_count": 1,
            "inventory_item_count": 0,
            "last_updated": "2026-05-06T12:15:34.912253Z",
            "latitude": null,
            "local_context_data": null,
            "location": null,
            "longitude": null,
            "module_bay_count": 0,
            "name": "chr1",
            "oob_ip": null,
            "parent_device": null,
            "platform": null,
            "position": null,
            "power_outlet_count": 0,
            "power_port_count": 0,
            "primary_ip": {
                "address": "10.20.0.2/24",
                "display": "10.20.0.2/24",
                "family": 4,
                "id": 1,
                "url": "http://91.210.106.55:8000/api/ipam/ip-addresses/1/"
            },
            "primary_ip4": {
                "address": "10.20.0.2/24",
                "display": "10.20.0.2/24",
                "family": 4,
                "id": 1,
                "url": "http://91.210.106.55:8000/api/ipam/ip-addresses/1/"
            },
            "primary_ip6": null,
            "rack": null,
            "rear_port_count": 0,
            "role": {
                "display": "router",
                "id": 1,
                "name": "router",
                "slug": "router",
                "url": "http://91.210.106.55:8000/api/dcim/device-roles/1/"
            },
            "serial": "",
            "site": {
                "display": "lab3",
                "id": 1,
                "name": "lab3",
                "slug": "lab3",
                "url": "http://91.210.106.55:8000/api/dcim/sites/1/"
            },
            "status": {
                "label": "Active",
                "value": "active"
            },
            "tags": [],
            "tenant": null,
            "url": "http://91.210.106.55:8000/api/dcim/devices/1/",
            "vc_position": null,
            "vc_priority": null,
            "virtual_chassis": null
        },
        {
            "airflow": null,
            "asset_tag": null,
            "cluster": null,
            "comments": "",
            "config_context": {},
            "config_template": null,
            "console_port_count": 0,
            "console_server_port_count": 0,
            "created": "2026-05-06T11:47:12.793851Z",
            "custom_fields": {
                "ospf_peer_ip": "10.20.0.2",
                "router_id": "2.2.2.2"
            },
            "description": "",
            "device_bay_count": 0,
            "device_role": {
                "display": "router",
                "id": 1,
                "name": "router",
                "slug": "router",
                "url": "http://91.210.106.55:8000/api/dcim/device-roles/1/"
            },
            "device_type": {
                "display": "CHR",
                "id": 1,
                "manufacturer": {
                    "display": "MikroTik",
                    "id": 1,
                    "name": "MikroTik",
                    "slug": "mikroMikroTiktik",
                    "url": "http://91.210.106.55:8000/api/dcim/manufacturers/1/"
                },
                "model": "CHR",
                "slug": "chr",
                "url": "http://91.210.106.55:8000/api/dcim/device-types/1/"
            },
            "display": "chr2",
            "face": null,
            "front_port_count": 0,
            "id": 2,
            "interface_count": 1,
            "inventory_item_count": 0,
            "last_updated": "2026-05-06T12:18:16.372949Z",
            "latitude": null,
            "local_context_data": null,
            "location": null,
            "longitude": null,
            "module_bay_count": 0,
            "name": "chr2",
            "oob_ip": null,
            "parent_device": null,
            "platform": null,
            "position": null,
            "power_outlet_count": 0,
            "power_port_count": 0,
            "primary_ip": {
                "address": "10.20.0.3/24",
                "display": "10.20.0.3/24",
                "family": 4,
                "id": 2,
                "url": "http://91.210.106.55:8000/api/ipam/ip-addresses/2/"
            },
            "primary_ip4": {
                "address": "10.20.0.3/24",
                "display": "10.20.0.3/24",
                "family": 4,
                "id": 2,
                "url": "http://91.210.106.55:8000/api/ipam/ip-addresses/2/"
            },
            "primary_ip6": null,
            "rack": null,
            "rear_port_count": 0,
            "role": {
                "display": "router",
                "id": 1,
                "name": "router",
                "slug": "router",
                "url": "http://91.210.106.55:8000/api/dcim/device-roles/1/"
            },
            "serial": "",
            "site": {
                "display": "lab3",
                "id": 1,
                "name": "lab3",
                "slug": "lab3",
                "url": "http://91.210.106.55:8000/api/dcim/sites/1/"
            },
            "status": {
                "label": "Active",
                "value": "active"
            },
            "tags": [],
            "tenant": null,
            "url": "http://91.210.106.55:8000/api/dcim/devices/2/",
            "vc_position": null,
            "vc_priority": null,
            "virtual_chassis": null
        }
    ],
    "ip_addresses": [
        {
            "address": "10.20.0.2/24",
            "assigned_object": {
                "_occupied": false,
                "cable": null,
                "device": {
                    "display": "chr1",
                    "id": 1,
                    "name": "chr1",
                    "url": "http://91.210.106.55:8000/api/dcim/devices/1/"
                },
                "display": "ether1",
                "id": 1,
                "name": "ether1",
                "url": "http://91.210.106.55:8000/api/dcim/interfaces/1/"
            },
            "assigned_object_id": 1,
            "assigned_object_type": "dcim.interface",
            "comments": "",
            "created": "2026-05-06T11:51:26.587596Z",
            "custom_fields": {},
            "description": "",
            "display": "10.20.0.2/24",
            "dns_name": "",
            "family": {
                "label": "IPv4",
                "value": 4
            },
            "id": 1,
            "last_updated": "2026-05-06T12:03:56.294141Z",
            "nat_inside": null,
            "nat_outside": [],
            "role": null,
            "status": {
                "label": "Active",
                "value": "active"
            },
            "tags": [],
            "tenant": null,
            "url": "http://91.210.106.55:8000/api/ipam/ip-addresses/1/",
            "vrf": null
        },
        {
            "address": "10.20.0.3/24",
            "assigned_object": {
                "_occupied": false,
                "cable": null,
                "device": {
                    "display": "chr2",
                    "id": 2,
                    "name": "chr2",
                    "url": "http://91.210.106.55:8000/api/dcim/devices/2/"
                },
                "display": "ether1",
                "id": 2,
                "name": "ether1",
                "url": "http://91.210.106.55:8000/api/dcim/interfaces/2/"
            },
            "assigned_object_id": 2,
            "assigned_object_type": "dcim.interface",
            "comments": "",
            "created": "2026-05-06T11:51:40.460706Z",
            "custom_fields": {},
            "description": "",
            "display": "10.20.0.3/24",
            "dns_name": "",
            "family": {
                "label": "IPv4",
                "value": 4
            },
            "id": 2,
            "last_updated": "2026-05-06T12:04:45.832926Z",
            "nat_inside": null,
            "nat_outside": [],
            "role": null,
            "status": {
                "label": "Active",
                "value": "active"
            },
            "tags": [],
            "tenant": null,
            "url": "http://91.210.106.55:8000/api/ipam/ip-addresses/2/",
            "vrf": null
        }
    ],
    "sites": [
        {
            "asns": [],
            "circuit_count": 0,
            "comments": "",
            "created": "2026-05-06T11:26:26.833386Z",
            "custom_fields": {},
            "description": "",
            "device_count": 2,
            "display": "lab3",
            "facility": "",
            "group": null,
            "id": 1,
            "last_updated": "2026-05-06T11:26:26.833436Z",
            "latitude": null,
            "longitude": null,
            "name": "lab3",
            "physical_address": "",
            "prefix_count": 0,
            "rack_count": 0,
            "region": null,
            "shipping_address": "",
            "slug": "lab3",
            "status": {
                "label": "Active",
                "value": "active"
            },
            "tags": [],
            "tenant": null,
            "time_zone": null,
            "url": "http://91.210.106.55:8000/api/dcim/sites/1/",
            "virtualmachine_count": 0,
            "vlan_count": 0
        }
    ]
}
```

CHR все так же между собой пингуются, я ничего не менял

<img width="870" height="342" alt="image" src="https://github.com/user-attachments/assets/32a3d90b-3895-43e2-a8bd-e74ff6135579" />

