# Создание ВПН сервера / Debian GNU/Linux 11
***
##  Первым делом на отдельном сервере необходимо развернуть удостоверяющий центр

### Для начала создадим ключевую пару для отправки файлов по протоколу *`SCP`* на сервер ВПН
+ вводим команду в терминале
```bash 
cd /home/$user/.ssh/
ssh-keygen -t rsa   >> #Вводим название (по умолчанию id_rsa) и парольную фразу
```
+ после создания ключевой пары копируем закрытый ключ в папку /.ssh root пользователя
```bash
sudo cp /home/$user/.ssh/id_rsa /root/.ssh/
```
+ далее публичный ключ вставляем в `authorized_keys` на сервере ВПН
### Теперь приступаем к работе со скриптом, который установит нужные программы, утилиты и создаст необходимые нам сертификаты и ключи
```bash
 #!/bin/bash
 echo "Этот скрипт создаст Центра сертификации с конфиг файлами, от вас потребуется указать количество клиентов и минимальные настройки"
 echo "К каждому пункту будет пояснение"
 echo "Для начала создадим пользователя"
       #Создадим нового пользователя openvpn с правами администратора
       #Проверка на наличие пользователя в системе, для отсутствия ошибок при повторном запуске
 username=$user #переменная с именем пользователя
 client_name=client #имя клиента
 answer=y #ответ пользователя
 grep "^$username:" /etc/passwd >/dev/null
 if [[ $? -ne 0 ]]; then
    adduser $user; sudo usermod -aG $user; passwd $user
    echo "Пользователь создан"
 else
    echo "Пользователь уже создан в системе"
 fi
       #Создание клиентов по умолчанию
 echo "Укажите количество клиентов по умолчанию. Потом можно добавить еще по необходимости"
 read quantity_client
       #Проверка-значение число, иначе сначала
 if [[ $quantity_client =~ ^[0-9]+$ ]]; then   #количество клиентов
    echo "Будут создано "$quantity_client" клиентских конфигураций с именами "$client_name"[X].ovpn"
 else
    echo "введённый символ не является числом, попробуйте снова"
    echo "Попробовать снова? (y/n/e)"
    read answer
    case $answer in
            "y")
               $0
               ;;
            "n")
               echo "bye"
               exit
               ;;
            "e")
               exit
               ;;
             *)
               echo "error"
               ;;
    esac
 fi
echo 'Установим утилиты необходимые для дальнейшей работы'
sudo apt-get install wget >/dev/null; sudo  apt-get install tar >/dev/null; sudo  apt-get install zip >/dev/null
      #Проверка наличия директории openvpn если есть то удаляем и создаем заново, иначе создаем
if [[ -e /etc/openvpn ]]; then
  sudo rm -rf /etc/openvpn
  sudo mkdir /etc/openvpn; sudo mkdir /etc/openvpn/keys; sudo chown -R $user:$user /etc/openvpn
   echo "Удалена старая директория openvpn, создана новая"
else
   sudo mkdir /etc/openvpn; sudo mkdir /etc/openvpn/keys; sudo chown -R $user:$user /etc/openvpn
   echo "создана новая дирктория openvpn"
fi

      #Скачиваем easy-rsa
sudo wget -P /etc/openvpn https://github.com/OpenVPN/easy-rsa/releases/download/v3.0.8/EasyRSA-3.0.8.tgz
sudo tar -xvzf /etc/openvpn/EasyRSA-3.0.8.tgz -C /etc/openvpn
sudo rm -rf /etc/openvpn/EasyRSA-3.0.8.tgz
      #Создадим файл vars, с настройками пользователя
sudo touch /etc/openvpn/EasyRSA-3.0.8/vars
echo "Укажите основные настройки создания сертификатов"
echo "Для каждого пункта есть настройки по умолчанию, их можно оставить"
echo "Страна(по умолчанию Italy):"; read country
if [[ -z $country ]]; then
   country="Italy"
fi
echo "Размер ключа(по умолчанию 2048):"; read key_size
if [[ $key_size =~ ^[0-9]+$ ]]; then #проверка на число
   echo "Установлен размер ключа:" $key_size
else
   key_size=2048; echo "Значение ключа установлено по умолчанию"
fi
echo "Укажите область\край(по умолчанию Toscana)"; read province
if [[ -z $province ]]; then
   province="Toscana"
fi
echo "Город(по умолчанию Florenzia)"; read city
if [[ -z $city ]]; then
   city="Florenzia"
fi
echo "email(по умолчанию user@email.com)"; read mail
if [[ -z $mail ]]; then
   mail="user@email.com"
fi
echo "срок действия сертификата, дней(по умолчанию 3650/10 лет): "; read expire
if [[ $expire =~ ^[0-9]+$ ]]; then
   echo "Срок действия сертификата" $expire "дней"
else
   expire=3650
fi
      #Набиваем vars
sudo cat  > /etc/openvpn/EasyRSA-3.0.8/vars <<EOF
set_var EASYRSA_REQ_COUNTRY $country
set_var EASYRSA_KEY_SIZE $key_size
set_var EASYRSA_REQ_PROVINCE $province
set_var EASYRSA_REQ_CITY $city
set_var EASYRSA_REQ_ORG $domain_name
set_var EASYRSA_REQ_EMAIL $mail
set_var EASYRSA_REQ_OU $domain_name
set_var EASYRSA_REQ_CN changeme
set_var EASYRSA_CERT_EXPIRE $expire
set_var EASYRSA_DH_KEY_SIZE $key_size
EOF

      #Теперь инициализируем инфраструктуру публичных ключей
cd /etc/openvpn/; sudo /etc/openvpn/EasyRSA-3.0.8/easyrsa init-pki
      #Создаем свой ключ
sudo /etc/openvpn/EasyRSA-3.0.8/easyrsa build-ca nopass
      #Создаем сертификат сервера
sudo /etc/openvpn/EasyRSA-3.0.8/easyrsa build-server-full server_cert nopass
      #Создаем Диффи Хелмана
sudo /etc/openvpn/EasyRSA-3.0.8/easyrsa gen-dh
      #crl для информации об активных/отозванных сертификатах
sudo /etc/openvpn/EasyRSA-3.0.8/easyrsa gen-crl
      #Теперь копируем все что создали в папку keys
sudo cp /etc/openvpn/pki/ca.crt /etc/openvpn/pki/crl.pem /etc/openvpn/pki/dh.pem /etc/openvpn/keys/
sudo cp /etc/openvpn/pki/issued/server_cert.crt /etc/openvpn/keys/
sudo cp /etc/openvpn/pki/private/server_cert.key /etc/openvpn/keys/

      #Начнем создавать клиентов
      #Диерктория для готовых конфигов
      if [[ -d "/home/$user/ready_conf" ]]
      then
                sudo rm -rf /etc/openvpn/ready_conf
                sudo mkdir /etc/openvpn/ready_conf
                echo "Удалена старая директоря для конифиг файлов, создана новая"
      else
                sudo mkdir /etc/openvpn/ready_conf
                echo "Создана директория для конфиг файлов"
      fi

         #Теперь функция создания клиентов
     create_client () {
     sudo /etc/openvpn/EasyRSA-3.0.8/easyrsa build-client-full "$client_name$quantity_client" nopass
     sudo touch /etc/openvpn/ready_conf/"$client_name$quantity_client".ovpn
     sudo chown -R $user:$user /etc/openvpn/ready_conf/*
     sudo echo '<ca>' > /etc/openvpn/ready_conf/"$client_name$quantity_client".ovpn
     sudo cat /etc/openvpn/pki/ca.crt >> /etc/openvpn/ready_conf/"$client_name$quantity_client".ovpn
     sudo echo '</ca>' >> /etc/openvpn/ready_conf/"$client_name$quantity_client".ovpn
     sudo echo '<cert>' >> /etc/openvpn/ready_conf/"$client_name$quantity_client".ovpn
     sudo cat /etc/openvpn/pki/issued/"$client_name$quantity_client".crt >> /etc/openvpn/ready_conf/"$client_name$quantity_client".ovpn
     sudo echo '</cert>' >> /etc/openvpn/ready_conf/"$client_name$quantity_client".ovpn
     sudo echo '<key>' >> /etc/openvpn/ready_conf/"$client_name$quantity_client".ovpn
     sudo cat /etc/openvpn/pki/private/"$client_name$quantity_client".key >> /etc/openvpn/ready_conf/"$client_name$quantity_client".ovpn
     sudo echo '</key>' >> /etc/openvpn/ready_conf/"$client_name$quantity_client".ovpn
     sudo echo '<dh>' >> /etc/openvpn/ready_conf/"$client_name$quantity_client".ovpn
     sudo cat /etc/openvpn/pki/dh.pem >> /etc/openvpn/ready_conf/"$client_name$quantity_client".ovpn
     sudo echo '</dh>' >> /etc/openvpn/ready_conf/"$client_name$quantity_client".ovpn
         }

      #Запускать функцию создания клиентов, по счетчику
           while [[ $quantity_client -ne 0 ]]; do
                 create_client
                 let "quantity_client=$quantity_client-1"
                 done
sudo /etc/openvpn/EasyRSA-3.0.8/easyrsa gen-crl #генерируем crl для информации об активных сертификатах
sudo cp /etc/openvpn/pki/crl.pem /etc/openvpn/keys/ #Копируем в директорию с активными сертификатами
cd /etc/openvpn/ready_conf/; sudo ls -alh ./
echo "сейчас вы в директории с готовыми файлами конфигураций, их уже можно использовать"
echo "скрипт завершен успешно"
echo "Теперь отправим файлы на сервер ВПН"
/home/$user/send.sh
exec bash
```
### Далее нам необходимо будет еще два скрипта.
+ Один из них будет закрывать доступ извне и подключаться сможете только вы по SSH.
```bash
#!/bin/bash
sudo apt-get update > /dev/null && echo "Обновили сервер, теперь установим помощник для Firewall"
sudo apt-get install iptables-persistent
if [ $? -eq 0 ]
then
        echo "Установили, теперь настройка доступа"
else
        echo "Пакет уже установлен, приступаем к настройке доступа"
fi
echo "Очистим таблицу перед настройкой"
sudo iptables -F
sudo iptables -A INPUT -p all -m state --state RELATED,ESTABLISHED -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 22 -j ACCEPT
sudo iptables -P INPUT DROP
sudo service netfilter-persistent save
if [ -d /home/$user/iptables ]
then
        sudo rm -rf /home/$user/iptables
else
        sudo mkdir /home/$user/iptables
fi
echo "Устновили блок на все пакеты кроме порта 22"
echo "Сохраним правила в отдельной папке"
mkdir /home/$user/iptables
sudo cp /etc/iptables/* /home/$user/iptables/
echo
echo "Готово"
```
+ Второй для отправки готовых подписанных сертификатов на сервер ВПН.
```bash
#!/bin/bash
sudo systemctl reload ssh
sudo systemctl restart sshd

send () {
        while [ $answer2=y ]
        do 
        read -p "Введите полный пусть до файла(ов): " files
        echo "Эта папка (у/n) ? "
        read answer
        case $answer in
                "y") all=-r ;;
                "n") all="" ;;
                *) $0 ;;
        esac
        read -p "Введите хост: " host
        read -p "Введите место на удаленном сервере: " place
        sudo scp $all $files $host:$place 
        echo "Файлы отправлены"
        break 
        done
}

read -p "Нужно ли отправить файлы? (y/n): " answer2
case $answer2 in 
        "y") send 
             $0
             ;;
        *) exit
             ;;
esac
```
:white_check_mark: ***Запускаем скрипты в порядке очереди и только после создания сервера ВПН***
***

## Создание сервера с OpenVPN.
### На сервере создадим нужные нам скрипты.
+ Скрипт установки программы и создания необходимых нам директории
```bash
#!/bin/bash
systemctl is-active --quiet openvpn-server@server.service
if [ $? -eq 0 ]
then
        echo "ВПН сервер запущен и работает"
        sudo systemctl restart openvpn-server@server.service
else
        echo "Либо в режиме inactive либо ВПН сервер не установлен"
        echo "Установим или перезапустим"
        sudo apt-get update > /dev/null
        sudo systemctl -f enable openvpn-server@server.service
        sudo systemctl restart openvpn-server@server.service || sudo apt-get install openvpn
        echo "Готово, теперь приступим к созданию конфиг файлов"
fi
echo "Создание директории где лежит пример конфиг файла клиентов"
if [ -d /home/$user/clients ]
then
        echo "Папка для образца уже существует"
        sudo cp /usr/share/doc/openvpn/examples/sample-config-files/client.conf /home/$user/clients/
        echo "Скопировали образец в нужную нам папку"
else
        mkdir /home/$user/clients
        echo "Папку для образца создали "
        echo "Теперь копируем в эту папку сам образец"
        sudo cp /usr/share/doc/openvpn/examples/sample-config-files/client.conf /home/$user/clients/
fi
sudo cp /usr/share/doc/openvpn/examples/sample-config-files/server.conf /etc/openvpn/server
echo "Заодно скопировали образец конфигурации для сервера"
make_config () {
        echo "Создание конфиг файлов готовых для клиентов"
        read -p "Введите название первого конфига с ключом: " "client"
        test -f /home/$user/clients/$client
        if [ $? -eq 0 ]
        then
                /home/$user/make_config $client
                if [ $? -eq 0 ]
                then
                        echo "Создали конфигурационный файл"
                        echo
                        read -p "Есть ли еще файлы для создания ? (y/n) " answer
                        case $answer in
                                "y") $0 ;;
                                *) echo "Продолжаем настройки" ;;
                        esac
                else
                        echo "Проверить файл make_config"
                fi
        else
                echo "Ошибка, такого файла не существует или перекиньте файлы с ЦС"
                exit 1
        fi
}
make_config
echo "Теперь приступим к настройке фаерволла"
/home/$user/Iptables.sh
echo "ВСЕ ГОТОВО"
exit 0
```
+ Скрипт установки правил на firewall для туннелирования трафика
```bash
#!/bin/bash

ip addr
read -p "Название интерфейса: " eth
read -p "Название протокола: " proto
read -p "Номер порта: " port

Tunnel () {

echo "Очистим таблицу и запустим туннелирование"

sudo iptables -F

sudo iptables -A INPUT -i "$eth" -m state --state NEW -p "$proto" --dport "$port" -j ACCEPT

#Allow TUN interface connections to the VPN-server

sudo iptables -A INPUT -i tun+ -j ACCEPT

#Allow TUN interface connections to be forwarded through other interfaces

sudo iptables -A FORWARD -i tun+ -j ACCEPT
sudo iptables -A FORWARD -i tun+ -o "$eth" -m state --state RELATED,ESTABLISHED -j ACCEPT
sudo iptables -A FORWARD -i "$eth" -o tun+ -m state --state RELATED,ESTABLISHED -j ACCEPT

#NAT the VPN client traffic to the internet

sudo iptables -t nat -A POSTROUTING -s 10.8.0.0/24 -o "$eth" -j MASQUERADE
}

read -p "Данные введены правильно? (y/n) " y
case $y in
        "y") Tunnel
        ;;
        "n") $0
        ;;
        *) echo "Завершение настроек"
        exit ;;
esac
echo "Туннелирование запущено"
echo "Приступим к настройке маршрутирования"
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
echo "Запустили маршрутирование"
```
+ Скрипт для создания готовых файлов в отдельной папке  
```bash
#!/bin/bash
echo "Создается папка для сохранения конфиг файлов готовых для клиентов"
if [ -d /home/$user/config_clients ]
then
        echo "Такая папка уже существует"
else
        mkdir /home/$user/config_clients
        echo "Создали папку"
fi

KEY_DIR=/home/$user/clients
OUTPUT_DIR=/home/$user/config_clients
BASE_CONF=/home/$user/clients/client.conf

echo "Прроверим есть ли нужные файлы"
test -f $KEY_DIR/"$1"
if [ $? -eq 0 ]
then
        sudo chown $user:$user $BASE_CONF
        echo "Приступаем к созданию конфиг файлов для клиентов"
else
        echo "Файлы еще не отправили"
        echo "Отправьте файлы и перезвпустите главный скрипт OPENVPN"
        exit 1
fi

cat $KEY_DIR/"$1" >> $BASE_CONF
if [ $? -eq 0 ]
then
        mv $BASE_CONF $OUTPUT_DIR/"$1"
        echo "Получилось"
else
        echo "Не получилось"
        exit 1
fi

echo "Конфигурационный файл клиента готов"
echo "скрипт make_config.sh завершен успешно "
```
+ Отдельно создаем дополнительный секретный ключ
  + в терминале вводим команду
  ```bash 
  /usr/sbin/openvpn --genkey --secret ta.key
  ```
  + копируем этот ключ в папку /etc/openvpn/server/ 
  ```bash
  cp ta.key /etc/openvpn/server/
  ```
  + добавляем в файл конфигурации созданных для клиентов
  ```bash
  cat /etc/openvpn/server/ta.key >> /путь_до_вашего_файла
  ```
+ В этом же файле введем изменения 
  + закомментируем строки
  ```bash
  ca ca.crt
  cert client.crt
  key client.key
  ```
  + изменим некоторые строки 
  ```bash 
  tls-auth >> tls-crypt
  cipher AES-256-CBC >> cipher AES-256-GCM
  ```
  + добавим после `cipher AES-256-GCM` строки 
  ```bash
  auth SHA256
  key-direction 1
  ```
+ Добавим некоторые изменения в файл конфигурации сервера `/etc/openvpn/server/server.conf` 
  + убрать комментарии в строках
  ```bash 
  redirect-gateway def1 bypass-dhcp
  push "dhcp-options DNS 8.8.8.8"
  push "dhcp-options DNS 8.8.4.4"
  user nobody
  group nobody
  ```
  + поменять название ключа Диффи-Хеллмана на название нашего *(у вас возможно другое название)*
  ```bash 
  dh dh2048.pem >> dh dh.pem
  ```
  + поменять строки 
  ```bash 
  tls-auth >> tls-crypt
  cipher AES-256-CBC >> cipher AES-256-GCM
  ```
  + добавить после `cipher AES-256-GCM` строку
  ```bash 
  auth SHA256
  ```
+ После всех изменений перезапускаем сервер и проверяем статус 
  + вводим в терминале команду
  ```bash
  sudo systemctl restart openvpn-server@server.service
  sudo systemctl status openvpn-server@server.service
  ```
:white_check_mark: ***Теперь можете использовать файл конфигурации для подключения к ВПН серверу с клиентского устройства***
### Установим `openvpn_exporter` для мониторинга состояния сервера ВПН.
+ Установим компилятор Golang
  + вводим в терминале команды
  ```bash
  wget https://golang.org/dl/go1.15.2.linux-amd64.tar.gz
  tar -C /usr/local -xzf go1.15.2.linux-amd64.tar.gz
  export PATH=$PATH:/usr/local/go/bin
  source ~/.bashrc
  ```
  + скачиваем сам `openvpn_exporter` с конфигурационными файлами в отдельную папку
  ```bash
  mkdir openvpn_exporter
  cd openvpn_exporter/
  wget https://github.com/kumina/openvpn_exporter/archive/v0.3.0.tar.gz
  tar -xzf v0.3.0.tar.gz
  ```
  + открываем с помощью редактора файл `main.go` 
  ```bash
  cd openvpn_exporter-0.3.0/
  vi main.go
  ```
  + проверяем где находится ваш файл журнала состояния
  ```py
  func main() {
  var (
    listenAddress      = flag.String("web.listen-address", ":9176", "Address to listen on for web interface and telemetry.")
    metricsPath        = flag.String("web.telemetry-path", "/metrics", "Path under which to expose metrics.")
    openvpnStatusPaths = flag.String("openvpn.status_paths", "/var/log/openvpn/status.log", "Paths at which OpenVPN places its status files.")
    ignoreIndividuals  = flag.Bool("ignore.individuals", false, "If ignoring metrics for individuals")
  )
  flag.Parse()
  ```
  + запускаем создание бинарного файла и копируем его в `/usr/local/bin/`
  ```bash
  go build -o openvpn_exporter main.go
  cp openvpn_exporter /usr/local/bin/
  ```
  + создаем отдельный `systemd unit` 
  ```bash
  [Unit]
  Description=Prometheus OpenVPN Node Exporter
  Wants=network-online.target
  After=network-online.target
 
  [Service]
  Type=simple
  ExecStart=/usr/local/bin/openvpn_exporter
 
  [Install]
  WantedBy=multi-user.target
  ```
  + добавляем в автозапуск и проверяем статус
  ```bash
  systemctl daemon-reload
  systemctl enable --now openvpn_exporter.service
  systemctl status openvpn_exporter
  ```
:white_check_mark: Теперь можем настроить сервер Prometheus.
***
## Настроим сервер Prometheus для просмотра состояния сервера ВПН.
### Скрипт установит все необходимые компоненты, от нас требуется заполнить все необходимые данные.
```bash
#!/bin/bash
echo "Обновим сервер"
sudo apt-get update > /dev/null
read -p "Установлена ли программа ? (y/n) " answer
case $answer in
        "y") echo "Перезагрузим сервисы"
             sudo systemctl restart prometheus.service
             sudo systemctl restart prometheus-node-exporter.service
             echo "Готово"
             ;;
        "n") echo "Установим программы"
             sudo apt-get install prometheus prometheus-node-exporter
             if [ $? -eq 0 ]
             then
                  echo "Добавим автозапуск во время загрузки"
                  sudo systemctl enable prometheus.service
                  sudo systemctl enable prometheus-node-exporter.service
                  echo "Проверим статус"
                  sudo systemctl status prometheus.service
             else
                  echo "Во время устновки что-то пошло не так"
             fi
             ;;
        *) echo "Неверный ввод"
           echo "Запустите заново"
           exit
           ;;
esac
echo
read -p "Надо ли добавлять Openvpn_exporter? (y/n) " answer2
case $answer2 in
        "y") read -p "Jb_name: " jobname #название впн-сервера
             read -p "Scrape_interval: " num #интервал для обновления метрик
             read -p "Ip_address: " ip #ip адрес сервера ВПН с которого будем получать метрики
             read -p "Port_number: " port #по умолчанию порт 9176
             echo
             echo "Добавляем вот этот Таргет"
             echo "  - job_name: '$jobname'" | sudo tee -a /etc/prometheus/prometheus.yml
             echo "    scrape_interval: $num" | sudo tee -a /etc/prometheus/prometheus.yml
             echo "    static_configs:" | sudo tee -a /etc/prometheus/prometheus.yml
             echo "      - targets: ['$ip:$port']" | sudo tee -a /etc/prometheus/prometheus.yml
             echo "Обновим сервер"
             sudo systemctl restart prometheus
             sudo systemctl status prometheus
             ;;
        *)   echo "Обновим сервер и можно проверить статус"
             sudo systemctl restart prometheus.service
             ;;
esac
```
### Открыв в браузере сайт мониторинга, должны увидеть `targets`:
<image src="http://soalcepat.com/wp-content/uploads/2022/01/Prometheus-openvpn-exporter.png" alt="Сервер мониторинга">
:white_check_mark: Мониторинг готов и можно просматривать нужные метрики.

***
## Сервер Backup *(резервного хранения)*.
### Необходим для повышения безопасности и облегчения автоматизации развёртывания сервисов.
+ Данный скрипт будет делать архивацию и сжатие необходимых файлов на сервере
```bash 
#!/bin/bash
dir=/home/$user
find $dir -name *.tar.gz -exec rm -rf {} \; #удаляем старый архив с данными
tar -czf /home/$user/name_backup_$(date +'%d%m%Y').tar.gz -C / home/$user/ #создаем новый сжатый архив с датой создания
scp /home/$user/$back_up_files.tar.gz $hostname@ip_address:/home/$place #Отправка на сервер Бэкапа
```
+ После создания скрипта бэкапирования необходимо добавить в планировщик задач *(например `Crontab`)* командой `crontab -e`
  + в данном примере выставлена команда делать резерв файлов раз в неделю
  ```bash
  0 0 * * 0 /home/путь/до/скрипта
  ```
:white_check_mark: Готовим Backup сервер на случай непредвиденных обстоятельств. 
