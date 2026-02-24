# wb-ext-conventions

**Универсальный слой абстракции устройств Wiren Board**

## Проблема

Wiren Board представляет устройства как плоские MQTT-контролы — отдельные топики с одним значением. Внешние платформы (Home Assistant, HomeKit, Алиса) работают с составными сущностями: термостат = температура + уставка + режим + нагрев.

Нет стандартного способа объявить, что набор контролов с разных модулей образует одно устройство.

## Решение

Каноническая модель устройств — платформо-независимое промежуточное представление:

```
WB MQTT контролы
       │
       ▼
  Discovery Engine ← bindings + overrides + profiles + fallback
       │
       ▼
  Канонические устройства (DeviceType + Capabilities + Properties)
       │
       ├──► Адаптер HA
       ├──► Адаптер HomeKit
       └──► Адаптер Алиса
```

Три компонента конвенции:

- **Типы устройств** — стандартные (switch, dimmer, thermostat и т.д.) + произвольные кастомные
- **[Discovery Engine](docs/discovery-engine.md)** — автоматическое обнаружение устройств; конфиг нужен только для кросс-модульных связок и переопределений
- **[Адаптерный интерфейс](docs/adapter-interface.md)** — контракт трансляции в формат целевой платформы

## Типы устройств

### Простые (один слот)

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

### Составные (несколько слотов)

| Тип | Обязательные слоты | Опциональные слоты |
|-----|-------------------|-------------------|
| `dimmer` | on_off, brightness | — |
| `rgb_light` | on_off, color | brightness |
| `thermostat` | current_temperature, target_temperature | is_heating, mode, on_off |
| `cover` | position | on_off |

### Кастомные типы

Произвольная строка в поле `type`, отличная от зарезервированных имён стандартных типов. Произвольный набор слотов. Адаптер сам решает, как транслировать, или игнорирует неизвестный тип.

## Конфигурация

Пустой конфиг `{}` валиден — всё обнаруживается автоматически. Конфиг нужен только для:

- **Связки** (`bindings[]`) — кросс-модульные составные устройства и кастомные типы
- **Переопределения** (`overrides[]`) — смена типа или имени авто-обнаруженного устройства
- **Настройки discovery** — исключения контролов и устройств

```json
{
  "bindings": [
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
  ],
  "overrides": [
    { "control": "wb-mr6cu_97/K3", "type": "cover", "name": "Шторы спальня" }
  ],
  "discovery": {
    "enabled": true,
    "exclude": ["wb-mr6cu_97/K5", "wb-mr6cu_97/K6"],
    "exclude_devices": ["system"]
  }
}
```

**Связка** — `name`, `type` и `map` (слот → WB-контрол). Для простых типов с одним слотом вместо `map` можно указать `control`.

**Override** — `control` (обязательно), `type` и/или `name` (опционально). Работает только для типов с одним обязательным слотом.

## Пример: путь термостата

```
MQTT-контролы (3 модуля)          Каноническое устройство            Платформы
─────────────────────────         ──────────────────────             ──────────
wb-msw-v3_1/Temperature  ──┐     CanonicalDevice:                  HA: MQTT Discovery
thermostat_setpoints/    ──┼──►    type: thermostat                 HomeKit: HeaterCooler
  living_room              │       properties:                      Алиса: devices.types.
wb-mr6cu_97/K1           ──┘         current_temperature: 23.5        thermostat
                                     is_heating: true
                                   capabilities:
                                     target_temperature: 22.0
                                     mode: "heat"
```

## Конвертация типов данных

WB MQTT передаёт значения как строки. Ядро конвертирует их в типы канонической модели:

| MQTT | `data_type` | Канонический тип |
|------|-------------|-----------------|
| `"0"` / `"1"` | `bool` | `false` / `true` |
| `"23.5"` | `float` | `23.5` |
| `"heat"` | `enum` | `"heat"` |
| `"255;128;0"` | `string` | `"255;128;0"` |
