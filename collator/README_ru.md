---
title: Как запустить коллатор Robonomics
contributors: [dergudzon, Leemo94]
tools:
  - Ubuntu 24.04.4
    https://releases.ubuntu.com/24.04/
  - Robonomics 3.3.0
    https://github.com/airalab/robonomics/releases/tag/v3.3.0
---

В настоящее время сеть Robonomics в основном поддерживается первоначальными разработчиками, но любой может поддержать проект. Каждый дополнительный полный узел блокчейна помогает ему стать более устойчивым и отказоустойчивым. Бинарные файлы узлов Robonomics доступны в [релизах](https://github.com/airalab/robonomics/releases).

## Что такое коллатор

Коллатор является частью парачейна Robonomics. Этот тип узла создает новые блоки для цепочки Robonomics.

>Коллаторы поддерживают парачейны, собирая транзакции парачейна от пользователей и создавая доказательства перехода состояния для валидаторов цепочки ретрансляции. Другими словами, коллаторы поддерживают парачейны, агрегируя транзакции парачейна в кандидаты на блоки парачейна и создавая доказательства перехода состояния для валидаторов на основе этих блоков.NODE_NAME

Вы можете узнать больше о коллаторах на соответствующей [странице вики Polkadot](https://wiki.polkadot.network/learn/learn-collator)

В парачейне Robonomics каждый коллатор получает вознаграждение (**0.001598184 XRT**) за каждый блок, который строит коллатор (вознаграждение начисляется при запечатывании блоков в цепочку).
Также коллатор, который создает блок, получает **50% от комиссий за транзакции**, содержащиеся в блоке, который они создают.

## Требования
В этой статье мы используем следующие характеристики:
+ 4 vCPU
+ 1Tb пространства NVMe для баз данных. Требуется возможность расширения этого дискового пространства.
+ 8 ГБ ОЗУ

На момент написания статьи полные ноды Робономики занимают достаточно много дискового пространства:
- в сети Polkadot: ~700G
- в сети Kusama: ~900G

При этом размер базы данных постоянно увеличивается, в связи с этим для разворачивания полной ноды Робономики требуется использовать диск с возможностью его увеличения.

## Важная информация
Мы используем некоторые переменные в этих инструкциях, и вам нужно будет заменить значения на свои во всех командах:
+ **${NODE_NAME}** - имя узла. Пример: *my-robonomics-collator*
+ **${BASE_PATH}** - путь к директории, в которой будет храниться база данных ноды. Как уже упоминалось выше, крайне важно, чтобы у вас была возможность увеличения размера диска, на котором находится данная директория. Пример: */mnt/HC_Volume_16056435/*
+ **${CHAIN}** - название сети, в которой вы разворачиваете узел. На момент написания статьи это может быть либо **polkadot** либо **kusama**. Обратите внимание, что нельзя одновременно использовать обе сети при настройке одной ноды, необходимо везде указать одну и ту же выбранную сеть.
+ **${POLKADOT_ACCOUNT_ADDRESS}** - адрес вашего аккаунта, на который будут приходить награды за сборку блоков, в формате SS58. Пример: `4Gp3QpacQhp4ZReGhJ47pzExQiwoNPgqTWYqEQca9XAvrYsu`

Запуск коллатора Робономики состоит из нескольких шагов:
1) Предварительная подготовка сервера.
2) Запуск коллатора, который можно выполнить в двух вариантах: в Docker либо как systemd сервис.

## 1. Предварительная подготовка сервера
1. Для начала обновите ПО на сервере и перезагрузите его, если это требуется:
``` 
apt update && apt -y upgrade
reboot
```
2. Убедитесь, что время на сервере синхронизировано:
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

## 2. Вариант 1: Запуск коллатора Робономики как systemd сервис

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
--chain=${CHAIN} \
--name="${NODE_NAME}" \
--collator \
--lighthouse-account="${POLKADOT_ACCOUNT_ADDRESS}" \
--telemetry-url="wss://telemetry.parachain.robonomics.network/submit/ 0" \
--base-path="${BASE_PATH}" \
-- \
--sync=warp    

[Install]
WantedBy=multi-user.target
```

3. Сохраните этот файл, затем включите и запустите службу:
```
systemctl enable robonomics.service
systemctl start robonomics.service
```

Коллатор запущен, его лог можно посмотреть при помощи следующей команды:
```
journalctl -u robonomics.service -f
```

URL телеметрии: https://telemetry.parachain.robonomics.network/

## 2. Вариант 2: Запуск коллатора Робономики в Docker
1. Установите Docker по инструкции с официального сайта: https://docs.docker.com/engine/install/ubuntu/#install-using-the-repository

2. Чтобы ранее созданный пользователь *robonomics* мог запускать контейнеры, создайте  группу *docker* и добавьте его в нее:
```
groupadd docker
usermod -aG docker robonomics
```

3. Перейдите в пользователя robonomics и создайте директорию *robonomics_collator*:
```
su robonomics
mkdir /home/robonomics/robonomics_collator && cd /home/robonomics/robonomics_collator
```

4. Создайте в этой директории файл *docker-compose.yml* со следующим содержимым:
```
services:
  robonomics-collator:
    container_name: robonomics-collator
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
      "--collator",
      "--name=${NODE_NAME}",
      "--lighthouse-account=${LIGHTHOUSE_ACCOUNT}",
      "--telemetry-url=wss://telemetry.parachain.robonomics.network/submit/ 0",
      "--base-path=/robonomics/base/",
      "--",
      "--sync=warp",
    ]
```
Не забудьте заменить переменные `${CHAIN}`, `${NODE_NAME}`, `${LIGHTHOUSE_ACCOUNT}` и `${BASE_PATH}` на нужные вам.

5. После сохранения файла останется только запустить docker compose как сервис. Это нужно сделать, находясь в той же директории (/home/robonomics/robonomics_collator):
```
docker compose up -d
```

Коллатор запущен, его лог можно посмотреть при помощи следующей команды:
```
docker compose logs -f -n50
```

URL телеметрии: https://telemetry.parachain.robonomics.network/

