# Bonus — Résolution de nom et communication réseau

Cette note complète la documentation de la Mission 5. Elle décrit la configuration observée pendant les tests pour accéder à Kibana depuis un poste Windows et les deux interfaces réseau de la VM ELK.

## 1. Interfaces de la VM ELK

La VM ELK (`kibana`) dispose de deux interfaces réseau :

| Interface | Adresse observée | Usage |
|---|---|---|
| `ens32` | `10.10.7.207/16` | Interface en mode bridge, sur le réseau physique/local |
| `ens34` | `192.168.100.30/24` | Réseau Host-Only utilisé pour le réseau de laboratoire |

L'adresse en bridge peut être attribuée par DHCP et donc changer. Vérifier l'adresse actuelle avec `ip -br addr` avant de configurer les postes clients.

## 2. Nom d'hôte et Avahi

Le nom d'hôte Linux a été défini sur `kibana` :

```bash
hostnamectl set-hostname kibana
```

Le service Avahi a été redémarré puis vérifié :

```bash
systemctl restart avahi-daemon
hostname
systemctl status avahi-daemon --no-pager
```

Le service est indiqué comme `active (running)`. Avahi permet la découverte/résolution locale par mDNS sur les réseaux compatibles. Cependant, le test Windows ci-dessous a montré que la résolution de `kibana.local` provenait d'une entrée locale : `nslookup` interroge le DNS configuré et ne consulte pas le fichier `hosts` de Windows.

## 3. Résolution de `kibana.local` sur Windows

Sur le poste Windows utilisé pour les tests, la commande `ping kibana.local` a résolu le nom vers `192.168.100.30`. Pour reproduire cette résolution sur un autre poste Windows, ajouter une entrée au fichier `hosts` avec des droits administrateur.

Ouvrir le fichier :

```powershell
notepad C:\Windows\System32\drivers\etc\hosts
```

Pour accéder à la VM par son interface Host-Only :

```text
192.168.100.30 kibana.local
```

Si les postes doivent plutôt accéder à l'interface bridge et qu'ils peuvent joindre cette adresse, utiliser l'adresse bridge actuelle, par exemple :

```text
10.10.7.207 kibana.local
```

Ne conserver qu'une seule entrée active pour ce nom afin d'éviter une résolution ambiguë. Après modification, vider le cache et tester :

```powershell
ipconfig /flushdns
ping kibana.local
Test-NetConnection kibana.local -Port 5601
```

Le résultat attendu est que le nom se résolve vers l'adresse choisie et que `TcpTestSucceeded` soit `True` pour le port `5601`.

> Important : chaque poste client doit disposer de la résolution nécessaire, sauf si un DNS ou un mécanisme mDNS commun est effectivement configuré et pris en charge sur le réseau. L'entrée `hosts` d'un poste n'est pas automatiquement partagée avec les autres.

## 4. Accès à Kibana

Kibana répond sur le port TCP `5601`. L'accès navigateur se fait avec :

```text
http://kibana.local:5601
```

Pendant les tests, les commandes `curl -I` vers `http://127.0.0.1:5601` et `http://192.168.100.30:5601` ont renvoyé `HTTP/1.1 302 Found` avec une redirection vers `/login?next=%2F`. Cela confirme que le service HTTP de Kibana répond sur ces adresses ; la redirection vers la page de connexion est normale.

## 5. Communication entre les machines du laboratoire

Le réseau Host-Only utilisé pour les échanges internes est `192.168.100.0/24` :

```text
VM-Cible   192.168.100.10
     |  UDP/5514 (envoi des journaux)
     v
VM-ELK     192.168.100.30
```

La réception des journaux a été testée avec `tcpdump` sur `ens34` et leur présence a été vérifiée dans Elasticsearch. Le mode bridge constitue un autre chemin d'accès, notamment depuis les postes du réseau local, sous réserve que le routage et les règles de pare-feu autorisent la communication.

## 6. Résultats observés

- Depuis Windows, `ping kibana.local` a répondu depuis `192.168.100.30`.
- `Test-NetConnection kibana.local -Port 5601` a retourné `TcpTestSucceeded : True`.
- `nslookup kibana.local` a retourné `Non-existent domain` auprès du DNS configuré : cela ne contredit pas une résolution par le fichier `hosts`.
- Kibana a répondu avec une redirection HTTP `302` vers sa page de connexion.

Ces résultats sont ceux des tests effectués dans l'environnement décrit. Ils ne garantissent pas que l'adresse bridge restera identique ni que tous les postes du réseau pourront l'atteindre sans configuration complémentaire.
