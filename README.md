# Audit de sécurité - HP ENVY 5000 series

Petit audit de posture de sécurité d'une imprimante tout-en-un grand public HP ENVY 5000, connectée en Wi-Fi sur un réseau domestique. L'objectif est d'identifier la surface d'attaque exposée par la configuration d'usine et de proposer des correctifs concrets.

Toutes les vérifications décrites ici sont passives ou effectuées sur son propre matériel. Aucune exploitation, aucune attaque.

## Périmètre

* Cible : HP ENVY 5000 series (imprimante jet d'encre multifonction)
* Raccordement : Wi-Fi, réseau local domestique
* Configuration : d'usine, jamais modifiée
* Type d'évaluation : revue de configuration et de surface d'attaque

## Contexte du modèle

L'ENVY 5000 est une imprimante grand public. Comme la plupart des modèles de cette gamme, elle embarque :

* un serveur web d'administration (EWS, Embedded Web Server) en HTTP et HTTPS
* l'impression réseau sur plusieurs protocoles
* le Wi-Fi Direct, qui crée un point d'accès sans fil dédié
* les services web HP (impression cloud, adresse e-mail d'impression)
* un agent SNMP pour la supervision

La plupart de ces services sont actifs par défaut, sans mot de passe.

## Surface d'attaque

Services généralement exposés sur le réseau en sortie d'usine :

| Service | Port | Rôle | Risque |
|---|---|---|---|
| EWS HTTP | 80/tcp | Interface d'administration | Accès admin sans mot de passe |
| EWS HTTPS | 443/tcp | Interface d'administration chiffrée | Certificat auto-signé |
| Impression brute | 9100/tcp | Envoi direct de travaux (RAW/JetDirect) | Impression non authentifiée |
| IPP | 631/tcp | Internet Printing Protocol | Impression non authentifiée |
| LPD | 515/tcp | Impression legacy | Impression non authentifiée |
| SNMP | 161/udp | Supervision | Communauté publique en lecture |
| mDNS/Bonjour | 5353/udp | Découverte réseau | Divulgation d'informations |

Les protocoles d'administration à distance type Telnet et FTP ne sont normalement pas présents sur ce modèle grand public, mais restent à vérifier lors du scan.

## Constats

### Priorité haute

1. **Interface d'administration sans mot de passe.** L'EWS est accessible par quiconque est sur le réseau, sans authentification. Toute la configuration de l'imprimante peut être lue et modifiée.

2. **Wi-Fi Direct actif avec identifiants par défaut.** L'imprimante diffuse son propre point d'accès sans fil. Un appareil à portée peut s'y connecter et imprimer sans jamais rejoindre le réseau principal, en contournant le Wi-Fi domestique.

3. **Firmware jamais mis à jour.** La configuration n'a jamais été touchée, donc le firmware est celui d'origine. Des correctifs de sécurité ont été publiés depuis la sortie du modèle.

### Priorité moyenne

4. **Impression non authentifiée (ports 9100, 631, 515).** N'importe quel appareil du réseau peut envoyer des travaux d'impression, ce qui permet gaspillage de consommables, déni de service local ou envoi de contenu indésirable.

5. **SNMP en communauté publique.** Si l'agent répond à la communauté par défaut, il divulgue des informations sur le matériel et sa configuration.

6. **Services web HP et impression cloud.** L'impression cloud attribue une adresse e-mail à l'imprimante et l'expose à des travaux venant d'Internet. À désactiver si non utilisée.

### Priorité basse

7. **Certificat HTTPS auto-signé.** L'EWS en HTTPS utilise un certificat non vérifiable, ce qui n'empêche pas l'écoute si un attaquant est déjà positionné.

8. **Divulgation d'informations via mDNS.** Le modèle et les services de l'imprimante sont annoncés sur le réseau local.

## Recommandations

Par ordre de priorité :

1. Définir un mot de passe administrateur sur l'EWS (interface web de l'imprimante).
2. Désactiver le Wi-Fi Direct s'il n'est pas utilisé, ou changer son mot de passe.
3. Mettre à jour le firmware via l'application HP Smart, puis activer la mise à jour automatique.
4. Désactiver les protocoles d'impression non utilisés (LPD, IPP si inutile), ne garder que le strict nécessaire.
5. Désactiver SNMP s'il n'est pas utilisé, sinon changer la communauté par défaut.
6. Désactiver les services web HP et l'impression cloud si non utilisés.
7. Isoler l'imprimante sur un réseau invité ou un VLAN dédié, séparé des postes principaux.
8. Désactiver WPS sur le routeur.

## Automatisation

Le dossier contient un playbook Ansible qui rejoue les vérifications réseau en lecture seule contre l'imprimante (ports ouverts, accès à l'EWS, réponse SNMP). Voir `audit-imprimante.yml`.

Prérequis : Ansible installé sur le poste de contrôle, et l'adresse IP de l'imprimante (visible dans le routeur ou sur un rapport de configuration réseau imprimé depuis l'écran de l'imprimante).

Lancement :

```
ansible-playbook -i inventory.ini audit-imprimante.yml
```

Le playbook ne modifie rien sur l'imprimante. Il se contente de tester les ports et de signaler ce qui est exposé.
