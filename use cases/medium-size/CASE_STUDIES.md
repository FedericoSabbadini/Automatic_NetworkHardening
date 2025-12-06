# Casi d'Uso Complessi - Network Hardening

Due scenari enterprise realistici per testare il planner in condizioni di complessità elevata.

---

## Caso 1: Data Center E-Commerce

**File:** `case_ecommerce_datacenter.json`

### Contesto
Un'azienda e-commerce deve preparare l'infrastruttura per il Black Friday. L'audit di sicurezza ha identificato diverse porte non conformi che devono essere chiuse prima dell'evento.

### Infrastruttura

| Componente | Host | Porte Critiche |
|------------|------|----------------|
| Load Balancer | 2 | HTTP (80) da migrare |
| Web Cluster | 4 | HTTP (80) da migrare |
| API Gateway | 1 | SNMP (161) da disattivare |
| Microservizi | 6 | SMTP, NetBIOS da gestire |
| Database | 3 | Telnet (23) da chiudere |
| Cache Redis | 2 | SNMP (161) su replica |
| Elasticsearch | 2 | NetBIOS (139) su nodo 2 |
| Message Queue | 1 | SNMP (161) critico |
| Monitoring | 3 | HTTP, Telnet da chiudere |
| Admin Bastion | 1 | RDP, VNC, TFTP da gestire |

**Totale: 25 host**

### Porte Vietate
- 80 (HTTP) → migrazione a 8080
- 23 (Telnet) → chiusura diretta
- 25 (SMTP) → disattivazione servizio
- 139 (NetBIOS) → disattivazione servizio
- 161 (SNMP) → disattivazione servizio
- 69 (TFTP) → disattivazione servizio
- 3389 (RDP) → migrazione a 33389
- 5900 (VNC) → migrazione a 59000

### Sfide
1. **Alta disponibilità**: molti servizi sono critici e non possono essere fermati
2. **Dipendenze complesse**: catena load balancer → web → API → microservizi → DB
3. **Mix di strategie**: servizi migrabili vs. servizi da disattivare
4. **Nodi replica**: alcuni nodi sono sacrificabili, altri no

### Risultato Atteso
Il planner deve trovare un piano che:
- Migra HTTP su 8080 per tutti i web server
- Disattiva SMTP sul notification service
- Disattiva NetBIOS su inventory ed elasticsearch_02
- Disattiva SNMP sui nodi non critici
- Chiude le porte Telnet sui database replica
- Gestisce admin bastion (RDP, VNC, TFTP)

---

## Caso 2: Infrastruttura Bancaria Multi-Site

**File:** `case_banking_multisite.json`

### Contesto
Una banca deve adeguarsi alle normative PCI-DSS e SWIFT CSP. L'infrastruttura include trading, ATM network, fraud detection e disaster recovery. Audit richiede chiusura immediata di protocolli legacy.

### Infrastruttura

| Componente | Host | Criticità |
|------------|------|-----------|
| Core Banking | 2 | CRITICO - no downtime |
| Trading Platform | 3 | CRITICO - real-time |
| Web Banking | 2 | HTTP da migrare |
| Mobile API | 1 | SNMP da disattivare |
| ATM Controllers | 2 | CRITICO - Telnet legacy |
| POS Gateway | 1 | FTP da migrare |
| SWIFT Gateway | 1 | CRITICO - compliance |
| Fraud Detection | 2 | CRITICO - real-time |
| AML Scanner | 1 | SMTP da disattivare |
| Report Generator | 1 | NetBIOS, SMB da gestire |
| Database Layer | 4 | Mix critico/non critico |
| Message Queue | 2 | SNMP su entrambi |
| Redis Session | 2 | SNMP su replica |
| HSM Connector | 1 | CRITICO - crittografia |
| LDAP Directory | 1 | CRITICO - porta 389 vietata! |
| SIEM | 1 | SNMP da disattivare |
| Admin | 1 | RDP, VNC, TFTP |
| DR Coordinator | 1 | Telnet legacy |

**Totale: 30 host**

### Porte Vietate
- 80 (HTTP) → migrazione
- 21 (FTP) → migrazione a 2121
- 23 (Telnet) → problema su ATM controller!
- 25 (SMTP) → disattivazione
- 139 (NetBIOS) → disattivazione
- 161 (SNMP) → disattivazione
- 69 (TFTP) → disattivazione
- 389 (LDAP) → migrazione a 3890
- 3389 (RDP) → migrazione
- 5900 (VNC) → migrazione

### Sfide Critiche

1. **ATM Controller con Telnet**: I controller ATM hanno porta 23 aperta ma il servizio telnet NON è nella lista servizi. È una porta legacy aperta ma non usata → chiusura diretta possibile.

2. **LDAP critico su porta vietata**: Il servizio LDAP è critico E usa porta 389 che è vietata. Deve essere MIGRATO (non può essere fermato).

3. **Database Oracle**: Non hanno servizi nella lista (solo ssh) ma hanno porte legacy aperte.

4. **Trading real-time**: Nessun downtime ammesso sui trading engine.

5. **Compliance SWIFT**: Il gateway SWIFT non può subire modifiche che ne compromettano la certificazione.

### Risultato Atteso
Piano complesso che:
- Migra HTTP su web banking
- Migra FTP su POS gateway
- Migra LDAP 389→3890 (servizio critico!)
- Migra RDP e VNC su admin workstation
- Disattiva SMTP su AML scanner
- Disattiva NetBIOS su report generator
- Disattiva SNMP sui nodi non critici
- Chiude Telnet su ATM, DB, DR coordinator
- Preserva tutti i servizi critici

---

## Come Eseguire

### Opzione 1: Copia nella cartella scenarios
```python
# Nel notebook, dopo aver generato gli scenari standard:
import shutil
shutil.copy('case_ecommerce_datacenter.json', 'scenarios/')
shutil.copy('case_banking_multisite.json', 'scenarios/')

# Poi riesegui run_all_scenarios()
```

### Opzione 2: Esecuzione singola
```python
# Carica e risolvi un singolo scenario
with open('case_ecommerce_datacenter.json') as f:
    scenario = json.load(f)

problem, domain, objects = setup_problem(scenario)
result, ground_actions = solve(problem)

# Mostra piano
if result and result.plan:
    for i, action in enumerate(result.plan.actions, 1):
        print(f"{i:3d}. {action}")
```

---

## Metriche Attese

| Scenario | Host | Goal | Azioni Stimate | Tempo Stimato |
|----------|------|------|----------------|---------------|
| E-Commerce | 25 | ~15 | 20-30 | 2-5 sec |
| Banking | 30 | ~20 | 25-40 | 3-8 sec |

---

## Note sulla Complessità

### Fattori che aumentano la complessità:
1. **Numero di host**: scala lineare con gli oggetti
2. **Dipendenze**: aumentano le precondizioni da verificare
3. **Servizi critici**: riducono le azioni disponibili
4. **Porte senza alternativa**: forzano disattivazioni costose

### Possibili risultati:
- **SOLVED**: piano trovato con costo ottimale
- **TIMEOUT**: troppo complesso (aumentare timeout)
- **UNSOLVABLE**: vincoli impossibili da soddisfare

Il caso banking potrebbe risultare UNSOLVABLE se LDAP non avesse alternativa configurata (ma ce l'ha: 389→3890).
