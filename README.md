# SecuEnterprise-ELK

Documentation de la **Mission 5 — Installation et configuration de l'ELK Stack** du projet SecuEnterprise.

> **Bonus réseau/DNS :** [Consulter la documentation sur la résolution de `kibana.local` et la communication entre les machines](BONUS-DNS-RESEAU.md).

## Architecture

| Machine | IP | Rôle |
|---|---|---|
| VM-Cible | 192.168.100.10 | Génération des logs |
| VM-ELK | 192.168.100.30 | Elasticsearch, Logstash, Kibana |

Réseau Host-Only : `192.168.100.0/24`

## 1. ELK Stack

- **Elasticsearch** : stockage et recherche des logs
- **Logstash** : réception et traitement des logs
- **Kibana** : visualisation et analyse
- **Watcher** : détection de conditions et déclenchement d'alertes

Elasticsearch fonctionne sur `https://127.0.0.1:9200`.

Kibana est accessible sur `http://192.168.100.30:5601`.

## 2. Logstash

Fichier : `/etc/logstash/conf.d/enterprise.conf`

```conf
input {
  tcp {
    port => 5044
    codec => json
  }

  udp {
    port => 5514
    type => "syslog"
    ecs_compatibility => "disabled"
  }
}

output {
  elasticsearch {
    hosts => ["https://127.0.0.1:9200"]
    index => "enterprise-logs-%{+YYYY.MM.dd}"
    user => "elastic"
    password => "TON_MDP"
    ssl_certificate_authorities => ["/etc/logstash/certs/http_ca.crt"]
  }

  stdout {
    codec => rubydebug
  }
}
```

Le mot de passe réel n'est pas stocké dans GitHub : `TON_MDP` est un placeholder.

Test de configuration :

```bash
su -s /bin/bash logstash -c '/usr/share/logstash/bin/logstash --path.settings /etc/logstash -t'
```

Résultat : `Configuration OK`.

Puis :

```bash
systemctl restart logstash
systemctl status logstash --no-pager
```

Résultat : service `active (running)`.

## 3. Import des logs

La VM-Cible envoie ses logs vers Logstash en UDP sur le port `5514`.

Test :

```bash
logger -n 192.168.100.30 -P 5514 -d "TEST LOG FINAL VM-CIBLE"
```

Les logs sont ensuite stockés dans les index :

```text
enterprise-logs-YYYY.MM.dd
```

La réception réseau a été vérifiée avec :

```bash
tcpdump -ni ens34 udp port 5514
```

Chaîne validée :

```text
VM-Cible (192.168.100.10)
        |
        | UDP 5514
        v
Logstash (192.168.100.30)
        |
        v
Elasticsearch
```

## 4. Kibana

Une Data View `enterprise-logs-*` a été créée.

Visualisations réalisées :

- graphique en barres des sources ;
- graphique temporel avec `@timestamp` ;
- graphique circulaire des événements ;
- tableau avec `event`, `source`, `message` et `@timestamp`.

Les visualisations sont regroupées dans un dashboard Kibana.

## 5. Watcher

Watch configuré : `enterprise-test-alert`.

Paramètres :

- déclenchement toutes les **1 minute** ;
- recherche dans `enterprise-logs-*` ;
- recherche des messages contenant `TEST` ;
- fenêtre temporelle : **5 minutes** ;
- condition : `hits.total > 0` ;
- action : journalisation d'une alerte.

Création :

```bash
curl -k -u 'elastic:TON_MDP' \
-X PUT 'https://127.0.0.1:9200/_watcher/watch/enterprise-test-alert' \
-H 'Content-Type: application/json' \
-d '{
  "trigger": {
    "schedule": {
      "interval": "1m"
    }
  },
  "input": {
    "search": {
      "request": {
        "indices": ["enterprise-logs-*"],
        "body": {
          "query": {
            "bool": {
              "must": [
                {"match": {"message": "TEST"}}
              ],
              "filter": [
                {
                  "range": {
                    "@timestamp": {
                      "gte": "now-5m",
                      "lte": "now"
                    }
                  }
                }
              ]
            }
          }
        }
      }
    }
  },
  "condition": {
    "compare": {
      "ctx.payload.hits.total": {
        "gt": 0
      }
    }
  },
  "actions": {
    "log_alert": {
      "logging": {
        "text": "ALERTE SECURITE : log TEST détecté sur VM-Cible"
      }
    }
  }
}'
```

Vérification :

```bash
curl -k -u 'elastic:TON_MDP' 'https://127.0.0.1:9200/_watcher/watch/enterprise-test-alert?pretty'
```

Exécution manuelle :

```bash
curl -k -u 'elastic:TON_MDP' -X POST 'https://127.0.0.1:9200/_watcher/watch/enterprise-test-alert/_execute?pretty'
```

Le test avec :

```bash
logger -n 192.168.100.30 -P 5514 -d "TEST WATCHER ALERTE VM-CIBLE"
```

a permis de valider la détection : résultat `hits.total = 1`, condition satisfaite et action exécutée.

## 6. Résultat de la Mission 5

| Élément | État |
|---|---|
| Elasticsearch | ✅ |
| Logstash | ✅ |
| Kibana | ✅ |
| Import des logs | ✅ |
| Data View | ✅ |
| Visualisations | ✅ |
| Dashboard | ✅ |
| Watcher | ✅ |
| Condition d'alerte | ✅ |
| Test de détection | ✅ |

**Mission 5 — ELK, Logstash, Kibana et Watcher configurés et testés.**

> Aucun mot de passe réel n'est présent dans ce dépôt.
