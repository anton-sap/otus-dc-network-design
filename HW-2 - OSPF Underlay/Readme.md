# Underlay. OSPF

## Цель
* Настроить OSPF для  Underlay сети

Ожидаемый результат:
1. Настроен OSPF в Underlay сети, IP связанность между устройствами.
2. В документации зафиксирован план работ, адресное пространство, схема сети, конфигурация устройств.
3. IP связанность между устройствами проверена и подтверждена.

## Схема сети
Кабельный журнал и IP-plan
![](images/HW-2-map.png)
Point-toPoint соединения

| **Соединение**                                           | **Подсеть**    | **Устройство A**                     | **Интерфейс** | **IP A**   | **Устройство B**                            | **Интерфейс** | **IP B**   |
|----------------------------------------------------------|----------------|--------------------------------------|---------------|------------|---------------------------------------------|---------------|------------|
| no-osl-dc1-f1-r01k01-spn01 <-> no-osl-dc1-f1-r03k01-lf01 | 10.16.2.0/31   | no-osl-dc1-f1-r01k01-spn01           | Ethernet 1    | 10.16.2.0  | no-osl-dc1-f1-r03k01-lf01                   | Ethernet 1    | 10.16.2.1  |
| no-osl-dc1-f1-r01k01-spn01 <-> no-osl-dc1-f1-r03k02-lf01 | 10.16.2.2/31   | no-osl-dc1-f1-r01k01-spn01           | Ethernet 2    | 10.16.2.2  | no-osl-dc1-f1-r03k02-lf01                   | Ethernet 1    | 10.16.2.3  |
| no-osl-dc1-f1-r01k01-spn01 <-> no-osl-dc1-f1-r03k03-lf01 | 10.16.2.4/31   | no-osl-dc1-f1-r01k01-spn01           | Ethernet 3    | 10.16.2.4  | no-osl-dc1-f1-r03k03-lf01                   | Ethernet 1    | 10.16.2.5  |
| no-osl-dc1-f1-r02k01-spn01 <-> no-osl-dc1-f1-r03k01-lf01 | 10.16.2.6/31   | no-osl-dc1-f1-r02k01-spn01           | Ethernet 1    | 10.16.2.6  | no-osl-dc1-f1-r03k01-lf01                   | Ethernet 2    | 10.16.2.7  |
| no-osl-dc1-f1-r02k01-spn01 <-> no-osl-dc1-f1-r03k02-lf01 | 10.16.2.8/31   | no-osl-dc1-f1-r02k01-spn01           | Ethernet 2    | 10.16.2.8  | no-osl-dc1-f1-r03k02-lf01                   | Ethernet 2    | 10.16.2.9  |
| no-osl-dc1-f1-r02k01-spn01 <-> no-osl-dc1-f1-r03k03-lf01 | 10.16.2.10/31  | no-osl-dc1-f1-r02k01-spn01           | Ethernet 3    | 10.16.2.10 | no-osl-dc1-f1-r03k03-lf01                   | Ethernet 2    | 10.16.2.11 |

Loopback интерфейсы для нужд OSPF и VTEP

| Устройство                 | Loopback 0 (OSPF) | Loopback 10 (VTEP) |
|----------------------------|-------------------|--------------------|
| no-osl-dc1-f1-r01k01-spn01 | 10.16.0.1/32      | -                  |
| no-osl-dc1-f1-r02k01-spn01 | 10.16.0.2/32      | -                  |
| no-osl-dc1-f1-r03k01-lf01  | 10.16.1.1/32      | 10.16.4.1/32       |
| no-osl-dc1-f1-r03k02-lf01  | 10.16.1.2/32      | 10.16.4.2/32       |
| no-osl-dc1-f1-r03k03-lf01  | 10.16.1.3/32      | 10.16.4.3/32       |

## Настройка OSPF
Настройка OSPF для устройств (внезапно :) ) реализована через Ansible. Идея заимствована из книги "Network Automation Cookbook" (Karim Okasha) и репозитория на Github “https://github.com/PacktPublishing/Network-Automation-Cookbook/tree/master/ch4_arista.”

Файл инвентаря inventory представлен в следующем виде

    [spine]
    no-osl-dc1-f1-r01k01-spn01 ansible_host=172.16.108.101
    no-osl-dc1-f1-r02k01-spn01 ansible_host=172.16.108.102
    
    [leaf]
    no-osl-dc1-f1-r03k01-lf01 ansible_host=172.16.108.111
    no-osl-dc1-f1-r03k02-lf01 ansible_host=172.16.108.112
    no-osl-dc1-f1-r03k03-lf01 ansible_host=172.16.108.113
    
    [arista:children]
    spine
    leaf

Переменные содержат следующие значения:
arista.yml

    ---
    ansible_user: ansible
    ansible_ssh_pass: ansible123
    
    ansible_network_os: eos
    ansible_connection: httpapi
    ansible_httpapi_use_ssl: yes
    ansible_httpapi_validate_certs: no

all.yml

    global:
        mgmt_vrf: mgmt
        
        lo_ip:
            no-osl-dc1-f1-r01k01-spn01: "10.16.0.1/32"
            no-osl-dc1-f1-r02k01-spn01: "10.16.0.2/32"
            no-osl-dc1-f1-r03k01-lf01: "10.16.1.1/32"
            no-osl-dc1-f1-r03k02-lf01: "10.16.1.2/32"
            no-osl-dc1-f1-r03k03-lf01: "10.16.1.3/32"
        
        vtep_lo_ip:
            no-osl-dc1-f1-r03k01-lf01: "10.16.4.1/32"
            no-osl-dc1-f1-r03k02-lf01: "10.16.4.2/32"
            no-osl-dc1-f1-r03k03-lf01: "10.16.4.3/32"
        
        p2p_prefix: 31
        
        p2p_ip:
            no-osl-dc1-f1-r01k01-spn01:
              - { port: Ethernet1, ip: 10.16.2.0/31 }
              - { port: Ethernet2, ip: 10.16.2.2/31 }
              - { port: Ethernet3, ip: 10.16.2.4/31 }
              no-osl-dc1-f1-r02k01-spn01:
              - { port: Ethernet1, ip: 10.16.2.6/31 }
              - { port: Ethernet2, ip: 10.16.2.8/31 }
              - { port: Ethernet3, ip: 10.16.2.10/31 }
              no-osl-dc1-f1-r03k01-lf01:
              - { port: Ethernet1, ip: 10.16.2.1/31 }
              - { port: Ethernet2, ip: 10.16.2.7/31 }
              no-osl-dc1-f1-r03k02-lf01:
              - { port: Ethernet1, ip: 10.16.2.3/31 }
              - { port: Ethernet2, ip: 10.16.2.9/31 }
              no-osl-dc1-f1-r03k03-lf01:
              - { port: Ethernet1, ip: 10.16.2.5/31 }
              - { port: Ethernet2, ip: 10.16.2.11/31 }
            
        
        ospf_router_id:
            no-osl-dc1-f1-r01k01-spn01: "10.16.0.1"
            no-osl-dc1-f1-r02k01-spn01: "10.16.0.2"
            no-osl-dc1-f1-r03k01-lf01: "10.16.0.3"
            no-osl-dc1-f1-r03k02-lf01: "10.16.0.4"
            no-osl-dc1-f1-r03k03-lf01: "10.16.0.5"

Плейбуки несут следующие функции:
01_setup_eapi.yml - включает eAPI на устройствах

        ---
        - name: "Enable eAPI on Arista Switches"
          hosts: arista
          become: yes
          vars:
            ansible_connection: network_cli
          tasks:
            - name: "Enable eAPI"
              eos_eapi:
                  https_port: 443
                  https: yes
                  state: started
        
            - name: "Enable eAPI under VRF"
              eos_eapi:
                  state: started
                  vrf: "{{global.mgmt_vrf}}"
            
            - name: "Create user for ansible"
              eos_user:
                  name: ansible
                  privilege: 15
                  configured_password: ansible123
                  state: present
        
            - name:
              eos_command:
                  commands: aaa authorization exec default local
02_create_interface_config.yml - создает и настраивает интерфейсы устройств и присваивает адреса
03_ospf.yml - настраивает OSPF