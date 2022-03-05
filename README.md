# Подъем собственного VPN-тоннеля
В этой инструкции описаны шаги, позволяющие поднять свой собственный VPN-тоннель на VDS или аппаратном устройстве (например, одноплатном компьютере) под управлением ОС Debian Linux или дистрибутива на его основе. Кроме того, к инструкции прилагаются скрипты, автоматизирующие генерацию и отзыв пользовательских сертификатов. Инструкция частично основана на [туториале от DigitalOcean](https://www.digitalocean.com/community/tutorials/how-to-set-up-an-openvpn-server-on-debian-10). Предполагается установка тоннеля OpenVPN. Он не так хорошо подходит для скрытия трафика, но при развертывании на неизвестном доменном имени и нестандартном для сервиса порту может эффективно исполнять функцию резервного тоннеля.

## Машинерия
Для подъема тоннеля необходимы:
- Постоянно подсоединенная к сети **машина**, аппаратная или виртуальная. Данная инструкция разрабатывалась и тестировалась на [OrangePi Lite 2](http://www.orangepi.org/) под управлением Armbian Buster. Подойдет любой виртуальный или аппаратный мини-сервер под управлением любой версии Linux, основанной на Debian 10 или выше. Машина должна находиться на территории или в юрисдикции, где гарантированно доступны нужные вам ресурсы (практически все европейские страны, США, большая часть Южной Америки). IP-адрес машины должен быть известен.
- Аккаунт на любом сервисе **DynamicDNS**, например, [noip.com](https://www.noip.com/), совместимый с вашей схемой маршрутизации (в пределе - с домашним роутером).
- **Роутер** (если маршрутизация не поднимается непосредственно на сервере, но это отдельная задача), поддерживающий проброс портов в TCP и/или UDP протоколах. В большинстве роутеров такой механизм поддерживается в настройках под наименованием **NAT/Virtual Server**.
- Доступ в сеть со средневзвешенной скоростью более 10 МБит/с (можно меньше, но при нескольких пользователях будет очень медленно). Подойдет даже Wi-Fi!

## Установка ПО
Программное обеспечение для подъема простейшего VPN-тоннеля уже есть в репозиториях Debian. Поставить надо пакеты OpenVPN и EasyRSA. Все команды отдаются от пользователя `root` или через `sudo`, так как требуют повышения привилегий:
```sh
apt install openvpn easy-rsa iptables-persistent
```

Далее создаем каталоги и копируем скрипты EasyRSA:
```sh
mkdir /etc/openvpn/pki
mkdir /etc/openvpn/pki/keys
cp -vr /usr/share/easy-rsa/* /etc/openvpn/pki/
cp -vr /etc/openvpn/pki/vars.example /etc/openvpn/pki/vars
```

Копируем скрипты генерации и отзыва пользовательских ключей из этого репозитория в каталог, указанный в `PATH`:
```sh
sudo cp -v certgen /usr/bin/certgen
sudo cp -v certrevoke /usr/bin/certrevoke
sudo chmod +x /usr/bin/certgen
sudo chmod +x /usr/bin/certrevoke
```

## Настройка EasyRSA
Правим в любом редакторе созданный файл `vars`. Это сборник системных переменных для скрипта EasyRSA. В скрипте следует проставить необходимые имена и пути к файлам для будущего сервиса. В примере приведены только те переменные, значения которых следует обязательно изменить. Остальные можно не трогать. Знак `#` означает начало комментария.
```
set_var EASYRSA_PKI		        "/etc/openvpn/pki/keys"     # Каталог, созданный ранее
set_var EASYRSA_REQ_COUNTRY	    "IT"                        # Код страны. Нужно подставить сюда двузначное обозначение страны, в которой расположен сервер
set_var EASYRSA_REQ_PROVINCE	"TS"                        # Код провинции. Произвольное обозначение. Пример приведен для итальянского города Триеста
set_var EASYRSA_REQ_CITY	    "Trieste"                   # Имя города, одно слово символами ASCII без пробелов
set_var EASYRSA_REQ_ORG		    "orgname_"                  # Имя "организации", которой принадлежит сервер, произвольная строка ASCII без пробелов
set_var EASYRSA_REQ_EMAIL	    "mail@examp.le"             # Адрес администратора для сертификатов, следует обычному формату e-mail без кириллицы
set_var EASYRSA_REQ_OU		    "Very Secret Name"          # "Операционное имя" сервера. Произвольная строка ASCII, возможны пробелы в составе
set_var EASYRSA_KEY_SIZE	    4096                        # Размер ключа. Чем больше, тем лучше, но медленнее. 4096 - приемлемое значение для большинства ПО общего назначения
set_var EASYRSA_ALGO		    ec                          # Алгоритм: Elliptical Curve (ec, рекомендуется) или RSA (rsa).
set_var EASYRSA_CURVE		    secp384r1                   # Вид эллиптической кривой, только при значении 'ec' в переменной EASYRSA_ALGO
set_var EASYRSA_CA_EXPIRE	    3650                        # Срок действия сертификата корневого блока, дней. Можно ставить по своему усмотрению, но ключи подлежат отзыву и перевыпуску по истечении срока действия
set_var EASYRSA_CERT_EXPIRE	    1080                        # Срок действия пользователького сертификата, дней
set_var EASYRSA_CERT_RENEW	    60                          # Число дней до истечения срока действия, по достижении которого сертификат можно обновить. Если поставить сюда значение, большее EASYRSA_CERT_EXPIRE, обновление запрещено и сертификат будет подлежать обязательному отзыву по истечении срока действия.
set_var EASYRSA_CRL_DAYS	    3650                        # Срок действия списка отозванных сертификатов
```

## Создание инфраструктуры шифрования с закрытыми ключами (PKI)
```sh
chmod -vR 0700 /etc/openvpn/pki
cd /etc/openvpn/pki
./easyrsa init-pki
./easyrsa build-ca nopass
```
В ответ на эти команды система попросит ввести имя машины (**Common Name**) для корневого сертификата (root CA). Можно согласиться с именем по умолчанию (`Easy-RSA CA`), или ввести свое. Поддерживаются произвольные строки латиницей, в том числе, с пробелами. После этого корневой сертификат нашей цепочки шифрования готов. Все остальные действия проведем на той же машине (что в реальной установке VPN-сервера не приветствуется, но сгодится на маленьком сервере "для друзей").

Далее генерируем шифр-запрос, которым VPN-сервер запрашивает у CA сертификат своей собственной принадлежности:
```sh
./easyrsa gen-req server nopass
```
Здесь система снова попросит ввести имя машины (**Common Name**), можно оставить по умолчанию `server`.

Далее подписываем сертификат сервера и генерируем его сертификат принадлежности:
```sh
./easyrsa sign-req server server
```
В ответ на запрос сервера печатаем `yes` (обязательно полностью), после чего сертификат будет сгенерирован. Следующая операция потребует, скорее всего, довольно длительного времени. Генерируем ключ алгоритма Диффи-Хеллмана, шаблон списка отозванных сертификатов и усиленный ключ TLS для шифрования на серверной стороне:
```sh
./easyrsa gen-dh
openvpn --genkey --secret ../ta.key
cp -v keys/dh.pem ./
./easyrsa gen-crl
```

## Конфигурация серверной части OpenVPN
На этом этапе правим файл `/etc/openvpn/server.conf`. Снова указываются только настройки, значения которых отличаются от значений по умолчанию. Комментарии в файле начинается с символа `#` в любом месте, или с точки с запятой `;` только, если комментарий занимает всю строку.
```
local *.*.*.*           # Сюда следует подставить IP-адрес машины в сети роутера (или во внешней сети, когда за маршрутизацию отвечает сам сервер)
port 1194               # Порт в выбранном протоколе. Можно настроить переброс этого порта на внешний порт на роутере, или указать здесь нестандартное значение. Рекомендуется открывать во внешнюю сеть порты с номерами более 50000. Диапазон возможных значений - 1024-65535
proto udp               # Протокол. TCP (значение tcp) или UDP (udp)
dev tun                 # Тип виртуального сетевого устройства, только туннелирование драйвером TUN. Работа с мостовыми TAP-драйверами в данном руководстве не рассматривается

# Адреса файлов с сертификатами и ключами
ca          /etc/openvpn/pki/keys/ca.crt
cert        /etc/openvpn/pki/keys/issued/server.crt
key         /etc/openvpn/pki/keys/private/server.key
dh          /etc/openvpn/pki/keys/dh.pem
crl-verify  /etc/openvpn/pki/keys/crl.pem

topology subnet         # Вид маршрутизации, только subnet

# IP-шаблон и маска подсети для клиентов
server  10.8.0.0 255.255.0.0

# Адрес статус-файла виртуальной сети
ifconfig-pool-persist /var/log/openvpn/ipp.txt

# Выкладка маршрутов и адресов DNS-серверов, передаваемая клиенту
push "route 10.8.0.0 255.255.0.0"
route 10.8.0.0 255.255.0.0
push "redirect-gateway local def1"
push "dhcp-option DNS 1.1.1.1"
push "dhcp-option DNS 8.8.8.8"
push "dhcp-option DNS 8.8.4.4"
push "dhcp-option DNS 208.67.222.222"
push "dhcp-option DNS 208.67.220.220"

client-to-client        # Разрешаем соединения между клиентами, нужно закомментировать, если даете соединения людям, которые не знают друг друга
keepalive 10 120        # Держим канал протокола открытым с помощью периодической передачи контрольных сетевых пакетов

# Настройка шифрования
cipher AES-256-CBC
auth SHA256

max-clients 16          # Максимальное число одновременно подключенных клиентов

# Снижаем привилегии серверного процесса для усложнения атаки на машину
user nobody
group nogroup

# Устанавливаем режим доступа к ресурсам сервера
persist-key
persist-tun

# Адреса файлов карты статуса сервера и журнала сообщений. При отладке канала в эти файлы сервер будет сбрасывать сообщения об ошибках
status /var/log/openvpn/openvpn-status.log
log /var/log/openvpn/openvpn.log
verb 4

# Уведомляем клиентов о перезапуске сервера
explicit-exit-notify 1

# Устанавливаем низкоуровневые опции стабильности сети
ncp-disable
mute-replay-warnings
sndbuf 393216
rcvbuf 393216
push "sndbuf 393216"
push "rcvbuf 393216"
```

## Конфигурирование скриптов генерации и отзыва пользовательских ключей
На этом этапе нужно исправить переменные, определенные в файле `/usr/bin/certgen`. Они объединены в группы в начале файла. Следует повторить здесь значения, установленные ранее в разделе "**Настройка EasyRSA**" (все значения должны совпасть).
```sh
OPERATION_COUNTRY="IT"
OPERATION_REGION="TS"
OPERATION_CITY="Trieste"
OPERATION_ORG="orgname_"
OPERATION_UNIT="Very Secret Name"
CERT_KEYLEN=4096
CERT_EXPIRATION=3650
```

Далее настраиваем доменные правила:
```sh
AUTH_DOMAIN="mydomain.net"  # Доменное имя, следует подставить сюда имя, выданное вашим провайдером DNS или DynamicDNS, например, noip.com
AUTH_PORT="50000"           # ВНЕШНИЙ порт для подключения к серверу. Он может не совпадать со значением, выставленном в /etc/openvpn/server.conf, так как его открывает, и прописывает правила передачи на него соединений - ваш маршрутизатор.
```
Разумеется, ВНЕШНИЙ порт `AUTH_PORT`, указанный при настройке скрипта `certgen`, нужно открыть в настройках маршрутизации вашего роутера, и связать его с ВНУТРЕННИМ портом сервера OpenVPN, указанном в файле `/etc/openvpn/server.conf`.

## Настройка сетевого стека
Включаем режим "маскарад" для того, чтобы наш сервер мог выступать в качестве конфигуратора виртуальной сети и пробрасывать через себя трафик. Для этого в файле `/etc/sysctl.conf` добавляем или раскомментируем строку:
```
net.ipv4.ip_forward = 1
```
Далее настраиваем правила файрволла
```sh
sudo iptables -A INPUT -i wlan0 -p udp -m state --state NEW -m udp --dport 1194 -j ACCEPT   # 1
sudo iptables -A INPUT -i tun+ -j ACCEPT                                                    # 2
sudo iptables -A FORWARD -s 10.8.0.0/24 -d 192.168.1.0/24 -j ACCEPT                         # 3
sudo iptables -A FORWARD -i tun+ -j ACCEPT                                                  # 4
sudo iptables -A FORWARD -i tun+ -o wlan0 -m state --state RELATED,ESTABLISHED -j ACCEPT    # 5
sudo iptables -A FORWARD -i wlan0 -o tun+ -m state --state RELATED,ESTABLISHED -j ACCEPT    # 6
sudo iptables -A OUTPUT -o tun+ -j ACCEPT                                                   # 7
sudo iptables -A POSTROUTING -s 10.8.0.0/16 -o wlan0 -j MASQUERADE                          # 8
sudo iptables -A POSTROUTING -s 10.8.0.0/24 -d 192.168.1.0/24 -j MASQUERADE                 # 9
```
Здесь нужно обратить особое внимание на правильность имен (контролируем последовательно по командам, подставляем имена, принятые в **нашей конкретной системе на конкретной машине**):
- `wlan0` - имя сетевого интерфейса в системе
- Аргумент опции `--dport` команды **1** - номер ВНУТРЕННЕГО порта, указанный ранее в разделе **Конфигурация серверной части OpenVPN**
- Аргумент ключа `-d` команды **3** - IP-адрес машины во ВНЕШНЕЙ сети и его бинарная маска (например, в сети вашего домашнего маршрутизатора)
- Аргумент ключа `-d` команды **9** - снова IP-адрес и маска машины во внешней сети

## Тестирование
После отдачи команд файрволлу тестируем полученный стек. Запускаем скрипт `certgen`:
```sh
sudo /usr/bin/certgen test-cert test@ema.il
```

Берем полученный на выходе файл `test.ovpn`, и пробуем подключиться с его помощью к серверу. Тестируем соединение, следим за сообщениями об ошибках. Если все в порядке - отзываем тестовый ключ:

```sh
sudo /usr/bin/certrevoke test-cert
sudo service openvpn restart
```

## Сохранение настроек сети
Отдаем команду на сохранение настроек файрволла при перезагрузке:
```sh
sudo iptables-save > /etc/iptables/rules.v4
```

# Использование

В полученной сети VPN **каждому устройству** необходим отдельный ключ (файл `.ovpn`) и уникальное имя (Common Name). Ключ создается скриптом `certgen`:
```sh
sudo /usr/bin/certgen <common-name> <email@addre.ss>
```
Адрес электронной почты должен иметь валидный формат. Адреса почты могут повторяться на одном сервере, но имена Common Name - нет. При необходимости обновить ключ устройства с тем же Common Name, его нужно отозвать, а затем создать заново.

Имя Common Name используется для отзыва ключа так:
```sh
sudo /usr/bin/certrevoke <common-name>
sudo service openvpn restart
```

