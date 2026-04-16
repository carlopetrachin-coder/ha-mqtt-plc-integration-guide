# PLC Health Check — Commissioning

## 1. Aggiungere un nuovo PLC al sistema di monitoraggio

### Step 1 — Input boolean (enable/disable)

In `ha_watchdog.yaml`:
```yaml
input_boolean:
  plc_<nuovo_id>_en:
    name: "PLC <nome> abilitato"
    icon: mdi:lan-connect
```

### Step 2 — Shell command (file .stop)

```yaml
shell_command:
  plc_enable_<nuovo_id>:  "rm -f /config/plc/plc_<nuovo_id>.stop"
  plc_disable_<nuovo_id>: "touch /config/plc/plc_<nuovo_id>.stop"
```

### Step 3 — Ping sensor

```yaml
command_line:
  - binary_sensor:
      name: plc_ping_<nuovo_id>
      unique_id: plc_ping_<nuovo_id>
      command: "ping -c 1 -W 1 <IP> > /dev/null 2>&1 && echo 'ON' || echo 'OFF'"
      payload_on: "ON"
      payload_off: "OFF"
      scan_interval: 30
      command_timeout: 2
```

### Step 4 — Package HA (comm_ok, comm_age)

Creare `/config/packages/plc/ha_<nuovo_id>.yaml`:

```yaml
mqtt:
  binary_sensor:
    - name: "plc_<nuovo_id>_link"
      state_topic: "plc/plc_<nuovo_id>/status"
      payload_on: "online"
      payload_off: "offline"
      device_class: connectivity

template:
  - binary_sensor:
      - name: plc_<nuovo_id>_stack_comm_ok
        unique_id: plc_<nuovo_id>_stack_comm_ok
        device_class: connectivity
        state: >
          {% set e = states.binary_sensor.plc_<nuovo_id>_link %}
          {% if e is not none and e.state not in ['unknown','unavailable'] %}
            {{ (as_timestamp(now()) - as_timestamp(e.last_reported, 0)) < 120 }}
          {% else %}
            false
          {% endif %}

  - sensor:
      - name: plc_<nuovo_id>_comm_age_s
        unique_id: plc_<nuovo_id>_comm_age_s
        unit_of_measurement: "s"
        state_class: measurement
        availability: "{{ is_state('input_boolean.plc_<nuovo_id>_en', 'on') }}"
        state: >
          {% set e = states.binary_sensor.plc_<nuovo_id>_link %}
          {% if e is not none and e.state not in ['unknown','unavailable','none',''] %}
            {{ [(as_timestamp(now()) - as_timestamp(e.last_reported, 0)) | int, 9999] | min }}
          {% else %}
            9999
          {% endif %}
```

### Step 5 — Automazione toggle

Aggiungere il trigger in `ha_watchdog.yaml` nell'automazione toggle:
```yaml
- platform: state
  entity_id: input_boolean.plc_<nuovo_id>_en
  id: <nuovo_id>
```

### Step 6 — Card System Check

Aggiungere l'IP al dizionario e il PLC al loop nella card:
```jinja
{% set plc_ips = {
  'rockwell_01': '10.44.200.2',
  's1200': '192.168.1.20',
  's1500': '192.168.1.21',
  '<nuovo_id>': '<nuovo_ip>'
} %}
```

### Step 7 — Riavviare HA

Le modifiche ai package, input_boolean e command_line richiedono un riavvio.

## 2. Rimuovere un PLC dal monitoraggio

1. Rimuovere input_boolean, shell_command, ping da `ha_watchdog.yaml`
2. Rimuovere il file `packages/plc/ha_<id>.yaml`
3. Rimuovere dalla card System Check
4. Riavviare HA
5. Pulire entita orfane: `Settings > Devices > Entities`

## 3. Soglie e allarmi

### Soglie comm_age nella card

| Valore | Colore | Significato |
|--------|--------|-------------|
| 0–15s | Verde | Comunicazione regolare |
| 15–30s | Arancione | Ritardo — verificare |
| > 30s | Rosso | Comunicazione persa |
| 9999 | Grigio (?) | Entita non disponibile |

### Aggiungere notifica per PLC offline

```yaml
automation:
  - alias: plc_offline_alert
    trigger:
      - platform: state
        entity_id: binary_sensor.plc_<id>_stack_comm_ok
        to: "off"
        for: "00:02:00"
    condition:
      - "{{ is_state('input_boolean.plc_<id>_en', 'on') }}"
    action:
      - service: persistent_notification.create
        data:
          title: "PLC {{ trigger.entity_id }} offline"
          message: "Comm_age > 120s da 2 minuti"
          notification_id: "plc_alert_{{ trigger.entity_id }}"
```

## 4. Pattern heartbeat (solo Siemens)

Per PLC Siemens con ciclo OB1, il heartbeat bidirezionale e piu affidabile del solo LWT:

| Direzione | Implementazione | Timeout |
|-----------|----------------|---------|
| PLC → HA | PLC toggle bit nel DB ogni ciclo | HA: `last_changed > 10s` = offline |
| HA → PLC | HA toggle switch ogni 5s (se comm_ok) | PLC: logica custom |

Per Rockwell non c'e heartbeat nativo — si usa solo LWT MQTT.

## 5. Checklist nuovo PLC

- [ ] IP configurato e pingabile da HA
- [ ] Bridge PLC addon installato e avviato
- [ ] Mosquitto broker attivo
- [ ] `plc_config.yaml` — PLC aggiunto con driver e punti
- [ ] `ha_watchdog.yaml` — input_boolean, shell_command, ping
- [ ] `packages/plc/ha_<id>.yaml` — comm_ok, comm_age
- [ ] Card System Check aggiornata
- [ ] `.storage/lovelace.dashboard_board` aggiornato (se card gia deployata)
- [ ] Riavvio HA
- [ ] Verifica: ping ON, link ON, comm_age < 5s
