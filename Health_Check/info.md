# PLC Health Check — Monitoraggio e Diagnostica

## Scopo

Sistema unificato per monitorare lo stato di salute di tutti i PLC integrati in HA.
Funziona indipendentemente dal driver (pylogix, snap7) — si basa su MQTT LWT e ping IP.

## Architettura a 3 livelli

```
Livello 1: PING        → IP raggiungibile?           (binary_sensor.plc_ping_*)
Livello 2: MQTT LWT    → Bridge addon connesso?       (binary_sensor.plc_*_link)
Livello 3: COMM AGE    → Dati recenti dal PLC?        (sensor.plc_*_comm_age_s)
```

| Livello | Entita | Significato ON | Significato OFF |
|---------|--------|---------------|-----------------|
| 1. Ping | `binary_sensor.plc_ping_<id>` | IP raggiungibile | Rete/hardware down |
| 2. Link | `binary_sensor.plc_<id>_link` | Bridge MQTT online | Bridge crash/stop |
| 3. Comm OK | `binary_sensor.plc_<id>_stack_comm_ok` | Dati < 120s | PLC non risponde |

## File di riferimento

| File | Percorso | Scopo |
|------|----------|-------|
| Watchdog | `/config/packages/plc/ha_watchdog.yaml` | Enable/disable, ping, startup |
| Rockwell | `/config/packages/plc/ha_rockwell_01.yaml` | Comm_ok, comm_age per Rockwell |
| S1200 | `/config/packages/plc/ha_s1200.yaml` | Comm_ok, comm_age + heartbeat |
| S1500 | `/config/packages/plc/ha_s1500.yaml` | Comm_ok, comm_age (stub) |

## Entita per PLC

Per ogni PLC vengono create automaticamente:

### Enable/Disable
```yaml
input_boolean:
  plc_<id>_en:
    name: "PLC <nome> abilitato"
    icon: mdi:lan-connect
```

### Ping IP
```yaml
command_line:
  - binary_sensor:
      name: plc_ping_<id>
      command: "ping -c 1 -W 1 <IP> > /dev/null 2>&1 && echo 'ON' || echo 'OFF'"
      scan_interval: 30
      command_timeout: 2
```

### MQTT Link (LWT)
```yaml
mqtt:
  binary_sensor:
    - name: "plc_<id>_link"
      state_topic: "plc/plc_<id>/status"
      payload_on: "online"
      payload_off: "offline"
      device_class: connectivity
```

### Comm OK + Comm Age
```yaml
template:
  - binary_sensor:
      - name: plc_<id>_stack_comm_ok
        device_class: connectivity
        state: >
          {% set e = states.binary_sensor.plc_<id>_link %}
          {{ (as_timestamp(now()) - as_timestamp(e.last_reported, 0)) < 120 }}

  - sensor:
      - name: plc_<id>_comm_age_s
        unit_of_measurement: "s"
        state: >
          {% set e = states.binary_sensor.plc_<id>_link %}
          {{ [(as_timestamp(now()) - as_timestamp(e.last_reported, 0)) | int, 9999] | min }}
```

## Automazioni

### Startup — abilita bridge se PLC raggiungibile
```yaml
automation:
  - trigger:
      - platform: homeassistant
        event: start
    action:
      - wait_template: "{{ is_state('binary_sensor.plc_ping_<id>', 'on') }}"
        timeout: "00:01:00"
      - if: "{{ wait.completed }}"
        then:
          - service: shell_command.plc_enable_<id>
          - service: hassio.addon_restart
            data: { addon: local_plc_bridge }
```

### Toggle manuale
```yaml
automation:
  - trigger:
      - platform: state
        entity_id: input_boolean.plc_<id>_en
    action:
      - choose:
          - conditions: "{{ trigger.to_state.state == 'on' }}"
            sequence:
              - service: shell_command.plc_enable_<id>
          - conditions: "{{ trigger.to_state.state == 'off' }}"
            sequence:
              - service: shell_command.plc_disable_<id>
      - service: hassio.addon_restart
        data: { addon: local_plc_bridge }
```

## Esempio concreto — 3 PLC

Dalla configurazione attuale del progetto Children Hospital:

| PLC | Driver | IP | Stato | Comm Strategy |
|-----|--------|----|-------|---------------|
| Rockwell LC50 | pylogix | 10.44.200.2 | attivo | LWT MQTT |
| S7-1200 | snap7 | 192.168.1.20 | attivo | Heartbeat PLC (toggle bit) |
| S7-1500 | snap7 | 192.168.1.21 | futuro | LWT MQTT (stub) |

### Card System Check

La dashboard "System Check" mostra una tabella unificata con:
- Nome PLC
- Enable (toggle)
- Ping (verde/rosso)
- Comm OK (verde/rosso)
- Comm Age (secondi, colorato per soglie)
- IP

```jinja
{% set plc_ips = {
  'rockwell_01': '10.44.200.2',
  's1200': '192.168.1.20',
  's1500': '192.168.1.21'
} %}
{% for plc, label in [
  ('rockwell_01','Rockwell LC50'),
  ('s1200','S7-1200'),
  ('s1500','S7-1500')
] %}
  ...tabella con ping/comm/age...
{% endfor %}
```
