# SNMPv3 (все устройства)
```bash
snmp-server enable snmp
snmp-server view view1 .1 included
snmp-server group reaskills v3 auth read view1
snmp-server user snmpuser group reaskills auth md5 snmppass
```

> **Доступ по SNMP:**  
> - Группа: `reaskills`  
> - Пользователь: `snmpuser` / Пароль: `snmppass` (настраивается на стороне клиента)  
> - Доступ разрешен только из сети управления (VLAN 100)


# Напишите скрипт вызова утилиты snmpwalk на CR-SRV

> - Скрипт должен быть доступен по имени get_info без явного указания пути и расширения, из любого места в системе.
> - В качестве аргумента скрипт должен принимать доменное имя устройства
> - По умолчанию скрипт должен выводить информацию на экран, но при использовании ключа --dump перед именем устройства, вывод должен быть сохранен в файл имяустройства_текущаядатаивремя_report.txt

```bash
nano /bin/get_info
chmod +x /bin/get_info
```

```bash
#!/bin/bash

# Обработка флага --dump
if [ "$1" = "--dump" ]; then
    DUMP=1
    shift
else
    DUMP=0
fi

# Проверка аргумента
if [ -z "$1" ]; then
    echo "Использование: get_info [--dump] <доменное_имя>"
    exit 1
fi

HOST="$1"
FILE="${HOST}_$(date +%Y%m%d_%H%M%S)_report.txt"

# Вызов snmpwalk для CR-SRV (предполагаем, что CR-SRV — это шлюз с доступом к устройству)
CMD="snmpwalk -v2c -c public $HOST"

# Вывод в файл или на экран
if [ $DUMP -eq 1 ]; then
    $CMD > "$FILE"
    echo "Сохранено в $FILE"
else
    $CMD
fi

```
