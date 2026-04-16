# Addon: Mosquitto Broker

## Scopo

Broker MQTT locale per la comunicazione tra il Bridge PLC e Home Assistant.
Tutti i dati PLC transitano via MQTT — Mosquitto e il componente centrale.

## Installazione

1. **Settings > Add-ons > Add-on Store**
2. Cercare **"Mosquitto broker"** (addon ufficiale HA)
3. Click **Install**
4. Abilitare **"Start on boot"** e **"Watchdog"**
5. Click **Start**

## Configurazione addon

Tab **Configuration** dell'addon:

```yaml
logins:
  - username: mqtt_plc_bridge
    password: "Bassora2000!"
customize:
  active: false
  folder: mosquitto
certfile: fullchain.pem
keyfile: privkey.pem
require_certificate: false
```

### Parametri chiave

| Parametro | Valore | Note |
|-----------|--------|------|
| `username` | `mqtt_plc_bridge` | Utente dedicato per il bridge PLC |
| `password` | (da personalizzare) | Usare password complessa in produzione |
| `require_certificate` | `false` | TLS opzionale su rete interna |

## Configurazione integrazione HA

Dopo aver avviato l'addon, HA dovrebbe rilevare automaticamente Mosquitto.
Se non lo fa:

1. **Settings > Devices & Services > Add Integration**
2. Cercare **"MQTT"**
3. Configurare:
   - Broker: `core-mosquitto` (hostname interno addon)
   - Porta: `1883`
   - Username: `mqtt_plc_bridge`
   - Password: (stessa dell'addon)

## Verifica funzionamento

### Da HA Developer Tools > Services

```yaml
service: mqtt.publish
data:
  topic: "test/ping"
  payload: "hello"
```

Poi in **Developer Tools > MQTT > Listen**:
- Topic: `test/ping`
- Click "Start Listening"
- Dovresti vedere il messaggio "hello"

### Topic PLC attesi

Una volta avviato il bridge, i topic sono:

| Topic | Tipo | Contenuto |
|-------|------|-----------|
| `plc/<plc_id>/status` | LWT | `online` / `offline` |
| `homeassistant/binary_sensor/<id>/config` | Discovery | Configurazione entita |
| `homeassistant/binary_sensor/<id>/state` | State | Valore entita |
| `homeassistant/sensor/<id>/config` | Discovery | Configurazione entita |
| `homeassistant/sensor/<id>/state` | State | Valore entita |
| `homeassistant/switch/<id>/config` | Discovery | Configurazione entita |
| `homeassistant/switch/<id>/state` | State | Valore entita |
| `homeassistant/switch/<id>/set` | Command | Comando da HA al PLC |

## Troubleshooting

| Problema | Causa | Soluzione |
|----------|-------|-----------|
| Addon non parte | Porta 1883 occupata | Verificare che non ci sia altro broker |
| "Connection refused" dal bridge | Username/password errati | Verificare `logins` nella config addon |
| Entita non appaiono in HA | Discovery prefix errato | Verificare `discovery_prefix: homeassistant` nel bridge |
| Messaggi persi | QoS 0 su rete instabile | Impostare QoS 1 nel bridge config |

## Note sicurezza

- In **produzione**: cambiare la password di default
- Per accesso esterno: abilitare TLS (`require_certificate: true`)
- Creare utenti separati per bridge e altri client MQTT
- Non esporre porta 1883 su internet senza TLS
