# Mission 06 — Capture et analyse réseau avec Wireshark/TShark

Cette page documente les manipulations effectuées sur VM-ELK. Les captures ont été réalisées alors que la carte réseau était en NAT pour permettre l'accès Internet ; les adresses relevées dans la capture Kibana appartiennent donc au réseau NAT, et non au réseau Host-only du laboratoire.

## 1. Outil

TShark (Wireshark en ligne de commande) est installé et exécutable sur VM-ELK. La version affichée lors du contrôle est **4.4.18**.

## 2. Capture du trafic Syslog

Le trafic UDP envoyé de VM-Cible vers Logstash a été capturé sur l'interface réseau du laboratoire avec le filtre `udp port 5514`.

Échange observé :

- Source : `192.168.100.10`
- Destination : `192.168.100.30`
- Protocole : UDP
- Port de destination : `5514`

La capture a été enregistrée sous `/tmp/mission06-syslog.pcapng`. Une lecture filtrée avec `udp.port == 5514` a permis de retrouver le datagramme.

Commandes d'analyse utilisées :

```bash
tshark -r /tmp/mission06-syslog.pcapng -Y 'udp.port == 5514'
tshark -r /tmp/mission06-syslog.pcapng -Y 'udp.port == 5514' -T fields \
  -e frame.number -e ip.src -e ip.dst -e udp.dstport -e data
```

## 3. Capture du trafic Kibana

Le trafic vers Kibana a été capturé sur l'interface NAT avec le filtre `tcp port 5601`. Le fichier utilisé pour l'analyse est `/tmp/mission06-kibana.pcapng`.

Les conversations TCP indiquent plusieurs connexions du client `192.168.58.1` vers le serveur Kibana `192.168.58.158:5601`. Le trafic observé est cohérent avec les requêtes du navigateur lors du chargement de l'interface.

Commande de synthèse :

```bash
tshark -r /tmp/mission06-kibana.pcapng -q -z conv,tcp
```

## 4. Codes HTTP relevés

Le filtre `http.response && http.response.code != 200` a relevé des réponses **302, 401, 202, 204, 500 et 503**.

Interprétation prudente :

- `302` : redirection, qui peut être normale dans un parcours de navigation ou d'authentification ;
- `401` : authentification requise ou refusée ;
- `202` et `204` : réponses HTTP valides pour certaines opérations ;
- `500` : erreur interne serveur ;
- `503` : service temporairement indisponible.

Les codes `500` et `503` méritent une investigation applicative, mais leur présence ne constitue pas à elle seule la preuve d'une attaque. De même, le code `401` n'est pas nécessairement malveillant sans examiner la requête et son contexte.

Commande utilisée :

```bash
tshark -r /tmp/mission06-kibana.pcapng \
  -Y 'http.response && http.response.code != 200' \
  -T fields -e frame.number -e ip.src -e ip.dst -e http.response.code
```

Pour examiner le détail des erreurs :

```bash
tshark -r /tmp/mission06-kibana.pcapng \
  -Y 'http.response.code == 500 || http.response.code == 503' -V
```

## 5. Bilan

| Activité | Résultat |
|---|---|
| Installation et exécution de TShark | Réalisé |
| Capture Syslog UDP/5514 | Réalisée ; échange VM-Cible → VM-ELK observé |
| Capture du trafic Kibana/TCP 5601 | Réalisée |
| Synthèse des conversations TCP | Réalisée |
| Relevé des codes HTTP non-200 | Réalisé |
| Investigation approfondie des erreurs et qualification d'éventuels paquets suspects | À poursuivre ; les codes observés ne suffisent pas à conclure à une attaque |

**Conclusion :** les captures et une première analyse des flux TCP/IP et HTTP ont été effectuées. Une conclusion de sécurité définitive nécessiterait de replacer les réponses anormales dans le contexte des requêtes et des journaux applicatifs.