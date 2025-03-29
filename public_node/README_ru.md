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

На данный момент у Робономики есть парачейны в двух сетях: Kusama и Polkadot, поэтому вы можете развернуть публичные ноды в них обеих.

Для этого нужно будет выполнить следующие шаги, которые мы более подробно распишем ниже:
1) Запустить публичную ноду на сервере, либо в виде systemd сервиса, либо в docker compose. Для этого потребуется сервер с соответствующими техническими характеристиками.
2) Настроить конфигурацию nginx и получить сертификат let's encrypt. Для этого потребуется направить на сервер доменное имя.
3) Отправить pull request в репозиторий [polkadot-js/apps](https://github.com/polkadot-js/apps) для включения развернутой ноды в список публичных нод на портале.

## 1. Запуск публичной ноды Робономики
### Требования
В этой статье мы используем следующие характеристики:
+ 4 vCPU
+ 1Tb пространства NVMe для баз данных. Требуется возможность расширения этого дискового пространства.
+ 8 ГБ ОЗУ

На данный момент полные ноды Робономики занимают достаточно много дискового пространства:
- в сети Polkadot: 660G
- в сети Kusama: 880G

При этом размер базы данных постоянно увеличивается, в связи с этим для разворачивания полной ноды Робономики требуется использовать диск с возможностью его увеличения.

### Важная информация
Мы используем некоторые переменные в этих инструкциях, и вам нужно будет заменить значения на свои во всех командах:
+ **%NODE_NAME%** - имя узла. Пример: *my-robonomics-kusama-collator*
+ **%BASE_PATH%** - путь к директории, в которой будет храниться база данных ноды. Как уже упоминалось выше, крайне ранее, чтобы у вас была возможность увеличения размера диска, на котором находится данная директория. Пример: */mnt/HC_Volume_16056435/*
+ **%CHAIN%** - название сети, в которой вы разворачиваете узел. На данный момент это может быть либо **polkadot** либо **kusama**.
+ **%DOMAIN_NAME%** - доменное имя, которое находится в вашем владении, и которое будет использоваться для доступа к публичной ноде

### Запуск публичной ноды Робономики как службы

1. Создайте пользователя для службы с домашним каталогом сделайте этого пользователя владельцем директории, в которой будет база данных публичной ноды
    ```
    useradd -m robonomics
    chown -R robonomics:robonomics %BASE_PATH%
    ```
2. Загрузите, извлеките и переместите исполняемый файл Robonomics в каталог */usr/local/bin/*. Вам нужно заменить *$ROBONOMICS_VERSION* на текущую версию Robonomics в командах в этом разделе. Текущую версию можно найти на [странице релизов репозитория Robonomics на github](https://github.com/airalab/robonomics/releases).
   ```
   wget https://github.com/airalab/robonomics/releases/download/v$ROBONOMICS_VERSION/robonomics-$ROBONOMICS_VERSION-x86_64-unknown-linux-gnu.tar.gz
   tar -xf robonomics-$ROBONOMICS_VERSION-x86_64-unknown-linux-gnu.tar.gz
   mv ./robonomics /usr/local/bin/
   ```

3. Создайте файл службы systemd с именем *robonomics.service*:
    ```
    nano /etc/systemd/system/robonomics.service
    ```

    И добавьте следующие строки в файл службы:
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
      --name="%NODE_NAME%" \
      --chain=%CHAIN% \
      --base-path="%BASE_PATH%" \
      --telemetry-url="wss://telemetry.parachain.robonomics.network/submit/ 0" \
      --rpc-external \
      --rpc-cors=all \
      --state-pruning=archive \
      --blocks-pruning=archive \
      -- \
      --chain=%CHAIN% \
      --sync=warp

    [Install]
    WantedBy=multi-user.target
    ```

4. Сохраните этот файл, затем включите и запустите службу:
    ```
    systemctl enable robonomics.service
    systemctl start robonomics.service
    ```

URL телеметрии, нода должна появиться спустя некоторое время (обычно это пара минут): https://telemetry.parachain.robonomics.network/

Логи севриса можно отслеживать с помощью данной команды: `journalctl -u robonomics.service -f`

После запуска сервиса, нода начнет синхронизацию с указанным в параметрах запуска чейном.

## 2. Настройка nginx и получение сертификата let's encrypt

На данном этапе необходимо, чтобы ваше доменное имя %DOMAIN_NAME% уже было направлено на ваш сервер. Это делается в настройках DNS в панели управления вашим доменным именем, при помощи А-записи

1. Установите nginx, certbot, python3-certbot-nginx и создайте конфигурационный файл nginx для вашего доменного имени
    ```
    apt install nginx certbot python3-certbot-nginx
    nano /etc/nginx/sites-available/%DOMAIN_NAME%.conf
    ```

2. Впишите в данный файл следующую заготовку:
    ```
    server {
      listen 80;
      listen [::]:80;

      server_name %DOMAIN_NAME%;
    }
    ```

3. Затем сохраните файл и выполните команду для запроса сертификата let's encrypt и его деплоя в конфиг nginx:
    ```
    certbot --nginx -d %DOMAIN_NAME% --agree-tos
    ```
В процессе потребуется указать email администратора доменного имени.

4. Далее, откройте ранее созданный файл, обратите внимание, что certbot создал в нем секцию с https в котором прописал полученные сертификаты. Нам остается только прописать нужные локации и сохранить файл.

TODO: скриншот с содержимым файла после выполнения certbot

```
окончательное содержимое файла 
```

5. Настало время включить хост, проверить конфигурацию nginx, если никаких ошибок в конфигурационных файлах nginx нет, перезагрузить сервис nginx

    ```
    ln -s /etc/nginx/sites-available/%DOMAIN_NAME%.conf /etc/nginx/sites-enabled/
    nginx -t
    systemctl reload nginx 
    ```

## 3. Отправка запроса на включение запущенной публичной ноды в polkadot{.js} портал.

1) fork https://github.com/polkadot-js/apps
2) git clone https://github.com/MINE/apps
3) edit - add new provider line, 
example pr: https://github.com/polkadot-js/apps/pull/10309/files
4) commit
5) push
6) pr
7) wait