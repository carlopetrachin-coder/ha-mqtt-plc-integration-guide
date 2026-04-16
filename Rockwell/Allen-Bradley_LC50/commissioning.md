# Allen-Bradley LC50 — Commissioning

## 1. Configurazione iniziale

### 1.1 Prerequisiti PLC
- IP statico configurato (es. 10.44.200.2)
- Slot processore noto (es. 0)
- Tag PLC creati e accessibili via EtherNet/IP
- Firewall: porta TCP 44818 aperta tra HA e PLC

### 1.2 Setup HA
1. Installare addon **Mosquitto broker** e avviarlo
2. Installare addon **Bridge PLC**
3. Verificare connettivita: `ping <IP_PLC>` dal terminale HA

## 2. Configurazione file

### 2.1 Registro PLC (`plc_config.yaml`)

Aggiungere il PLC alla lista:

```yaml
- id: plc_rockwell_01
  enable: true
  name: "Allen-Bradley LC50"
  driver: pylogix
  host: 10.44.200.2
  slot: 0
  points:
    - { id: plc_rw01_pm1_marcia, name: "PM1 Marcia", kind: binary_sensor, group: fast,
        logix: { tag: "PM1_Running", type: bool } }
    - { id: plc_rw01_v1_level_mm, name: "V1 Livello", kind: sensor, group: medium,
        logix: { tag: "V1_Level_Raw", type: int } }
```

### 2.2 Data points (`specs/rockwell_01_waste/data_plc.yaml`)

Documentare ogni punto con:
- `plc_tag`: nome tag nel PLC (case-sensitive)
- `ha_id`: nome entita HA
- `description`: descrizione funzionale
- `kind`: binary_sensor | sensor | switch
- `group`: fast (1s) | medium (5s) | slow (30s)

### 2.3 Package HA (`packages/plc/ha_rockwell_01.yaml`)

Contiene:
- MQTT binary_sensor per LWT (link status)
- Template binary_sensor per `stack_comm_ok` (< 120s)
- Template sensor per `comm_age_s`
- Template sensor per scaling analogici (raw -> m -> %)

## 3. Aggiungere entita

### 3.1 Aggiungere un ingresso digitale (DI)

1. **Nel PLC**: creare/verificare il tag (es. `PM9_Running`)
2. **In `data_plc.yaml`**: documentare il punto
3. **In `plc_config.yaml`**: aggiungere alla lista `points`:
   ```yaml
   - { id: plc_rw01_pm9_marcia, name: "PM9 Marcia", kind: binary_sensor, group: fast,
       logix: { tag: "PM9_Running", type: bool } }
   ```
4. **Riavviare il bridge addon** (o HA)

### 3.2 Aggiungere un analogico (AI)

Come sopra, ma con `kind: sensor` e `type: int` o `real`:
```yaml
- { id: plc_rw01_v6_level_mm, name: "V6 Livello", kind: sensor, group: medium,
    logix: { tag: "V6_Level_Raw", type: int } }
```

Per lo scaling, aggiungere un template sensor in `ha_rockwell_01.yaml`.

### 3.3 Aggiungere un comando (DO)

```yaml
- { id: plc_rw01_pm9_cmd, name: "PM9 Comando", kind: switch, group: fast,
    logix: { tag: "PM9_Command", type: bool } }
```

### 3.4 Rimuovere entita

1. Rimuovere il punto da `plc_config.yaml`
2. Riavviare il bridge
3. L'entita diventa `unavailable` — rimuoverla da HA: `Settings > Devices > Entities`

## 4. Gruppi di polling

| Gruppo | Intervallo | Uso tipico |
|--------|-----------|------------|
| `fast` | 1.0s | Stati pompe, comandi, allarmi critici |
| `medium` | 5.0s | Livelli analogici, timer |
| `slow` | 30.0s | Contatori ore, diagnostica |

**Limiti**: pylogix legge un tag alla volta via CIP. Con 77 punti:
- fast (60 punti): ~60 letture/s — sostenibile
- Oltre 200 punti fast: considerare block read o rallentare a medium

## 5. Limiti e considerazioni

| Aspetto | Limite | Nota |
|---------|--------|------|
| Protocollo | EtherNet/IP (CIP) | Solo Allen-Bradley |
| Tag type | bool, int, real, dint | No strutture complesse |
| Connessioni | Max ~10 CIP session per PLC | Bridge usa 1 sessione |
| Latenza | ~50-100ms per tag read | Rete LAN locale |
| Scritture | Singolo tag per comando | No block write |
| Failover | Nessuno | Se bridge cade, dati MQTT stale |
| Sicurezza | Nessuna auth EtherNet/IP | Segregare la rete PLC |

## 6. Troubleshooting

| Problema | Causa | Soluzione |
|----------|-------|-----------|
| `plc_link = off` | Bridge addon non avviato o crash | Controllare log addon |
| `comm_age > 120s` | PLC non raggiungibile o tag errato | Verificare ping + nome tag |
| Valore sempre 0 | Tag name case-sensitive errato | Verificare maiuscole/minuscole nel PLC |
| `unavailable` dopo rimozione | Entita orfana in HA | Rimuovere da Settings > Entities |
