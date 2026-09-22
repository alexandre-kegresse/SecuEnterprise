# SecuEnterprise-ELK

Documentation du projet **SecuEnterprise**, incluant les missions 5 et 6 réalisées sur VM-ELK.

> **Bonus réseau/DNS :** [Documentation sur `kibana.local` et la communication entre les machines](BONUS-DNS-RESEAU.md).

# Architecture du laboratoire

| Machine | IP Host-Only | Rôle |
|---|---|---|
| VM-Cible | 192.168.100.10 | Génération des logs |
| VM-ELK | 192.168.100.30 | Elasticsearch, Logstash, Kibana, TShark |

Réseau Host-Only : `192.168.100.0/24`. Une interface NAT a aussi été utilisée ponctuellement pour l'accès Internet. Les captures NAT affichent les adresses du réseau NAT.

# Mission 5 — Installation et configuration de l'ELK Stack

## ELK Stack

- **Elasticsearch** : stockage et recherche des logs, sur `https://127.0.0.1:9200`.
- **Logstash** : réception et traitement des logs.
- **Kibana** : visualisation et analyse, accessible sur `http://192.168.100.30:5601`.
- **Watcher** : détection de conditions et déclenchement d'alertes.

## Logstash

Configuration : `/etc/logstash/conf.d/enterprise.conf`

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
  stdout { codec => rubydebug }
}
```

`TON_MDP` est un placeholder : aucun mot de passe réel n'est versionné.

Test de configuration et contrôle du service :

```bash
su -s /bin/bash logstash -c '/usr/share/logstash/bin/logstash --path.settings /etc/logstash -t'
systemctl restart logstash
systemctl status logstash --no-pager
```

Résultats observés : `Configuration OK` et service `active (running)`.

## Import des logs

La VM-Cible envoie ses logs à Logstash en UDP/5514 :

```bash
logger -n 192.168.100.30 -P 5514 -d "TEST LOG FINAL VM-CIBLE"
tcpdump -ni ens34 udp port 5514
```

Chaîne validée : **VM-Cible (192.168.100.10) → UDP/5514 → Logstash (192.168.100.30) → Elasticsearch**. Les événements sont stockés dans `enterprise-logs-YYYY.MM.dd`.

## Kibana

La Data View `enterprise-logs-*` a été créée. Les visualisations réalisées sont :

- graphique en barres des sources ;
- graphique temporel basé sur `@timestamp` ;
- graphique circulaire des événements ;
- tableau avec `event`, `source`, `message` et `@timestamp`.

Elles sont regroupées dans un dashboard Kibana.

## Watcher

Watch : `enterprise-test-alert`.

- Exécution planifiée toutes les 1 minute.
- Recherche dans `enterprise-logs-*` des messages contenant `TEST` dans les 5 dernières minutes.
- Condition : `hits.total > 0`.
- Action : journalisation d'une alerte.

Création :

```bash
curl -k -u 'elastic:TON_MDP' \
-X PUT 'https://127.0.0.1:9200/_watcher/watch/enterprise-test-alert' \
-H 'Content-Type: application/json' \
-d '{
  "trigger": {"schedule": {"interval": "1m"}},
  "input": {"search": {"request": {"indices": ["enterprise-logs-*"], "body": {"query": {"bool": {
    "must": [{"match": {"message": "TEST"}}],
    "filter": [{"range": {"@timestamp": {"gte": "now-5m", "lte": "now"}}}]
  }}}}}},
  "condition": {"compare": {"ctx.payload.hits.total": {"gt": 0}}},
  "actions": {"log_alert": {"logging": {"text": "ALERTE SECURITE : log TEST détecté sur VM-Cible"}}}
}'
```

Vérification et exécution manuelle :

```bash
curl -k -u 'elastic:TON_MDP' 'https://127.0.0.1:9200/_watcher/watch/enterprise-test-alert?pretty'
curl -k -u 'elastic:TON_MDP' -X POST 'https://127.0.0.1:9200/_watcher/watch/enterprise-test-alert/_execute?pretty'
```

Test émis depuis VM-Cible : `logger -n 192.168.100.30 -P 5514 -d "TEST WATCHER ALERTE VM-CIBLE"`.

Résultat observé : `hits.total = 1`, condition satisfaite et action exécutée.

## Bilan Mission 5

| Élément | État |
|---|---|
| Elasticsearch, Logstash et Kibana | Configurés et opérationnels |
| Import des logs et dashboard | Validés |
| Watcher et test d'alerte | Validés |

# Mission 6 — Capture et analyse réseau avec Wireshark/TShark

## Installation

TShark, l'outil en ligne de commande de Wireshark, est installé et exécutable sur VM-ELK. Version contrôlée : **4.4.18**.

## Capture Syslog

Le trafic VM-Cible → VM-ELK a été capturé avec le filtre `udp port 5514`.

- Source : `192.168.100.10`
- Destination : `192.168.100.30`
- Transport : UDP, port destination `5514`
- Fichier de capture : `/tmp/mission06-syslog.pcapng`

Analyse effectuée :

```bash
tshark -r /tmp/mission06-syslog.pcapng -Y 'udp.port == 5514'
tshark -r /tmp/mission06-syslog.pcapng -Y 'udp.port == 5514' -T fields \
  -e frame.number -e ip.src -e ip.dst -e udp.dstport -e data
```

## Capture Kibana et analyse HTTP

Le trafic TCP/5601 a été capturé sur l'interface NAT dans `/tmp/mission06-kibana.pcapng`. Les conversations affichent des connexions entre le client NAT `192.168.58.1` et Kibana `192.168.58.158:5601`. Ces adresses correspondent au réseau NAT actif pendant la capture, et non au réseau Host-Only.

Synthèse des conversations :

```bash
tshark -r /tmp/mission06-kibana.pcapng -q -z conv,tcp
```

Le filtre `http.response && http.response.code != 200` a relevé les codes **302, 401, 202, 204, 500 et 503**. Les redirections et réponses 401 doivent être interprétées selon le contexte d'authentification ; les codes 500 et 503 peuvent justifier une vérification applicative, mais ne prouvent pas à eux seuls une attaque.

```bash
tshark -r /tmp/mission06-kibana.pcapng \
  -Y 'http.response && http.response.code != 200' \
  -T fields -e frame.number -e ip.src -e ip.dst -e http.response.code

tshark -r /tmp/mission06-kibana.pcapng \
  -Y 'http.response.code == 500 || http.response.code == 503' -V
```

## Captures de preuve

Les captures de terminal fournies pour la Mission 6 sont regroupées ci-dessous : capture Syslog, conversations/flux Kibana et résultats d'analyse HTTP.

![Montage des captures Mission 6 : Syslog UDP, échanges TCP Kibana, codes HTTP et preuve de capture](screenshots/mission06-captures.gif)

## Bilan Mission 6

| Activité | Résultat |
|---|---|
| Installation et contrôle de TShark | Réalisé — version 4.4.18 |
| Capture Syslog UDP/5514 | Réalisée ; flux VM-Cible → VM-ELK observé |
| Capture Kibana/TCP 5601 | Réalisée sur l'interface NAT |
| Analyse des conversations TCP | Réalisée |
| Relevé des codes HTTP non-200 | Réalisé |
| Qualification approfondie des anomalies | À poursuivre : les codes HTTP seuls ne suffisent pas à conclure à une attaque |

**Conclusion :** les captures et une première analyse TCP/IP et HTTP ont été réalisées. Une conclusion de sécurité définitive nécessite de replacer les réponses atypiques dans le contexte des requêtes et des journaux applicatifs.

---

> Aucun mot de passe réel n'est présent dans ce dépôt. Remplacer `TON_MDP` localement par le secret approprié.