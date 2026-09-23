# SecuEnterprise-ELK

Documentation du projet **SecuEnterprise**, incluant les missions 5, 6, 7 et 8 réalisées sur VM-ELK.

> **Bonus réseau/DNS :** [Documentation sur `kibana.local` et la communication entre les machines](BONUS-DNS-RESEAU.md).

---

## Architecture du laboratoire

| Machine | IP Host-Only | Rôle |
|---|---|---|
| VM-Cible | 192.168.100.10 | Génération des logs |
| VM-ELK | 192.168.100.30 | Elasticsearch, Logstash, Kibana, TShark |

Réseau Host-Only : `192.168.100.0/24`. Une interface NAT a également été utilisée ponctuellement pour l'accès Internet. Les captures réalisées sur cette interface affichent donc les adresses du réseau NAT et non celles du réseau Host-Only.

---

# Mission 5 — Installation et configuration de l'ELK Stack

## 1. Les composants

- **Elasticsearch** : stockage et recherche des logs
- **Logstash** : réception et traitement des logs
- **Kibana** : visualisation et analyse
- **Watcher** : détection de conditions et déclenchement d'alertes

Elasticsearch fonctionne sur `https://127.0.0.1:9200`.
Kibana est accessible sur `http://192.168.100.30:5601`.

## 2. Installation

Installation d'Elasticsearch puis de Kibana sur VM-ELK, avec génération du token d'enrôlement pour relier Kibana au cluster.

![Installation de Kibana et génération du token d'enrôlement](images/M5_24.png)

## 3. Logstash

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

Au démarrage, Logstash se connecte à Elasticsearch, crée le template d'index `enterprise-logs-%{+YYYY.MM.dd}` et ouvre ses écouteurs.

![Démarrage du pipeline Logstash et connexion à Elasticsearch](images/M5-M6_16.png)

## 4. Import des logs

La VM-Cible envoie ses logs vers Logstash en UDP sur le port `5514`.

Test :

```bash
logger -n 192.168.100.30 -P 5514 -d "TEST LOG FINAL VM-CIBLE"
```

Les logs sont ensuite stockés dans les index `enterprise-logs-YYYY.MM.dd`.

La réception réseau a été vérifiée avec :

```bash
tcpdump -ni ens34 udp port 5514
```

La capture ci-dessous montre les deux côtés : à gauche le `tcpdump` sur VM-ELK, à droite l'envoi du log depuis VM-Cible.

![tcpdump sur VM-ELK et envoi du log depuis VM-Cible](images/M5-M6_46.png)

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

## 5. Kibana

Une Data View `enterprise-logs-*` a été créée.

Visualisations réalisées :

- graphique en barres des sources ;
- graphique temporel avec `@timestamp` ;
- graphique circulaire des événements ;
- tableau avec `event`, `source`, `message` et `@timestamp`.

Les visualisations sont regroupées dans un dashboard Kibana.

![Dashboard Kibana avec les quatre visualisations](images/M5-M6_35.png)

## 6. Watcher

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
  "trigger": {"schedule": {"interval": "1m"}},
  "input": {
    "search": {
      "request": {
        "indices": ["enterprise-logs-*"],
        "body": {
          "query": {
            "bool": {
              "must": [{"match": {"message": "TEST"}}],
              "filter": [{"range": {"@timestamp": {"gte": "now-5m", "lte": "now"}}}]
            }
          }
        }
      }
    }
  },
  "condition": {"compare": {"ctx.payload.hits.total": {"gt": 0}}},
  "actions": {
    "log_alert": {
      "logging": {"text": "ALERTE SECURITE : log TEST détecté sur VM-Cible"}
    }
  }
}'
```

Un log de test est envoyé depuis la VM-Cible pour déclencher le watch :

```bash
logger -n 192.168.100.30 -P 5514 -d "TEST WATCHER ALERTE VM-CIBLE"
```

![Envoi du log de test depuis VM-Cible](images/M5-M6_70.png)

Exécution manuelle du watch :

```bash
curl -k -u 'elastic:TON_MDP' -X POST 'https://127.0.0.1:9200/_watcher/watch/enterprise-test-alert/_execute?pretty'
```

La sortie montre `"met": true`, `ctx.payload.hits.total = 1` et l'action `log_alert` exécutée.

![Résultat de l'exécution du Watcher : condition remplie et action exécutée](images/M5-M6_75.png)

## 7. Bilan de la Mission 5

| Élément | État |
|---|---|
| Elasticsearch | ✅ |
| Logstash | ✅ |
| Kibana | ✅ |
| Import des logs | ✅ |
| Data View et visualisations | ✅ |
| Dashboard | ✅ |
| Watcher et test d'alerte | ✅ |

**Mission 5 — ELK, Logstash, Kibana et Watcher configurés et testés.**

---

# Mission 6 — Capture et analyse réseau avec Wireshark/TShark

## 1. Installation et contrôle de l'outil

TShark (Wireshark en ligne de commande) est installé et exécutable sur VM-ELK. La version affichée lors du contrôle est **4.4.18**.

```bash
tshark -v
```

![Contrôle de la version de TShark](images/M7_14.png)

## 2. Capture du trafic Syslog

Le trafic UDP envoyé de VM-Cible vers Logstash a été capturé sur l'interface du réseau Host-Only avec le filtre `udp port 5514`.

```bash
tshark -i ens34 -f "udp port 5514" -w /tmp/mission06-syslog.pcapng
tshark -r /tmp/mission06-syslog.pcapng -Y "udp.port == 5514"
```

Échange observé :

- Source : `192.168.100.10`
- Destination : `192.168.100.30`
- Protocole : UDP
- Port de destination : `5514`

La même capture montre ensuite le démarrage de la capture Kibana sur `tcp port 5601`.

![Capture du trafic Syslog UDP/5514 puis du trafic Kibana TCP/5601](images/M7_17.png)

Les deux fichiers de capture produits :

![Fichiers de capture mission06-syslog.pcapng et mission06-kibana.pcapng](images/M7_20.png)

## 3. Capture du trafic Kibana

Le trafic vers Kibana a été capturé sur l'interface NAT avec le filtre `tcp port 5601`. Le fichier utilisé pour l'analyse est `/tmp/mission06-kibana.pcapng`.

Les requêtes HTTP du navigateur vers Kibana sont bien visibles (chargement des bundles, appels à l'API).

![Requêtes HTTP du navigateur vers Kibana](images/M7_22.png)

Les conversations TCP indiquent plusieurs connexions du client `192.168.58.1` vers le serveur Kibana `192.168.58.158:5601`. Ce trafic est cohérent avec les requêtes du navigateur lors du chargement de l'interface. Ces adresses sont celles du réseau NAT utilisé pendant cette capture.

```bash
tshark -r /tmp/mission06-kibana.pcapng -q -z conv,tcp
```

## 4. Codes HTTP relevés

Le filtre `http.response && http.response.code != 200` a relevé les réponses **302, 401, 202, 204, 500 et 503**.

```bash
tshark -r /tmp/mission06-kibana.pcapng \
  -Y 'http.response && http.response.code != 200' \
  -T fields -e frame.number -e ip.src -e ip.dst -e http.response.code
```

La capture ci-dessous montre la synthèse des conversations TCP puis la liste des codes non-200.

![Conversations TCP et codes HTTP non-200](images/M7_24.png)

Interprétation :

- `302` : redirection, potentiellement normale lors de la navigation ou de l'authentification ;
- `401` : authentification requise ou refusée ;
- `202` et `204` : réponses HTTP valides pour certaines opérations ;
- `500` : erreur interne serveur ;
- `503` : service temporairement indisponible.

Les codes `500` et `503` justifient une investigation applicative, mais leur présence seule ne prouve pas une attaque. De même, un code `401` n'est pas nécessairement malveillant sans examiner la requête et son contexte.

## 5. Bilan de la Mission 6

| Activité | Résultat |
|---|---|
| Installation et exécution de TShark | Réalisé — version 4.4.18 |
| Capture Syslog UDP/5514 | Réalisée ; échange VM-Cible → VM-ELK observé |
| Capture du trafic Kibana/TCP 5601 | Réalisée sur l'interface NAT |
| Synthèse des conversations TCP | Réalisée |
| Relevé des codes HTTP non-200 | Réalisé |
| Investigation approfondie des erreurs et qualification de paquets suspects | À poursuivre : les codes seuls ne permettent pas de conclure à une attaque |

**Conclusion :** les captures et une première analyse des flux TCP/IP et HTTP ont été effectuées. Une conclusion de sécurité définitive nécessiterait de replacer les réponses atypiques dans le contexte des requêtes et des journaux applicatifs.

---

# Mission 7 — Tendance des connexions échouées

## 1. Recherche des événements

Dans Kibana Discover, la Data View `enterprise-logs` a été utilisée pour rechercher les messages contenant le terme `failed`, avec la requête KQL :

```kql
message: *failed*
```

La recherche a fait apparaître un événement syslog généré sur la VM-Cible : **`SECURITY TEST failed login for user admin from 192.168.100.10`**. Il s'agit d'un événement de test explicitement généré pour valider la recherche ; il ne constitue pas, à lui seul, la preuve d'une tentative malveillante réelle.

![Recherche message: *failed* dans Discover](images/BONUS_09.png)

## 2. Visualisation temporelle

Une visualisation Lens de type **graphique en barres** a été configurée avec :

- **Axe horizontal :** `@timestamp` ;
- **Axe vertical :** nombre d'enregistrements ;
- **Filtre :** `message: *failed*` ;
- **Période affichée :** les 7 derniers jours.

![Graphique en barres des connexions échouées sur 7 jours](images/BONUS_15.png)

La visualisation a été enregistrée sous le titre **« Mission 07 - Tendance des connexions échouées »**, ajoutée à la bibliothèque et associée au **Dashboard Kibana 1**.

![Enregistrement de la visualisation Lens et ajout au dashboard](images/BONUS_13.png)

![Dashboard Kibana 1 avec le filtre message: *failed*](images/BONUS_17.png)

La visualisation a affiché une occurrence dans la période sélectionnée. Cela valide l'affichage de la donnée filtrée, mais une tendance statistique fiable nécessiterait un historique plus important et plusieurs événements répartis dans le temps.

## 3. Bilan de la Mission 7

| Élément | Résultat |
|---|---|
| Recherche des messages de connexion échouée dans Discover | Réalisée — événement de test retrouvé |
| Visualisation temporelle Lens | Créée — barres, `@timestamp` et nombre d'enregistrements |
| Enregistrement et ajout au dashboard | Réalisés — `Dashboard Kibana 1` |
| Analyse de tendance sur un volume conséquent | À approfondir : une occurrence observée |

**Conclusion :** la recherche d'un événement `failed` et la visualisation temporelle correspondante sont configurées. Le résultat observé est une preuve fonctionnelle de la chaîne de recherche et de visualisation, et non une conclusion qu'une attaque a eu lieu.

---

# Mission 8 — Réponse à incident

Deux playbooks de réponse à incident, construits sur l'infrastructure des missions 5 à 7 (ELK, Watcher, TShark).

Chaque incident a été **déclenché réellement** sur le laboratoire, puis détecté par la chaîne de supervision. Les captures montrent la détection telle qu'elle s'est produite.

Les procédures suivent les trois temps demandés : **contenir, éradiquer, récupérer**.

## Playbook A — Intrusion Borg (tentatives de connexion répétées)

### 1. Le scénario

Une série de connexions échouées sur le compte `admin`, toutes depuis la même adresse. C'est le comportement typique d'une attaque par force brute : l'attaquant essaie des mots de passe à la chaîne.

Simulation de l'attaque depuis VM-Cible :

```bash
for i in $(seq 1 10); do
  logger -n 192.168.100.30 -P 5514 -d "SECURITY BORG failed login for user admin from 192.168.100.66"
  sleep 1
done
```

### 2. Détection

Un watch `borg-bruteforce` a été créé dans Elasticsearch :

| Paramètre | Valeur |
|---|---|
| Fréquence | toutes les minutes |
| Index | `enterprise-logs-*` |
| Recherche | `failed` dans le champ `message` |
| Fenêtre | 5 minutes |
| Condition | plus de 5 occurrences |
| Action | écrit une alerte dans le journal |

**Pourquoi un seuil à 5 ?** Une connexion échouée isolée, c'est une faute de frappe. Cinq en cinq minutes, non. Le seuil sert à séparer l'erreur humaine de l'attaque. Trop bas, on crée des fausses alertes ; trop haut, on laisse passer l'attaque.

Création du watch :

```bash
read -s -p "Mot de passe elastic : " ES_PWD; echo
curl -k -u "elastic:$ES_PWD" \
-X PUT 'https://127.0.0.1:9200/_watcher/watch/borg-bruteforce' \
-H 'Content-Type: application/json' \
-d '{
  "trigger": {"schedule": {"interval": "1m"}},
  "input": {
    "search": {
      "request": {
        "indices": ["enterprise-logs-*"],
        "body": {
          "query": {
            "bool": {
              "must": [{"match": {"message": "failed"}}],
              "filter": [{"range": {"@timestamp": {"gte": "now-5m", "lte": "now"}}}]
            }
          }
        }
      }
    }
  },
  "condition": {"compare": {"ctx.payload.hits.total": {"gt": 5}}},
  "actions": {
    "log_alert": {
      "logging": {"text": "ALERTE BORG : tentatives de connexion repetees detectees"}
    }
  }
}'
```

Le mot de passe n'est pas écrit dans la commande : `read -s` le demande au clavier sans l'afficher, et `curl` le récupère dans la variable. Cela évite de le retrouver dans les captures d'écran et dans l'historique du dépôt.

![Exécution du watch](images/M8_A1_commande.png)

### 3. Résultat de la détection

Le watch s'exécute, la condition est remplie et l'action part.

![Condition remplie et alerte déclenchée](images/M8_A2_resultat.png)

- `"met" : true` — la condition est remplie
- `"ctx.payload.hits.total" : 32` — 32 événements `failed` dans la fenêtre de 5 minutes
- `"id" : "log_alert"` en `"status" : "success"` — l'alerte a bien été écrite

Les 32 événements sont ceux de la **fenêtre de 5 minutes** du watch. Dans Discover, sur 7 jours, on en compte 50 : ce sont tous les événements envoyés depuis le début des tests.

![Recherche des événements BORG dans Discover](images/M8_A3_discover.png)

L'écart entre 50 et 32 n'est pas une erreur, c'est la différence de fenêtre. Le watch ne compte pas une attaque : il compte ce qui s'est produit sur les 5 dernières minutes. C'est ce qui lui permet de détecter un pic plutôt qu'un total.

### 4. Qualifier avant d'agir

Avant de bloquer quoi que ce soit, quatre questions. Bloquer une adresse légitime, c'est créer une panne en croyant régler un incident.

| Question | Comment y répondre |
|---|---|
| Combien d'adresses sources ? | une seule → attaque ciblée ; plusieurs → attaque distribuée |
| Le compte visé existe-t-il vraiment ? | si non, l'attaque est vouée à l'échec |
| L'adresse source est-elle connue ? | un poste interne mal configuré donne le même signal |
| **Une connexion a-t-elle réussi après les échecs ?** | c'est la question qui change tout |

Cette dernière est la plus importante. Une série d'échecs suivie d'un succès signifie que l'attaquant a trouvé le mot de passe. On ne traite plus une tentative, on traite une intrusion.

Dans Kibana Discover :

```kql
message: *failed*
message: *Accepted*
```

### 5. Contenir

| Action | Raison |
|---|---|
| Bloquer l'adresse source au pare-feu | arrête l'attaque en cours |
| Suspendre le compte visé | empêche l'accès même si le mot de passe est trouvé |
| Vérifier si le compte a des droits d'administration | détermine la gravité |

**Ne pas éteindre la machine.** C'est le réflexe naturel et c'est une erreur : on perd la mémoire vive et les connexions en cours, donc une partie des traces.

### 6. Éradiquer

Deux cas, selon la qualification.

**Aucune connexion réussie** — il n'y a rien à éradiquer, l'attaque a échoué. On garde les traces et on renforce : mot de passe plus long, limitation du nombre d'essais.

**Une connexion a réussi** — l'attaquant est entré. Il faut :
- changer le mot de passe du compte et de tous ceux qui partagent le même ;
- chercher ce qu'il a laissé derrière lui : tâches planifiées, clés SSH ajoutées, nouveaux comptes ;
- en cas de doute sur l'étendue, réinstaller la machine à partir d'une sauvegarde saine.

### 7. Récupérer

1. Réactiver le compte avec un nouveau mot de passe.
2. Lever le blocage de l'adresse une fois l'incident clos.
3. Vérifier que les logs remontent toujours dans `enterprise-logs-*` — une chaîne de supervision cassée pendant l'incident, c'est un second incident.
4. Ajuster le seuil du watch si l'incident a montré qu'il était mal réglé.

### 8. Traces à conserver

- l'export Discover des événements de la période ;
- la sortie du watch (elle horodate la détection) ;
- une capture réseau TShark si l'attaque est encore en cours.

### 9. Limite de ce playbook

L'action du watch se contente d'écrire dans un journal. **Personne n'est prévenu en temps réel** : il faut que quelqu'un aille lire. Dans un environnement réel, l'action enverrait un mail ou appellerait un webhook vers un outil de ticketing.

C'est la limite assumée de ce laboratoire — la détection fonctionne, la notification reste à construire.

---

## Playbook B — Sabotage Klingon (arrêt de la remontée de logs)

### 1. Le scénario

Un attaquant déjà présent sur la machine coupe l'envoi des logs vers le serveur de supervision. Objectif : continuer à agir sans laisser de trace. Ce n'est pas un incident bruyant comme le bruteforce — au contraire, il cherche le silence.

Simulation depuis VM-Cible, en bloquant la sortie UDP 5514 :

```bash
nft add table inet sabotage
nft add chain inet sabotage output '{ type filter hook output priority 0; }'
nft add rule inet sabotage output udp dport 5514 drop
logger -n 192.168.100.30 -P 5514 -d "KLINGON sabotage en cours"
```

![Blocage de la sortie des logs](images/M8_B1_sabotage.png)

Le `logger` échoue avec `Opération non permise` : le message ne sort pas de la machine. C'est la preuve que le sabotage fonctionne. À partir de cet instant, tout ce qui se passe sur VM-Cible est invisible depuis Kibana.

### 2. Détection

Ici on ne peut pas détecter un pic : il n'y a plus rien à compter. Il faut détecter **l'absence**.

Un watch `klingon-sabotage` a été créé :

| Paramètre | Valeur |
|---|---|
| Fréquence | toutes les minutes |
| Index | `enterprise-logs-*` |
| Recherche | tous les événements, sans filtre de contenu |
| Fenêtre | 5 minutes |
| Condition | 0 événement (`lte: 0`) |
| Action | écrit une alerte dans le journal |

La seule différence avec le playbook A tient dans la condition : `gt: 5` devient `lte: 0`. On ne cherche plus un excès, on cherche un vide.

```bash
read -s -p "Mot de passe elastic : " ES_PWD; echo
curl -k -u "elastic:$ES_PWD" \
-X PUT 'https://127.0.0.1:9200/_watcher/watch/klingon-sabotage' \
-H 'Content-Type: application/json' \
-d '{
  "trigger": {"schedule": {"interval": "1m"}},
  "input": {
    "search": {
      "request": {
        "indices": ["enterprise-logs-*"],
        "body": {
          "query": {
            "bool": {
              "filter": [{"range": {"@timestamp": {"gte": "now-5m", "lte": "now"}}}]
            }
          }
        }
      }
    }
  },
  "condition": {"compare": {"ctx.payload.hits.total": {"lte": 0}}},
  "actions": {
    "log_alert": {
      "logging": {"text": "ALERTE KLINGON : plus aucun log recu depuis 5 minutes"}
    }
  }
}'
```

### 3. Résultat de la détection

Cinq minutes après le blocage, la fenêtre est vide et le watch se déclenche.

![Condition remplie : plus aucun log reçu](images/M8_B2_resultat.png)

- `"met" : true` — la condition est remplie
- `"ctx.payload.hits.total" : 0` — aucun événement sur les 5 dernières minutes
- l'alerte `ALERTE KLINGON : plus aucun log recu depuis 5 minutes` est écrite

Il faut attendre que la fenêtre se vide complètement. Tant qu'un seul log de moins de 5 minutes subsiste, le compteur n'est pas à zéro et l'alerte ne part pas. C'est aussi le délai de détection : ce type d'alerte ne peut pas être instantané.

### 4. Qualifier avant d'agir

Le silence n'est pas forcément une attaque. C'est la principale difficulté de ce playbook : la même alerte se déclenche pour une panne banale.

| Cause possible | Comment la reconnaître |
|---|---|
| La machine est éteinte ou a redémarré | elle ne répond pas au ping |
| Le service d'envoi des logs est arrêté | la machine répond, mais `systemctl` montre le service inactif |
| Panne réseau entre les deux machines | les autres flux sont coupés aussi |
| Disque plein côté Elasticsearch | plus aucune machine ne remonte de logs, pas seulement celle-ci |
| **Sabotage** | la machine fonctionne, le réseau fonctionne, mais une règle de pare-feu bloque le port |

Le test qui tranche : **est-ce que les autres machines remontent encore des logs ?** Si oui, le problème est local à cette machine. Si non, il est côté serveur.

Pour vérifier la présence d'une règle de blocage sur la machine suspecte :

```bash
nft list ruleset
```

### 5. Contenir

| Action | Raison |
|---|---|
| Isoler la machine du reste du réseau | si c'est un sabotage, l'attaquant y a les droits root |
| Renforcer la surveillance des autres machines | le sabotage précède souvent une action plus large |
| Récupérer les logs locaux de la machine | ils n'ont pas été envoyés, mais ils existent peut-être encore sur place |

Ce dernier point est important : les logs bloqués en sortie sont souvent toujours présents dans `/var/log` sur la machine. C'est la seule façon de reconstituer ce qui s'est passé pendant l'aveuglement.

### 6. Éradiquer

Supprimer la règle ne suffit pas. **Poser une règle de pare-feu demande les droits root** : il y a donc une compromission antérieure, et c'est elle le vrai incident.

1. Supprimer la règle de blocage.
2. Chercher comment l'attaquant a obtenu les droits : historique des commandes, comptes créés, tâches planifiées, clés SSH.
3. Remonter à l'intrusion d'origine — le sabotage n'est qu'une conséquence.
4. Si l'origine reste introuvable, réinstaller la machine.

### 7. Récupérer

```bash
nft delete table inet sabotage
```

Puis :

1. Vérifier que les logs remontent à nouveau, avec un `logger` de test suivi d'une recherche dans Discover.
2. Mesurer la durée de l'aveuglement : l'horodatage du dernier log reçu en marque le début.
3. Récupérer les logs locaux de la période si la machine les a conservés.
4. Accepter que la période soit peut-être définitivement perdue, et le noter dans le rapport d'incident.

### 8. Traces à conserver

- la règle de pare-feu trouvée (`nft list ruleset`) ;
- l'horodatage du dernier log reçu, qui date le début du sabotage ;
- la sortie du watch, qui date la détection ;
- les logs locaux de la machine sur la période aveugle.

L'écart entre le dernier log reçu et le déclenchement de l'alerte donne le délai de détection. C'est un chiffre à connaître : il mesure la qualité de la supervision.

### 9. Limites de ce playbook

**Le watch ne distingue pas un sabotage d'une panne.** Il signale un silence, la qualification reste humaine. Un watch par machine permettrait au moins de savoir laquelle s'est tue.

**Si l'attaquant s'en prend à VM-ELK elle-même, plus rien ne détecte quoi que ce soit.** Le superviseur n'est pas supervisé. Dans une vraie infrastructure, c'est un serveur tiers qui vérifie que la chaîne de supervision est en vie.

---

## Ce que les deux playbooks ont en commun

| | Playbook A — Borg | Playbook B — Klingon |
|---|---|---|
| Signal | trop d'événements | plus aucun événement |
| Condition | `gt: 5` | `lte: 0` |
| Détection | quasi immédiate | 5 minutes minimum |
| Risque d'erreur | bloquer une adresse légitime | confondre panne et attaque |

Les deux reposent sur la même chaîne : Logstash collecte, Elasticsearch indexe, le Watcher compare à un seuil. Seule la condition change.

La supervision ne dit jamais « il y a une attaque ». Elle dit « ce que j'observe sort de l'ordinaire ». La qualification, puis la décision de contenir, restent humaines — et c'est pour cela que ces procédures s'écrivent à l'avance, à froid, plutôt que pendant l'incident.

---

> Aucun mot de passe réel n'est présent dans ce dépôt. Les chaînes `TON_MDP` sont des placeholders à remplacer localement.
