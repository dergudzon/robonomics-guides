## Устранение неполадок
### Ошибка: "Ошибка базы данных состояния: Слишком много вставленных блоков-соседей"
Для исправления этой ошибки можно просто запустить свой коллатор в режиме архива:

1) Сначала необходимо остановить службу Robonomics:

    root@robokusama-collator-screencast:~# systemctl stop robonomics.service


2) Затем добавьте параметр `--state-pruning=archive` к части служебного файла парачейна. Пример отредактированного служебного файла:
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
    --parachain-id=2048 \
    --name="%NODE_NAME%" \
    --validator \
    --lighthouse-account="%POLKADOT_ACCOUNT_ADDRESS%" \
    --telemetry-url="wss://telemetry.parachain.robonomics.network/submit/ 0" \
    --base-path="%BASE_PATH%" \
    --state-cache-size=0 \
    --execution=Wasm \
    --state-pruning=archive \
    -- \
    --database=RocksDb \
    --execution=Wasm

    [Install]
    WantedBy=multi-user.target
    ```

3) Перезагрузите конфигурацию менеджера systemd:
    ```
    root@robokusama-collator-screencast:~# systemctl daemon-reload
    ```

4) Удалите существующую базу данных парачейна:
    ```
    root@robokusama-collator-screencast:~# rm -rf %BASE_PATH%/chains/robonomics/db/
    ```

5) Запустите службу Robonomics:
    ```
    root@robokusama-collator-screencast:~# systemctl start robonomics.service
    ```

    После этого нужно подождать синхронизации базы данных парачейна.

### Ошибка: "не удается создать модуль: настройки компиляции несовместимы с хостом"
Эта ошибка связана с параметрами виртуализации. Необходимо использовать тип "host-model" эмулируемого процессора. Вы можете настроить это на хосте виртуализации.

Однако, если вы столкнулись с этой ошибкой на любом хостинге, вам следует обратиться к технической поддержке только по этой проблеме.