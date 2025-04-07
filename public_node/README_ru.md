---
title: Как запустить публичную ноду Робономики
contributors: [dergudzon]
tools:
  - Ubuntu 24.04.4
    https://releases.ubuntu.com/24.04/
  - Robonomics 3.3.0
    https://github.com/airalab/robonomics/releases/tag/v3.3.0
---

В настоящее время сеть Robonomics в основном поддерживается первоначальными разработчиками, но любой может поддержать проект. Каждый дополнительный полный узел блокчейна помогает ему стать более устойчивым и отказоустойчивым. Исполняемые файлы узлов Robonomics доступны в [релизах](https://github.com/airalab/robonomics/releases).

## Кратко о публичных нодах Робономики
Под публичной нодой Робономики мы подразумеваем полную архивную ноду Робономики, включенную в список публичных нод на портале [polkadot{.js}](https://polkadot.js.org/apps/)

Подробнее о вариантах запуска нод в экосистеме Polkadot вы можете прочитать на соответствующей странице их документации: [Running a Node](https://docs.polkadot.com/infrastructure/running-a-node/)

На момент написания статьи у Робономики есть парачейны в двух сетях: Kusama и Polkadot, поэтому вы можете развернуть публичные ноды и там и там.

Для этого нужно будет выполнить следующие шаги, которые мы более подробно распишем ниже:
1) Запустить публичную ноду на сервере, либо в виде systemd сервиса, либо в docker compose. Для этого потребуется сервер с соответствующими техническими характеристиками.
2) Настроить конфигурацию nginx и получить сертификат let's encrypt. Для этого потребуется направить на сервер доменное имя.
3) Отправить pull request в репозиторий [polkadot-js/apps](https://github.com/polkadot-js/apps) для включения запущенной ноды в список публичных нод на портале.


## 1. Запуск публичной ноды Робономики
### Требования
В этой статье мы используем следующие характеристики:
+ 4 vCPU
+ 1Tb пространства NVMe для баз данных. Требуется возможность расширения этого дискового пространства.
+ 8 ГБ ОЗУ

На момент написания статьи полные ноды Робономики занимают достаточно много дискового пространства:
- в сети Polkadot: ~700G
- в сети Kusama: ~900G

При этом размер базы данных постоянно увеличивается, в связи с этим для разворачивания полной ноды Робономики требуется использовать диск с возможностью его увеличения.

### Важная информация
Мы используем некоторые переменные в этих инструкциях, и вам нужно будет заменить значения на свои во всех командах:
+ **${NODE_NAME}** - имя узла. Пример: *my-robonomics-polkadot-public-node*
+ **${BASE_PATH}** - путь к директории, в которой будет храниться база данных ноды. Как уже упоминалось выше, крайне важно, чтобы у вас была возможность увеличения размера диска, на котором находится данная директория. Пример: */mnt/HC_Volume_16056435/*
+ **${CHAIN}** - название сети, в которой вы разворачиваете узел. На момент написания статьи это может быть либо **polkadot** либо **kusama**. Обратите внимание, что нельзя одновременно использовать обе сети при настройке одной ноды, необходимо везде указать одну и ту же выбранную сеть.
+ **${DOMAIN_NAME}** - доменное имя, предназначенное использования на вашей публичной ноде, и направленное на целевой сервер

Запуск публичной ноды Робономики состоит из нескольких шагов:
1) Предварительная подготовка сервера.
2) Запуск публичной ноды, который можно выполнить в двух вариантах: в Docker либо как systemd сервис.
3) Добавление запущенной публичной ноды в список на портале https://polkadot.js.org/apps

## 1. Предварительная подготовка сервера
1. Для начала обновите ПО на сервере и перезагрузите его, если это требуется:
``` 
apt update && apt -y upgrade
reboot
```

2. Установите необходимое ПО:
```
apt install -y nginx certbot python3-certbot-nginx
```
    
3. Убедитесь, что время на сервере синхронизировано:
```
timedatectl
```
Если в выводе данной команды есть строки **`System clock synchronized: yes`** и **`NTP service: active`**, то все в порядке. Если это не так, то нужно будет синхронизировать время, например, при помощи **`ntpd`**:
```
timedatectl set-ntp no
apt install ntp
```
3. Создайте пользователя для запуска коллатора и сделайте его владельцем директории, в которой планируется хранение базы данных коллатора:
```
useradd robonomics -u 4200 -m -s /bin/bash
chown -R robonomics:robonomics ${BASE_PATH}
```

На этом предварительная подготовка сервера закончена, можно переходить непосредственно к запуску коллатора Робономики. Как уже говорилось выше, коллатор Робономики можно запустить либо как systemd сервис, либо при помощи Docker. Опишем оба этих варианта.

## 2. Вариант 1: Запуск публичной ноды Робономики как systemd сервис
1. Скачайте, извлеките и переместите бинарный файл Робономики в каталог */usr/local/bin/*. В данной инструкции используется версия v3.3.0, ниже представлены команды по ее скачиванию и распаковке. Актуальную версию Робономики можно найти на [странице релизов проекта на github](https://github.com/airalab/robonomics/releases).
```
wget https://github.com/airalab/robonomics/releases/download/v3.3.0/robonomics-v3.3.0-ubuntu-x86_64.tar.gz
tar -xvf robonomics-v3.3.0-ubuntu-x86_64.tar.gz
mv robonomics /usr/local/bin/
```

2. Создайте файл службы systemd с именем *robonomics.service*:
```
nano /etc/systemd/system/robonomics.service
```

И добавьте в него следующие строки (не забудьте заменить необходимые переменные на ваши значения):
```
[Unit]
Description=robonomics
After=network.target

[Service]
User=robonomics
Group=robonomics
Type=simple
Restart=on-failure

ExecStart=/usr/local/bin/robonomics \
  --name="${NODE_NAME}" \
  --chain=${CHAIN} \
  --base-path="${BASE_PATH}" \
  --telemetry-url="wss://telemetry.parachain.robonomics.network/submit/ 0" \
  --rpc-external \
  --rpc-cors=all \
  --state-pruning=archive \
  --blocks-pruning=archive \
  -- \
  --chain=${CHAIN} \
  --sync=warp

[Install]
WantedBy=multi-user.target
```

3. Сохраните этот файл, затем включите и запустите службу:
```
systemctl enable robonomics.service
systemctl start robonomics.service
```

Публичная нода запущена, ее лог можно посмотреть при помощи следующей команды:
```
journalctl -u robonomics.service -f
```

URL телеметрии: https://telemetry.parachain.robonomics.network/

## 2. Вариант 2: Запуск публичной ноды Робономики в Docker
1. Установите Docker по инструкции с официального сайта: https://docs.docker.com/engine/install/ubuntu/#install-using-the-repository

2. Чтобы ранее созданный пользователь *robonomics* мог запускать контейнеры, создайте  группу *docker* и добавьте его в нее:
```
groupadd docker
usermod -aG docker robonomics
```

3. Перейдите в пользователя robonomics и создайте директорию *robonomics_public_node*:
```
su robonomics
mkdir /home/robonomics/robonomics_public_node && cd /home/robonomics/robonomics_public_node
```

4. Создайте в этой директории файл *docker-compose.yml* со следующим содержимым:
```
services:
  robonomics-public-node:
    container_name: robonomics-public-node
    image: robonomics/robonomics:v3.3.0
    user: 4200:4200
    restart: always
    ports:
      - 30333:30333 # p2p port
      - 9944:9944 # ws port
    volumes:
      - ${BASE_PATH}:/robonomics:rw
    command: [
      "robonomics",
      "--chain=${CHAIN}",
      "--name=${NODE_NAME}",
      "--telemetry-url=wss://telemetry.parachain.robonomics.network/submit/ 0",
      "--base-path=/robonomics/base/",
      "--rpc-external",
      "--rpc-cors=all",
      "--state-pruning=archive",
      "--blocks-pruning=archive",
      "--",
      "--sync=warp",
    ]
```
Не забудьте заменить переменные `${CHAIN}`, `${NODE_NAME}` и `${BASE_PATH}` на нужные вам.

5. После сохранения файла останется только запустить docker compose как сервис. Это нужно сделать, находясь в той же директории (/home/robonomics/robonomics_public_node):
```
docker compose up -d
```

Коллатор запущен, его лог можно посмотреть при помощи следующей команды:
```
docker compose logs -f -n50
```

URL телеметрии: https://telemetry.parachain.robonomics.network/


## 3. Настройка nginx и получение сертификата let's encrypt

На данном этапе необходимо, чтобы ваше доменное имя `${DOMAIN_NAME}` уже было направлено на ваш сервер. Это обычно делается в настройках DNS в панели управления регистратора вашего доменного имени, при помощи А-записи.

1. Установите nginx, certbot, python3-certbot-nginx и создайте конфигурационный файл nginx для вашего доменного имени
```
apt install nginx certbot python3-certbot-nginx
nano /etc/nginx/sites-available/${DOMAIN_NAME}.conf
```

2. Впишите в данный файл следующую заготовку:
```
server {
  listen 80;
  listen [::]:80;

  server_name ${DOMAIN_NAME};
}
```

3. Сохраните файл, создайте ссылку на него в директории /etc/nginx/sites-enabled/ и перезапустите nginx
```
ln -s /etc/nginx/sites-available/${DOMAIN_NAME}.conf /etc/nginx/sites-enabled/
systemctl reload nginx
```

4. Выполните команду для запроса сертификата let's encrypt и его деплоя в конфиг nginx:
```
certbot --nginx -d ${DOMAIN_NAME}
```
В процессе нужно будет указать email администратора доменного имени.

5. Далее, откройте ранее созданный файл, обратите внимание, что certbot создал в нем секцию с https в котором прописал полученные сертификаты. Нам остается только прописать нужные локации и сохранить файл.

Примерно так выглядит содержимое файла после выполнения certbot:
  
  ```
  server {

    server_name ${DOMAIN_NAME};

    listen [::]:443 ssl ipv6only=on; # managed by Certbot
    listen 443 ssl; # managed by Certbot
    ssl_certificate /etc/letsencrypt/live/${DOMAIN_NAME}/fullchain.pem; # managed by Certbot
    ssl_certificate_key /etc/letsencrypt/live/${DOMAIN_NAME}/privkey.pem; # managed by Certbot
    include /etc/letsencrypt/options-ssl-nginx.conf; # managed by Certbot
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem; # managed by Certbot
  }
  server {
    if ($host = ${DOMAIN_NAME}) {
        return 301 https://$host$request_uri;
    } # managed by Certbot

    listen 80;
    listen [::]:80;

    server_name ${DOMAIN_NAME};
      return 404; # managed by Certbot
  }
  ```

Нам остается только добавить **location /** в секцию https сервера, итоговое содержимое файла получится таким:
```
server {
  server_name ${DOMAIN_NAME};

  location / {
      proxy_pass http://127.0.0.1:9944;
      proxy_set_header X-Real-IP $remote_addr;
      proxy_set_header Host $host;
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

      proxy_http_version 1.1;
      proxy_set_header Upgrade $http_upgrade;
      proxy_set_header Connection "upgrade";

      proxy_connect_timeout       300;
      proxy_send_timeout          300;
      proxy_read_timeout          300;
      send_timeout                300;
  }

  listen [::]:443 ssl ipv6only=on; # managed by Certbot
  listen 443 ssl; # managed by Certbot
  ssl_certificate /etc/letsencrypt/live/${DOMAIN_NAME}/fullchain.pem; # managed by Certbot
  ssl_certificate_key /etc/letsencrypt/live/${DOMAIN_NAME}/privkey.pem; # managed by Certbot
  include /etc/letsencrypt/options-ssl-nginx.conf; # managed by Certbot
  ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem; # managed by Certbot
}

server {
  if ($host = ${DOMAIN_NAME}) {
      return 301 https://$host$request_uri;
  } # managed by Certbot

  listen 80;
  listen [::]:80;

  server_name ${DOMAIN_NAME};
  return 404; # managed by Certbot
}
```

6. Настало время проверить конфигурацию nginx, если никаких ошибок в конфигурационных файлах nginx нет, перезагрузить сервис nginx.
```
nginx -t
systemctl reload nginx 
```
Если все хорошо, то ваша нода должна быть доступна на портале polkadot по следующей ссылке 
```
https://polkadot.js.org/apps/?rpc=wss%3A%2F%2F${DOMAIN_NAME}/#/explorer 
```
**Замените `${DOMAIN_NAME}` в ссылке на ваше доменное имя.**


## 4. Отправка запроса на включение запущенной публичной ноды в polkadot{.js} портал.

Для того, чтобы запущенная вами публичная нода Робономики, была добавлена в список публичных нод на портале polkadot, вам необходимо будет отправить pull request в их репозиторий. Для этого нужно иметь аккаунт на Github

Краткая интрукция:
1) Создать форк проекта https://github.com/polkadot-js/apps в своем аккаунте.

2) В созданном форке нужно добавить в проект строку со ссылкой на свой эндпоинт. Если вы развернули коллатор в **Robonomics Kusama**, то нужно внести правки в файле https://github.com/polkadot-js/apps/blob/master/packages/apps-config/src/endpoints/productionRelayKusama.ts в секцию с parad **2048**. Если вы развернули коллатор **Robonomics Polkadot**,то править необходимо в файле https://github.com/polkadot-js/apps/blob/master/packages/apps-config/src/endpoints/productionRelayPolkadot.ts секцию с parad **3388**. 

3) Сделать коммит со сделанными изменениями и отправить в исходный репозиторий pull request с этим коммитом. Вот, для примера, ссылка на старый pull request с добавлением эндпоинта в парачейн Robonomics Kusama: https://github.com/polkadot-js/apps/pull/10309/files

4) После этого останется только подождать, когда отправленный PR будет принят.