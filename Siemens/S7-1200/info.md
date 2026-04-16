# Siemens S7-1200 — Integrazione Home Assistant

## Scopo

Integrazione PLC Siemens S7-1200 con Home Assistant tramite bridge MQTT custom.
Driver: **snap7** (protocollo S7comm nativo, accesso DB tramite indirizzi assoluti).

## Architettura

```
PLC (S7comm)  ──TCP──>  Bridge Addon (snap7)  ──MQTT──>  Home Assistant
                                                <──MQTT──  (comandi)
```

Il bridge legge/scrive byte in un DB condiviso (es. DB1000) e pubblica su MQTT.
HA consuma i topic MQTT come entita.

## Requisiti

| Componente | Dettaglio |
|------------|-----------|
| PLC | Siemens S7-1200 con Ethernet integrata |
| IP esempio | 192.168.1.20 (rack 0, slot 1) |
| Addon HA | Bridge PLC custom (vedi `plc_config.yaml`) |
| MQTT Broker | Mosquitto (addon core-mosquitto, porta 1883) |
| Driver | snap7 |
| TIA Portal | DB con "Optimized block access" **DISABILITATO** |

## Addon da installare

1. **Mosquitto broker** — `Settings > Add-ons > Mosquitto broker`
2. **Bridge PLC** — addon custom che legge `plc_config.yaml` e pubblica su MQTT

## Prerequisito TIA Portal

Nel progetto TIA Portal, per ogni DB usato dal bridge:
1. Aprire le proprieta del DB
2. Tab **Attributi**
3. **Disabilitare** "Optimized block access"
4. Compilare e scaricare

Senza questo, snap7 non puo usare indirizzi assoluti (byte.bit).

## File di riferimento

| File | Percorso | Scopo |
|------|----------|-------|
| Config PLC | `/config/plc/plc_config.yaml` | Registro centrale PLC |
| Data points | `/config/plc/specs/plc_s1200/data_plc.yaml` | Mappa byte DB1000 -> entita HA |
| Package HA | `/config/packages/plc/ha_s1200.yaml` | Template comm_ok, heartbeat, cycle label |
| I/O custom | `/config/packages/plc/ha_s1200_io.yaml` | Scaling AI, icone, etichette device |
| Comandi | `/config/packages/plc/ha_s1200_commands.yaml` | input_number, sync automazioni |
| Watchdog | `/config/packages/plc/ha_watchdog.yaml` | Enable/disable, ping, avvio |

## Struttura DB condiviso (DB1000)

```
Byte 0-1    HA → PLC    DO comandi manuali (Q0.0–Q1.1)
Byte 2-3    HA → PLC    Ciclo: enable, heartbeat HA, periodo ms
Byte 4-5    PLC → HA    DI ingressi (I0.0–I1.7)
Byte 6-7    PLC → HA    DO feedback (Q0.0–Q1.1 stato reale)
Byte 8      PLC → HA    Heartbeat PLC (toggle ogni ciclo)
Byte 9      PLC → HA    Ciclo step attivo (0–9)
Byte 10-11  PLC → HA    AI 0 raw (IW64, 0–27648)
Byte 12-13  PLC → HA    AI 1 raw (IW66, 0–27648)
```

## Entita create

### Binary Sensors
- 16 DI: `binary_sensor.plc_s7_01_i0_0` ... `i1_7`
- 10 DO feedback: `binary_sensor.plc_s7_01_q0_0_sts` ... `q1_1_sts`
- Heartbeat PLC: `binary_sensor.plc_s7_01_sts_plc_hb`
- Connectivity: `binary_sensor.plc_s1200_link`, `plc_s1200_stack_comm_ok`

### Sensors
- 2 AI raw: `sensor.plc_s7_01_ai0_raw`, `ai1_raw`
- 2 AI scaled: `sensor.plc_s7_01_ai0_scaled`, `ai1_scaled`
- Ciclo step: `sensor.plc_s7_01_cycle_step`
- Step label: `sensor.plc_s1200_cycle_step_label`
- Comm age: `sensor.plc_s1200_comm_age_s`

### Switches / Numbers
- 10 DO comandi: `switch.plc_s7_01_q0_0_cmd` ... `q1_1_cmd`
- Ciclo enable: `switch.plc_s7_01_cmd_cycle_enable`
- Heartbeat HA: `switch.plc_s7_01_cmd_ha_hb`
- Periodo ciclo: `input_number.plc_s1200_cycle_ms` (100–10000ms)

## Heartbeat bidirezionale

```
PLC: toggle sts_plc_hb ogni ciclo (500ms default)
HA:  monitora last_changed di sts_plc_hb
     se > 10s → comm_age alto → allarme
HA:  toggle cmd_ha_hb ogni 5s (se comm_ok)
PLC: monitora cmd_ha_hb, se fermo > 10s → HA offline
```

## Automazioni di esempio

### Sync comando al PLC
```yaml
automation:
  - trigger:
      - platform: state
        entity_id: input_number.plc_s1200_cycle_ms
    action:
      - service: number.set_value
        target:
          entity_id: number.plc_s7_01_cmd_cycle_ms
        data:
          value: "{{ states('input_number.plc_s1200_cycle_ms') | int }}"
```
