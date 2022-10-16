# Настройка роутера MikroTik для дома #

По умолачниюю подключение осуществляется по адресу 192.168.88.1/24 по любому порту.

Логин: admin

Пароль пустой

Подключиться можно как по ssh,так и через web, так и через winbox.

Подключаемся через терминал и выполняем команды описанные далее.

### Сброc конфигурации (полный сброс) ###

	/system reset-configuration no-defaults=yes skip-backup=yes

### Конфигурирование беспроводных сетей ###
Создаем профиль безопасности для беспроводной сети

	/interface wireless security-profiles add name=MySecurityProfile authentication-types=wpa2-psk \
  unicast-ciphers=aes-ccm group-ciphers=aes-ccm mode=dynamic-keys\
  wpa2-pre-shared-key=MyPassword

Включаем адаптер беспроводной связи и настриваем его

	/interface wireless enable wlan1;
	set wlan1 band=2ghz-b/g/n channel-width=20/40mhz-XX distance=indoors \
	mode=ap-bridge ssid=MikroTik wireless-protocol=802.11 \
	security-profile=Swan frequency-mode=regulatory-domain \
	set country=etsi antenna-gain=3 hide-ssid=yes \
	wds-mode=disabled wds-default-bridge=none wds-ignore-ssid=no bridge-mode=enabled
	
	/interface wireless enable wlan2;
	set wlan1 band=5ghz-a/n/ac channel-width=20/40/80mhz-XXXX distance=indoors \
	mode=ap-bridge ssid=MikroTik wireless-protocol=802.11 \
	security-profile=Swan frequency-mode=regulatory-domain \
	set country=etsi antenna-gain=3 hide-ssid=yes \
	wds-mode=disabled wds-default-bridge=none wds-ignore-ssid=no bridge-mode=enabled
  
  (Optional) for not hidden ssid change parametr: hide-ssid=no

### Конфигурирование адресного пространства ###

* Добавление логическоих интерфейсов (bridge interface). 

Они необходимы для объединения физических интерфейсов и интерфесов беспроводной сети в один интерфейс.
	
	/interface bridge add name=LAN

* Добавление логического порта (bridge port). 

Они необходимы для назначения соответсвия физических интерфейсов / интерфесов беспроводной сети и логического интерфейса.
	
	/interface bridge port add interface=ether1 bridge=WAN
	/interface bridge port add interface=ether2 bridge=LAN
	/interface bridge port add interface=ether3 bridge=LAN
	/interface bridge port add interface=ether4 bridge=LAN
	/interface bridge port add interface=ether5 bridge=LAN
	/interface bridge port add interface=wlan1 bridge=LAN
	/interface bridge port add interface=wlan2 bridge=LAN

* Настройка адресации на интерфейсах
Так как у нас фсе физические интерфейсы объеденены в логическое, то настраиваться будут только bridge interfaces.

WAN интерфейс не настривается, так как для него адреса будут получаться от провайдера по DHCP.
	
	/ip address add address=192.168.0.1/24 interface=LAN

### Конфигурирование DHCP-Серверов ###

	/ip dhcp-server setup [enter]
	dhcp server interface: LAN [enter]
	dhcp address space: 192.168.0.0/24 [enter]
	gateway for dhcp network: 192.168.0.1 [enter]
	addresses to give out: 192.168.0.2-192.168.0.254 [enter]
	dns servers: 192.168.0.1 [enter]
	lease time: 10m [enter]

### Конфигурирование Internet соединения ###
#### Динамический (серый) IP ####
	
	/ip dhcp-client add disabled=no interface=ether1
	
#### Статический (белый) IP ####

	/ip address add address=100.108.39.100/24 interface=ether1
	/ip route add gateway=100.108.39.15
	/ip dns set servers=8.8.8.8
	
#### PPPoE соеинение ####
	
	/interface pppoe-client
	  add disabled=no interface=ether1 user=me password=123 add-default-route=yes use-peer-dns=yes

#### L2tp соеинение ####		
Настройка L2TP клиента на роутере. Подключиение надежнее осуществлять к доменному имени, в случае смены ip-адреса будет бесшовный переход.

	/interface l2tp-client add connect-to=tp.internet.beeline.ru disabled=no name=L2tp-beeline password=MyPassword user=MyUsername /
	max-mtu=1450 max-mru=1450 mrru=disabled add-default-route=yes default-route-distance=1 allow=mschap2 /
	l2tp-proto-version=l2tpv2 l2tpv3-digest-hash=md5
	
### Конфигурирование NAT ###
Просто включаем маскарад на порту, адрес которого должен натировать в LAN

(Вариант для серго IP)

	/ip firewall nat add chain=srcnat out-interface=ether1 action=masquerade

(Вариант для белого IP)

	/ip firewall nat add action=src-nat chain=srcnat out-interface=ether1 to-addresses=10.20.1.20
	
При необходимости можно прокинуть порты (на примере RDP подключения)

	/ip firewall nat add chain=dstnat protocol=tcp port=3389 in-interface=ether1 action=dst-nat to-address=192.168.0.15

### Нстройка пользователей ###
Добавляем пользователя

	/user add name=dreyner password=mypassword group=full
	
Удаляем пользователя по умолчанию

	/user remove admin

### Настройка подключения по MAC ###
	
	/interface list add name=LAN_list
	/interface list member add list=LAN_list interface=LAN
	tool mac-server set allowed-interface-list=LAN_list
	tool mac-server mac-winbox set allowed-interface-list=LAN_list
	/ip neighbor discovery-settings set discover-interface-list=LAN_list
	
### Настройка безопасности ###
	
Устанавливаем адрес/пул адресов? разрешенный для подключения к коммутатору пользователю

	/user set 0 allowed-address=192.168.0.0/24

Настриваем Межсетевой Экран:

	add chain=forward connection-state=established,related action=fasttrack-connection hw-offload=yes /
	protocol=tcp comment="Fast-track for established,related TCP";
	add chain=forward connection-state=established,related action=fasttrack-connection hw-offload=yes /
	protocol=udp comment="Fast-track for established,related UDP";
	add chain=forward connection-state=established,related action=accept comment="Accept established,related";
	add chain=forward action=drop connection-state=invalid comment="Drop invalid forward";
	add chain=forward action=drop connection-state=new connection-nat-state=!dstnat in-interface=ether1 comment="Drop access to clients behind NAT from WAN";
	add chain=input connection-state=established,related action=accept comment="Accept established,related INPUT";
	add chain=input connection-state=invalid action=drop comment="Drop invalide INPUT";
	add chain=input in-interface=ether1 protocol=icmp action=accept comment="Allow ICMP";
	add chain=input in-interface=ether1 action=drop in-interface=!LAN comment="Block everything not from LAN";
  
Отключаем неиспользуемые и небезопасные сервисы

	/ip service disable telnet, ftp, www, www-ssl, api
	
(Опционально) Также можно изменить порт для подключения по SSH

	/ip service set ssh port=2200

Включаем строгую криптографию для ssh

	/ip ssh set strong-crypto=yes
	
Разрешаем подключение к WinBox только с определенного пула адресов
	
	/ip service set winbox address=192.168.0.0/24
	
Далее идет отключение сервисов, которые отключены по умолчанию (так на всякий случай)

	/tool bandwidth-server set enabled=no
	/ip dns set allow-remote-requests=no
	/lcd set enabled=no
	/ip proxy set enabled=no
	/ip socks set enabled=no
	/ip upnp set enabled=no
	/ip cloud set ddns-enabled=no update-time=no
	
(Опционально) Также при необходимости можно погасить неиспользуемы сетевые порты
	
	/interface print
	/interface set x disabled=yes
	
(Опционально) Бокироваку нежелательных сайтов можно выполнить следующим образом:
1. Настриваем прокси

	/ip firewall nat add chain=dst-nat protocol=tcp dst-port=80 src-address=192.168.0.128/24 action=redirect to-ports=8080

2. Включаем web-Proxy и добавлчяем сайты для блокировки

	/ip proxy set enabled=yes
	/ip proxy access add dst-host=www.facebook.com action=deny
	/ip proxy access add dst-host=*.youtube.* action=deny
	/ip proxy access add dst-host=:vimeo action=deny


### Создание резервной копии конфигурации ###

	system backup save name=backup_date password=123
	
### Восстановление конфигурации из резервной копии ###
	
	system backup load name=backup_date password=123
	system reboot
