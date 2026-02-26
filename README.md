# wb-ext-conventions

> **Внимание:** Это черновик конвенции. Она ещё не принята официально и может существенно измениться.

**Универсальный слой абстракции устройств для Wiren Board**

## Проблема

Wiren Board представляет устройства как плоские MQTT-контролы — отдельные топики с единственным значением. Внешние платформы (Home Assistant, HomeKit, Яндекс Алиса) работают с составными сущностями: термостат = температура + уставка + режим + нагрев.

Нет стандартного способа объявить, что набор контролов из разных модулей образует одно устройство.

## Решение

Каноническая модель устройств — платформонезависимое промежуточное представление:

```
MQTT-контролы WB
       │
       ▼
  Discovery Engine ← bindings + профили + fallback
       │
       ▼
  Канонические устройства (DeviceType + Capabilities + Properties)
       │                        ▲
       │                  overrides (пост-обработка)
       │
       ├──► Адаптер Home Assistant
       ├──► Адаптер HomeKit
       └──► Адаптер Яндекс Алисы
```

Два компонента конвенции:

- **Типы устройств** — стандартные (switch, dimmer, thermostat и т.д.) + произвольные пользовательские типы
- **Discovery Engine** — автоматическое обнаружение устройств

## Автоматическое обнаружение

Большинство устройств обнаруживается без конфигурации. Discovery Engine обрабатывает контролы в порядке приоритета:

1. **Bindings** (`bindings[]`) — ручные, из конфигурации. Для кросс-модульных устройств и пользовательских типов.
2. **Профили модулей** — автоматически. Профиль описывает структуру известного модуля WB и создаёт составные устройства.
3. **Per-control fallback** — автоматически. Каждый оставшийся контрол становится отдельным устройством по таблице маппинга.

Контрол, обработанный на более высоком уровне, пропускается нижними уровнями. Контролы модуля, не покрытые профилем, проваливаются в fallback.

После обнаружения применяются **overrides** (`overrides[]`) — изменение типа или имени fallback-устройств.

## Типы устройств

### Простые (один слот)

| Тип | Категория | Слот | Тип данных | Описание |
|-----|-----------|------|------------|----------|
| `switch` | switch | on_off | bool | Выключатель |
| `temperature_sensor` | sensor | temperature | float | Датчик температуры |
| `humidity_sensor` | sensor | humidity | float | Датчик влажности |
| `power_sensor` | sensor | power | float | Датчик мощности |
| `voltage_sensor` | sensor | voltage | float | Датчик напряжения |
| `illuminance_sensor` | sensor | illuminance | float | Датчик освещённости |
| `binary_sensor` | sensor | state | bool | Бинарный датчик |
| `contact_sensor` | sensor | contact | bool | Датчик открытия |
| `motion_sensor` | sensor | motion | bool | Датчик движения |
| `leak_sensor` | sensor | leak | bool | Датчик протечки |

### Составные (несколько слотов)

| Тип | Категория | Обязательные слоты | Опциональные слоты |
|-----|-----------|-------------------|-------------------|
| `dimmer` | light | brightness (float) | on_off (bool) |
| `rgb_light` | light | on_off (bool), color (string) | brightness (float) |
| `thermostat` | climate | current_temperature (float), target_temperature (float) | is_heating (bool), mode (enum), on_off (bool) |
| `cover` | cover | position (float) | on_off (bool) |

Слот `color` — строка в формате WB `"R;G;B"` (например, `"255;128;0"`).

### Пользовательские типы

Произвольная строка в поле `type`, отличная от зарезервированных стандартных имён. Произвольный набор слотов. Адаптер сам решает, как транслировать, или игнорирует неизвестный тип.

## Конфигурация

Пустой конфиг `{}` валиден — всё обнаруживается автоматически. Конфиг нужен только для:

- **Bindings** (`bindings[]`) — кросс-модульные составные устройства и пользовательские типы
- **Overrides** (`overrides[]`) — изменение типа или имени fallback-устройства
- **Настройки discovery** — исключения для контролов и устройств

```json
{
  "bindings": [
    {
      "name": "Термостат гостиной",
      "type": "thermostat",
      "map": {
        "current_temperature": "wb-msw-v3_1/Temperature",
        "target_temperature": "thermostat_setpoints/living_room",
        "is_heating": "wb-mr6cu_97/K1",
        "mode": "thermostat_modes/living_room"
      }
    },
    {
      "name": "Телевизор гостиной",
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
    { "control": "wb-mr6cu_97/K3", "type": "cover", "name": "Шторы спальни" }
  ],
  "discovery": {
    "enabled": true,
    "exclude": ["wb-mr6cu_97/K5", "wb-mr6cu_97/K6"],
    "exclude_devices": ["system"]
  }
}
```

**Binding** — `name`, `type` и `map` (слот → контрол WB). Для простых типов с одним слотом можно использовать `control` вместо `map`.

**Override** — `control` (обязательно), `type` и/или `name` (опционально). Идентифицирует устройство по одному контролу, поэтому применяется только к fallback-устройствам.

## Discovery Engine

Автоматическое обнаружение устройств Wiren Board и создание канонических устройств.

### Приоритеты

```
1. Bindings (bindings[])          ← кросс-модульные + пользовательские
2. Профили модулей               ← известные модули WB
3. Per-control fallback           ← таблица маппинга type → DeviceType
```

Если контрол уже привязан на более высоком уровне, нижние уровни его пропускают. Контролы модуля, не покрытые профилем, проваливаются в fallback.

После обнаружения применяются overrides (`overrides[]`) — изменение типа или имени fallback-устройств.

При `discovery.enabled: false` автоматическое обнаружение (профили + fallback) отключается; работают только bindings.

### Профили модулей

Профиль описывает, как модель устройства WB преобразуется в набор канонических устройств. Профиль знает структуру модуля и создаёт **составные устройства** (например, диммер = реле + канал яркости).

#### Формат

```yaml
model: wb-mdm3                          # совпадает с префиксом MQTT-имени
vendor: Wiren Board
description: "3-канальный диммер"
aliases:                                 # альтернативные модели (опционально)
  - wb-mdm3-mini

devices:
  - name_template: "{module_title} Dimmer {n}"
    type: dimmer
    repeat: 3                            # n = 1..3
    map:
      on_off: "K{n}"
      brightness: "Channel {n}"
```

Результат для MQTT-устройства `wb-mdm3_1` — 3 канонических устройства:
- `WB-MDM3 Dimmer 1` (dimmer: K1 + Channel 1)
- `WB-MDM3 Dimmer 2` (dimmer: K2 + Channel 2)
- `WB-MDM3 Dimmer 3` (dimmer: K3 + Channel 3)

#### Пример: WB-LED (режим RGB+W)

```yaml
model: wb-led
vendor: Wiren Board
description: "4-канальный LED-контроллер (режим RGB+W)"

devices:
  - name_template: "{module_title} RGB"
    type: rgb_light
    map:
      on_off: "On"
      color: "RGB Palette"
      brightness: "RGB Brightness"
  - name_template: "{module_title} White"
    type: dimmer
    map:
      on_off: "On"
      brightness: "Channel 4"
```

Результат для MQTT-устройства `wb-led_1` — 2 канонических устройства:
- `WB-LED RGB` (rgb_light: On + RGB Palette + RGB Brightness)
- `WB-LED White` (dimmer: On + Channel 4)

#### Поля

| Поле | Описание |
|------|----------|
| `model` | Идентификатор модели, kebab-case (например, `wb-mdm3`) |
| `aliases` | Альтернативные модели с идентичной структурой |
| `devices[].name_template` | Шаблон имени |
| `devices[].type` | Тип канонического устройства |
| `devices[].repeat` | Количество повторений (подставляет `{n}`) |
| `devices[].control` / `map` | Один слот или маппинг слот → контрол |

Переменные шаблона: `{n}`, `{module_title}`, `{device_name}`, `{address}`.

#### Извлечение модели

MQTT-имена устройств WB следуют формату `<model>_<address>` (regex `^(.+)_(\d+)$`). При поиске профиля проверяются и `model`, и `aliases`. Если имя не соответствует шаблону, профили не применяются.

### Per-control fallback

Если контрол не привязан через binding или профиль — один контрол = одно каноническое устройство.

| Тип контрола WB | Единица | DeviceType |
|-----------------|---------|------------|
| `switch` | — | `switch` |
| `range` | — | `dimmer` |
| `temperature` | `°C` | `temperature_sensor` |
| `rel_humidity` | `%` | `humidity_sensor` |
| `power` | `W` | `power_sensor` |
| `voltage` | `V` | `voltage_sensor` |
| `lux` | `lx` | `illuminance_sensor` |

Тип контрола берётся из метаданных MQTT (`.../meta/type`). Единица измерения уточняет маппинг для неоднозначных типов (например, `value`). Контролы, отсутствующие в таблице, игнорируются.

### Исключения

**Исключения** (`discovery.exclude`, `discovery.exclude_devices`) — контролы и целые MQTT-устройства, которые движок игнорирует. Bindings имеют приоритет: контрол в `exclude` + в `bindings[]` обрабатывается через binding.

### Именование и идентификаторы

| Источник | Формат имени | Формат ID |
|----------|-------------|-----------|
| Binding | `name` из конфига | slugify от `name` |
| Профиль | `name_template` из профиля | `{device_name}_{slug(type)}_{n}` |
| Fallback | `{device}/{control}` | `auto_{device}_{control}` |
| Override | оригинал или `name` из override | оригинал (без изменений) |
