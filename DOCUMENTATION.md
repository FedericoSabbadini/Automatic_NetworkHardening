# Network Hardening con Automated Planning

## Documentazione del Progetto

---

# 1. Introduzione

## 1.1 Descrizione del Progetto

**Network Hardening Planner** è un sistema di pianificazione automatica per la messa in sicurezza di infrastrutture di rete. Il sistema utilizza tecniche di Automated Planning (AI Planning) per generare piani ottimali che chiudono le porte di rete considerate insicure, minimizzando l'impatto operativo sui servizi in esecuzione.

## 1.2 Problema Affrontato

In un'infrastruttura di rete aziendale, alcune porte sono considerate insicure per policy di sicurezza (es. porta 80 HTTP non cifrato, porta 23 Telnet, porta 21 FTP). La chiusura di queste porte non è banale perché:

- Alcuni servizi attivi dipendono da queste porte
- Alcuni servizi sono critici e non possono essere fermati
- Esistono dipendenze tra servizi (es. un'applicazione web dipende dal database)
- Ogni azione ha un costo operativo diverso

Il sistema trova automaticamente la sequenza ottimale di azioni per raggiungere lo stato di sicurezza desiderato.

## 1.3 Approccio Utilizzato

Il problema è modellato come un problema di **Classical Planning** con costi sulle azioni:

- **Stato iniziale**: configurazione attuale della rete (porte aperte, servizi attivi)
- **Goal**: tutte le porte vietate devono essere chiuse
- **Azioni**: chiusura porte, disattivazione servizi, migrazione servizi
- **Metrica**: minimizzare il costo totale delle azioni

Il solver utilizzato è **Fast-Downward**, uno dei più efficienti planner domain-independent.

---

# 2. Architettura del Sistema

## 2.1 Componenti

```
network_hardening.ipynb
├── Configurazione
│   ├── SERVICE_PORT_MAPPING    # Associazione servizi → porte
│   ├── ALTERNATIVE_PORTS       # Porte alternative per migrazione
│   └── ACTION_COSTS            # Costi delle azioni
│
├── Modello di Planning
│   ├── NetworkHardeningDomain  # Definizione tipi e fluent
│   └── setup_problem()         # Costruzione problema PDDL
│
├── Generazione Scenari
│   ├── SCENARIOS               # Definizione scenari di test
│   └── generate_scenarios()    # Creazione file JSON
│
└── Esecuzione
    ├── solve()                 # Interfaccia con Fast-Downward
    └── run_all_scenarios()     # Esecuzione batch
```

## 2.2 Modello Formale

### Tipi
- `Host`: server o macchina nella rete
- `Port`: porta di rete (es. 80, 443, 22)
- `Service`: servizio software (es. http, mysql, ssh)

### Fluent (Predicati)
| Fluent | Tipo | Descrizione |
|--------|------|-------------|
| `porta_aperta(host, port)` | Bool | La porta è aperta sull'host |
| `servizio_attivo(host, service)` | Bool | Il servizio è in esecuzione |
| `servizio_critico(host, service)` | Bool | Il servizio non può essere fermato |
| `servizio_usa_porta(host, service, port)` | Bool | Il servizio utilizza la porta |
| `dipende_da(host, srv1, srv2)` | Bool | srv1 dipende da srv2 |

### Azioni
| Azione | Costo | Precondizioni | Effetti |
|--------|-------|---------------|---------|
| `chiudi_porta(h, p)` | 1 | porta aperta, nessun servizio attivo su p | porta chiusa |
| `disattiva_servizio(h, s)` | 5 | servizio attivo, non critico, nessun dipendente attivo | servizio inattivo |
| `migra_servizio(h, s, p1, p2)` | 3 | servizio attivo, p1 aperta, p2 chiusa | servizio su p2, p1 chiusa |

---

# 3. Manuale di Installazione

## 3.1 Requisiti

- Python 3.8 o superiore
- pip (gestore pacchetti Python)
- Connessione internet (per scaricare le dipendenze)

## 3.2 Installazione su Google Colab

1. Aprire Google Colab: https://colab.research.google.com
2. Caricare il file `network_hardening.ipynb`
3. Eseguire la prima cella per installare le dipendenze:

```python
!pip install -q unified-planning up-fast-downward pandas matplotlib
```

4. Eseguire le celle successive in ordine

## 3.3 Installazione Locale

### Passo 1: Creare ambiente virtuale (opzionale ma consigliato)

```bash
python -m venv venv
source venv/bin/activate  # Linux/Mac
venv\Scripts\activate     # Windows
```

### Passo 2: Installare le dipendenze

```bash
pip install unified-planning up-fast-downward pandas matplotlib
```

### Passo 3: Eseguire il notebook

```bash
jupyter notebook network_hardening.ipynb
```

Oppure convertire in script Python:

```bash
jupyter nbconvert --to script network_hardening.ipynb
python network_hardening.py
```

## 3.4 Dipendenze

| Pacchetto | Versione | Descrizione |
|-----------|----------|-------------|
| unified-planning | ≥1.0.0 | Framework di planning |
| up-fast-downward | ≥0.4.0 | Solver Fast-Downward |
| pandas | ≥1.3.0 | Analisi dati |
| matplotlib | ≥3.4.0 | Grafici |

## 3.5 Verifica Installazione

Eseguire il seguente codice per verificare che tutto sia installato correttamente:

```python
from unified_planning.shortcuts import *
from unified_planning.engines import PlanGenerationResultStatus
print("Installazione completata con successo!")
```

---

# 4. Guida all'Uso

## 4.1 Esecuzione Base

Il notebook è suddiviso in sezioni eseguibili in sequenza:

1. **Setup**: installa e importa le librerie
2. **Configurazione**: definisce mapping e costi
3. **Modello**: definisce il dominio di planning
4. **Scenari**: genera gli scenari di test
5. **Esecuzione**: risolve tutti gli scenari
6. **Analisi**: mostra risultati e grafici

## 4.2 Creare uno Scenario Personalizzato

Per aggiungere un nuovo scenario, modificare la lista `SCENARIOS`:

```python
{
    'name': 'mio_scenario',
    'description': 'Descrizione dello scenario',
    'hosts': [
        # (id, [porte_aperte], [servizi], is_critical)
        ('webserver', [80, 443, 22], ['http', 'https', 'ssh'], False),
        ('database', [3306, 22], ['mysql', 'ssh'], True),
    ],
    'forbidden': [80],  # Porte da chiudere
    'critical': {'database': ['mysql']},  # Servizi che non possono essere fermati
    'dependencies': {
        'webserver': {'http': ['mysql']}  # http dipende da mysql
    },
}
```

## 4.3 Interpretare i Risultati

### Output Testuale
```
Scenario: 01_all_actions_basic
  Risultato: RISOLTO in 0.526s
  Piano: 4 azioni, costo 10
  Dettaglio: 1 migrazioni, 1 stop, 2 chiusure
```

### File Generati
- `scenarios/*.json`: scenari in formato JSON
- `results/plan_*.txt`: piani generati
- `results/summary.csv`: riepilogo risultati
- `results/analysis.png`: grafici

---

# 5. Scenari di Test

## 5.1 Panoramica

| # | Nome | Host | Descrizione |
|---|------|------|-------------|
| 1 | all_actions_basic | 3 | Scenario base, tutte le azioni |
| 2 | deps_all_actions | 4 | Con dipendenze tra servizi |
| 3 | enterprise_mixed | 8 | Infrastruttura enterprise |
| 4 | security_incident | 6 | Post-breach hardening |
| 5 | healthcare_gdpr | 7 | Compliance GDPR/HIPAA |
| 6 | financial_pci | 9 | Compliance PCI-DSS |
| 7 | stress_test | 14 | Test di scalabilità |
| 8 | impossible | 1 | Scenario irrisolvibile |

## 5.2 Scenari per Tipo di Azione

Ogni scenario è progettato per richiedere tutte e tre le azioni:

- **migra_servizio**: porte con alternativa (80→8080, 21→2121, ecc.)
- **disattiva_servizio**: porte senza alternativa (25, 139, 161, 69)
- **chiudi_porta**: porte aperte senza servizi attivi

---

# 6. Note Tecniche

## 6.1 Scelte Progettuali

### Tre azioni invece di cinque
La versione iniziale includeva 5 azioni:
1. chiudi_porta
2. disattiva_servizio
3. riattiva_servizio
4. migra_porta_servizio (per critici)
5. migra_con_restart (per non critici)

Sono state ridotte a 3 perché:
- `riattiva_servizio` non veniva mai usata (nessun goal richiede servizi attivi)
- Le due migrazioni avevano lo stesso effetto pratico

### Porte senza alternativa
Alcune porte non hanno alternativa configurata:
- **25 (SMTP)**: protocollo legacy, si consiglia TLS su 587
- **139 (NetBIOS)**: protocollo obsoleto
- **161 (SNMP)**: si consiglia SNMPv3 su porte diverse
- **69 (TFTP)**: protocollo non sicuro

Questo forza il planner a usare `disattiva_servizio`.

### Dipendenze intra-host
Le dipendenze sono limitate allo stesso host per semplicità. Una webapp che dipende da un database su un altro server non è modellata.

## 6.2 Limitazioni

- Le migrazioni sono considerate atomiche (nessun downtime)
- Non sono gestite finestre temporali di manutenzione
- Non è modellato il rollback in caso di fallimento
- Le dipendenze cross-host non sono supportate

## 6.3 Possibili Estensioni

1. **Priorità servizi**: minimizzare impatto su servizi importanti
2. **Vincoli temporali**: pianificare durante finestre di manutenzione
3. **Dipendenze cross-host**: modellare dipendenze tra server diversi
4. **Rollback**: includere azioni di ripristino

---

# 7. Riferimenti

## 7.1 Librerie Utilizzate

- **Unified Planning**: https://github.com/aiplan4eu/unified-planning
- **Fast-Downward**: https://www.fast-downward.org/

## 7.2 Documentazione

- Unified Planning Documentation: https://unified-planning.readthedocs.io/
- PDDL Reference: https://planning.wiki/

## 7.3 Articoli di Riferimento

- Helmert, M. (2006). "The Fast Downward Planning System"
- Geffner, H. & Bonet, B. (2013). "A Concise Introduction to Models and Methods for Automated Planning"

---

# 8. Changelog

## Versione 4.1 (Finale)
- Ridotte le azioni da 5 a 3
- Rimossa `riattiva_servizio` (mai utilizzata)
- Unificate le due migrazioni in una sola
- Aggiunto supporto dipendenze tra servizi
- Ottimizzato il codice rimuovendo ridondanze

## Versione 4.0
- Aggiunto fluent `dipende_da` per dipendenze
- Aggiornate precondizioni di `disattiva_servizio`

## Versione 3.0
- Aggiunte 5 azioni distinte
- Gestione servizi critici
- Correzione bug su `servizio_usa_porta`

---

# 9. Licenza

Questo progetto è rilasciato a scopo didattico.

---

*Documentazione generata per il progetto Network Hardening Planner*
