# Siemens S7-1200 — Commissioning

## 1. Configurazione iniziale

### 1.1 Prerequisiti PLC (TIA Portal)
- IP statico configurato (es. 192.168.1.20)
- DB condiviso creato (es. DB1000, 14 byte)
- **"Optimized block access" DISABILITATO** sul DB
- PUT/GET abilitato: `Dispositivo > Proprieta > Protezione > Permetti accesso PUT/GET`
- Firewall: porta TCP 102 aperta tra HA e PLC

### 1.2 Setup HA
1. Installare addon **Mosquitto broker** e avviarlo
2. Installare addon **Bridge PLC**
3. Verificare connettivita: `ping <IP_PLC>` dal terminale HA

## 2. Configurazione file

### 2.1 Registro PLC (`plc_config.yaml`)

```yaml
- id: plc_s1200
  enable: true
  name: "Siemens S7-1200"
  driver: snap7
  host: 192.168.1.20
  rack: 0
  slot: 1
  points:
    - { id: plc_s7_01_i0_0, name: "I0.0", kind: binary_sensor, group: fast,
        snap7: { db: 1000, byte: 4, bit: 0 } }
    - { id: plc_s7_01_ai0_raw, name: "AI0 Raw", kind: sensor, group: medium,
        snap7: { db: 1000, byte: 10, type: int16 } }
    - { id: plc_s7_01_q0_0_cmd, name: "Q0.0 Cmd", kind: switch, group: fast,
        snap7: { db: 1000, byte: 0, bit: 0 } }
```

### 2.2 Mappa DB (`specs/plc_s1200/data_plc.yaml`)

Documentare ogni byte/bit del DB condiviso:
```
Byte.Bit | Direzione | ha_id              | Descrizione
---------|-----------|--------------------|-----------
4.0      | PLC→HA    | plc_s7_01_i0_0     | Ingresso I0.0
0.0      | HA→PLC    | plc_s7_01_q0_0_cmd | Uscita Q0.0 manuale
10-11    | PLC→HA    | plc_s7_01_ai0_raw  | Analogico IW64 (0-27648)
```

## 3. Aggiungere entita

### 3.1 Aggiungere un DI

1. **Nel PLC (TIA)**: copiare il valore nel DB condiviso (es. byte 14, bit 0)
2. **In `data_plc.yaml`**: documentare
3. **In `plc_config.yaml`**: aggiungere punto:
   ```yaml
   - { id: plc_s7_01_i2_0, name: "I2.0", kind: binary_sensor, group: fast,
       snap7: { db: 1000, byte: 14, bit: 0 } }
   ```
4. **Allargare il DB** nel PLC se necessario (aggiungere byte)
5. Riavviare bridge addon

### 3.2 Aggiungere un AI

1. **Nel PLC**: copiare il valore INT nel DB (2 byte, es. byte 14-15)
2. **In `plc_config.yaml`**:
   ```yaml
   - { id: plc_s7_01_ai2_raw, name: "AI2 Raw", kind: sensor, group: medium,
       snap7: { db: 1000, byte: 14, type: int16 } }
   ```
3. **In `ha_s1200_io.yaml`**: aggiungere scaling template + input_number per range

### 3.3 Aggiungere un DO (comando)

1. **Nel PLC**: leggere il bit dal DB nel ciclo
2. **In `plc_config.yaml`**:
   ```yaml
   - { id: plc_s7_01_q2_0_cmd, name: "Q2.0 Cmd", kind: switch, group: fast,
       snap7: { db: 1000, byte: 2, bit: 0 } }
   ```

### 3.4 Rimuovere entita

1. Rimuovere da `plc_config.yaml`
2. Non serve toccare il DB PLC (il byte/bit non viene piu letto)
3. Riavviare bridge
4. Rimuovere entita orfana da HA

## 4. Scaling analogici

I valori AI arrivano come INT (0–27648 per 0–10V / 4–20mA S7-1200).

Configurazione in `ha_s1200_io.yaml`:
```yaml
input_number:
  plc_s1200_ai0_inl:  { min: 0, max: 65535, initial: 0 }      # raw min
  plc_s1200_ai0_inh:  { min: 0, max: 65535, initial: 27648 }   # raw max
  plc_s1200_ai0_outl: { min: -1000, max: 1000, initial: 0 }    # eng min
  plc_s1200_ai0_outh: { min: -1000, max: 1000, initial: 100 }  # eng max
```

Formula template:
```jinja
{% set inl = states('input_number.plc_s1200_ai0_inl') | float %}
{% set inh = states('input_number.plc_s1200_ai0_inh') | float %}
{% set outl = states('input_number.plc_s1200_ai0_outl') | float %}
{% set outh = states('input_number.plc_s1200_ai0_outh') | float %}
{% set raw = states('sensor.plc_s7_01_ai0_raw') | float(0) %}
{% if inh != inl %}
  {{ (outl + (raw - inl) * (outh - outl) / (inh - inl)) | round(2) }}
{% else %}
  0
{% endif %}
```

## 5. Heartbeat — configurazione

| Parametro | PLC (TIA) | HA |
|-----------|-----------|-----|
| Toggle bit | DB1000.DBX8.0 ogni ciclo OB1 | `switch.plc_s7_01_cmd_ha_hb` ogni 5s |
| Periodo | default 500ms (OB1 cycle) | configurabile: `input_number.plc_s1200_cycle_ms` |
| Timeout detect | HA: `last_changed > 10s` | PLC: logica custom su cmd_ha_hb |

## 6. Limiti e considerazioni

| Aspetto | Limite | Nota |
|---------|--------|------|
| Protocollo | S7comm (TCP 102) | Solo Siemens S7-1200/1500/300/400 |
| Accesso | DB con indirizzi assoluti | Optimized block access DEVE essere OFF |
| Connessioni | Max 8 (S7-1200 Basic) / 16 (Standard) | Bridge usa 1 connessione |
| DB size | Max 64KB per DB | Piu che sufficiente |
| Tipi dato | bool, int8, int16, int32, real, string | No UDT diretti |
| Lettura | Block read per DB (tutti i byte in 1 PDU) | Molto efficiente |
| Latenza | ~10-30ms per block read | Dipende da dimensione DB |
| Sicurezza | Nessuna auth S7comm | Segregare la rete PLC |

## 7. Troubleshooting

| Problema | Causa | Soluzione |
|----------|-------|-----------|
| `plc_link = off` | Bridge non si connette | Verificare IP, rack, slot |
| Connessione rifiutata | PUT/GET disabilitato | Abilitare in TIA Portal |
| Tutti i valori 0 | Optimized block access ON | Disabilitare sul DB |
| Heartbeat fermo | Ciclo PLC in STOP | Mettere PLC in RUN |
| AI fuori range | Cavo scollegato (> 27648) | Aggiungere wire-break detection |
| `comm_age` alto | PLC in STOP o rete lenta | Verificare stato PLC e ping |
