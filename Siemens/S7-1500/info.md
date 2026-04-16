# Siemens S7-1500 — Integrazione Home Assistant

## Scopo

Integrazione PLC Siemens S7-1500 con Home Assistant tramite bridge MQTT custom.
Driver: **snap7** (stesso driver S7-1200, protocollo S7comm).

## Differenze rispetto a S7-1200

| Aspetto | S7-1200 | S7-1500 |
|---------|---------|---------|
| Connessioni max | 8/16 | 32+ |
| DB max size | 64KB | 64KB (extendable) |
| CPU slot | tipicamente 1 | tipicamente 1 |
| Performance | ~10ms/block read | ~5ms/block read |
| TIA Portal | V13+ | V13+ |
| Optimized block access | Default ON (disabilitare) | Default ON (disabilitare) |

La configurazione e identica a S7-1200 — cambia solo IP e potenzialmente rack/slot.

## Requisiti

| Componente | Dettaglio |
|------------|-----------|
| PLC | Siemens S7-1500 con Ethernet integrata |
| IP esempio | 192.168.1.21 (rack 0, slot 1) |
| Addon HA | Bridge PLC custom |
| MQTT Broker | Mosquitto |
| Driver | snap7 |
| TIA Portal | V13+ con "Optimized block access" **DISABILITATO** |

## Addon da installare

Identici a S7-1200:
1. **Mosquitto broker**
2. **Bridge PLC**

## File di riferimento

| File | Percorso | Scopo |
|------|----------|-------|
| Config PLC | `/config/plc/plc_config.yaml` | Registro centrale PLC |
| Config dedicata | `/config/plc/plc_s1500.yaml` | Config standalone (alternativa) |
| Data points | `/config/plc/specs/plc_s1500/data_plc.yaml` | Mappa byte DB -> entita HA |
| Package HA | `/config/packages/plc/ha_s1500.yaml` | Template comm_ok, comm_age |
| Watchdog | `/config/packages/plc/ha_watchdog.yaml` | Enable/disable, ping |

## Stato attuale

**Non ancora configurato** (`enable: false` in plc_config.yaml).
Il DB condiviso e la mappa punti vanno definiti quando il PLC viene messo in servizio.

## Entita create (stub)

- `binary_sensor.plc_s1500_link` — LWT MQTT
- `binary_sensor.plc_s1500_stack_comm_ok` — comunicazione OK
- `sensor.plc_s1500_comm_age_s` — eta ultimo dato

Le entita dati verranno aggiunte con la mappa punti del DB.

## Note TIA Portal per S7-1500

1. **Security**: S7-1500 ha protezione accesso piu restrittiva del 1200.
   - Andare in `Dispositivo > Proprieta > Protezione e sicurezza`
   - Impostare livello accesso su "Accesso completo (nessuna protezione)"
   - Oppure: abilitare "Consenti accesso PUT/GET dal partner remoto"

2. **Optimized DB**: su S7-1500 e il default — **DEVE** essere disabilitato:
   - Click destro sul DB > Proprieta > Attributi
   - Deselezionare "Accesso al blocco ottimizzato"

3. **Profinet**: se il PLC usa Profinet IO, l'interfaccia Ethernet potrebbe essere dedicata.
   Verificare che l'IP usato per snap7 sia raggiungibile dalla rete HA.
