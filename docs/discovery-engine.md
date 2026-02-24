# Discovery Engine — движок обнаружения устройств

## Обзор

Discovery Engine автоматически обнаруживает устройства Wiren Board и создаёт канонические устройства без ручного описания каждого контрола в конфиге. Движок работает в двух режимах и использует три уровня приоритета для определения того, как WB-контролы превращаются в канонические устройства.

### Два режима работы

| Режим | Назначение | Пример |
|-------|-----------|--------|
| **Автоматический** (runtime) | Устройства появляются в рантайме, передаются всем адаптерам | Home Assistant: подключил → всё смаплено → работает |
| **Генерация конфига** (`--scan`) | Создаёт JSON-конфиг для ручной доработки | Алиса/Маруся: сгенерировал → поправил → запустил |

### Три уровня приоритета

```
┌─────────────────────────────────────┐
│  1. Ручной конфиг (devices[])       │  ← всегда выигрывает
├─────────────────────────────────────┤
│  2. Профили модулей                 │  ← известные WB-модули
├─────────────────────────────────────┤
│  3. Поконтрольный фоллбэк          │  ← таблица type → DeviceType
└─────────────────────────────────────┘
```

Если контрол уже привязан на более высоком уровне, нижние уровни его не трогают.

## Режимы использования

### Автоматический режим (runtime)

Включается настройкой `discovery.enabled: true` (по умолчанию). При запуске и при появлении новых MQTT-устройств движок:

1. Сканирует все MQTT-топики `/devices/+/controls/+/meta/+`
2. Для каждого MQTT-устройства проверяет уровни приоритета (конфиг → профиль → фоллбэк)
3. Создаёт канонические устройства и передаёт их адаптерам через `OnDeviceAdded`

Устройства обнаруживаются динамически: подключение нового модуля к шине вызывает появление новых MQTT-топиков, движок обрабатывает их и уведомляет адаптеры.

### Режим генерации конфига (`--scan`)

Запуск:

```bash
wb-ext-bridge --scan > /tmp/discovered.json
```

Движок выполняет однократное сканирование MQTT и выводит JSON-массив обнаруженных устройств на stdout. Вывод совместим с форматом `devices[]` конфига — его можно перенести в `/etc/wb-ext-bridge.conf` целиком или частично.

Типичный сценарий для платформ с конфигом (Алиса, Маруся):

1. `wb-ext-bridge --scan > /tmp/discovered.json`
2. Отредактировать: убрать лишнее, переименовать, добавить комнаты
3. Перенести в `/etc/wb-ext-bridge.conf`
4. Перезапустить сервис

## Профили модулей

### Концепция

Профиль модуля описывает, как конкретная модель WB-устройства (WB-MDM3, WB-MR6C и т.д.) превращается в набор канонических устройств. В отличие от поконтрольного фоллбэка, профиль знает структуру модуля и создаёт **составные устройства** (например, диммер = реле + канал яркости).

### Формат профиля

Профили хранятся в директории `profiles/`, по одному YAML-файлу на модель.

```yaml
model: wb-mdm3
vendor: Wiren Board
description: "3-канальный диммер"
aliases:                           # опционально, дополнительные модели
  - wb-mdm3-mini

devices:
  - name_template: "{module_title} Диммер {n}"
    type: dimmer
    repeat: 3                      # повторить для n=1..3
    map:
      on_off: "K{n}"
      brightness: "Channel {n}"
```

### Поля профиля

| Поле | Тип | Обязательное | Описание |
|------|-----|:---:|----------|
| `model` | string | да | Идентификатор модели (в нижнем регистре) |
| `vendor` | string | нет | Производитель |
| `description` | string | нет | Описание модуля |
| `aliases` | string[] | нет | Альтернативные модели с идентичной структурой |
| `devices` | object[] | да | Список устройств, создаваемых из модуля |

### Поля устройства в профиле

| Поле | Тип | Обязательное | Описание |
|------|-----|:---:|----------|
| `name_template` | string | да | Шаблон имени устройства |
| `type` | string | да | Тип канонического устройства |
| `repeat` | int | нет | Количество повторений (для многоканальных модулей) |
| `control` | string | нет | WB-контрол (для простых устройств с одним слотом) |
| `map` | object | нет | Маппинг слотов на контролы (для составных устройств) |

Устройство использует либо `control` (один слот), либо `map` (несколько слотов) — аналогично конфигу `devices[]`.

### Шаблонные переменные

| Переменная | Описание | Пример |
|------------|----------|--------|
| `{n}` | Номер повторения (1, 2, 3, ...) | `Channel {n}` → `Channel 1` |
| `{module_title}` | Человекочитаемое название модуля из профиля | `WB-MDM3` |
| `{device_name}` | Имя MQTT-устройства | `wb-mdm3_1` |
| `{address}` | Адрес модуля на шине (извлечённый из имени) | `1` |

### Примеры профилей

#### WB-MDM3 — 3-канальный диммер

```yaml
model: wb-mdm3
vendor: Wiren Board
description: "3-канальный диммер"

devices:
  - name_template: "{module_title} Диммер {n}"
    type: dimmer
    repeat: 3
    map:
      on_off: "K{n}"
      brightness: "Channel {n}"
```

MQTT-устройство `wb-mdm3_1` → 3 канонических устройства:

| Устройство | Тип | on_off | brightness |
|------------|-----|--------|------------|
| WB-MDM3 Диммер 1 | dimmer | `wb-mdm3_1/K1` | `wb-mdm3_1/Channel 1` |
| WB-MDM3 Диммер 2 | dimmer | `wb-mdm3_1/K2` | `wb-mdm3_1/Channel 2` |
| WB-MDM3 Диммер 3 | dimmer | `wb-mdm3_1/K3` | `wb-mdm3_1/Channel 3` |

#### WB-MR6C — 6-канальное реле

```yaml
model: wb-mr6c
vendor: Wiren Board
description: "6-канальное реле"
aliases:
  - wb-mr6cu

devices:
  - name_template: "{module_title} Реле {n}"
    type: switch
    repeat: 6
    control: "K{n}"
```

MQTT-устройство `wb-mr6cu_97` → 6 канонических устройств типа `switch`.

#### WB-MSW-v3 — мультисенсор

```yaml
model: wb-msw-v3
vendor: Wiren Board
description: "Мультисенсор (температура, влажность, освещённость, движение)"

devices:
  - name_template: "{module_title} Температура"
    type: temperature_sensor
    control: "Temperature"

  - name_template: "{module_title} Влажность"
    type: humidity_sensor
    control: "Humidity"

  - name_template: "{module_title} Освещённость"
    type: illuminance_sensor
    control: "Illuminance"

  - name_template: "{module_title} Движение"
    type: motion_sensor
    control: "Motion"
```

MQTT-устройство `wb-msw-v3_1` → 4 канонических устройства (по одному на датчик).

#### WB-MRGBW-D — RGB+W контроллер

```yaml
model: wb-mrgbw-d
vendor: Wiren Board
description: "RGB+W контроллер"

devices:
  - name_template: "{module_title} RGB"
    type: rgb_light
    map:
      on_off: "ON"
      color: "RGB"
      brightness: "White"
```

## Извлечение модели из имени MQTT-устройства

Имена MQTT-устройств Wiren Board следуют конвенции `<модель>_<адрес>`. Движок извлекает модель с помощью регулярного выражения:

```
^(.+?)_(\d+)$
```

| MQTT-устройство | Модель | Адрес |
|-----------------|--------|-------|
| `wb-mdm3_1` | `wb-mdm3` | `1` |
| `wb-mr6cu_97` | `wb-mr6cu` | `97` |
| `wb-msw-v3_1` | `wb-msw-v3` | `1` |
| `wb-mrgbw-d_12` | `wb-mrgbw-d` | `12` |

Если имя не соответствует паттерну (нет суффикса `_<число>`), профили модулей не применяются, и контролы обрабатываются поконтрольным фоллбэком.

При поиске профиля учитываются как поле `model`, так и `aliases`. Например, MQTT-устройство `wb-mr6cu_97` находит профиль `wb-mr6c.yaml`, потому что `wb-mr6cu` указан в `aliases`.

## Поконтрольный фоллбэк

Если контрол не привязан ни через конфиг, ни через профиль модуля, применяется поконтрольный фоллбэк. Каждый контрол анализируется индивидуально: один контрол = одно каноническое устройство.

### Таблица соответствий

| Тип WB-контрола | Единица измерения | DeviceType | Категория | Описание |
|-----------------|-------------------|------------|-----------|----------|
| `switch` | — | `switch` | switch | Реле, выключатель |
| `range` | — | `dimmer` | light | Диммер (0-100%) |
| `temperature` | `°C` | `temperature_sensor` | sensor | Датчик температуры |
| `rel_humidity` | `%` | `humidity_sensor` | sensor | Датчик влажности |
| `power` | `W` | `power_sensor` | sensor | Датчик мощности |
| `voltage` | `V` | `voltage_sensor` | sensor | Датчик напряжения |
| `lux` | `lx` | `illuminance_sensor` | sensor | Датчик освещённости |
| `concentration` | `ppm` | — | — | Нет стандартного типа (расширяемо) |
| `pressure` | `mmHg` | — | — | Нет стандартного типа (расширяемо) |
| `value` | `V` | `voltage_sensor` | sensor | Датчик напряжения (при однозначном определении) |
| `value` | `%` | — | — | Неоднозначный маппинг |

Контролы, не попавшие в таблицу, игнорируются. Для них необходимо добавить запись в конфиг вручную или расширить таблицу.

### Правила

1. **Тип контрола определяет DeviceType.** Используется поле `type` из MQTT-метаданных (`/devices/<device>/controls/<control>/meta/type`).

2. **Единица измерения уточняет маппинг.** Если тип контрола неоднозначен (например, `value`), единица помогает определить DeviceType. Если однозначное соответствие найти не удаётся, контрол игнорируется.

3. **Один контрол = одно устройство.** Фоллбэк всегда создаёт ровно одно каноническое устройство на один WB-контрол.

## Исключения

### Исключение контролов

Поле `discovery.exclude` содержит список контролов, которые движок должен игнорировать:

```json
{
  "discovery": {
    "exclude": [
      "wb-mr6cu_97/K5",
      "wb-mr6cu_97/K6"
    ]
  }
}
```

Исключённые контролы не обрабатываются ни профилями, ни фоллбэком. Ручной конфиг (`devices[]`) имеет приоритет: если контрол указан и в `exclude`, и в `devices[]`, он будет обработан по конфигу.

### Исключение устройств

Поле `discovery.exclude_devices` исключает целые MQTT-устройства (все их контролы):

```json
{
  "discovery": {
    "exclude_devices": [
      "wb-mr6cu_97",
      "system"
    ]
  }
}
```

## Именование

### Устройства из профилей

Имя формируется по шаблону `name_template` из профиля:

| Шаблон | MQTT-устройство | Результат |
|--------|-----------------|-----------|
| `{module_title} Диммер {n}` | `wb-mdm3_1` | `WB-MDM3 Диммер 1` |
| `{module_title} Реле {n}` | `wb-mr6cu_97` | `WB-MR6C Реле 1` |
| `{module_title} Температура` | `wb-msw-v3_1` | `WB-MSW-v3 Температура` |

### Устройства из поконтрольного фоллбэка

Имя формируется по шаблону `{device}/{control}`:

| MQTT-устройство | Контрол | Имя |
|-----------------|---------|-----|
| `wb-mr6cu_97` | `K2` | `wb-mr6cu_97/K2` |
| `wb-msw-v3_1` | `Temperature` | `wb-msw-v3_1/Temperature` |

### Идентификаторы

Поле `id` канонического устройства формируется автоматически:

- **Профильные устройства**: `{device_name}_{slug(type)}_{n}` — например, `wb-mdm3_1_dimmer_1`
- **Фоллбэк**: `auto_{device}_{control}` — например, `auto_wb-mr6cu_97_K2`

## Алгоритм работы

Для каждого MQTT-устройства, обнаруженного в брокере:

```
1. Собрать список контролов устройства
2. Убрать контролы, указанные в discovery.exclude
3. Проверить: устройство в discovery.exclude_devices? → пропустить всё
4. Убрать контролы, уже привязанные через devices[]
5. Извлечь модель из имени устройства (regex)
6. Найти профиль по модели/aliases
7. Если профиль найден:
   a. Создать канонические устройства по профилю
   b. Пометить использованные контролы
8. Для оставшихся контролов — применить поконтрольный фоллбэк
```

## Примеры

### WB-MDM3: 3 диммера вместо 6 отдельных сущностей

**Было** (существующий wb-home-assistant):
- `wb-mdm3_1/K1` → Switch
- `wb-mdm3_1/K2` → Switch
- `wb-mdm3_1/K3` → Switch
- `wb-mdm3_1/Channel 1` → Number
- `wb-mdm3_1/Channel 2` → Number
- `wb-mdm3_1/Channel 3` → Number

**Стало** (Discovery Engine с профилем):
- `WB-MDM3 Диммер 1` → dimmer (K1 + Channel 1)
- `WB-MDM3 Диммер 2` → dimmer (K2 + Channel 2)
- `WB-MDM3 Диммер 3` → dimmer (K3 + Channel 3)

### WB-MR6C: 6 реле

MQTT-устройство `wb-mr6cu_97` с контролами `K1`..`K6`.

Профиль `wb-mr6c.yaml` (по alias `wb-mr6cu`) создаёт 6 устройств:
- `WB-MR6C Реле 1` → switch (K1)
- `WB-MR6C Реле 2` → switch (K2)
- ...
- `WB-MR6C Реле 6` → switch (K6)

### WB-MSW-v3: мультисенсор

MQTT-устройство `wb-msw-v3_1` с контролами `Temperature`, `Humidity`, `Illuminance`, `Motion`, `Sound Level`, `CO2` и другими.

Профиль создаёт 4 устройства для известных контролов:
- `WB-MSW-v3 Температура` → temperature_sensor
- `WB-MSW-v3 Влажность` → humidity_sensor
- `WB-MSW-v3 Освещённость` → illuminance_sensor
- `WB-MSW-v3 Движение` → motion_sensor

Контролы `Sound Level`, `CO2` и другие, не описанные в профиле, обрабатываются поконтрольным фоллбэком (если есть совпадение в таблице) или игнорируются.

### Смешанный сценарий

Конфиг:

```json
{
  "devices": [
    {
      "name": "Термостат гостиная",
      "type": "thermostat",
      "map": {
        "current_temperature": "wb-msw-v3_1/Temperature",
        "target_temperature": "thermostat_setpoints/living_room",
        "is_heating": "wb-mr6cu_97/K1",
        "mode": "thermostat_modes/living_room"
      }
    }
  ],
  "discovery": {
    "enabled": true,
    "exclude": ["wb-mr6cu_97/K5", "wb-mr6cu_97/K6"]
  }
}
```

Результат обнаружения:

| Устройство | Тип | Источник | Пояснение |
|------------|-----|----------|-----------|
| Термостат гостиная | thermostat | config | Из `devices[]` |
| WB-MDM3 Диммер 1 | dimmer | profile | Профиль wb-mdm3 |
| WB-MDM3 Диммер 2 | dimmer | profile | Профиль wb-mdm3 |
| WB-MDM3 Диммер 3 | dimmer | profile | Профиль wb-mdm3 |
| WB-MR6C Реле 2 | switch | profile | Профиль wb-mr6c (K1 занят конфигом) |
| WB-MR6C Реле 3 | switch | profile | Профиль wb-mr6c |
| WB-MR6C Реле 4 | switch | profile | Профиль wb-mr6c |
| WB-MSW-v3 Влажность | humidity_sensor | profile | Профиль wb-msw-v3 (Temperature занят конфигом) |
| WB-MSW-v3 Освещённость | illuminance_sensor | profile | Профиль wb-msw-v3 |
| WB-MSW-v3 Движение | motion_sensor | profile | Профиль wb-msw-v3 |

Контролы `K5`, `K6` исключены через `discovery.exclude`. Контролы `K1` и `Temperature` заняты ручным конфигом термостата.

### Пример вывода `--scan`

```bash
$ wb-ext-bridge --scan
```

```json
[
  {
    "name": "WB-MDM3 Диммер 1",
    "type": "dimmer",
    "map": {
      "on_off": "wb-mdm3_1/K1",
      "brightness": "wb-mdm3_1/Channel 1"
    }
  },
  {
    "name": "WB-MDM3 Диммер 2",
    "type": "dimmer",
    "map": {
      "on_off": "wb-mdm3_1/K2",
      "brightness": "wb-mdm3_1/Channel 2"
    }
  },
  {
    "name": "WB-MDM3 Диммер 3",
    "type": "dimmer",
    "map": {
      "on_off": "wb-mdm3_1/K3",
      "brightness": "wb-mdm3_1/Channel 3"
    }
  },
  {
    "name": "WB-MR6C Реле 1",
    "type": "switch",
    "control": "wb-mr6cu_97/K1"
  },
  {
    "name": "WB-MR6C Реле 2",
    "type": "switch",
    "control": "wb-mr6cu_97/K2"
  },
  {
    "name": "WB-MR6C Реле 3",
    "type": "switch",
    "control": "wb-mr6cu_97/K3"
  },
  {
    "name": "WB-MR6C Реле 4",
    "type": "switch",
    "control": "wb-mr6cu_97/K4"
  },
  {
    "name": "WB-MR6C Реле 5",
    "type": "switch",
    "control": "wb-mr6cu_97/K5"
  },
  {
    "name": "WB-MR6C Реле 6",
    "type": "switch",
    "control": "wb-mr6cu_97/K6"
  },
  {
    "name": "WB-MSW-v3 Температура",
    "type": "temperature_sensor",
    "control": "wb-msw-v3_1/Temperature"
  },
  {
    "name": "WB-MSW-v3 Влажность",
    "type": "humidity_sensor",
    "control": "wb-msw-v3_1/Humidity"
  },
  {
    "name": "WB-MSW-v3 Освещённость",
    "type": "illuminance_sensor",
    "control": "wb-msw-v3_1/Illuminance"
  },
  {
    "name": "WB-MSW-v3 Движение",
    "type": "motion_sensor",
    "control": "wb-msw-v3_1/Motion"
  }
]
```

Вывод `--scan` — это массив устройств в формате `devices[]`. Его можно перенести в конфиг целиком или выборочно.
