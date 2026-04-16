# Siemens S7-1500 — Commissioning

## 1. Configurazione iniziale

Seguire la stessa procedura di S7-1200 (vedi `../S7-1200/commissioning.md`).

Differenze specifiche S7-1500:

### 1.1 Prerequisiti PLC (TIA Portal)
- IP statico (es. 192.168.1.21, rack 0, slot 1)
- DB condiviso creato con **"Optimized block access" DISABILITATO**
- Accesso PUT/GET abilitato (piu restrittivo del 1200, doppio check)
- Firewall/protezione: livello "Accesso completo" o PUT/GET esplicito
- Porta TCP 102 aperta

### 1.2 Abilitare in HA

Attualmente il PLC e disabilitato. Per attivarlo:

1. In `plc_config.yaml`: cambiare `enable: false` → `enable: true`
2. In `plc_s1500.yaml`: stessa modifica
3. Creare la mappa punti in `specs/plc_s1500/data_plc.yaml`
4. Aggiungere i punti alla sezione `points:` in plc_config.yaml
5. Riavviare bridge addon

## 2. Aggiungere/rimuovere entita

Identico a S7-1200 — vedi `../S7-1200/commissioning.md` sezioni 3 e 4.

L'unica differenza e l'indirizzamento snap7:
```yaml
snap7: { db: <numero_db>, byte: <offset_byte>, bit: <offset_bit> }
snap7: { db: <numero_db>, byte: <offset_byte>, type: int16 }  # per analogici
```

## 3. Limiti specifici S7-1500

| Aspetto | Limite | Nota |
|---------|--------|------|
| Protezione | Piu livelli rispetto a S7-1200 | Verificare OGNI livello |
| Profinet | IP potrebbe essere su interfaccia dedicata | Verificare raggiungibilita |
| Firmware | Firmware recenti piu restrittivi su PUT/GET | Aggiornare config se necessario |
| DB retentive | DB retentivi possono avere layout diverso | Testare con DB non-retentivo |

## 4. Troubleshooting

Stessi problemi di S7-1200, con aggiunta:

| Problema | Causa | Soluzione |
|----------|-------|-----------|
| Connessione rifiutata | Protezione S7-1500 attiva | Verificare TUTTI i livelli di sicurezza in TIA |
| Timeout connessione | IP su interfaccia Profinet dedicata | Usare IP dell'interfaccia standard |
| Errore "DB not found" | DB non ancora scaricato nel PLC | Scaricare il progetto TIA |
