# Architecture Decision Record (ADR)

Le format des [ADR](https://adr.github.io/) prend en compte à la fois la date de prise de décision, du statut de la décision, les différentes options envisagées voire dans certains cas les conséquences de la décision par rapport au contexte du produit numérique construit.

### Pourquoi ?

- Une décision d'architecture est un choix de conception d'un logiciel qui répond à une exigence fonctionnelle ou non fonctionnelle importante sur le plan architectural. 

- Un enregistrement de décision d'architecture  capture une seule décision architecturale. L'ensemble des ADR créés et conservés dans le cadre d'un projet constitue son journal des décisions. 

**Une ADR est immuable** : seul son statut peut changer (c'est-à-dire qu'elle peut devenir obsolète ou remplacée). Ainsi, l'on dispose de l'historique complet du projet en lisant simplement son journal de décisions dans l'ordre chronologique. 


### Le statut d'une ADR peut avoir les états suivants :

```mermaid
    flowchart LR;
        A[Brouillon] --> B[Proposée];
        B[Proposée] --> C[Rejetée];
        B[Proposée] --> D[Acceptée];
        D[Acceptée] --> E[Obsolète];
        D[Acceptée] --> F[Remplacée];
```

Voici un exemple de modèle d'ADR: le [lien](./adr-template.md)

Ce repo se propose de regrouper des propositions d'architecture sur différents patrons rencontrés dans le cadre de Cloud Pi Native, par exemple :
 - La gestion des [architectures asynchrones](./architecture_asynchrone/adr-asynchrone.md)
 - Les API managers dans les architectures projets
 - PostgreSQL en mode Cloud Native

Ce repo est en cours de mise en place et s'enrichiera au fur et à mesure.
