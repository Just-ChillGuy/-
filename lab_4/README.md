# Ход работы

На пк другой я установил себе виртуал бокс и вагрант

Далее в ВМ я сначала просто запустил виртуалку

<img width="862" height="634" alt="image" src="https://github.com/user-attachments/assets/b871964a-c191-4f59-805b-80e2fce0dacb" />

Вообще правильнее было сделать так:

```
vagrant up
```

Это я далее и сделал, просто не с первого раза завелось. 

И уже полноценно запустить: 

```
vagrant ssh release
```

Всё, теперь мы в виртуалке, склонировал репозиторий

```
git clone https://github.com/p4lang/tutorials.git
cd ~/tutorials/exercises/basic_tunnel
```

Чтобы в следующих двух заданиях всё у нас заработало, надо починить basic_tunnel.p4. В парсере: 
```
state parse_ethernet {
    packet.extract(hdr.ethernet);
    transition select(hdr.ethernet.etherType) {
        0x1212: parse_tunnel;
        0x0800: parse_ipv4;
        default: accept;
    }
}
```
Было изменено для того, чтобы коммутатор мог различать обычные IPv4-пакеты и пакеты с пользовательским туннельным заголовком myTunnel.
Если etherType = 0x1212, пакет передаётся в обработчик туннеля.
```
state parse_tunnel {
    packet.extract(hdr.myTunnel);
    transition select(hdr.myTunnel.proto_id) {
        0x0800: parse_ipv4;
        default: accept;
    }
}
```
Добавлено извлечение заголовка myTunnel.
Поле proto_id используется для определения типа инкапсулированного протокола.
При 0x0800 дополнительно извлекается IPv4-заголовок.
```
state parse_ipv4 {
    packet.extract(hdr.ipv4);
    transition accept;
}
```
Используется для извлечения IPv4-заголовка и завершения парсинга пакета.

В Action:

```
action myTunnel_forward(egressSpec_t port) {
    standard_metadata.egress_spec = port;
}
```
Добавлено действие пересылки пакета через указанный порт.
Порт передаётся из control plane через таблицу маршрутизации.

В Table:

```
table myTunnel_exact {
    key = {
        hdr.myTunnel.dst_id: exact;
    }

    actions = {
        myTunnel_forward;
        drop;
    }

    size = 1024;
    default_action = drop();
}
```
Добавлена таблица маршрутизации для туннельных пакетов.
Коммутатор выполняет точное сравнение (exact match) поля dst_id и выбирает выходной порт.
Если совпадение отсутствует — пакет отбрасывается.

В apply:

```
apply {
    if (hdr.myTunnel.isValid()) {
        myTunnel_exact.apply();
    } else if (hdr.ipv4.isValid()) {
        ipv4_lpm.apply();
    }
}
```

Добавлена логика выбора способа маршрутизации:

если пакет содержит myTunnel — используется таблица myTunnel_exact;

если туннеля нет — используется обычная IPv4-маршрутизация.

И порядок вывода: 

```
packet.emit(hdr.ethernet);
packet.emit(hdr.myTunnel);
packet.emit(hdr.ipv4);
```

Итоговый конфиг:

```
/* -*- P4_16 -*- */
#include <core.p4>
#include <v1model.p4>

const bit<16> TYPE_MYTUNNEL = 0x1212;
const bit<16> TYPE_IPV4 = 0x0800;

/*************************************************************************
************************ H E A D E R S ***********************************
*************************************************************************/

typedef bit<9>  egressSpec_t;
typedef bit<48> macAddr_t;
typedef bit<32> ip4Addr_t;

header ethernet_t {
    macAddr_t dstAddr;
    macAddr_t srcAddr;
    bit<16>   etherType;
}

header myTunnel_t {
    bit<16> proto_id;
    bit<16> dst_id;
}

header ipv4_t {
    bit<4>    version;
    bit<4>    ihl;
    bit<8>    diffserv;
    bit<16>   totalLen;
    bit<16>   identification;
    bit<3>    flags;
    bit<13>   fragOffset;
    bit<8>    ttl;
    bit<8>    protocol;
    bit<16>   hdrChecksum;
    ip4Addr_t srcAddr;
    ip4Addr_t dstAddr;
}

struct metadata { }

struct headers {
    ethernet_t ethernet;
    myTunnel_t myTunnel;
    ipv4_t ipv4;
}

/*************************************************************************
************************ P A R S E R *************************************
*************************************************************************/

parser MyParser(packet_in packet,
                out headers hdr,
                inout metadata meta,
                inout standard_metadata_t standard_metadata) {

    state start {
        transition parse_ethernet;
    }

    state parse_ethernet {
        packet.extract(hdr.ethernet);
        transition select(hdr.ethernet.etherType) {
            TYPE_MYTUNNEL: parse_tunnel;
            TYPE_IPV4: parse_ipv4;
            default: accept;
        }
    }

    state parse_tunnel {
        packet.extract(hdr.myTunnel);
        transition select(hdr.myTunnel.proto_id) {
            TYPE_IPV4: parse_ipv4;
            default: accept;
        }
    }

    state parse_ipv4 {
        packet.extract(hdr.ipv4);
        transition accept;
    }
}

/*************************************************************************
******************** CHECKSUM VERIFY ************************************
*************************************************************************/

control MyVerifyChecksum(inout headers hdr, inout metadata meta) {
    apply { }
}

/*************************************************************************
******************** INGRESS *********************************************
*************************************************************************/

control MyIngress(inout headers hdr,
                  inout metadata meta,
                  inout standard_metadata_t standard_metadata) {

    action drop() {
        mark_to_drop(standard_metadata);
    }

    action ipv4_forward(macAddr_t dstAddr, egressSpec_t port) {
        standard_metadata.egress_spec = port;
        hdr.ethernet.srcAddr = hdr.ethernet.dstAddr;
        hdr.ethernet.dstAddr = dstAddr;
        hdr.ipv4.ttl = hdr.ipv4.ttl - 1;
    }

    table ipv4_lpm {
        key = {
            hdr.ipv4.dstAddr: lpm;
        }
        actions = {
            ipv4_forward;
            drop;
            NoAction;
        }
        size = 1024;
        default_action = drop();
    }

    /******** NEW TUNNEL ACTION *********/
    action myTunnel_forward(egressSpec_t port) {
        standard_metadata.egress_spec = port;
    }

    /******** NEW TABLE *********/
    table myTunnel_exact {
        key = {
            hdr.myTunnel.dst_id: exact;
        }
        actions = {
            myTunnel_forward;
            drop;
        }
        size = 1024;
        default_action = drop();
    }

    apply {
        if (hdr.myTunnel.isValid()) {
            myTunnel_exact.apply();
        }
        else if (hdr.ipv4.isValid()) {
            ipv4_lpm.apply();
        }
    }
}

/*************************************************************************
******************** EGRESS **********************************************
*************************************************************************/

control MyEgress(inout headers hdr,
                 inout metadata meta,
                 inout standard_metadata_t standard_metadata) {
    apply { }
}

/*************************************************************************
******************** CHECKSUM ********************************************
*************************************************************************/

control MyComputeChecksum(inout headers hdr, inout metadata meta) {
    apply {
        update_checksum(
            hdr.ipv4.isValid(),
            {
                hdr.ipv4.version,
                hdr.ipv4.ihl,
                hdr.ipv4.diffserv,
                hdr.ipv4.totalLen,
                hdr.ipv4.identification,
                hdr.ipv4.flags,
                hdr.ipv4.fragOffset,
                hdr.ipv4.ttl,
                hdr.ipv4.protocol,
                hdr.ipv4.srcAddr,
                hdr.ipv4.dstAddr
            },
            hdr.ipv4.hdrChecksum,
            HashAlgorithm.csum16
        );
    }
}

/*************************************************************************
******************** DEPARSER ********************************************
*************************************************************************/

control MyDeparser(packet_out packet, in headers hdr) {
    apply {
        packet.emit(hdr.ethernet);
        packet.emit(hdr.myTunnel);
        packet.emit(hdr.ipv4);
    }
}

/*************************************************************************
******************** SWITCH **********************************************
*************************************************************************/

V1Switch(
    MyParser(),
    MyVerifyChecksum(),
    MyIngress(),
    MyEgress(),
    MyComputeChecksum(),
    MyDeparser()
) main;
```

Запуск:

В

```
nano ../../utils/Makefile
```

--p4runtime-files build/$*.p4.p4info.txtpb  поменял на --p4runtime-files build/$*.p4.p4info.txt нового формата

Далее очистка всех данных и запуск:

```
make stop
sudo mn -c
rm -rf build
make run
```

В mininet>  проверяем командой pingall 1 часть задания: 

<img width="393" height="125" alt="image" src="https://github.com/user-attachments/assets/c2939721-f990-4550-a5cf-452b80e862cd" />

Проверка 2 части задания:

Проверка обычной маршрутизации:

```
h2 python3 receive.py &
h1 python3 send.py 10.0.2.2 "normal packet"
```
<img width="623" height="712" alt="image" src="https://github.com/user-attachments/assets/510ed2b7-30a4-4a11-b5e0-f4378ba32bac" />

Проверка tunneling:

```
h1 python3 send.py 10.0.2.2 "tunnel packet" --dst_id 2
```

<img width="881" height="760" alt="image" src="https://github.com/user-attachments/assets/6747599d-8d33-449e-988b-454bd72e2446" />

Проверка tunnel override IP:

```
h1 python3 send.py 10.0.3.3 "forced tunnel" --dst_id 2
```

<img width="649" height="765" alt="image" src="https://github.com/user-attachments/assets/ee8f31e1-0680-4711-87fb-0fd22fa11639" />


В выводах видно, что сначала пакет маршрутизируется по обычному IPv4, затем через custom tunnel header MyTunnel, а в последнем тесте пакет доставляется по dst_id, игнорируя IP-адрес назначения, что подтверждает корректную реализацию туннелирования в P4.
