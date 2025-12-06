# Stress Test Estremi - Network Hardening

Due scenari progettati per portare il planner al limite delle sue capacità. Tempo di esecuzione stimato: 30-90 secondi ciascuno.

---

## Scenario 1: Multinazionale 4 Region

**File:** `stress_multinational_60host.json`

### Contesto
Una multinazionale con presenza globale deve uniformare le policy di sicurezza su tutte le region prima di un audit ISO 27001. L'infrastruttura è cresciuta organicamente negli anni, con sistemi legacy particolarmente presenti in LATAM.

### Architettura

```
                         ┌─────────────────┐
                         │  GLOBAL (4)     │
                         │  DNS, Monitor,  │
                         │  SIEM, Admin    │
                         └────────┬────────┘
                                  │
        ┌─────────────┬───────────┼───────────┬─────────────┐
        │             │           │           │             │
   ┌────▼────┐   ┌────▼────┐ ┌────▼────┐ ┌────▼────┐
   │  EU     │   │  US     │ │  APAC   │ │  LATAM  │
   │  (15)   │   │  (15)   │ │  (15)   │ │  (11)   │
   └─────────┘   └─────────┘ └─────────┘ └─────────┘
```

### Distribuzione Host per Region

| Region | Host | Componenti |
|--------|------|------------|
| **EU** | 15 | 2 LB, 3 Web, 2 App, 2 ERP, 1 CRM, 2 DB, 2 Redis, 1 MQ |
| **US** | 15 | 2 LB, 3 Web, 2 App, 2 Finance, 2 HR, 2 DB, 1 Redis, 1 MQ |
| **APAC** | 15 | 2 LB, 3 Web, 2 App, 2 Supply Chain, 2 Warehouse, 2 DB, 1 Redis, 1 MQ |
| **LATAM** | 11 | 1 LB, 2 Web, 1 App, 2 Legacy, 2 DB, 1 Redis |
| **Global** | 4 | 2 DNS, 1 Monitoring, 1 SIEM, 1 Backup, 1 Admin |

**Totale: 60 host**

### Porte Vietate

| Porta | Servizio | Strategia | Host Coinvolti |
|-------|----------|-----------|----------------|
| 80 | HTTP | Migra → 8080 | 16 (tutti i LB e Web) |
| 21 | FTP | Migra → 2121 | 3 (Supply, Legacy, Backup) |
| 23 | Telnet | Chiudi | 6 (DB replica, Legacy) |
| 25 | SMTP | Disattiva | 3 (CRM, HR, Legacy) |
| 139 | NetBIOS | Disattiva | 6 (DB replica, Warehouse, Legacy) |
| 161 | SNMP | Disattiva | 15+ (monitoring su molti host) |
| 69 | TFTP | Disattiva | 2 (Supply, Legacy) |
| 389 | LDAP | Migra → 3890 | 2 (HR) |
| 3389 | RDP | Migra → 33389 | 2 (Finance) |
| 5900 | VNC | Migra → 59000 | 1 (Finance) |

### Servizi Critici (21 host)

```
EU:    eu_app_01, eu_erp_01, eu_erp_02, eu_db_master, eu_redis_01, eu_mq_01
US:    us_app_01, us_finance_01, us_hr_01, us_db_master, us_redis_01, us_mq_01
APAC:  apac_app_01, apac_supply_01, apac_db_master, apac_redis_01, apac_mq_01
LATAM: latam_app_01, latam_db_master
Global: global_dns_01, global_siem
```

### Dipendenze (39 relazioni)

Catena tipica per region:
```
LB.http → Web.nodejs → App.tomcat → MQ.rabbitmq
                                  → DB.postgresql/mysql
```

Dipendenze speciali:
- `eu_crm_01.tomcat` → `smtp` (CRM invia email)
- `apac_warehouse.tomcat` → `netbios`, `smb` (accesso file share)
- `latam_legacy.tomcat` → `ftp`, `smtp`, `netbios`, `tftp` (sistemi legacy)
- `global_backup.rsync` → `ftp` (backup via FTP)

### Sfide Principali

1. **Volume**: 60 host con ~35 goal da soddisfare
2. **LDAP critico**: US HR usa LDAP su porta 389 (deve migrare, non fermare)
3. **Sistemi legacy LATAM**: concentrazione di protocolli obsoleti
4. **SNMP diffuso**: presente su ~15 host per monitoring
5. **Dipendenze cross-servizio**: warehouse dipende da NetBIOS

### Metriche Attese

| Metrica | Valore Stimato |
|---------|----------------|
| Oggetti UP | ~180 |
| Azioni ground | ~200 |
| Goal | ~35 |
| Piano | 50-70 azioni |
| Tempo | 20-60 secondi |

---

## Scenario 2: Cloud Provider Infrastructure

**File:** `stress_cloud_provider_80host.json`

### Contesto
Un cloud provider deve superare l'audit ISO 27001 e SOC2. L'infrastruttura include compute, storage, networking e security stack. Tutti i protocolli legacy devono essere eliminati mantenendo l'operatività 24/7.

### Architettura

```
┌─────────────────────────────────────────────────────────────┐
│                        EDGE (6)                              │
│              EU-1, EU-2, US-1, US-2, APAC-1, APAC-2         │
└─────────────────────────────┬───────────────────────────────┘
                              │
┌─────────────────────────────▼───────────────────────────────┐
│                     API GATEWAY (3)                          │
│                    + AUTH SERVICE (3)                        │
└─────────────────────────────┬───────────────────────────────┘
                              │
     ┌────────────────────────┼────────────────────────┐
     │                        │                        │
┌────▼────┐            ┌──────▼──────┐          ┌─────▼─────┐
│ COMPUTE │            │   STORAGE   │          │  MESSAGE  │
│ K8s(11) │            │ Ceph(9)     │          │  QUEUE(6) │
│         │            │ NFS/SMB(4)  │          │ Kafka(3)  │
└────┬────┘            └──────┬──────┘          │ Rabbit(3) │
     │                        │                 └─────┬─────┘
     │                        │                       │
┌────▼────────────────────────▼───────────────────────▼─────┐
│                      DATABASE (9)                          │
│         PostgreSQL(3) + MySQL(3) + MongoDB(3)              │
└────────────────────────────┬──────────────────────────────┘
                             │
┌────────────────────────────▼──────────────────────────────┐
│                       CACHE (6)                            │
│                    Redis Cluster                           │
└────────────────────────────┬──────────────────────────────┘
                             │
     ┌───────────────────────┼───────────────────────┐
     │                       │                       │
┌────▼────┐           ┌──────▼──────┐         ┌─────▼─────┐
│   ELK   │           │  MONITORING │         │ SECURITY  │
│   (6)   │           │     (4)     │         │    (6)    │
└─────────┘           └─────────────┘         └───────────┘
                             │
                      ┌──────▼──────┐
                      │ MANAGEMENT  │
                      │     (6)     │
                      └─────────────┘
```

### Distribuzione Host per Cluster

| Cluster | Host | Dettaglio |
|---------|------|-----------|
| **Edge PoP** | 6 | 2 per region (EU, US, APAC) |
| **API Gateway** | 3 | Load balanced, 2 critici |
| **Auth Service** | 3 | LDAP integration, 2 critici |
| **Billing** | 2 | 1 critico |
| **Kubernetes** | 11 | 3 master (critici) + 8 worker |
| **Ceph Storage** | 9 | 3 monitor (critici) + 6 OSD |
| **File Servers** | 4 | 2 NFS + 2 SMB |
| **PostgreSQL** | 3 | 2 critici |
| **MySQL** | 3 | 2 critici |
| **MongoDB** | 3 | 2 critici |
| **Redis** | 6 | 3 critici |
| **Kafka** | 3 | tutti critici |
| **RabbitMQ** | 3 | 1 critico |
| **ELK Stack** | 6 | 2 Elastic critici |
| **Monitoring** | 4 | Prometheus critico |
| **Security** | 6 | Vault(2), WAF(2), IDS(2) |
| **Management** | 6 | Bastion, Ansible, Jenkins, Nexus, Backup |

**Totale: 80 host**

### Porte Vietate

| Porta | Servizio | Strategia | Host Coinvolti |
|-------|----------|-----------|----------------|
| 80 | HTTP | Migra → 8080 | 14 (Edge, K8s worker, Kibana, Grafana, WAF) |
| 21 | FTP | Migra → 2121 | 2 (Nexus, Backup) |
| 23 | Telnet | Chiudi | 9 (tutti i DB) |
| 25 | SMTP | Disattiva | 2 (Billing, Alertmanager) |
| 139 | NetBIOS | Disattiva | 4 (NFS, SMB servers) |
| 161 | SNMP | Disattiva | ~35 (quasi tutti i server) |
| 69 | TFTP | Disattiva | 1 (Backup) |
| 389 | LDAP | Migra → 3890 | 3 (Auth service) |
| 3389 | RDP | Migra → 33389 | 2 (Bastion) |
| 5900 | VNC | Migra → 59000 | 2 (Bastion) |

### Servizi Critici (31 host)

```
Gateway:  api_gateway_01, api_gateway_02
Auth:     auth_service_01, auth_service_02
Billing:  billing_01
K8s:      k8s_master_01, k8s_master_02, k8s_master_03
Ceph:     ceph_mon_01, ceph_mon_02, ceph_mon_03
DB:       db_postgres_01, db_postgres_02, db_mysql_01, db_mysql_02, 
          db_mongo_01, db_mongo_02
Redis:    redis_cluster_01, redis_cluster_02, redis_cluster_03
Kafka:    mq_kafka_01, mq_kafka_02, mq_kafka_03
Rabbit:   mq_rabbit_01
ELK:      elk_elastic_01, elk_elastic_02
Monitor:  monitor_prometheus_01
Security: security_vault_01, security_vault_02, security_waf_01, security_ids_01
```

### Dipendenze (32 relazioni)

Pattern principali:
- `edge.http` → `https` (redirect HTTP→HTTPS)
- `auth.tomcat` → `ldap`, `https`
- `k8s_worker.http` → `https`
- `nfs/smb.service` → `netbios` (protocollo legacy)
- `kibana` → `elasticsearch`
- `grafana` → `prometheus`

### Sfide Principali

1. **Volume estremo**: 80 host, ~50 goal
2. **SNMP massivo**: ~35 host da cui rimuovere SNMP
3. **HTTP su WAF critico**: security_waf_01 ha HTTP critico (deve migrare!)
4. **LDAP critico**: auth_service usa LDAP 389 (migrazione obbligatoria)
5. **Storage legacy**: NFS/SMB dipendono da NetBIOS
6. **Kubernetes**: 8 worker con HTTP da migrare
7. **Database cluster**: 9 nodi con Telnet legacy

### Correzione Applicata

Il file originale conteneva un **deadlock**:
- `billing_01.tomcat` era critico
- `billing_01.tomcat` dipendeva da `smtp`
- `smtp` usa porta 25 (vietata, senza alternativa)

**Fix:** Rimossa dipendenza `tomcat → smtp`. Il billing può inviare email in modo asincrono tramite message queue.

### Metriche Attese

| Metrica | Valore Stimato |
|---------|----------------|
| Oggetti UP | ~240 |
| Azioni ground | ~300 |
| Goal | ~50 |
| Piano | 70-100 azioni |
| Tempo | 40-90 secondi |

---

## Come Eseguire

### Preparazione (aumentare timeout)

```python
# Modifica la funzione solve nel notebook
def solve(problem, timeout=120):  # 2 minuti
    try:
        with Compiler(problem_kind=problem.kind, 
                      compilation_kind=CompilationKind.GROUNDING) as compiler:
            compiled = compiler.compile(problem)
            with OneshotPlanner(name='fast-downward') as planner:
                result = planner.solve(compiled.problem, timeout=timeout)
                if result.plan:
                    result.plan = result.plan.replace_action_instances(
                        compiled.map_back_action_instance)
                return result, len(list(compiled.problem.actions))
    except Exception as e:
        print(f"Errore: {e}")
        return None, 0
```

### Esecuzione Singola

```python
import json
import time

# Carica scenario
with open('stress_multinational_60host.json') as f:
    scenario = json.load(f)

print(f"Scenario: {scenario['scenario_name']}")
print(f"Host: {len(scenario['hosts'])}")

# Setup e risolvi
problem, domain, objects = setup_problem(scenario)
print(f"Oggetti: {len(problem.all_objects)}")
print(f"Azioni: {len(problem.actions)}")
print(f"Goal: {len(problem.goals)}")

start = time.time()
result, ground = solve(problem, timeout=120)
elapsed = time.time() - start

if result and result.plan:
    plan = result.plan.actions
    print(f"\n✅ RISOLTO in {elapsed:.1f}s")
    print(f"Piano: {len(plan)} azioni")
    
    # Conta tipi azione
    migra = sum(1 for a in plan if 'migra' in str(a))
    stop = sum(1 for a in plan if 'disattiva' in str(a))
    chiudi = sum(1 for a in plan if 'chiudi' in str(a))
    print(f"Migrazioni: {migra}, Stop: {stop}, Chiusure: {chiudi}")
else:
    print(f"\n❌ Non risolto dopo {elapsed:.1f}s")
```

### Esecuzione Batch

```python
# Copia nella cartella scenarios
import shutil
for f in ['stress_multinational_60host.json', 'stress_cloud_provider_80host.json']:
    shutil.copy(f, 'scenarios/')

# Esegui tutti (con timeout aumentato)
df_results = run_all_scenarios()
```

---

## Confronto Scenari

| Caratteristica | Multinational | Cloud Provider |
|----------------|---------------|----------------|
| Host | 60 | 80 |
| Region/Cluster | 4 + Global | 12 |
| Porte vietate | 10 | 10 |
| Goal stimati | ~35 | ~50 |
| Servizi critici | 21 | 31 |
| Dipendenze | 39 | 32 |
| SNMP da rimuovere | ~15 | ~35 |
| Complessità | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |

---

## Troubleshooting

### Scenario UNSOLVABLE

Se uno scenario risulta irrisolvibile, verificare:

1. **Servizio critico su porta vietata senza alternativa**
   ```python
   # Porte senza alternativa: 25, 139, 161, 69
   # Se un servizio critico usa queste porte → deadlock
   ```

2. **Dipendenza critica da servizio da disattivare**
   ```python
   # Se A è critico e dipende da B
   # E B deve essere disattivato → deadlock
   ```

3. **Porta alternativa già occupata**
   ```python
   # Se porta 80 deve migrare a 8080
   # Ma 8080 è già aperta → impossibile
   ```

### Timeout

Se il planner va in timeout:
- Aumentare `timeout` a 180-300 secondi
- Verificare che Fast-Downward sia installato correttamente
- Considerare di ridurre il numero di host per test

---

## Note sulla Complessità Computazionale

| Fattore | Impatto |
|---------|---------|
| Numero host | Lineare sugli oggetti |
| Porte per host | Lineare sulle azioni |
| Dipendenze | Esponenziale sulle precondizioni |
| Servizi critici | Riduce spazio delle soluzioni |
| SNMP diffuso | Molte azioni disattiva_servizio |

Fast-Downward usa euristiche (h^FF, landmark) per navigare efficientemente lo spazio degli stati. Con 80 host, lo spazio teorico è ~10^20 stati, ma le euristiche lo riducono drasticamente.

---

*Stress test progettati per validare la scalabilità del Network Hardening Planner*
