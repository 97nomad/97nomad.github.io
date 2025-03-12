+++
date = 2025-03-12
title = "Подключаем к Home Assistant Tasmota и странные нонейм девайсы"
[taxonomies]
tags = [ 'tasmota', 'smarthome', 'ru', 'hass' ]
+++

# Проблематика

Однажды, один очень нехороший человек решил, что подключить весь дом (включая электроплиту) на один 2.5mm² — это очень хорошая идея.

Правила пожарной безопасности и автоматические предохранители же так не считали. 

Поэтому одновременное включение бойлера и духовки вызывает полный power outage в отдельно взятой квартире, что меня неистово фрустрирует, а каждый раз бегать включать-выключать бойлер из розетки — определённо не то, чем я хотела бы заниматься.

Так что будем делать умный дом, чтобы рулить этим делом хотя бы со смартфона.

# Железяки

Быстрый поиск по Амазону выдал мне `SONOFF Zigbee Bridge 3.0 Gateway Hub, ,Soporte de Protocolo Dual,WiFi, Compatible con Dispositivos Zigbee Pro, Alexa & Google`, он же `SONOFF ZBBridge-P` всего за 35€. \
Быстрогуглинг показал, что на неё есть какая-то опенсорсная прошивка. "Надо брать", — подумала я.

Попутно в корзину отправились `ZigBee Enchufe Inteligente Alexa 16A, 3680W Smart Plug con Monitor de Energía, Enchufe Temporizador con Control Remoto, Control por Voz y Funciones de Temporización, apoyo Alexa & Google Home` (4 штуки за 45€), позже оказавшиеся `Tuya TS011F`.

И ещё на сдачу была докинута `Garza Smart - Bombilla LED Zigbee Estándar A60, 11W (equivale a 75W de incandescencia), E27, Requiere Puente/Bridge, RGB + CCT, Intensidad regulable, Programable, Control por voz y app, Alexa/Google`, которая зачем-то прикидывалась `Tuya TZ3210`. \
Чем именно является эта `TZ3210` установить так и не удалось, потому что в разных источниках пишут, что это либо выключатель, либо датчик влажности и температуры, либо всё таки RGB лампочка.

# Лечим бридж от облака

Так как сливать данные умного дома на какое-то непонятное облако — это моветон, первым делом нужно прошить бридж.

Я руководствовалась вот этой весьма подробной инструкцией [How to flash Tasmota on Sonoff ZB Bridge Pro](https://notenoughtech.com/home-automation/tasmota-on-sonoff-zb-bridge-pro/).

Единственное, что нормально не проговаривается — это пин `GPIO00`. Он должен быть подтянут к `GND` во время запуска железки, чтобы загнать её в режим прошивки через UART. \
У меня были трудности с этим, так что успешность операции можно отследить элегантным `screen /dev/ttyACM0 115200` и посмотрев на логи запуска.

Я не буду описывать полный процесс прошивки, ибо он отлично описан по ссылке выше, но...

{{ figure(src="/blog/tasmota-tuya-perversions/how-to-get-heart-attack.jpg", caption="Рис 1. Как схватить инфаркт на ровном месте") }}

К счастью, этот баг уже давным-давно пофиксили в новой версии и кто-то просто долго не обновляла прошивку.

# Настраиваем секурный MQTT

Несмотря на то, что ESP32 уже имеет реализацию Wireguard'а, Tasmota его всё ещё не поддерживает, так что придётся выставить MQTT наружу и настроить на нём SSL, чтобы рандомные вандереры не развлекались с моими розетками.

Поэтому лёгким движением `nixos-rebuild switch ...` поднимаем на внешнем сервере с статическим IP Mosquitto

```nix
users.users.mosquitto.extraGroups = [ "nginx" ];
services.mosquitto = {
  enable = true;
  listeners = [
    {
      address = "123.456.789.123";
      port = 1883;
      users = {
        tasmota = {
          acl = [
            "readwrite tasmota/#"
          ];
          password = "changeme";
        };
      };
      settings = {
        "allow_anonymous" = false;
        "require_certificate" = false;
        "cafile" = "/var/lib/acme/mqtt.example.com/fullchain.pem";
        "certfile" = "/var/lib/acme/mqtt.example.com/cert.pem";
        "keyfile" = "/var/lib/acme/mqtt.example.com/key.pem";
        "tls_version" = "tlsv1.2";
      };
    }
    {
      address = "10.20.0.1";
      port = 1883;
      users = {
        hass = {
          acl = [
            "readwrite #"
          ];
          password = "changeme";
        };
      };
    }
  ];
};
```

Юзер `mosquitto` отправляется в группу `nginx` просто чтобы иметь возможность подтягивать Let's Encrypt сертификаты.

Также я использую систему из двух листенеров. Один защищён SSL, ограничен по самые помидоры и смотрит наружу, а второй во внутренней VPN сети спокойно даёт подсосаться к нему при помощи бриджей, но об этом чуть позже

Ещё стоит отметить строки `"require_certificate" = false;` и `"tls_version" = "tlsv1.2`. \
Первая отключает авторизацию через TLS, оставляя стандартный пароль, а вторая включает шифрование трафика. \
Без первой Tasmota не может подключться к MQTT, а без второй Mosquitto будет игнорировать все прописанные сертификаты и работать в незащищённом режиме.

На стороне Home Assistant'а же я настроила бридж, который будет всасывать сообщения из промежуточного MQTT и засасывать в основной. \
Просто потому что у меня уже был настроен MQTT для локальных штук и я не хотела его трогать.

```nix
services.mosquitto = {
  enable = true;
  listeners = [
    ...
  ];
  bridges = {
    "external2internal" = {
      addresses = [
        { address = "10.20.0.1"; port = 1883; }
      ];
      topics = [
        "tasmota/# in"
        "tasmota/cmnd/# out"
      ];
      settings = {
        remote_username = "hass";
        remote_password = "changeme";
      };
    };
  };
};
```

С такой настройкой топиков я не затягиваю внутрь ничего лишнего, а наружу могут попадать только непосредственные команды для Tasmota.

Но это, разумеется, можно пропустить, если MQTT всего один.

**И НЕ ЗАБУДЬ ОТКРЫТЬ ПОРТ НА ФАЕРВОЛЛЕ!**

# Подключаем бридж к секурному MQTT

{{ image(src="/blog/tasmota-tuya-perversions/tasmota-mqtt-settings.png") }}

Тут и так всё очевидно и понятно, кроме пары моментов.

Обязательно прожимаем чекбокс `MQTT TLS`, чтобы оно вообще подключилось к нашему секурному Mosquitto.

И в поле `Full Topic` следует в начало дописать `tasmota/`, чтобы все сообщения аккуратно сваливать в одну группу и не прописывать в ACL каждый бридж.

После сопряжения устройств с бриджом (Просто нажми кнопку `Zigbee Permit Join` и включай железяку) не лишним будет проверить, действительно ли всё работает как положено и сообщения от устройств попадают в MQTT. \
Подписываемся на `tasmota/#` и ждём, пока туда что-нибудь не посыпется.

Если не посыпалось, возвращаемся к началу и долго думаем, что же пошло не так. 

Повторять до полного удовлетворения.

# Изучаем устройства

{{ figure(src="/blog/tasmota-tuya-perversions/sonoff-bridge.jpg", caption="Рис. 2. SONOFF ZB Bridge-P") }}

Вот на этом этапе и начинается Адъ и Израиль.

Дело в том, что Tasmota не умеет в auto discovery. Точнее умеет, но только для конечных устройств и через свой собственный плагин. \
А так как у нас бридж, то из этого следует два факта:

1. Плагин нам не нужен, сырого MQTT более чем достаточно
2. Нам никто не поможет

Но для начала следует вооружиться консолью на самом бридже и посмотреть, что же там у нас имеется.

{{ figure(src="/blog/tasmota-tuya-perversions/tasmota-console.png", caption="Рис. 3. Зелёный и страшный") }}

Идём в `Tools > Console` в веб интерфейсе бриджа и начинаем тыкать всё вокруг палочкой.

Полный список команд лежит вот тут: [Tasmota Commands](https://tasmota.github.io/docs/Commands/), но нам сейчас интересна исключительно часть про [Zigbee](https://tasmota.github.io/docs/Commands/#zigbee).

А если быть точнее, то команды `ZbInfo`, `ZbName` и `ZbSend`.

### ZbInfo

Вбив в терминал `ZbInfo` можно получить всю информацию о подключенных устройствах. \
Результатом обычно получается вот такая портянка

```json
20:03:18.524 MQT: tasmota/tele/tasmota_6A65EC/7E20/SENSOR = {"ZbInfo":{"0x7E20":{"Device":"0x7E20","Name":"main_light_0","IEEEAddr":"0xA4C138E23EE90340","ModelId":"TS0505B","Manufacturer":"_TZ3210_4sku4dkz","Endpoints":[1,242],"Config":["O01","~01.1","L01"],"Power":1,"Dimmer":254,"Hue":0,"Sat":0,"X":24218,"Y":15601,"CT":499,"ColorMode":1,"RGB":"FFFFFF","RGBb":"FFFFFF","Reachable":true,"LastSeen":4,"LastSeenEpoch":1741806194,"LinkQuality":91}}}
20:03:18.536 MQT: tasmota/tele/tasmota_6A65EC/CD47/SENSOR = {"ZbInfo":{"0xCD47":{"Device":"0xCD47","Name":"aquarium_light","IEEEAddr":"0xA4C1382214F48A12","Endpoints":[1],"Config":["P01","O01"],"RMSVoltage":226,"ActivePower":10,"Power":1,"Reachable":true,"LastSeen":6,"LastSeenEpoch":1741806192,"LinkQuality":185}}}
20:03:18.548 MQT: tasmota/tele/tasmota_6A65EC/9AD4/SENSOR = {"ZbInfo":{"0x9AD4":{"Device":"0x9AD4","Name":"boiler_switch","IEEEAddr":"0xA4C13877C8106300","ModelId":"TS011F","Manufacturer":"_TZ3000_cehuw1lw","Endpoints":[1],"Config":["P01","O01"],"RMSVoltage":226,"ActivePower":0,"Power":0,"Reachable":true,"LastSeen":73,"LastSeenEpoch":1741806125,"LinkQuality":98}}}
20:03:18.563 MQT: tasmota/tele/tasmota_6A65EC/3B30/SENSOR = {"ZbInfo":{"0x3B30":{"Device":"0x3B30","Name":"main_light_1","IEEEAddr":"0xA4C1381A825F2ADE","ModelId":"TS0505B","Manufacturer":"_TZ3210_4sku4dkz","Endpoints":[1,242],"Config":["~01.1","O01","L01"],"Power":0,"Dimmer":254,"Hue":0,"Sat":254,"X":38449,"Y":21476,"CT":283,"ColorMode":2,"RGB":"FF0000","RGBb":"FF0000","Reachable":true,"LastSeen":136,"LastSeenEpoch":1741806062,"LinkQuality":163}}}
20:03:18.578 MQT: tasmota/tele/tasmota_6A65EC/898A/SENSOR = {"ZbInfo":{"0x898A":{"Device":"0x898A","Name":"studio_light","IEEEAddr":"0xA4C1389F99E433CE","ModelId":"TS0505B","Manufacturer":"_TZ3210_4sku4dkz","Endpoints":[1,242],"Config":["O01","~01.1","L01"],"Power":0,"Dimmer":42,"Hue":0,"Sat":254,"X":13578,"Y":14165,"CT":283,"ColorMode":2,"RGB":"FF0000","RGBb":"2A0000","Reachable":true,"LastSeen":150,"LastSeenEpoch":1741806048,"LinkQuality":65}}}
```

Если посидеть, повтыкать внимательно, то можно обнаружить:

- `Device` — Короткий адрес устройства (0xAAAA). Обычно используем его, чтобы обратиться к девайсу.
- `ModelId` и `Manufacturer` — собственно, описание модели. Если посмотреть в начало статьи, то можно увидеть, что дешёвую китайскую Tuya можно встретить под разными пафосными брендами, но тут железяки палятся мгновенно.
- `Dimmer`, `Hue`, `Sat`, `X`, `Y` и все остальные железяко-специфичные параметры, которые эта самая железяка будет слать нам в MQTT.

Последнее нам особенно пригодится, но об этом чуть позже.

### ZbName

Сейчас самое время дать всем устройствам читаемые имена, чтобы потом не запутаться что есть что.

`ZbName short_address,name` нам в этом поможет. Потом этот name будет ещё всплывать в каждом ивенте и можно будет с этим что-то делать, но главная цель — не запутаться.

### ZbSend

А теперь мы можем потыкать палочкой в девайс уже активно, ибо `ZbSend` позволяет отправить какое-нибудь сообщения и посмотреть на реакцию железки.

Можно, например, потыкать палочкой в умную розетку.

`ZbSend {"device": "0x9AD4", "send": { "Power": "ON" }}` будет оную розетку включать, а `ZbSend {"device": "0x9AD4", "send": { "Power": "OFF" }}`, соответственно, выключать.

Как мы узнали, что нужно отправить именно `{ "Power": "ON" }` — это самый интересный вопрос.

И ответом на него будет "Угадали", но это немного не так.

Давайте посмотрим на JSON из вывода команды `ZbInfo 0x9AD4` более пристально

```json
{
  "ZbInfo": {
    "0x9AD4": {
      "Device": "0x9AD4",
      "Name": "boiler_switch",
      "IEEEAddr": "0xA4C13877C8106300",
      "ModelId": "TS011F",
      "Manufacturer": "_TZ3000_cehuw1lw",
      "Endpoints": [
        1
      ],
      "Config": [
        "P01",
        "O01"
      ],
      "RMSVoltage": 226,
      "ActivePower": 0,
      "Power": 0,
      "Reachable": true,
      "LastSeen": 73,
      "LastSeenEpoch": 1741806125,
      "LinkQuality": 98
    }
  }
}
```

Среди всего прочего тут можно увидеть поле `Power` со значением `0`, тобишь "Выключено". \
А штуку с более читаемыми `ON` и `OFF` я просто в интернетах подсмотрела.

Если смотреть на JSON от лампочки, там больше интересного

```json
{
  "ZbInfo": {
    "0x7E20": {
      "Device": "0x7E20",
      "Name": "main_light_0",
      "IEEEAddr": "0xA4C138E23EE90340",
      "ModelId": "TS0505B",
      "Manufacturer": "_TZ3210_4sku4dkz",
      "Endpoints": [
        1,
        242
      ],
      "Config": [
        "O01",
        "~01.1",
        "L01"
      ],
      "Power": 1,
      "Dimmer": 254,
      "Hue": 0,
      "Sat": 0,
      "X": 24218,
      "Y": 15601,
      "CT": 499,
      "ColorMode": 1,
      "RGB": "FFFFFF",
      "RGBb": "FFFFFF",
      "Reachable": true,
      "LastSeen": 4,
      "LastSeenEpoch": 1741806194,
      "LinkQuality": 91
    }
  }
}
```

Помимо `Power` мы также имеем `Dimmer`, `Hue`, `Sat`, `X`, `Y`, `CT`, `ColorMode`, `RGB` и `RGBb`. \
Спойлер: не всё из этого можно точно так же поменять, но общее направление для копания под девайс у нас имеется.

### Ещё один момент (Backlog)

Ах, да. Тут стоить упомянуть ещё один момент.

Несмотря на то, что мы можем сделать \
`ZbSend {"device": "0x7E20", "send": { "Hue": "128" }}`, \
сделать \
`ZbSend {"device": "0x7E20", "send": { "Hue": "128", "Sat": "254" }}` \
мы уже не сможем, так как за один запрос можно поменять только один параметр.

Как быстрый воркараунд вокруг проблемы, можно использовать [Backlog](https://tasmota.github.io/docs/Commands/#the-power-of-backlog), встроенный язык...

Короче, в эту штуку можно просто несколько команд подряд передать, а потом они последовательно выполнятся. \
И в результате наша команда будет выглядеть примерно так:
```
Backlog ZbSend {"device": "0x7E20", "send": { "Hue": "128" }}; ZbSend {"device": "0x7E20", "send": { "Sat": "254" }}
```

Хорошо для быстрых хаков, но на практике получается так, что отправка двух команд — это не самый быстрый процесс и вместо этого  иногда можно обнаружить "недокументированные" команды.

В случае той же лампочки существует команда `Color` которая позволяет одновременно установить значения `X` и `Y`, но об этом опять чуть позже.

# Home Assistant

{{ figure(src="/blog/tasmota-tuya-perversions/home-assistant.png", caption="Рис. 4. Home Assistant") }}

Помните, я говорила, что Tasmota не умеет в автодискавери? Так вот. \
Сейчас будем писать коньфиги. \
Вручную. \
И да поможет нам Nix, и его возможность писать человеческие функции для автогенерации YAML'а.

## MQTT топики

По-сути нам нужны только два топика: `tasmota/tele/tasmota_${gw_addr}/SENSOR` для чтения и `tasmota/cmnd/tasmota_${gw_addr}/ZbSend` для отправки команд. \
Возможно ещё третий `tasmota/cmnd/tasmota_${gw_addr}/Backlog` для, собственно, Backlog команд.

Да, любую из списка команд можно вызвать через MQTT, отправив в соответствующий топик, где последняя часть и является названием команды.

Слать будем JSON'ы, парсить их тоже. Давайте к практике.

### Выключатели и сенсоры

{{ figure(src="/blog/tasmota-tuya-perversions/smart-socket.jpg", caption="Рис 5. Выключатели с сенсорами") }}

Давайте вспомним наш JSON из розетки

```json
{
  "ZbInfo": {
    "0x9AD4": {
      "Device": "0x9AD4",
      "Name": "boiler_switch",
      "IEEEAddr": "0xA4C13877C8106300",
      "ModelId": "TS011F",
      "Manufacturer": "_TZ3000_cehuw1lw",
      "Endpoints": [
        1
      ],
      "Config": [
        "P01",
        "O01"
      ],
      "RMSVoltage": 226,
      "ActivePower": 0,
      "Power": 0,
      "Reachable": true,
      "LastSeen": 73,
      "LastSeenEpoch": 1741806125,
      "LinkQuality": 98
    }
  }
}
```

Помимо `Power`, который управляет включением\выключением розетки, у нас есть ещё: `RMSVoltage` (меряет количество вольтов) и `ActivePower` (меряет количество амперов).

Поэтому начинаем писать конфиг:

```nix
mqtt = {
  switch = [
    {
      unique_id = "boiler_outlet_switch";
      name = "Boiler Outlet Switch";
      state_topic = "tasmota/tele/tasmota_6A65EC/SENSOR";
      value_template = "{{ value_json.ZbReceived['0x9AD4'].Power }}";
      command_topic = "tasmota/cmnd/tasmota_6A65EC/ZbSend";
      payload_on = ''{"device":"0x9AD4","send":{"Power": "ON"}}'';
      payload_off = ''{"device":"0x9AD4","send":{"Power": "OFF"}}'';
      retain = true;
      device_class = "outlet";
      device = {
        name = "Boiler Outlet";
        model = "TS011F";
        manufacturer = "Tuya";
        identifiers = [ "0x9AD4" ];
      };
    }
  ];
};
```

Полное объяснение каждого параметра можно найти в соответствующей [документации](https://www.home-assistant.io/integrations/switch.mqtt/), так что пробегусь по важному:

`state_topic` и `command_topic` — те самые топики для чтения состояния и отправки команд. Обычно сразу после получения команды девайс кинет сообщение с изменённым состоянием, что позволяет не надеяться на "оптимистичную" стратегию Home Assistant и, пускай и с небольшой задержкой, но знать актуальное состояние железяки.

`value_template` — шаблон для вытаскивания значения параметра из JSON'а. Так как JSON'ы генерирует сам гетвей, можно смело копипастить шаблон из примера и только менять последнюю часть. Да, я тоже у кого-то это скопипастила.

`payload_on` и `payload_off` — тут тоже всё понятно, однако чуть позже научимся их тоже шаблонизировать.

Разумеется, для каждой розетки писать такую портянку будет неприятно, поэтому оборачиваем всё в функцию:

```nix
ts011f_switch = id: name: short_addr: gw_addr:
  {
    unique_id = id;
    name = "${name} Switch";
    state_topic = "tasmota/tele/tasmota_${gw_addr}/SENSOR";
    value_template = "{{ value_json.ZbReceived['${short_addr'].Power }}";
    command_topic = "tasmota/cmnd/tasmota_${gw_addr}/ZbSend";
    payload_on = ''{"device":"${short_addr}","send":{"Power": "ON"}}'';
    payload_off = ''{"device":"${short_addr}","send":{"Power": "OFF"}}'';
    retain = true;
    device_class = "outlet";
    device = {
      name = name;
      model = "TS011F";
      manufacturer = "Tuya";
      identifiers = [ short_addr ];
    };
  };
```

Но у нас же ещё сенсоры остались. Так что пишем ещё функцю для них:

```nix
ts011f_sensors = id: name: short_addr: gw_addr: 
  [
    {
      unique_id = "${id}_power";
      name = "${name} Power";
      state_topic = "tasmota/tele/tasmota_${gw_addr}/SENSOR";
      value_template = "{{ value_json.ZbReceived['${short_addr}'].ActivePower }}";
      unit_of_measurement = "W";
      device = {
        identifiers = [ "${short_addr}" ];
      };
    }
    {
      unique_id = "${id}_voltage";
      name = "${name} Voltage";
      state_topic = "tasmota/tele/tasmota_${gw_addr}/SENSOR";
      value_template = "{{ value_json.ZbReceived['${short_addr}'].RMSVoltage }}";
      unit_of_measurement = "V";
      device = {
        identifiers = [ "${short_addr}" ];
      };
    }
  ];
```

В итоге наш непосредственный конфиг выглядит красивенько и можем добавлять столько розеток, сколько у нас имеется (а у меня их как раз четыре штуки):

```nix
mqtt = {
  switch = [
    (ts011f_switch "boiler_switch" "Boiler" "0x9AD4" "6A65EC")
  ];
  sensor = (ts011f_sensors "boiler" "Boiler" "0x9AD4" "6A65EC")
};
```

## Лампочки

{{ figure(src="/blog/tasmota-tuya-perversions/smart-bulb.jpg", caption="Рис 6. Лампочка") }}

А теперь к самому сложному. Лампочки. \
Лампочки — это совершенно проклятые устройства. \
Они выглядят достаточно просто, но эта простота обманчива и заманивает несчастных на скалы безумия.

### Скалы безумия

Давайте ещё раз на JSON посмотрим

```json
{
  "ZbInfo": {
    "0x7E20": {
      "Device": "0x7E20",
      "Name": "main_light_0",
      "IEEEAddr": "0xA4C138E23EE90340",
      "ModelId": "TS0505B",
      "Manufacturer": "_TZ3210_4sku4dkz",
      "Endpoints": [
        1,
        242
      ],
      "Config": [
        "O01",
        "~01.1",
        "L01"
      ],
      "Power": 1,
      "Dimmer": 254,
      "Hue": 0,
      "Sat": 0,
      "X": 24218,
      "Y": 15601,
      "CT": 499,
      "ColorMode": 1,
      "RGB": "FFFFFF",
      "RGBb": "FFFFFF",
      "Reachable": true,
      "LastSeen": 4,
      "LastSeenEpoch": 1741806194,
      "LinkQuality": 91
    }
  }
}
```

Конкретно эта лампочка умеет менять мощность, температуру свечения и свой цвет.

На самом деле делает она это не одновременно, а переключаясь между режимами, которые можно достать из `ColorMode`. \
Впрочем, эта информация нам нужна только для исследования и отладки.

Первый режим — это яркое свечение во всю мощщу с переменной температурой. За это отвечают параметры `Dimmer` и `CT` (Color Temperature) соответственно.

Второй и третий — это цветные. Отличаются они только способом выбора цвета: Hue\Sat или X\Y.

Логично предположить, что мы хотим менять параметры `Hue` и `Sat` или `X` и `Y`, но тут всплывает не самый очевидный с первого раза вопрос.

**А какие именно значения может принимать этот параметр?**

С одной стороны, тут можно положиться на здравый смысл. \
Hue — это круг, значит диапазон значений должен быть 0..360. \
А координатная система XY использует значения 0.0..1.0.

Но жизнь как всегда оказывается гораздо непредсказуемее, чем теория. \
И мы таки имеем дело с лампочкой, которая одновременно выключатель и датчик влажности.

Так что единственный адекватный способ узнать настоящий диапазон значений — это тыкать палочкой и смотреть что получится.

К примеру, `Dimmer` имеет диапазон значений 0..254. Но при попытке выставить его в 255 лампочка начинает возвращать null, что весьма странно, ибо это ведь явно 8-битная переменная...

Мои попытки подобрать значения `Hue` и `Sat` ни к чему конструктивному не привели, а вот `X` и `Y` при попытке запихнуть в них значение не в диапазоне 0..65279, просто отказываются воспринимать команду. \
Почему не 0..65535, как кто-то предложил в интернете, думая что это 16-битная переменная? \
Ну вот потому что!

### Собираем всё в Home Assistant

В результате, собрав все полученные в процессе тыкания палочкой знания, мы готовы написать функцию

```nix
tz3210_lamp = id: name: short_addr: gw_addr:
  {
    unique_id = id;
    name = name;
    command_topic = "tasmota/cmnd/tasmota_${gw_addr}/ZbSend";
    payload_on = ''{"device": "${short_addr}", "send": {"Power": "ON"}}'';
    payload_off = ''{"device": "${short_addr}", "send": {"Power": "OFF"}}'';
    brightness_command_topic = "tasmota/cmnd/tasmota_${gw_addr}/ZbSend";
    brightness_command_template = ''{"device": "${short_addr}", "send": { "Dimmer": "{{ value - 1 }}" }}'';
    brightness_state_topic = "tasmota/tele/tasmota_${gw_addr}/SENSOR";
    brightness_value_template = "{{ value_json.ZbReceived['${short_addr}'].Dimmer }}";
    xy_command_topic = "tasmota/cmnd/tasmota_${gw_addr}/ZbSend";
    xy_command_template = ''{"Device": "${short_addr}", "send":{ "Color": "{{ x * 65279 }},{{ y * 65279 }}"}}'';
    xy_state_topic = "tasmota/tele/tasmota_${gw_addr}/SENSOR";
    xy_value_template = "{{ value_json.ZbReceived['${short_addr}'].X / 65279 }},{{ value_json.ZbReceived['${short_addr}'].Y / 65279 }}";
    color_temp_command_topic = "tasmota/cmnd/tasmota_${gw_addr}/ZbSend";
    color_temp_command_template = ''{"device": "${short_addr}", "send": { "CT": "{{ value - 1 }}" }}'';
    color_temp_state_topic = "tasmota/tele/tasmota_${gw_addr}/SENSOR";
    color_temp_value_template = "{{ value_json.ZbReceived['${short_addr}'].CT }}";
    device = {
      name = "RGB Lamp";
      model = "TZ3210";
      manufacturer = "Tuya";
        identifiers = [ short_addr ];
    };
  }
```

Эта функция — просто сборник хаков, для того чтобы железка хоть как-то заработала. \
Давайте пойдём по-порядку

`Dimmer`, по мнению Home Assistant, должен принимать значения 0..255, но как мы уже знаем, максимальная циферка там 254, так что просто вычитаем единичку, всё равно оно в интерфейсе меньше чем в 3% не выставляется.\
То же самое для `CT`.

`X` и `Y`, по мнению того же Home Assistant, принимают значения от 0 до 1.0, поэтому преобразование туда\обратно делается элементарным умножением на максимально возможное значение

Кроме того, в `xy_command_template` тут используется функция `Color`, которая позволяет передать значения `X` и `Y` за одну команду, случайно найденная уже не помню где в обсуждении уже не помню какого устройства.

Если бы её не было, конфиг бы выглядел как-то так:

```nix
xy_command_topic = "tasmota/cmnd/tasmota_${gw_addr}/Backlog";
xy_command_template = ''ZbSend {"Device": "${short_addr}", "send":{ "X": "{{ x * 65279 }}"}}; ZbSend {"Device": "${short_addr}", "send":{ "Y": "{{ y * 65279 }}"}}'';
```

# Это конец

И на этой ноте мои приключения наконец-то закончились. 

Лампочки моргают, бойлер выключается и даже аквариум обзавёлся своим удалённым управлением.

Мне категорически не понравилось умный дом делать, он какой-то слишком тупой оказался.

А наебалово с ценами огорчило меня ещё больше. \
Пойду лучше на AliExpress железок докупать, там хотя бы модель явно пишут.

{{ figure(src="/blog/tasmota-tuya-perversions/bisexual-lighting.png", caption="Рис. 7. Бисексуальное освещение") }}
