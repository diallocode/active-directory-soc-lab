# Laboratoire SOC multi-LAN — Active Directory et Elastic

J’ai construit ce laboratoire sous VMware pour suivre le parcours d’un événement de sécurité, depuis sa génération sur un poste jusqu’à son analyse dans Kibana. Le lab simule un réseau d’entreprise avec trois LAN reliés par un routeur, un domaine Active Directory commun et un SIEM central.

## Architecture

| Segment | Machines et rôle |
| --- | --- |
| LAN 1 | Contrôleur de domaine et serveur DNS `DC01`, poste Windows `CLIENT01` et premier collecteur Windows |
| LAN 2 | Poste Windows `CLIENT02` et second collecteur Windows |
| LAN 3 | Poste Ubuntu intégré au domaine |
| Service central | Elasticsearch et Kibana sur un serveur Ubuntu |

Les postes rejoignent le domaine `corp.test`. Les deux collecteurs Windows regroupent les événements avant leur envoi au SIEM. Le poste Ubuntu dispose de son propre agent de collecte.

<!-- Pour afficher le schéma, ajoutez-le dans docs/architecture.png puis décommentez la ligne suivante.
![Architecture du laboratoire SOC multi-LAN](docs/architecture.png)
-->

## Collecte et analyse des journaux

- **Windows** — J’ai activé l’audit de création de processus et la journalisation des lignes de commande par GPO. Sysmon complète les journaux natifs avec des événements comme les créations de processus et les requêtes DNS. Windows Event Forwarding (WEF) transmet les événements aux collecteurs Windows Event Collector (WEC). Winlogbeat lit les événements transférés et les envoie vers Elasticsearch.
- **Linux** — Filebeat collecte les journaux du poste Ubuntu et les envoie directement vers Elasticsearch.
- **Elastic** — Les ingest pipelines traitent les événements à l’arrivée dans Elasticsearch. Dans Kibana Discover, j’ai vérifié la remontée des journaux Windows, Sysmon et Linux avant de les exploiter dans les règles de détection.

## Scénarios de détection testés

| Scénario | Ce que la règle recherche |
| --- | --- |
| Échecs d’authentification répétés | Au moins cinq événements Windows `4625` pour un même compte en dix minutes |
| Énumération du système | Exécutions successives de commandes comme `whoami`, `net user`, `ipconfig` et `systeminfo` par un même compte |
| Création d’une tâche planifiée | Événement signalant la création d’une tâche planifiée sur un poste Windows |

Pour valider une règle, j’ai généré l’activité dans le lab, vérifié le déclenchement de l’alerte, puis retrouvé dans Kibana les événements qui l’expliquaient. Dans le scénario d’authentification, j’ai provoqué des échecs de connexion sur un même compte de test et recherché les événements `4625` correspondants.

## Dépannage de la chaîne de collecte

Lors de la mise en place de WEF, certains postes apparaissaient dans l’état de l’abonnement sans transmettre leurs événements. Le journal `Microsoft-Windows-Forwarding/Operational` signalait l’événement `102` avec le code `5004`. J’ai corrigé les permissions de lecture des journaux pour le compte de service concerné, puis vérifié la présence des événements dans `Forwarded Events` et leur remontée dans Kibana.
