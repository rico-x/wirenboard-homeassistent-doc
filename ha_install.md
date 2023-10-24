# Установка Home Assistant на контроллер Wiren Board 7 с помощью docker compose

И так у нас есть контроллер Wiren Board 7 ![с расширенным корневым разделом ](https://wirenboard.com/wiki/Wbincludes:Wiren_Board_6_and_7_Rootfs_Increasing) версии wb-2310 или новее, 
после загрузки контроллера на него необходимо ![войти по ssh](https://wirenboard.com/wiki/index.php/SSH)


## Установка Docker
Статья частично повторяет описанное в ![официальной документации](https://wirenboard.com/wiki/Docker) 


### Подготовка к установке

Установите необходимые зависимости:
```
apt update && apt install ca-certificates curl gnupg lsb-release
```

Добавьте репозиторий с пакетами docker:
```
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/debian $(lsb_release -cs) stable" | tee /etc/apt/sources.list.d/docker.list > /dev/null
```

Добавьте GPG ключ для репозитория:
```
curl -fsSL https://download.docker.com/linux/debian/gpg | gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
```

Docker пока не поддерживает nftables, который заменил iptables в новом bullseye — поэтому, если у вас релиз wb-2304 и новее, надо выполнить по очереди:
```
apt install -y iptables
update-alternatives --set iptables /usr/sbin/iptables-legacy
update-alternatives --set ip6tables /usr/sbin/ip6tables-legacy
```

### Предварительная настройка

Чтобы не потерять установленный софт и его данные, обновляйте ПО контроллера только через менеджер пакетов apt.

Встроенный флеш-накопитель контроллера разбит на разделы и для пользователя отведён самый большой из них, который монтируется в папку /mnt/data. Нужно учесть эту особенность при установке программ, а также при обновлении прошивки контроллера.

Настройте симлинк для папки конфигурации:
```
mkdir /mnt/data/etc/docker && ln -s /mnt/data/etc/docker /etc/docker
```

Мы будем хранить образы на встроенном накопителе, если вам нужно больше места под образы — используйте внешнюю флешку.

Создайте папку для хранения образов:
```
mkdir /mnt/data/.docker
```

Укажите в файле настроек daemon.json созданную выше папку:

Откройте файл в редакторе:
```
nano /etc/docker/daemon.json
```

Вставьте в него строки:
```
{
  "data-root": "/mnt/data/.docker"
}
```

Сохраните и закройте файл: Ctrl+S и Ctrl+X.

### Установка

После того, как мы указали, где будут хранится контейнеры, устанавливаем сам docker:
```
apt update && apt install docker-ce docker-ce-cli containerd.io
```

## Docker compose
Для управления сценариями запуска контейнеров удобно использовать ![Docker Compose](https://docs.docker.com/compose/),
который следит за правильным порядком запусков контейнеров, имеет удобный конфиг и описание параметров запуска

### Установка

Узнаем актуальный релиз ![тут](https://github.com/docker/compose/releases/),
затем скачиваем собранный вариант нужной версии для архитектуры armv7
```
wget https://github.com/docker/compose/releases/download/v2.23.0/docker-compose-linux-armv7
mv docker-compose-linux-armv7 /usr/local/bin/docker-compose
chmod +x /usr/local/bin/docker-compose
```

### Настройка

Создаем рабочую дирректорию нвшего проекта и переходим в нее
```
mkdir /root/home-assistant-docker-compose
cd /root/home-assistant-docker-compose
```
Создаем окружение для работы:

**docker-compose.yaml** - размещен основной сценарий запуска в формате yaml, будьте внимательны при заполнении, количество пробелов имеет значение

**.env** - фаил с переменными, где удобно хранить пароли и прочие веременные для использования в конфиге, чтоб менять в одном месте не правя весь сценарий

__/config__ - директория, в которой будут размещены все конфиги запускаемых приложений

__config/home-assistant__ - конфигурационные файлы Home Assistent и ее компонентов

__config/hass-configurator__ - конфигурационные файлы ![конфигуратора](https://github.com/CausticLab/hass-configurator-docker)

__/store__ - директория с данными приложений, например файлы базы данных

__store/media__ - директория для медиаконтента Home Assistant

__store/mariadb__ - Файлы базы данных
```
mkdir config
mkdir store
mkdir config/home-assistant
mkdir config/hass-configurator
mkdir store/media
mkdir store/mariadb
nano docker-compose.yaml
```
Вставляем туда наш конфиг запуска
```
version: '3'
services:
  # HomeAssistant
  homeassistant:
    container_name: home-assistant
    image: homeassistant/home-assistant:latest
    volumes:
      # Local path where your home assistant config will be stored
      - ./config/home-assistant:/config
      - ./store/media:/media
      - /etc/localtime:/etc/localtime:ro
      - /run/dbus:/run/dbus:ro # <-- Bluetooth адаптер
    restart: unless-stopped
    network_mode: host
    environment:
      TZ: "${MYTZ}"
    privileged: true
    depends_on:
      - mariadb # MariaDB is optional (только если используем MariaDB для хранения метрик).
    labels:
      - "com.centurylinklabs.watchtower.monitor-only=true"

  # MariaDB
  mariadb:
    container_name: mariadb
    image: yobasystems/alpine-mariadb:latest
    restart: unless-stopped
    ports:
      - "3306:3306/tcp"
    environment:
      MYSQL_ROOT_PASSWORD: "${MYSQL_ROOT_PASSWORD}"
      MYSQL_DATABASE: "${HA_MYSQL_DATABASE}"
      MYSQL_USER: "${HA_MYSQL_USER}"
      MYSQL_PASSWORD: "${HA_MYSQL_PASSWORD}"
      user: "${LOCAL_USER}:${LOCAL_USER}"
      TZ: "${MYTZ}"
    volumes:
      # Local path where the database will be stored.
      - ./store/mariadb:/var/lib/mysql
    labels:
      - "com.centurylinklabs.watchtower.monitor-only=true"

  # hass-configurator
  hass-configurator:
    container_name: hass-configurator
    image: causticlab/hass-configurator-docker:latest
    restart: unless-stopped
    ports:
      - "3218:3218/tcp"
    environment:
      - HC_BASEPATH=/hass-config
      - DIRSFIRST=true
    volumes:
      - ./config/home-assistant:/hass-config
      - ./config/hass-configurator:/config
    privileged: true

# watchtower
  watchtower:
    image: containrrr/watchtower
    container_name: watchtower
    restart: unless-stopped
    volumes:
      - '/var/run/docker.sock:/var/run/docker.sock'
    environment:
      - TZ=${MYTZ}
      #- WATCHTOWER_MONITOR_ONLY=true
      - NO_COLOR=true
      - WATCHTOWER_CLEANUP=true
      - WATCHTOWER_NO_STARTUP_MESSAGE=true
      - WATCHTOWER_NOTIFICATIONS_LEVEL=info
      - WATCHTOWER_NOTIFICATIONS=shoutrrr
      - WATCHTOWER_NOTIFICATION_URL=telegram://BOTTOKEN@telegram/?channels=123456 # <-- Свой токен бота и канал в телеге куда слать оповещение о выходе обновлений
      - WATCHTOWER_SCHEDULE=0 30 10 * * *
```
Сохраните и закройте файл: Ctrl+S и Ctrl+X.


Создаем фаил с переменными
```
nano .env
```

Заполняем его примерно так:
```
MYSQL_ROOT_PASSWORD=MyVeryStrongPass123
HA_MYSQL_PASSWORD=MyHAMYSQLPass456
HA_MYSQL_DATABASE=ha_db
HA_MYSQL_USER=homeassistant
LOCAL_USER=0
MYTZ="Europe/Moscow"
```
Сохраните и закройте файл: Ctrl+S и Ctrl+X.


### Запуск
```
docker compose up -d
```

Ожидаем пока docker compose скачает нужные файлы и запустит наш проект, около 10 минут, зависит от скорости интернета

