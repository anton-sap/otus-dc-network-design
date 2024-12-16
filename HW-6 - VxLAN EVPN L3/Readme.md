# VxLAN. EVPN L3

## Цель

* Настроить маршрутизацию в рамках Overlay между клиентами

**Ожидаемый результат**
* Каждый клиент настроен в своем VNI.
* Настроена маршрутизация между клиентами.
* В документации зафиксирован план работ, адресное пространство, схема сети и конфигурация устройств.

## Достижение результата

### Введение
За основу используется топология из предыдущего задания [Часть 5. VxLAN. EVPN L2](https://github.com/anton-sap/otus-dc-network-design/tree/master/HW-5%20-%20VxLAN%20EVPN%20L2) с небольшими изменениями.

![](images/HW-6-map.png)

Коротко о нововведениях:
* no-osl-dc1-f1-r03k01-lf01
  * За leaf'ом no-osl-dc1-f1-r03k01-lf01 появилась сеть 192.168.11.0/24.
  * На leaf no-osl-dc1-f1-r03k01-lf01 настроен vlan11. Порт Ethernet 4 настроен в режиме access для vlan11
  * На leaf no-osl-dc1-f1-r03k01-lf01 настроен интерфейс Vlan11 с адресом 192.168.11.254/24
  * К коммутатору на порт Ethernet 4 подключен PC с адресом 192.168.11.1/24 и gw 192.168.11.254

* no-osl-dc1-f1-r03k02-lf02
    * За leaf'ом no-osl-dc1-f1-r03k02-lf02 появилась сеть 192.168.41.0/24.
    * На leaf no-osl-dc1-f1-r03k02-lf02 настроен vlan11. Порт Ethernet 4 настроен в режиме access для vlan41
    * На leaf no-osl-dc1-f1-r03k02-lf02 настроен интерфейс Vlan11 с адресом 192.168.41.254/24
    * К коммутатору на порт Ethernet 4 подключен PC с адресом 192.168.41.1/24 и gw 192.168.41.254

### Шаблоны конфигурации

Как и прошлом задании стараемся придерживать шаблонной конфигурации устройств и ренедеринга из Netbox.

Вводятся новые понятия в Netbox:
* `IPAM -> VRFS -> VRFs`
![](images/netbox_vrfs.png)

* `IPAM -> VRFS -> Route Targets`
![](images/netbox_route_targets.png)

Для корректности конфигурации нам нужно, чтобы в конфигурацию устройств добавились секции:
* vrf
* interface Vlan[x] с привязкой к нужному VRF
* секция в BGP для работы с VRF

Шаблон конфигурации Spine-коммутаторов остается неизменным.

Описание нового [шаблона для leaf-коммутаторов](files/hw6_netbox_leaf_bgp_template.jinja2):
1. Добавлен блок для генерации VRF:

`
{%- set interface_vrf_names = [] %}
{%- for interface in device.interfaces.all() %}
    {%- for ip in interface.ip_addresses.all() %}
        {%- if ip.vrf and ip.vrf.name not in interface_vrf_names %}
            {%- set _ = interface_vrf_names.append(ip.vrf.name) %}
        {%- endif %}
    {%- endfor %}
{%- endfor %}
{%- for vrf_name in interface_vrf_names %}
vrf instance {{ vrf_name }}
!
{%- endfor %}
`

2. В блок генерации интерфейсов и IP-адресов добавлена генерация для vlan-интерфейсов 


      {%- elif interface.name.startswith('Vlan') %}
    interface {{ interface.name }}
        {%- for ip in interface.ip_addresses.all() %}
      ip address {{ ip.address }}
      vrf {{ interface.ip_addresses.first().vrf.name }}
        {%- endfor %}
        {%- if interface.description %}
      description {{ interface.description }}
        {%- endif %}

3. В блок генерации настроек BGP добавлена логика генерации для VRF


    {%- for interface in device.interfaces.all() %}
      {%- if interface.name.startswith('Vlan') and interface.tags.filter(slug='vxlan').exists() %}
       vrf {{ interface.ip_addresses.first().vrf.name }}
          rd {{ device.interfaces.get(name="Loopback0").ip_addresses.first().address.ip }}:{{ interface.ip_addresses.first().vrf.id }}
          {%- set all_targets = [] %}
          {%- for target in interface.ip_addresses.first().vrf.import_targets.all() %}
            {%- if target not in all_targets %}
              {%- set _ = all_targets.append(target) %}
            {%- endif %}
          {%- endfor %}
          {%- for target in interface.ip_addresses.first().vrf.export_targets.all() %}
            {%- if target not in all_targets %}
              {%- set _ = all_targets.append(target) %}
            {%- endif %}
          {%- endfor %}
          {%- for target in all_targets %}
          route-target import evpn {{ target }}
          route-target export evpn {{ target }}
          {%- endfor %}
          redistribute connected
        !
      {%- endif %}
    {%- endfor %}

4. Другие небольшие изменения в виде фильтров для связки vlan-vni.

### Конфигурационные файлы



### Проверка настроек