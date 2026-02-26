# wb-ext-conventions

> **Note:** This is a draft convention. It has not been officially adopted yet and may change significantly.

**Universal device abstraction layer for Wiren Board**

## Problem

Wiren Board represents devices as flat MQTT controls — individual topics with a single value. External platforms (Home Assistant, HomeKit, Yandex Alice) work with composite entities: a thermostat = temperature + setpoint + mode + heating.

There is no standard way to declare that a set of controls from different modules forms a single device.

## Solution

A canonical device model — a platform-independent intermediate representation:

```
WB MQTT controls
       │
       ▼
  Discovery Engine ← bindings + profiles + fallback
       │
       ▼
  Canonical devices (DeviceType + Capabilities + Properties)
       │                        ▲
       │                  overrides (post-processing)
       │
       ├──► Home Assistant adapter
       ├──► HomeKit adapter
       └──► Yandex Alice adapter
```

Three components of the convention:

- **Device types** — standard (switch, dimmer, thermostat, etc.) + arbitrary custom types
- **Discovery Engine** — automatic device detection
- **Adapter interface** — contract for translating to the target platform's format

## Automatic Discovery

Most devices are discovered without configuration. The Discovery Engine processes controls in priority order:

1. **Bindings** (`bindings[]`) — manual, from config. For cross-module devices and custom types.
2. **Module profiles** — automatic. A profile describes the structure of a known WB module and creates composite devices.
3. **Per-control fallback** — automatic. Each remaining control becomes an individual device based on a mapping table.

A control processed at a higher level is skipped by lower levels. Module controls not covered by a profile fall through to the fallback.

After discovery, **overrides** (`overrides[]`) are applied — changing the type or name of fallback devices.

## Device Types

### Simple (single slot)

| Type | Category | Slot | Data type | Description |
|------|----------|------|-----------|-------------|
| `switch` | switch | on_off | bool | Switch |
| `temperature_sensor` | sensor | temperature | float | Temperature sensor |
| `humidity_sensor` | sensor | humidity | float | Humidity sensor |
| `power_sensor` | sensor | power | float | Power sensor |
| `voltage_sensor` | sensor | voltage | float | Voltage sensor |
| `illuminance_sensor` | sensor | illuminance | float | Illuminance sensor |
| `binary_sensor` | sensor | state | bool | Binary sensor |
| `contact_sensor` | sensor | contact | bool | Contact sensor |
| `motion_sensor` | sensor | motion | bool | Motion sensor |
| `leak_sensor` | sensor | leak | bool | Leak sensor |

### Composite (multiple slots)

| Type | Category | Required slots | Optional slots |
|------|----------|---------------|----------------|
| `dimmer` | light | brightness (float) | on_off (bool) |
| `rgb_light` | light | on_off (bool), color (string) | brightness (float) |
| `thermostat` | climate | current_temperature (float), target_temperature (float) | is_heating (bool), mode (enum), on_off (bool) |
| `cover` | cover | position (float) | on_off (bool) |

The `color` slot is a string in WB convention format `"R;G;B"` (e.g., `"255;128;0"`).

### Custom Types

An arbitrary string in the `type` field, different from the reserved standard type names. An arbitrary set of slots. The adapter decides how to translate, or ignores the unknown type.

## Configuration

An empty config `{}` is valid — everything is discovered automatically. Config is only needed for:

- **Bindings** (`bindings[]`) — cross-module composite devices and custom types
- **Overrides** (`overrides[]`) — changing the type or name of a fallback device
- **Discovery settings** — exclusions for controls and devices

```json
{
  "bindings": [
    {
      "name": "Living Room Thermostat",
      "type": "thermostat",
      "map": {
        "current_temperature": "wb-msw-v3_1/Temperature",
        "target_temperature": "thermostat_setpoints/living_room",
        "is_heating": "wb-mr6cu_97/K1",
        "mode": "thermostat_modes/living_room"
      }
    },
    {
      "name": "Living Room TV",
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
    { "control": "wb-mr6cu_97/K3", "type": "cover", "name": "Bedroom Curtains" }
  ],
  "discovery": {
    "enabled": true,
    "exclude": ["wb-mr6cu_97/K5", "wb-mr6cu_97/K6"],
    "exclude_devices": ["system"]
  }
}
```

**Binding** — `name`, `type`, and `map` (slot → WB control). For simple types with a single slot, `control` can be used instead of `map`.

**Override** — `control` (required), `type` and/or `name` (optional). Identifies a device by a single control, therefore only applies to fallback devices.

## Discovery Engine

Automatic discovery of Wiren Board devices and creation of canonical devices.

### Priorities

```
1. Bindings (bindings[])          ← cross-module + custom
2. Module profiles                ← known WB modules
3. Per-control fallback           ← type → DeviceType mapping table
```

If a control is already bound at a higher level, lower levels skip it. Module controls not covered by a profile fall through to the fallback.

After discovery, overrides (`overrides[]`) are applied — changing the type or name of fallback devices.

When `discovery.enabled: false`, automatic discovery (profiles + fallback) is disabled; only bindings work.

### Module Profiles

A profile describes how a WB device model is converted into a set of canonical devices. A profile knows the module's structure and creates **composite devices** (e.g., dimmer = relay + brightness channel).

#### Format

```yaml
model: wb-mdm3                          # matches MQTT name prefix
vendor: Wiren Board
description: "3-channel dimmer"
aliases:                                 # alternative models (optional)
  - wb-mdm3-mini

devices:
  - name_template: "{module_title} Dimmer {n}"
    type: dimmer
    repeat: 3                            # n = 1..3
    map:
      on_off: "K{n}"
      brightness: "Channel {n}"
```

Result for MQTT device `wb-mdm3_1` — 3 canonical devices:
- `WB-MDM3 Dimmer 1` (dimmer: K1 + Channel 1)
- `WB-MDM3 Dimmer 2` (dimmer: K2 + Channel 2)
- `WB-MDM3 Dimmer 3` (dimmer: K3 + Channel 3)

#### Fields

| Field | Description |
|-------|-------------|
| `model` | Model identifier, kebab-case (e.g., `wb-mdm3`) |
| `aliases` | Alternative models with identical structure |
| `devices[].name_template` | Name template |
| `devices[].type` | Canonical device type |
| `devices[].repeat` | Repeat count (substitutes `{n}`) |
| `devices[].control` / `map` | Single slot or slot-to-control mapping |

Template variables: `{n}`, `{module_title}`, `{device_name}`, `{address}`.

#### Model Extraction

WB MQTT device names follow the convention `<model>_<address>` (regex `^(.+)_(\d+)$`). When looking up a profile, both `model` and `aliases` are checked. If the name doesn't match the pattern, profiles are not applied.

### Per-Control Fallback

If a control is not bound via a binding or a profile — one control = one canonical device.

| WB control type | Unit | DeviceType |
|----------------|------|------------|
| `switch` | — | `switch` |
| `range` | — | `dimmer` |
| `temperature` | `°C` | `temperature_sensor` |
| `rel_humidity` | `%` | `humidity_sensor` |
| `power` | `W` | `power_sensor` |
| `voltage` | `V` | `voltage_sensor` |
| `lux` | `lx` | `illuminance_sensor` |

The control type is taken from MQTT metadata (`.../meta/type`). The unit of measurement refines the mapping for ambiguous types (e.g., `value`). Controls not in the table are ignored.

### Exclusions

**Exclusions** (`discovery.exclude`, `discovery.exclude_devices`) — controls and entire MQTT devices that the engine ignores. Bindings take priority: a control in `exclude` + in `bindings[]` is processed via the binding.

### Naming and Identifiers

| Source | Name format | ID format |
|--------|------------|-----------|
| Binding | `name` from config | slugify of `name` |
| Profile | `name_template` from profile | `{device_name}_{slug(type)}_{n}` |
| Fallback | `{device}/{control}` | `auto_{device}_{control}` |
| Override | original or `name` from override | original (unchanged) |

## Adapter Interface

An adapter is a component that translates canonical devices into the format of an external platform.

### Interface

```
interface Adapter {
    OnDeviceAdded(device CanonicalDevice)
    OnStateChanged(device_id string, slot string, value any)
    OnDeviceRemoved(device_id string)
    SetCommandHandler(handler func(device_id string, slot string, value any))
}
```

| Method | When called |
|--------|------------|
| `OnDeviceAdded` | Device discovered, all required slots have received initial values |
| `OnStateChanged` | Slot value changed (MQTT control updated) |
| `OnDeviceRemoved` | Device removed (binding removed, module disconnected) |
| `SetCommandHandler` | Initialization — the core sets a callback for commands from the platform |

**Command flow:** platform → adapter → `handler(device_id, slot, value)` → core → MQTT publish.

### CanonicalDevice

```
CanonicalDevice {
    id:           string              // slugify of name
    display_name: string              // name from binding, profile, or auto-generated
    type:         string              // device type identifier
    device_type:  DeviceType | null   // type definition (null for custom)
    source:       string              // "binding" | "override" | "profile" | "auto"
    properties:   map[string]any      // read-only slot values
    capabilities: map[string]any      // read-write slot values
}
```

### DeviceType

```
DeviceType {
    id:       string              // "thermostat", "switch", ...
    category: string              // "climate", "light", "switch", "cover", "sensor"
    slots:    map[string]SlotDef
}

SlotDef {
    data_type:   string           // "bool", "float", "enum", "string", "int"
    access:      string           // "rw" (capability) or "ro" (property)
    required:    bool
    constraints: Constraints      // optional
}

Constraints {
    min:    number
    max:    number
    step:   number
    values: []string              // for enum
}
```

The `access` field determines whether the value goes into `capabilities` (`rw`) or `properties` (`ro`).

### Lifecycle

```
OnDeviceAdded(device)  →  OnStateChanged(id, slot, value) ...  →  OnDeviceRemoved(id)
```

`OnDeviceAdded` is called after all required slots have received their initial values from MQTT. At runtime, discovery may call `OnDeviceAdded` at any moment when a new MQTT device appears.

### Custom Types

For custom devices, `device_type = null`. The adapter receives slots as-is and decides: map to a native entity, create a generic device, or ignore.

## Data Type Conversion

WB MQTT transmits values as strings. The core converts them to canonical model types:

| MQTT | `data_type` | Canonical type |
|------|-------------|---------------|
| `"0"` / `"1"` | `bool` | `false` / `true` |
| `"23.5"` | `float` | `23.5` |
| `"42"` | `int` | `42` |
| `"heat"` | `enum` | `"heat"` |
| `"255;128;0"` | `string` | `"255;128;0"` |

## Example: Thermostat Path

```
MQTT controls                        Canonical device                   Platforms
─────────────────────────            ──────────────────────             ──────────
wb-msw-v3_1/Temperature  ──┐        CanonicalDevice:                   HA: MQTT Discovery
thermostat_setpoints/    ──┤          type: thermostat                  HomeKit: HeaterCooler
  living_room              ├──►      properties:                        Alice: devices.types.
wb-mr6cu_97/K1           ──┤          current_temperature: 23.5           thermostat
thermostat_modes/        ──┘          is_heating: true
  living_room                        capabilities:
                                       target_temperature: 22.0
                                       mode: "heat"
```
