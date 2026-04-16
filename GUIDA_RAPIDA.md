# Collegare un PLC a Home Assistant — Guida passo passo

## A chi serve questa guida

Sei un utente Home Assistant e vuoi leggere dati da un PLC industriale
(Siemens, Rockwell/Allen-Bradley) direttamente nella tua dashboard.
Non serve essere programmatori — basta seguire i passi in ordine.

---

## Come funziona (in 30 secondi)

```
PLC  ───rete───>  Bridge (addon HA)  ───MQTT───>  Home Assistant
                                      <───MQTT───  (comandi)
```

1. Il **PLC** ha i dati (stati pompe, livelli, temperature...)
2. Un **addon bridge** installato in HA legge questi dati via rete
3. Li pubblica su **MQTT** (un sistema di messaggi interno)
4. **Home Assistant** li riceve e li mostra come entita (sensori, switch, etc.)

Non scrivi codice nel PLC. Devi solo sapere **quali dati leggere e dove trovarli**.

---

## Prima di iniziare — cosa ti serve

### Da parte tua
- [ ] Home Assistant installato (HAOS consigliato)
- [ ] PLC raggiungibile via rete dallo stesso PC dove gira HA
- [ ] IP del PLC (chiedilo a chi ha configurato la rete)

### Dal programmatore PLC (o dal manuale)
- [ ] **Rockwell**: lista dei nomi tag da leggere (es. `PM1_Running`, `V1_Level`)
- [ ] **Siemens**: numero DB + mappa byte/bit dei dati condivisi
- [ ] Tipo di ogni dato: booleano (on/off), intero (numero), reale (decimale)

> **Non hai queste informazioni?** Fermati qui e chiedile. Senza sapere
> COSA leggere e DOVE si trova nel PLC, non puoi procedere.

---

## STEP 1 — Installa gli addon (una volta sola)

### 1.1 Mosquitto Broker (il postino dei messaggi)

1. Vai in **Settings > Add-ons > Add-on Store**
2. Cerca **"Mosquitto broker"**
3. Click **Install**, poi abilita **Start on boot** e **Watchdog**
4. Tab **Configuration** — aggiungi un utente:
   ```yaml
   logins:
     - username: mqtt_plc_bridge
       password: "la_tua_password"
   ```
5. Click **Save**, poi **Start**

> Dettagli completi: [`Addons/mosquitto_broker.md`](Addons/mosquitto_broker.md)

### 1.2 PLC Bridge (il traduttore PLC → MQTT)

1. Sempre in **Add-on Store**, aggiungi il repository custom del bridge
2. Cerca **"PLC Bridge"** e installa
3. Abilita **Start on boot** e **Watchdog**
4. **NON avviarlo ancora** — prima va configurato (Step 2)

> Dettagli completi: [`Addons/plc_bridge.md`](Addons/plc_bridge.md)

---

## STEP 2 — Configura il tuo PLC

### 2.1 Scegli il tuo caso

| Il tuo PLC e un... | Driver | Vai a |
|---------------------|--------|-------|
| Allen-Bradley (CompactLogix, MicroLogix) | pylogix | [Rockwell](Rockwell/Allen-Bradley_LC50/info.md) |
| Siemens S7-1200 | snap7 | [S7-1200](Siemens/S7-1200/info.md) |
| Siemens S7-1500 | snap7 | [S7-1500](Siemens/S7-1500/info.md) |
| Siemens S7-300/400 | snap7 | Stessa procedura S7-1200, cambia rack/slot |

### 2.2 Verifica la connessione di rete

Dal **terminale HA** (Settings > Add-ons > Terminal):

```bash
ping 10.44.200.2      # metti l'IP del tuo PLC
```

Se risponde, procedi. Se non risponde:
- Verifica cavo di rete
- Verifica che PLC e HA siano sulla stessa rete/VLAN
- Verifica firewall (Rockwell: porta 44818, Siemens: porta 102)

### 2.3 Prerequisiti specifici Siemens (solo per S7-1200/1500)

**Devi fare queste operazioni in TIA Portal PRIMA di collegare il bridge:**

1. Apri il progetto TIA Portal
2. Sul DB condiviso: click destro > **Proprieta** > **Attributi**
   → **Disabilita** "Optimized block access"
3. Sul dispositivo PLC: **Proprieta** > **Protezione**
   → **Abilita** "Consenti accesso PUT/GET"
4. **Compila e scarica** nel PLC

> Senza questi passaggi, snap7 non puo leggere il PLC. E l'errore piu comune.

### 2.4 Crea il file di configurazione

Modifica il file `/config/plc/plc_config.yaml`.
Copia l'esempio dal tuo caso e adatta IP e punti:

**Rockwell — esempio minimo:**
```yaml
plcs:
  - id: plc_lc50
    enable: true
    name: "Il mio PLC Rockwell"
    driver: pylogix
    host: 10.44.200.2        # ← metti il TUO IP
    slot: 0
    points:
      - { id: plc_lc50_pm1_marcia, name: "Pompa 1 Marcia",
          kind: binary_sensor, group: fast,
          logix: { tag: "PM1_Running", type: bool } }
```

**Siemens — esempio minimo:**
```yaml
plcs:
  - id: plc_s1200
    enable: true
    name: "Il mio PLC Siemens"
    driver: snap7
    host: 192.168.1.20        # ← metti il TUO IP
    rack: 0
    slot: 1
    points:
      - { id: plc_s1200_i0_0, name: "Ingresso I0.0",
          kind: binary_sensor, group: fast,
          snap7: { db: 1000, byte: 4, bit: 0 } }
```

> File esempio completi nella cartella `files/` di ogni PLC.

---

## STEP 3 — Avvia e verifica

### 3.1 Avvia il bridge

1. **Settings > Add-ons > PLC Bridge > Start**
2. Controlla il **Log** dell'addon — deve mostrare:
   ```
   Connected to MQTT broker
   PLC plc_lc50: connected (10.44.200.2)
   Polling started...
   ```

### 3.2 Verifica le entita in HA

1. **Settings > Devices & Services > MQTT**
2. Dovresti vedere le entita create automaticamente:
   - `binary_sensor.plc_lc50_pm1_marcia`
   - `sensor.plc_lc50_v1_level_mm`
   - etc.

### 3.3 Test rapido

Vai in **Developer Tools > States** e cerca `plc_lc50`:
- Le entita devono avere un valore (`on`/`off`, numeri), non `unavailable`
- Se sono `unavailable`: il bridge non riesce a leggere → controlla IP e tag/DB

---

## STEP 4 — Aggiungi il monitoraggio salute

Questo ti permette di sapere se il PLC e connesso e risponde.

### 4.1 Copia i file health check

Copia nella cartella `/config/packages/plc/` i file:
- `ha_lc50_example.yaml` (o `ha_s1200_example.yaml` per Siemens)
- `ha_lc50_watchdog.yaml` (o `ha_s1200_watchdog.yaml`)

### 4.2 Riavvia HA

**Settings > System > Restart**

### 4.3 Verifica

Dopo il riavvio, in **Developer Tools > States** cerca:

| Entita | Valore atteso | Significato |
|--------|---------------|-------------|
| `binary_sensor.plc_ping_lc50` | `on` | PLC raggiungibile via rete |
| `binary_sensor.plc_lc50_link` | `on` | Bridge addon connesso al PLC |
| `binary_sensor.plc_lc50_stack_comm_ok` | `on` | Dati recenti (< 120s) |
| `sensor.plc_lc50_comm_age_s` | `0-5` | Secondi dall'ultimo dato |

> Dettagli: [`Health_Check/commissioning.md`](Health_Check/commissioning.md)

---

## STEP 5 — Usa i dati nella dashboard

Ora puoi aggiungere i sensori PLC a qualsiasi dashboard HA:

### Card semplice (stato pompa)
```yaml
type: entities
entities:
  - entity: binary_sensor.plc_lc50_pm1_marcia
    name: Pompa 1
  - entity: binary_sensor.plc_lc50_pm2_marcia
    name: Pompa 2
```

### Card livello vasca
```yaml
type: gauge
entity: sensor.plc_lc50_v1_level_pct
name: Vasca 1
min: 0
max: 100
severity:
  green: 20
  yellow: 60
  red: 80
```

### Card System Check (tabella completa)

Usa il file [`Health_Check/files/card_plc_system_check.yaml`](Health_Check/files/card_plc_system_check.yaml)
— mostra tutti i PLC con ping, comm, age in una tabella colorata.

---

## Riepilogo: cosa va dove

```
/config/
├── plc/
│   └── plc_config.yaml          ← registro PLC + punti (STEP 2)
├── packages/plc/
│   ├── ha_lc50_example.yaml     ← comm_ok, scaling (STEP 4)
│   └── ha_lc50_watchdog.yaml    ← ping, enable/disable (STEP 4)
└── (dashboard)                  ← card con entita PLC (STEP 5)
```

---

## Problemi comuni

| Sintomo | Causa probabile | Soluzione |
|---------|----------------|-----------|
| Entita non appaiono | Bridge non avviato | Settings > Add-ons > PLC Bridge > Start |
| Tutto `unavailable` | IP sbagliato o PLC spento | Verifica con `ping` |
| Rockwell: valori sempre 0 | Nome tag errato (case-sensitive!) | Controlla maiuscole nel PLC |
| Siemens: errore connessione | PUT/GET o Optimized block access | Vedi Step 2.3 |
| `comm_age` cresce | PLC in STOP o rete instabile | Metti PLC in RUN, verifica cavo |
| Addon crash loop | YAML malformato | Valida `plc_config.yaml` con un YAML linter |

---

## Prossimi passi

- **Aggiungere punti**: vedi il `commissioning.md` del tuo PLC
- **Aggiungere un secondo PLC**: aggiungi un blocco alla lista `plcs:` in plc_config.yaml
- **Creare automazioni**: usa le entita PLC come trigger (es. allarme pompa → notifica)
- **Scaling analogici**: vedi gli esempi nei file `ha_*_example.yaml`
