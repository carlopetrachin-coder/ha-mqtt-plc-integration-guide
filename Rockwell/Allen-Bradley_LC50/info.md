# Allen-Bradley LC50 — Integrazione Home Assistant

## Scopo

Integrazione PLC Allen-Bradley CompactLogix (LC50) con Home Assistant tramite bridge MQTT custom.
Driver: **pylogix** (protocollo EtherNet/IP nativo).

## Architettura

```
PLC (EtherNet/IP)  ──TCP──>  Bridge Addon (pylogix)  ──MQTT──>  Home Assistant
                                                       <──MQTT──  (comandi)
```

Il bridge addon legge/scrive tag PLC ciclicamente e pubblica su MQTT.
HA consuma i topic MQTT come entita (sensor, binary_sensor, switch).

## Requisiti

| Componente | Dettaglio |
|------------|-----------|
| PLC | Allen-Bradley CompactLogix / MicroLogix con EtherNet/IP |
| IP esempio | 10.44.200.2 (slot 0) |
| Addon HA | Bridge PLC custom (vedi `plc_config.yaml`) |
| MQTT Broker | Mosquitto (addon core-mosquitto, porta 1883) |
| Driver | pylogix |

## Addon da installare

1. **Mosquitto broker** — `Settings > Add-ons > Mosquitto broker`
2. **Bridge PLC** — addon custom che legge `plc_config.yaml` e pubblica su MQTT

## File di riferimento

| File | Percorso | Scopo |
|------|----------|-------|
| Config PLC | `/config/plc/plc_config.yaml` | Registro centrale PLC (IP, driver, punti) |
| Data points | `/config/plc/specs/rockwell_01_waste/data_plc.yaml` | Mappa tag PLC -> entita HA |
| Package HA | `/config/packages/plc/ha_rockwell_01.yaml` | Template comm_ok, comm_age, scaling livelli |
| Watchdog | `/config/packages/plc/ha_watchdog.yaml` | Enable/disable, ping, automazioni avvio |

## Entita create

### Binary Sensors (ingressi digitali)
- Stato pompe: `binary_sensor.plc_rw01_pm1_marcia` ... `pm8_marcia`
- Trip pompe: `binary_sensor.plc_rw01_pm1_scatto` ... `pm8_scatto`
- Valvole: `binary_sensor.plc_rw01_ev_carico`, `ev_scarico`, `ev_lavaggio`
- Allarmi: `binary_sensor.plc_rw01_al_pressostato`, `al_allagamento`, etc.
- Livelli: `binary_sensor.plc_rw01_sr_pre_lh`, `sr_pre_ll`, etc.

### Sensors (analogici)
- Livelli vasche (raw mm): `sensor.plc_rw01_v1_level_mm` ... `v5_level_mm`
- Livelli scalati (m e %): `sensor.plc_rw01_v1_level_m`, `v1_level_pct`
- Timer: `sensor.plc_rw01_pm1_timer_s` ... `pm8_timer_s`

### Connectivity
- `binary_sensor.plc_rockwell_01_link` — LWT MQTT (bridge online/offline)
- `binary_sensor.plc_rockwell_01_stack_comm_ok` — comunicazione OK (< 120s)
- `sensor.plc_rockwell_01_comm_age_s` — eta ultimo dato

## Automazioni di esempio

### Enable/Disable PLC
```yaml
input_boolean:
  plc_rockwell_01_en:
    name: "PLC Rockwell 01 abilitato"
    icon: mdi:lan-connect
```

### Ping reachability
```yaml
command_line:
  - binary_sensor:
      name: plc_ping_rockwell_01
      command: "ping -c 1 -W 1 10.44.200.2 > /dev/null 2>&1 && echo 'ON' || echo 'OFF'"
      scan_interval: 30
```

## Scaling livelli (esempio)

Formula per trasformare raw 4-20mA in distanza (m) e percentuale (%):
```yaml
# raw 13115 (4mA) = 0m, raw 65535 (20mA) = 5.0m
# Vasca max utile: 3.30m = 100%
state: >
  {% set raw = states('sensor.plc_rw01_v1_level_mm') | float(0) %}
  {% set m = ((raw - 13115) / (65535 - 13115)) * 5.0 %}
  {{ [([m, 0] | max), 5.0] | min | round(2) }}
```
