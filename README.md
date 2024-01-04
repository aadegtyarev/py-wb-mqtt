# python2wb
## Описание
Обёртка для paho-mqtt с помощью которой можно работать с MQTT [Wiren Board](https://wirenboard.com) из Python.

В контроллере Wiren Board обмен информацией идёт через MQTT-брокер, где согласно конвенции создаются устройства. У Wiren Board есть штатное средство для создания сценариев автоматизации [wb-rules](https://wirenboard.com/wiki/Wb-rules), но у него есть недостатки: нет модулей сообщества под разные задачи, нельзя запустить и отладить скрипты на компьютере. Использование Python позволяет писать скрипты так, как вы привыкли и отлаживать их в привычной IDE: запускаете скрипты локально на комьютере и подключаетесь к контроллеру по MQTT.

Файлы в репозитории:
- Модуль `module/python2wb.py`
- Пример скрипта `script.py`

Делалось «для себя», без гарантий и техподдержки, только для тех, кто понимает, что делает.

## Начало работы
Чтобы начать работу:
1. клонируйте репозиторий;
2. создайте рядом с папкой `module` скрипт;
3. подключите модуль к своему скрипту, создайте объект и укажите параметры подключения к MQTT-брокеру. 

Минимальный пример скрипта:
```python
# Импорт модуля
from module.python2wb import WbMqtt

# Создание объекта и передача параметров подключения
wb = WbMqtt("wirenboard-a25ndemj.local", 1883)  # server, port, username, password

# Объявление функции для обработки подписки
def log(device_id, control_id, new_value):
    print("log: %s %s %s" % (device_id, control_id, new_value))

# Подписка на все топики устройства system
wb.subscribe('system/+', log)
# Подписка на значение LVA15 устройства metrics
wb.subscribe('metrics/load_average_15min', log)

try:
    # Зацикливание скрипта, чтобы он не завершался
    wb.loop_forever()
finally:
    # Очистка винтуальных устройств и подписок перед выходом
    wb.clear()
```

## Разворачивание проекта на контроллере
После написания и отладки проекта на компьютере надо его переместить на контроллер. Допустим, скрипт у нас будет лежать в папке `/mnt/data/bin/python2wb/`:
1. Копируем на контроллер наши файлы, например, так: `scp -r ./* root@wirenboard-a25ndemj.local:/mnt/data/bin/python2wb`
2. Снова заходим в консоль контроллера и делаем файл скрипта исполняемым `chmod +x /mnt/data/bin/python2wb/script.py`
3. Далее создаём описание сервиса `nano /etc/systemd/system/python2wb.service`, например с таким содержанием:
```
[Unit]
Description=python2wb
After=network.target

[Service]
ExecStart=python3 /mnt/data/bin/python2wb/script.py
WorkingDirectory=/mnt/data/bin/python2wb/
StandardOutput=inherit
StandardError=inherit
Restart=always
User=root

[Install]
WantedBy=multi-user.target
```
5. Запускаем сервис и помещаем его в автозапуск `systemctl start python2wb ; systemctl enable python2wb`

## Работа с контролами устройств
Обёртка скрывает от пользователя длинные имена топиков, предоставляя простой индерфейс взаимодействия с контролами устройств, позволяя читать и писать данные используя короткую запись пути `device_id/control_id`:
```python
# Запись значения в контрол устройства
wb.set("wb-mr6c_226/K1", new_value)

# Чтение значения из контрола
print(wb.get("wb-mr6c_226/K1"))
```

## Подписка на контролы
Кроме этого можно подписаться на один или несколько контролов, в том числе с использованием символа подстановки `+`. Обработка события происходит в функции обратного вызова, которую нужно задать. Функция возвращает:
- `device_id` — идентификатор устройства;
- `control_id` — идентификатор контрола `wb-gpio/A1_OUT` или список контролов `["wb-gpio/A1_OUT", "wb-gpio/A2_OUT"]`;
- `new_value` — новое значение контрола, преобразованное к одному из типов (`float`, `int`, `str`).

К одной функции можно привязать несколько подписок:
```python
# Функция обратного вызова
def log(device_id, control_id, new_value):
    print("log: %s %s %s" % (device_id, control_id, new_value))

# Подписка с использованием символов подстановки
wb.subscribe('wb-gpio/+', log)

# Подписка на один контрол
wb.subscribe("wb-mr6c_226/K1", log)

# Подписка на несколько контролов
wb.subscribe(["wb-gpio/A1_OUT", "wb-gpio/A2_OUT"], log)
```

Также можнго отписаться от контрола, если это нужно: 
```python
# Отписаться от одного контрола
wb.unsubscribe("wb-mr6c_226/K1")

# Отписаться от нескольких контролов
wb.unsubscribe(["wb-gpio/A1_OUT", "wb-gpio/A2_OUT"])
```

## Подписка на ошибки
При работе с устройствами через драйвер wb-mqtt-serial можно получать ошибки обмена, которые публикуются драйвером в MQTT:
- r — ошибка чтения из устройства;
- w — ошибка записи в устройство;
- p — драйвер не смог выдержать заданные период опроса.

В одном сообщении может быть несколько ошибок, например `rwp` или `rp`.

Подписаться на ошибки определённого контрола:
```python

# Функция обратного вызова
def log_errors(device_id, control_id, error_value):
    print("log_errors: %s %s %s" % (device_id, control_id, error_value))

# Подписка на ошибки контрола wb-mr6c_226/K1
wb.subscribe_errors("wb-mr6c_226/K1", log_errors)

```
Можно использовать символ подстановки  `+`, например:
```python
# Подписка на все ошибки модуля wb-mr6c_226
wb.subscribe_errors("wb-mr6c_226/+", log_errors)
```

Также можно подписаться на ошибки нескольких контролов через список:
```python
# Подписка на ошибки нескольких контролов
wb.subscribe_errors(["wb-gpio/A1_OUT", "wb-gpio/A2_OUT"], log_errors)
```

Отписаться от одного или нескольких контролов:
```python
# Описка от ошибок контрола wb-mr6c_226/K1
wb.unsubscribe_errors("wb-mr6c_226/K1")

# Описка от ошибок контролов с символом подстановки
wb.unsubscribe_errors("wb-mr6c_226/+")

# Отписка от ошибок нескольких контролов
wb.unsubscribe_errors(["wb-gpio/A1_OUT", "wb-gpio/A2_OUT"])
```

## Подписка на командный топик /on
Если вы используете модуль в конвернете, полезно будет подписаться на командный топик, чтобы обрабатывать действия в веб-интерфейсе или стороннем софте. Можно использовать символ подстановки `+`.

```python

# Функция обратного вызова
def log_on(device_id, control_id, error_value):
    print("log_on: %s %s %s" % (device_id, control_id, error_value))

# Подписка на командный топик контрола wb-mr6c_226/K1
wb.subscribe_on("wb-mr6c_226/K1", log_on)

```

## Подписка на произвольные MQTT-топики
Иногда нужно работать с топиками сторонних устройств, которые ничего не знают про конвенцию Wiren Board. Для этого есть функции подписки, отписки и публикации с указанием полного имени топика. Обработка события происходит в функции обратного вызова, которую нужно задать. Функция возвращает два параметра:
- `mqtt_topic` — полный путь к топику;
- `new_value` — новое значение типа str.

При подписке можно использовать символы подстановки `+` — подписаться на один уровень и `#` — подписаться на все уровни ниже.

Пример:
```python
# Функция обратного вызова
def log_raw(mqtt_topic, new_value):
    print("log_raw: %s %s" % (mqtt_topic, new_value))

# Подписка на топик
wb.subscribe_raw('/wbrules/#', log_raw)

# Публикация нового значения в топик
wb.publish_raw('/wbrules/log/info', 'New Log Value')

# Отписка от топика
wb.unsubscribe_raw('/wbrules/#')

```

## Виртуальные устройства

Также можно создавать виртуальные устройства с произвольным количеством контролов и использовать их для взаимодействия с пользователем или хранения данных:
```python
# функция задания
wb.create_virtual_device(
    "my-device", # идентификатор устройства
    {"ru": "Моё устройство", "en": "My Device"}, # заголовок устройства (title)
    [
        {
            "name": "temp", # идентификатор контрола в mqtt. Обязательно.     
            "title": {"ru": "Температура", "en": "Temperature"}, # заголовок контрола (title). Обязательно.
            "type": "value", # тип контрола. Обязательно.
            "default": 50, # значение по умолчанию
            "order": 1, # порядковый номер для сортировки
            "units": "°C" # единица измерения          
        },
        {
            "name": "set_temp",
            "title": {"ru": "Уставка", "en": "Set Temperature"},
            "type": "value",
            "readonly": False, # запрещает редактировать контрол. По умолчанию True.
            "default": 12.5,
            "order": 2,
            "units": "°C",
            "min": 0,
            "max": 100        
        },        
        {
            "name": "slider",
            "title": {"ru": "Ползунок", "en": "Slider"},
            "type": "range",
            "default": 13,
            "order": 3,
            "units": "%",
            "min": 10,
            "max": 35
        },          
        {
            "name": "switch",
            "title": {"ru": "Переключатель", "en": "Switch"},
            "type": "switch",
            "default": 1,
            "order": 4,
        },
        {
            "name": "log",
            "title": "Text Control", # можно и одну строчку для всех языков
            "type": "text",
            "default": "",
            "order": 5,
        },      
    ],
)
```
Заголовок (title) устройства и контролов можно указывать строкой `"My Device Title"` или словарём с указанием языков `{"ru": "Переключатель", "en": "Switch 1"}`.

Контролы передаются массивом словарей. Описание контролов соответствует текущей [конвенции Wiren Board](https://github.com/wirenboard/conventions), за исключением `default` для `switch`, в модуле надо использовать значения `0` и `1`.

Список типов и доступные параметры для каждого типа:
| type       | Возможные значения               | default | order | units | min/max | readonly |
|------------|----------------------------------|---------|-------|-------|---------|----------|
| value      | любые числа: int, float          | +       |+      |+      |+        |+         |
| range      | любые числа: int, float          | +       |+      |+      |+        |+         |
| rgb        | формат "R;G;B", каждое 0...255   | +       |+      |+      |         |+         |
| text       | текст                            | +       |+      |+      |         |+         |
| alarm      | текст                            | +       |+      |       |         |          |
| switch     | 0 или 1                          | +       |+      |       |         |+         |
| pushbutton | 1                                | +       |+      |       |         |          |

Список доступных `units`:
| Значение  | Описание, EN |
|---        |---          |
| mm/h      | mm per hour, precipitation rate (rainfall rate) |
| m/s       | meter per second, speed |
| W         | watt, power |
| kWh       | kilowatt hour, power consumption |
| V         | voltage |
| mV        | voltage (millivolts) |
| m^3/h     | cubic meters per hour, flow |
| m^3       | cubic meters, volume |
| Gcal/h    | giga calories per hour, heat power |
| cal       | calories, energy |
| Gcal      | giga calories, energy |
| Ohm       | resistance |
| mOhm      | resistance (milliohms) |
| bar       | pressure |
| mbar      | pressure (100Pa) |
| s         | second |
| min       | minute |
| h         | hour |
| m         | meter |
| g         | gram |
| kg        | kilo gram |
| mol       | mole, amount of substance |
| cd        | candela, luminous intensity |
| %, RH     | relative humidity |
| deg C     | temperature |
| %         | percent |
| ppm       | parts per million |
| ppb       | parts per billion |
| A         | ampere, current |
| mA        | milliampere, current |
| deg       | degree, angle |
| rad       | radian, angle |

## Известные проблемы
Не всегда удаляются созданные виртуальные устройства. Возможно, скрипт завершается раньше, чем успевают отработать процедуры удаления.

При установке надо создать описание спрвиса для автозапуска, поэтому обновлять софт контроллера надо строго через apt. При обновлении с флешки или в веб-интерфейсе описание сервиса будет удалено. Как костыль, можно на wb-rules написать скрипт, который будет запускать скрипт на питоне :D
