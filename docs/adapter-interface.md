# Спецификация адаптерного интерфейса

Адаптер — компонент, отвечающий за трансляцию канонических устройств wb-ext в формат конкретной внешней платформы (Home Assistant, Apple HomeKit, Яндекс Алиса и др.).

## Интерфейс

Каждый адаптер реализует четыре метода:

```
interface Adapter {
    OnDeviceAdded(device CanonicalDevice)
    OnStateChanged(device_id string, slot string, value any)
    OnDeviceRemoved(device_id string)
    SetCommandHandler(handler func(device_id string, slot string, value any))
}
```

### OnDeviceAdded

Вызывается при обнаружении нового устройства. Адаптер получает полное описание канонического устройства и должен зарегистрировать его на целевой платформе.

**Параметры:**
- `device` — каноническое устройство (см. структуру CanonicalDevice ниже)

**Действия адаптера:**
- Создать сущность на целевой платформе
- Сопоставить capabilities и properties с нативными атрибутами платформы
- Опубликовать discovery-информацию (если платформа поддерживает автообнаружение)

### OnStateChanged

Вызывается при изменении значения любого слота устройства (capability или property). Адаптер обновляет состояние на целевой платформе.

**Параметры:**
- `device_id` — идентификатор устройства (генерируется из `name` через slugify)
- `slot` — имя слота (например, `current_temperature`, `mode`)
- `value` — новое значение (тип зависит от `data_type` слота)

**Действия адаптера:**
- Обновить состояние соответствующего атрибута на целевой платформе

### OnDeviceRemoved

Вызывается при удалении устройства (например, при удалении записи из конфига или отключении модуля).

**Параметры:**
- `device_id` — идентификатор удаляемого устройства

**Действия адаптера:**
- Удалить сущность с целевой платформы
- Очистить ресурсы, связанные с устройством

### SetCommandHandler

Устанавливает callback для обработки команд от внешней платформы. Ядро wb-ext вызывает этот метод один раз при инициализации адаптера.

**Параметры:**
- `handler` — функция обратного вызова с сигнатурой `(device_id, slot, value)`

**Контракт:**
- Когда пользователь управляет устройством через внешнюю платформу, адаптер вызывает `handler` с нужными параметрами
- Ядро wb-ext транслирует команду в MQTT-сообщение для соответствующего WB-контрола

## Структура CanonicalDevice

```
CanonicalDevice {
    id:           string              // Уникальный идентификатор (slugify от name)
    display_name: string              // Человекочитаемое имя (из поля name конфига)
    type:         string              // Тип устройства (ссылка на DeviceType)
    device_type:  DeviceType          // Полное описание типа устройства
    source:       string              // Источник: "config", "profile" или "auto"
    properties:   map[string]any      // Текущие значения properties
    capabilities: map[string]any      // Текущие значения capabilities
}
```

Поле `id` генерируется автоматически из `name` через slugify (транслитерация + замена пробелов на дефисы, например `"Термостат гостиная"` → `"termostat-gostinaya"`).

Поле `source` указывает, каким образом устройство было создано:
- `"config"` — явная запись в `devices[]`
- `"profile"` — обнаружено через профиль модуля (см. [docs/discovery-engine.md](discovery-engine.md))
- `"auto"` — обнаружено поконтрольным фоллбэком

Поля `properties` и `capabilities` содержат карту `имя_слота -> текущее_значение`. Ключи соответствуют именам, объявленным в DeviceType.

## Структура DeviceType

`DeviceType` описывает тип устройства — набор слотов с их типами данных, ограничениями и разделением на capabilities (управляемые) и properties (только для чтения).

```
DeviceType {
    id:       string              // Идентификатор типа (например, "thermostat", "switch")
    category: string              // Категория устройства (см. ниже)
    metadata: map[string]any      // Произвольные метаданные типа
    slots:    map[string]SlotDef  // Описание слотов
}

SlotDef {
    data_type:   string           // Тип данных: "bool", "float", "enum", "string", "int"
    access:      string           // "rw" (capability) или "ro" (property)
    required:    bool             // Обязателен ли слот
    constraints: Constraints      // Ограничения значений (опционально)
}

Constraints {
    min:    number                // Минимальное значение (для float/int)
    max:    number                // Максимальное значение (для float/int)
    step:   number                // Шаг изменения (для float/int)
    values: []string              // Допустимые значения (для enum)
}
```

Поле `access` определяет, является ли слот capability (управляемый, `"rw"`) или property (только для чтения, `"ro"`). Это разделение напрямую влияет на то, в какую карту попадает значение в `CanonicalDevice` — `capabilities` или `properties`.

### Значения `category`

| Категория | Описание | Примеры типов |
|-----------|----------|---------------|
| `switch` | Коммутация | `switch` |
| `light` | Освещение | `dimmer`, `rgb_light` |
| `climate` | Климат | `thermostat` |
| `cover` | Шторы/жалюзи | `cover` |
| `sensor` | Датчики | `temperature_sensor`, `humidity_sensor`, `motion_sensor` и др. |

Для кастомных типов (не входящих в стандартные 14) `device_type` равен `null`. Адаптер получает слоты как есть и сам решает, как их интерпретировать (см. раздел «Обработка кастомных типов»).

## Жизненный цикл устройства

```
Устройство появилось                  Устройство удалено
       │                                     │
       ▼                                     ▼
  OnDeviceAdded(device)              OnDeviceRemoved(device_id)
       │
       ▼
  OnStateChanged(id, slot, value)  ◄── повторяется при каждом
  OnStateChanged(id, slot, value)      изменении состояния
  OnStateChanged(id, slot, value)
       ...
```

1. **Появление устройства.** При загрузке конфига ядро создаёт канонические устройства. Как только все обязательные (required) слоты получают начальные значения из MQTT, вызывается `OnDeviceAdded`. В автоматическом режиме обнаружения (`discovery.enabled: true`) `OnDeviceAdded` может вызываться не только при старте, но и при появлении нового MQTT-устройства в рантайме.

2. **Обновление состояния.** При каждом изменении MQTT-контрола, привязанного через конфиг, вызывается `OnStateChanged` с идентификатором устройства, именем слота и новым значением.

3. **Удаление устройства.** При удалении записи из конфига или потере связи с модулем вызывается `OnDeviceRemoved`.

## Поток команд

Когда пользователь управляет устройством через внешнюю платформу, команда проходит следующий путь:

```
Внешняя система (HA / HomeKit / Алиса)
       │
       ▼
    Адаптер
       │  (вызывает handler, установленный через SetCommandHandler)
       ▼
  handler(device_id, slot, value)
       │
       ▼
    Ядро wb-ext
       │  (находит устройство в конфиге, определяет WB-контрол)
       ▼
  MQTT publish → /devices/<device>/controls/<control>/on → value
```

Пример: пользователь устанавливает температуру 24°C в Home Assistant:

1. HA публикует в `wb-ext/living_room_thermostat/target_temperature/set` значение `24`
2. Адаптер HA получает команду и вызывает `handler("living_room_thermostat", "target_temperature", 24.0)`
3. Ядро находит устройство `living_room_thermostat` в конфиге, слот `target_temperature` → `thermostat_setpoints/living_room`
4. Ядро публикует `24` в `/devices/thermostat_setpoints/controls/living_room/on`

## Рекомендации по написанию адаптера

### Регистрация устройства

В `OnDeviceAdded` адаптер должен:

1. Определить нативный тип сущности по `device.device_type.metadata.category`:
   - `climate` → термостат
   - `light` → свет
   - `switch` → выключатель
   - `cover` → шторы
   - `sensor` → датчик

2. Для каждого слота найти соответствие на целевой платформе по имени слота.

3. Учитывать `constraints` (min, max, step) при настройке элементов управления.

4. Учитывать поле `required` — обязательные слоты должны быть всегда представлены на целевой платформе.

### Обновление состояния

В `OnStateChanged` адаптер должен:

1. Найти зарегистрированное устройство по `device_id`.
2. Определить нативный атрибут по имени слота.
3. При необходимости преобразовать значение (например, `bool` в числовой формат HomeKit).
4. Обновить состояние на целевой платформе.

### Обработка команд

Адаптер подписывается на команды от целевой платформы и транслирует их через callback:

1. Извлечь идентификатор устройства и слот из нативной команды.
2. Преобразовать значение из формата платформы в канонический формат.
3. Вызвать `handler(device_id, slot, value)`.

### Обработка кастомных типов

Кастомные устройства (тип не входит в стандартные 14) приходят в адаптер со следующими особенностями:

- `device.type` — произвольная строка (например, `"tv"`, `"fan"`, `"irrigation"`)
- `device.device_type` — `null` (описание типа отсутствует)
- `device.capabilities` и `device.properties` — заполнены на основе значений WB-контролов из `map`, но без семантической информации о `data_type`, `access` и `constraints`

Рекомендуемое поведение адаптера:

1. **Если адаптер умеет маппить тип** — создать нативную сущность (например, `"tv"` → пульт ДУ в Home Assistant).
2. **Если тип неизвестен** — залогировать warning и либо пропустить устройство, либо создать generic-сущность с прямым доступом к слотам.

Пример обработки в `OnDeviceAdded`:

```python
def on_device_added(self, device: CanonicalDevice):
    if device.device_type is not None:
        # Стандартный тип — маппинг по category и слотам
        self._register_standard(device)
    elif device.type in self.custom_handlers:
        # Известный кастомный тип — специализированный обработчик
        self.custom_handlers[device.type](device)
    else:
        # Неизвестный кастомный тип
        logger.warning(
            "Unknown device type '%s' for device '%s', skipping",
            device.type, device.display_name
        )
```

### Обработка ошибок

- Если устройство не найдено по `device_id` — залогировать и игнорировать.
- Если слот не распознан — залогировать и игнорировать.
- Если значение не проходит `constraints` — отклонить команду.

## Примеры трансляции

Ниже приведены краткие примеры того, как адаптер транслирует каноническое устройство в формат целевой платформы.

### Home Assistant

При `OnDeviceAdded` для термостата адаптер публикует MQTT Discovery:

```json
{
  "name": "Термостат гостиная",
  "unique_id": "wb_ext_living_room_thermostat",
  "modes": ["off", "heat", "cool", "auto"],
  "current_temperature_topic": "wb-ext/living_room_thermostat/current_temperature",
  "temperature_state_topic": "wb-ext/living_room_thermostat/target_temperature",
  "temperature_command_topic": "wb-ext/living_room_thermostat/target_temperature/set",
  "min_temp": 5,
  "max_temp": 35,
  "temp_step": 0.5
}
```

Маппинг слотов →HA:
- `on_off` → `switch` / `light` (в зависимости от category)
- `temperature` → `current_temperature`
- `temperature_setting` → `temperature`
- `brightness` → `brightness` (пересчёт 0-100 → 0-255)
- `mode` → `hvac_mode`

### Apple HomeKit

При `OnDeviceAdded` для термостата адаптер создаёт аксессуар с сервисом `HeaterCooler`:

```json
{
  "accessory": "wb_ext_living_room_thermostat",
  "service": "HeaterCooler",
  "characteristics": {
    "Active": 1,
    "CurrentTemperature": 22.5,
    "HeatingThresholdTemperature": 23.0,
    "CurrentHeaterCoolerState": 2,
    "TargetHeaterCoolerState": 1
  }
}
```

Маппинг слотов →HomeKit:
- `on_off` → `Active` / `On`
- `temperature` → `CurrentTemperature`
- `temperature_setting` → `HeatingThresholdTemperature` / `CoolingThresholdTemperature`
- `brightness` → `Brightness`
- `color_rgb` → `Hue` + `Saturation` (требуется конвертация `"R;G;B"` → HSV)

### Яндекс Алиса

При `OnDeviceAdded` для термостата адаптер формирует описание устройства:

```json
{
  "id": "living_room_thermostat",
  "name": "Термостат гостиная",
  "type": "devices.types.thermostat",
  "capabilities": [
    {
      "type": "devices.capabilities.range",
      "parameters": {
        "instance": "temperature",
        "unit": "unit.temperature.celsius",
        "range": {"min": 5, "max": 35, "precision": 0.5}
      }
    }
  ],
  "properties": [
    {
      "type": "devices.properties.float",
      "parameters": {"instance": "temperature", "unit": "unit.temperature.celsius"}
    }
  ]
}
```

Маппинг слотов →Алиса:
- `on_off` → `devices.capabilities.on_off`
- `temperature` → `devices.properties.float` (instance: `temperature`)
- `temperature_setting` → `devices.capabilities.range` (instance: `temperature`)
- `brightness` → `devices.capabilities.range` (instance: `brightness`)
- `color_rgb` → `devices.capabilities.color_setting` (instance: `rgb`)
- `mode` → `devices.capabilities.mode`
