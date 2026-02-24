# wb-ext-conventions

**Универсальный слой абстракции устройств Wiren Board**

## Проблема

Контроллер Wiren Board представляет все устройства как плоские MQTT-контролы. Каждый контрол — это отдельный топик `/devices/<device>/controls/<control>`, несущий одно значение: температуру, состояние реле, уставку и т.д.

Внешние платформы автоматизации (Home Assistant, Apple HomeKit, Яндекс Алиса) работают с составными сущностями. Например, термостат в Home Assistant — это единый объект с текущей температурой, уставкой, режимом работы и состоянием нагрева. В HomeKit это `HeaterCooler` с набором характеристик. В Алисе — устройство с capabilities и properties.

Сейчас нет стандартного способа объявить, что набор WB-контролов (температурный датчик на одном модуле, реле на другом, виртуальный контрол уставки на третьем) образует одно логическое устройство — термостат.

## Решение

**wb-ext-conventions** определяет каноническую модель устройств — платформо-независимое промежуточное представление:

- **Типы устройств** — 14 стандартных типов (switch, dimmer, thermostat и т.д.) + произвольные кастомные типы
- **Один JSON-конфиг** — маппинг слотов канонической модели на физические WB-контролы
- **Адаптерный интерфейс** — контракт для трансляции канонической модели в формат целевой платформы

Мосты получают готовые составные устройства через единый интерфейс и транслируют их в формат своей платформы.

## Архитектура

```
┌──────────────────────────────────────────────────────────┐
│                    Внешние системы                        │
│          HA          HomeKit         Алиса               │
│      ┌────────┐   ┌──────────┐   ┌──────────┐           │
│      │Адаптер │   │ Адаптер  │   │ Адаптер  │           │
│      │   HA   │   │ HomeKit  │   │  Алиса   │           │
│      └───┬────┘   └────┬─────┘   └────┬─────┘           │
│          │              │              │                  │
│          ▼              ▼              ▼                  │
│  ┌───────────────────────────────────────────────────┐   │
│  │            Каноническая модель                     │   │
│  │      DeviceType + Capabilities + Properties        │   │
│  └───────────────────────┬───────────────────────────┘   │
│                          │                               │
│                          ▼                               │
│  ┌───────────────────────────────────────────────────┐   │
│  │             WB Control Tracker                     │   │
│  │     MQTT /devices/+/controls/+ → состояния         │   │
│  └───────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────┘
```

**Три слоя:**

1. **WB Control Tracker** — подписывается на MQTT-топики Wiren Board, отслеживает появление и изменение контролов.
2. **Каноническая модель** — по конфигурации собирает контролы в составные устройства, валидирует их по описаниям типов.
3. **Адаптеры** — получают канонические устройства и транслируют их в формат целевой платформы.

## Конфигурация

Пользователь описывает устройства один раз в едином конфиге. Сервис `wb-ext-bridge` читает конфиг, собирает канонические устройства и передаёт их **всем подключённым адаптерам одновременно**. Один конфиг → wb-ext-bridge → канонические устройства → Home Assistant + HomeKit + Алиса (и любые другие адаптеры). Добавлять устройства отдельно на каждой платформе не нужно.

Конфиг — один JSON-файл `/etc/wb-ext-bridge.conf`. Схема (`schema/wb-ext-bridge.schema.json`) используется встроенным веб-редактором Wiren Board (wb-mqtt-confed) для генерации UI с автоподстановкой контролов.

### Простые устройства — 3 поля

Для устройств с одним слотом (выключатель, датчик) достаточно трёх полей:

```json
{
  "name": "Свет кухня",
  "type": "switch",
  "control": "wb-mr6cu_97/K2"
}
```

- `name` — отображаемое имя устройства
- `type` — тип устройства (выбирается из выпадающего списка в UI)
- `control` — WB-контрол в формате `"устройство/контрол"` (соответствует структуре MQTT-топика `/devices/<устройство>/controls/<контрол>`). Символы `+` и `#` запрещены — это MQTT wildcard-символы. Формат валидируется паттерном `^[^/+#]+/[^/+#]+$` в JSON Schema.

### Составные устройства — поле `map`

Устройства с несколькими слотами (термостат, диммер, RGB-лента, шторы) используют объект `map`, где ключи — имена слотов, значения — WB-контролы:

```json
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
```

### Полный пример конфига

Пример включает стандартные типы (простые и составные) и кастомный тип (`tv`). Полная версия — `examples/wb-ext-bridge.conf`.

```json
{
  "devices": [
    {
      "name": "Свет кухня",
      "type": "switch",
      "control": "wb-mr6cu_97/K2"
    },
    {
      "name": "Температура спальня",
      "type": "temperature_sensor",
      "control": "wb-msw-v3_1/Temperature"
    },
    {
      "name": "Диммер гостиная",
      "type": "dimmer",
      "map": {
        "on_off": "wb-mdm3_1/K1",
        "brightness": "wb-mdm3_1/Channel 1"
      }
    },
    {
      "name": "Термостат гостиная",
      "type": "thermostat",
      "map": {
        "current_temperature": "wb-msw-v3_1/Temperature",
        "target_temperature": "thermostat_setpoints/living_room",
        "is_heating": "wb-mr6cu_97/K1",
        "mode": "thermostat_modes/living_room"
      }
    },
    {
      "name": "Телевизор гостиная",
      "type": "tv",
      "map": {
        "power": "wb-mir_1/Power",
        "volume_up": "wb-mir_1/Volume Up",
        "volume_down": "wb-mir_1/Volume Down",
        "mute": "wb-mir_1/Mute"
      }
    }
  ]
}
```

### Типы устройств

#### Простые (один слот — поле `control`)

| Тип | Слот | Описание |
|-----|------|----------|
| `switch` | on_off | Выключатель |
| `temperature_sensor` | temperature | Датчик температуры |
| `humidity_sensor` | humidity | Датчик влажности |
| `power_sensor` | power | Датчик мощности |
| `voltage_sensor` | voltage | Датчик напряжения |
| `illuminance_sensor` | illuminance | Датчик освещённости |
| `binary_sensor` | state | Бинарный датчик |
| `contact_sensor` | contact | Датчик открытия |
| `motion_sensor` | motion | Датчик движения |
| `leak_sensor` | leak | Датчик протечки |

#### Составные (несколько слотов — поле `map`)

| Тип | Обязательные слоты | Опциональные слоты |
|-----|-------------------|-------------------|
| `dimmer` | on_off, brightness | — |
| `rgb_light` | on_off, color | brightness |
| `thermostat` | current_temperature, target_temperature | is_heating, mode, on_off |
| `cover` | position | on_off |

### Кастомные типы

Для устройств, не попадающих в стандартные 14 типов (телевизор через ИК-пульт, медиаплеер, вентилятор и т.д.), используется произвольный тип. Поле `type` — любая строка, отличная от зарезервированных имён стандартных типов. Поле `map` — произвольный набор слотов:

```json
{
  "name": "Телевизор гостиная",
  "type": "tv",
  "map": {
    "power": "wb-mir_1/Power",
    "volume_up": "wb-mir_1/Volume Up",
    "volume_down": "wb-mir_1/Volume Down",
    "mute": "wb-mir_1/Mute"
  }
}
```

Кастомные устройства передаются адаптерам как есть — адаптер сам решает, как транслировать произвольные слоты в формат целевой платформы. Если адаптер не знает тип устройства, он может его проигнорировать или отобразить как generic-устройство.

## Авто-обнаружение

Простые устройства (один контрол = одно устройство) могут быть автоматически распознаны без явной записи в конфиге. Правила авто-обнаружения описаны в [docs/auto-discovery.md](docs/auto-discovery.md).

Таблица сопоставления WB-контролов с типами устройств:

| Тип WB-контрола | Единица | DeviceType |
|-----------------|---------|------------|
| `switch` | — | `switch` |
| `range` | — | `dimmer` |
| `temperature` | `°C` | `temperature_sensor` |
| `rel_humidity` | `%` | `humidity_sensor` |
| `power` | `W` | `power_sensor` |
| `voltage` | `V` | `voltage_sensor` |
| `lux` | `lx` | `illuminance_sensor` |

Составные устройства (термостат, RGB-лента) **всегда** требуют явной записи в конфиге, так как объединяют контролы с нескольких физических модулей.

## Адаптерный интерфейс

Адаптер — компонент, который транслирует канонические устройства в формат целевой платформы. Интерфейс содержит четыре метода:

```
OnDeviceAdded(device CanonicalDevice)
OnStateChanged(device_id string, slot string, value any)
OnDeviceRemoved(device_id string)
SetCommandHandler(handler func(device_id string, slot string, value any))
```

Подробная спецификация: [docs/adapter-interface.md](docs/adapter-interface.md).

## Пример сквозной трансляции

Рассмотрим путь термостата от WB-контролов до представления на внешних платформах.

### Шаг 1. Физические контролы WB

Контроллер публикует в MQTT:

```
/devices/wb-msw-v3_1/controls/Temperature        → 23.5
/devices/thermostat_setpoints/controls/living_room → 22.0
/devices/wb-mr6cu_97/controls/K1                  → 1
/devices/thermostat_modes/controls/living_room     → heat
```

### Шаг 2. Конфиг описывает устройство

Запись в `/etc/wb-ext-bridge.conf`:

```json
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
```

### Шаг 3. Каноническое устройство

Система формирует каноническое устройство:

```
CanonicalDevice:
  id:           termostat-gostinaya
  display_name: "Термостат гостиная"
  type:         thermostat
  properties:
    current_temperature: 23.5
    is_heating:          true
  capabilities:
    target_temperature:  22.0
    mode:                heat
```

### Шаг 4. Адаптеры транслируют в целевой формат

**Home Assistant** (MQTT Discovery):

```json
{
  "name": "Термостат гостиная",
  "unique_id": "wb_ext_termostat-gostinaya",
  "modes": ["off", "heat", "cool", "auto"],
  "current_temperature_topic": "wb-ext/termostat-gostinaya/current_temperature",
  "temperature_state_topic": "wb-ext/termostat-gostinaya/target_temperature",
  "temperature_command_topic": "wb-ext/termostat-gostinaya/target_temperature/set",
  "min_temp": 5,
  "max_temp": 35,
  "temp_step": 0.5
}
```

**Apple HomeKit** (HeaterCooler):

```json
{
  "accessory": "wb_ext_termostat-gostinaya",
  "service": "HeaterCooler",
  "characteristics": {
    "Active": 1,
    "CurrentTemperature": 23.5,
    "HeatingThresholdTemperature": 22.0,
    "CurrentHeaterCoolerState": 2,
    "TargetHeaterCoolerState": 1
  }
}
```

**Яндекс Алиса** (IoT API):

```json
{
  "id": "termostat-gostinaya",
  "name": "Термостат гостиная",
  "type": "devices.types.thermostat",
  "capabilities": [
    {
      "type": "devices.capabilities.range",
      "parameters": {
        "instance": "temperature",
        "unit": "unit.temperature.celsius",
        "range": {"min": 5, "max": 35, "precision": 0.5}
      },
      "state": {"instance": "temperature", "value": 22.0}
    },
    {
      "type": "devices.capabilities.mode",
      "parameters": {
        "instance": "thermostat",
        "modes": [
          {"value": "heat"},
          {"value": "cool"},
          {"value": "auto"}
        ]
      },
      "state": {"instance": "thermostat", "value": "heat"}
    }
  ],
  "properties": [
    {
      "type": "devices.properties.float",
      "parameters": {"instance": "temperature", "unit": "unit.temperature.celsius"},
      "state": {"instance": "temperature", "value": 23.5}
    }
  ]
}
```

## Структура репозитория

```
wb-ext-conventions/
├── README.md                          # Спецификация
├── schema/
│   └── wb-ext-bridge.schema.json      # JSON Schema для конфига (UI wb-editor)
├── examples/
│   └── wb-ext-bridge.conf             # Пример конфига
└── docs/
    ├── adapter-interface.md           # Спецификация адаптерного интерфейса
    └── auto-discovery.md              # Правила авто-обнаружения
```

## Конвертация типов данных

WB MQTT передаёт все значения как строки (`"0"`, `"1"`, `"23.5"`, `"heat"`). WB Control Tracker отвечает за конвертацию строк в типы канонической модели:

| MQTT строка | `data_type` | Каноническое значение |
|-------------|-------------|-----------------------|
| `"0"`, `"1"` | `bool` | `false`, `true` |
| `"23.5"` | `float` | `23.5` |
| `"heat"` | `enum` | `"heat"` |
| `"255;128;0"` | `string` (color_rgb) | `"255;128;0"` |

Адаптеры получают уже типизированные значения и не работают с сырыми MQTT-строками.

## Расширяемость

### Добавление нового типа устройства

Каждый стандартный тип описан как отдельный definition в `schema/wb-ext-bridge.schema.json`. Чтобы добавить новый тип:

#### 1. Создайте definition

Каждый definition — JSON-объект со следующей структурой:

- `title` — человекочитаемое название (используется в UI и переводах)
- `properties.name` — `$ref` на `#/definitions/device_name`
- `properties.type` — строка с `enum` из одного значения (имя нового типа). Поле `default` равно тому же значению, `options.hidden: true` скрывает его в UI (тип выбирается через `oneOf`-дискриминацию, а не вручную)
- `properties.control` или `properties.map` — для простого или составного устройства
- `required` — обязательные поля
- `additionalProperties: false`

Пример definition для простого типа `co2_sensor`:

```json
"co2_sensor": {
  "type": "object",
  "title": "CO2 sensor",
  "properties": {
    "name": { "$ref": "#/definitions/device_name" },
    "type": {
      "type": "string",
      "enum": ["co2_sensor"],
      "default": "co2_sensor",
      "options": { "hidden": true }
    },
    "control": {
      "$ref": "#/definitions/control_field",
      "title": "CO2"
    }
  },
  "required": ["name", "type", "control"],
  "additionalProperties": false
}
```

#### 2. Добавьте в `oneOf`

В `properties.devices.items.oneOf` добавьте `$ref` на новый definition (перед `custom`):

```json
{ "$ref": "#/definitions/co2_sensor" }
```

Дискриминация `oneOf` работает через поле `type`: каждый стандартный тип имеет `enum` из одного значения, поэтому JSON Schema однозначно определяет, какой definition применить к конкретному устройству.

#### 3. Обновите `not.enum` в `custom`

Definition `custom` содержит `not.enum` — список всех зарезервированных имён стандартных типов. При добавлении нового стандартного типа **обязательно** добавьте его имя в этот список, чтобы кастомные устройства не могли использовать зарезервированное имя.

#### 4. Добавьте переводы

В секцию `translations.ru` добавьте перевод `title` и названий слотов.

### Добавление нового адаптера

Реализуйте четыре метода адаптерного интерфейса (см. [docs/adapter-interface.md](docs/adapter-interface.md)) для трансляции канонических устройств в формат целевой платформы.
