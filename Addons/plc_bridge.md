# Addon: PLC Bridge

## Scopo

Addon custom che legge/scrive PLC (Siemens snap7, Rockwell pylogix) e pubblica
i dati su MQTT con discovery automatico per Home Assistant.

## Installazione

1. **Settings > Add-ons > Add-on Store** (icona menu in alto a destra)
2. **Repositories** > aggiungere l'URL del repository custom del bridge
3. Cercare **"PLC Bridge"** e installare
4. Abilitare **"Start on boot"** e **"Watchdog"**
5. Click **Start**

## File di configurazione

Il bridge legge un unico file di configurazione:

```
/config/plc/plc_config.yaml
```

### Struttura del file

```yaml
# ── Connessione MQTT ────────────────────────────────────────
mqtt:
  host: core-mosquitto          # hostname interno addon Mosquitto
  port: 1883
  username: mqtt_plc_bridge
  password: "Bassora2000!"      # deve corrispondere a Mosquitto logins
  base_topic: plc               # prefisso topic: plc/<plc_id>/status
  discovery_prefix: homeassistant  # per auto-discovery entita HA
  client_id: plc_bridge_v3      # univoco per ogni istanza bridge

# ── Gruppi di polling ───────────────────────────────────────
poll_groups:
  fast: 1.0       # secondi — stati pompe, comandi, allarmi
  medium: 5.0     # secondi — livelli analogici, timer
  slow: 30.0      # secondi — contatori ore, diagnostica

# ── Lista PLC ───────────────────────────────────────────────
plcs:
  - id: plc_lc50
    enable: true
    name: "Allen-Bradley LC50"
    driver: pylogix
    host: 10.44.200.2
    slot: 0
    points:
      - { id: plc_lc50_pm1_marcia, name: "PM1 Marcia", kind: binary_sensor,
          group: fast, logix: { tag: "PM1_Running", type: bool } }
      # ... altri punti ...
```

### Parametri MQTT

| Parametro | Valore | Note |
|-----------|--------|------|
| `host` | `core-mosquitto` | Nome DNS interno dell'addon Mosquitto |
| `port` | `1883` | Porta standard MQTT (non-TLS) |
| `username` | `mqtt_plc_bridge` | Deve esistere in Mosquitto logins |
| `password` | (stringa) | Stessa di Mosquitto |
| `base_topic` | `plc` | LWT pubblicato su `plc/<id>/status` |
| `discovery_prefix` | `homeassistant` | Standard HA per auto-discovery |
| `client_id` | `plc_bridge_v3` | Univoco — non usare lo stesso su 2 istanze |

### Parametri polling

| Gruppo | Default | Uso |
|--------|---------|-----|
| `fast` | 1.0s | DI, DO, heartbeat, allarmi |
| `medium` | 5.0s | AI, livelli, timer |
| `slow` | 30.0s | Ore funzionamento, diagnostica |

### Parametri PLC

| Campo | Tipo | Note |
|-------|------|------|
| `id` | stringa | Identificativo univoco (usato nei topic MQTT) |
| `enable` | bool | `true` per abilitare, `false` per ignorare |
| `name` | stringa | Nome descrittivo (mostrato in HA) |
| `driver` | `pylogix` / `snap7` | Dipende dal PLC |
| `host` | IP | Indirizzo PLC |
| `slot` | int | Slot CPU (Rockwell) o rack/slot (Siemens) |
| `rack` | int | Solo Siemens (default 0) |

### Parametri punti — pylogix (Rockwell)

```yaml
logix:
  tag: "PM1_Running"    # Nome tag nel PLC (case-sensitive!)
  type: bool            # bool, int, real, dint
```

### Parametri punti — snap7 (Siemens)

```yaml
snap7:
  db: 1000              # Numero DB
  byte: 4               # Offset byte nel DB
  bit: 0                # Offset bit (solo per bool)
  type: bool            # bool, int16, int32, real
```

## Enable/Disable PLC senza toccare plc_config.yaml

Il bridge supporta file `.stop` per disabilitare un PLC a runtime:

```bash
# Disabilita (bridge ignora questo PLC al prossimo restart)
touch /config/plc/plc_lc50.stop

# Riabilita
rm -f /config/plc/plc_lc50.stop

# Riavvia bridge per applicare
ha addons restart local_plc_bridge
```

In HA questo e gestito da shell_command + input_boolean (vedi watchdog).

## LWT (Last Will and Testament)

Il bridge pubblica automaticamente:

| Evento | Topic | Payload |
|--------|-------|---------|
| Connessione | `plc/<id>/status` | `online` |
| Disconnessione | `plc/<id>/status` | `offline` |

HA usa il LWT per determinare se il bridge e attivo (`binary_sensor.plc_<id>_link`).

## Log e debug

- **Log addon**: Settings > Add-ons > PLC Bridge > Log
- **MQTT debug**: Developer Tools > MQTT > Listen to topic: `plc/#`
- **Entita create**: Settings > Devices & Services > MQTT > Entities

## Troubleshooting

| Problema | Causa | Soluzione |
|----------|-------|-----------|
| Addon non parte | plc_config.yaml syntax error | Validare YAML (yamllint) |
| "Connection refused" MQTT | Mosquitto non avviato | Avviare addon Mosquitto prima |
| PLC non raggiungibile | IP errato o firewall | Verificare ping da terminale HA |
| Tag non trovato (Rockwell) | Nome case-sensitive | Verificare esattamente il nome nel PLC |
| DB error (Siemens) | Optimized block access ON | Disabilitare in TIA Portal |
| Entita non appaiono | discovery_prefix errato | Deve essere `homeassistant` |
| Bridge crash loop | .stop file corrotto | Rimuovere tutti i .stop e riavviare |

## Aggiornamento configurazione

1. Modificare `/config/plc/plc_config.yaml`
2. Riavviare l'addon: `Settings > Add-ons > PLC Bridge > Restart`
   oppure da terminale: `ha addons restart local_plc_bridge`
3. Le nuove entita appaiono in HA automaticamente (discovery)
4. Le entita rimosse restano come `unavailable` — eliminarle manualmente
