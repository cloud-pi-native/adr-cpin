# Utilisation d'un Mesh sur CpIN

Cet AdR montre comment utiliser un service Mesh permettant de chiffrer les communication entre les nodes d'un cluster CPiN

## Statut

Proposé

## Date

|Date | Status | Commentaires |
|---|---|---|
| 2025-11-20 | Proposé | Première version |


## Participant

Les équipes CpiN suivante ont participé à cet AdR :
 - ServiceTeam
 - Exploitation

## Contexte et Problème

Les communications inter nodes des clusters Openshift sont par defaut non chiffrés. le but de cet AdR est de proposer une solution, la plus transparente pour les projets, permettant de chiffrer le contenu des communications entre les noeuds d'un cluster CPiN afin de se prémunir contre un attanquant qui arriverait à écouter le réseau.

De façon plus large, cette AdR présente comment chiffrer les flux sur l'ensemble de la chaine de communication d'un projet depuis le navigateur client jusqu'aux éléments applicatifs (POD)

Le schéma suivant présente les différentes étapes d'une requêtes dans CPiN

![schéma general](./img/istio.png)

le trajet d'une requête HTTP(S) peut se découper en plusieurs phase :

| Phase | Source | Destination |Chiffré | Explications |
|-------|-----|-----|----|--------------|
| 1 | Navigateur Client | CDS | Oui | La CDS est le point de terminaison SSL d'une requête HTTPS depuis l'extérieur. Ce flux est *forcément* chiffré via TLS |
| 2 | CDS | Ingress/Route | A la main du projet | Suivant la configration de la CDS et de la route Openshift il est possible de configurer la CDS pour renvoyer le flux en https vers les ingress |
| 3 | Ingres | Pods applicatifs | Non | 
| 4 | Pods | Pods | A la main du projet | En activant le mesh, le flux passe par un ztunnel et en mTLS |


## Options Considérées

Lister les différentes options analysées avant d'arrêter la décision. 
Chaque alternative peut inclure :
- Une description
- Les avantages
- Les inconvénients
- Les raisons pour lesquelles elle a été rejetée (le cas échéant)

TODO

### Istio

TODO mise en place sur le cluster

### Configuration pour les projets

TODO mise en place par les projets


## Décision

Confirmer le mode d'installation avec exploitation : est ce que l'opérateur est systématiquement mis en place ou à la demande ?

Les projets ont la main sur le chiffrement des communications entre leurs PODs.

> Cependant il reste un point de vulnérabilité sur le chiffrement des flux entre les ingress et le premier pod.


## Liens et Références

Fournit des références vers des documents complémentaires, discussions, tickets de suivi, ou toute autre ressource pertinente liée à cette décision.
