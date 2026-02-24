# Адаптерный интерфейс

Адаптер — компонент, транслирующий канонические устройства в формат внешней платформы.

## Интерфейс

```
interface Adapter {
    OnDeviceAdded(device CanonicalDevice)
    OnStateChanged(device_id string, slot string, value any)
    OnDeviceRemoved(device_id string)
    SetCommandHandler(handler func(device_id string, slot string, value any))
}
```

| Метод | Когда вызывается |
|-------|-----------------|
| `OnDeviceAdded` | Устройство обнаружено, все required-слоты получили начальные значения |
| `OnStateChanged` | Значение слота изменилось (MQTT-контрол обновился) |
| `OnDeviceRemoved` | Устройство удалено (связка убрана, модуль отключён) |
| `SetCommandHandler` | Инициализация — ядро устанавливает callback для команд от платформы |

**Поток команд:** платформа → адаптер → `handler(device_id, slot, value)` → ядро → MQTT publish.

## CanonicalDevice

```
CanonicalDevice {
    id:           string              // slugify от name
    display_name: string              // имя из связки, профиля или автогенерация
    type:         string              // идентификатор типа устройства
    device_type:  DeviceType | null   // описание типа (null для кастомных)
    source:       string              // "binding" | "override" | "profile" | "auto"
    properties:   map[string]any      // значения ro-слотов
    capabilities: map[string]any      // значения rw-слотов
}
```

## DeviceType

```
DeviceType {
    id:       string              // "thermostat", "switch", ...
    category: string              // "climate", "light", "switch", "cover", "sensor"
    slots:    map[string]SlotDef
}

SlotDef {
    data_type:   string           // "bool", "float", "enum", "string", "int"
    access:      string           // "rw" (capability) или "ro" (property)
    required:    bool
    constraints: Constraints      // опционально
}

Constraints {
    min:    number
    max:    number
    step:   number
    values: []string              // для enum
}
```

Поле `access` определяет, попадает ли значение в `capabilities` (`rw`) или `properties` (`ro`).

## Жизненный цикл

```
OnDeviceAdded(device)  →  OnStateChanged(id, slot, value) ...  →  OnDeviceRemoved(id)
```

`OnDeviceAdded` вызывается после того, как все required-слоты получат начальные значения из MQTT. В runtime-режиме discovery может вызвать `OnDeviceAdded` в любой момент при появлении нового MQTT-устройства.

## Кастомные типы

Для кастомных устройств `device_type = null`. Адаптер получает слоты как есть и сам решает: маппить в нативную сущность, создать generic-устройство или проигнорировать.
