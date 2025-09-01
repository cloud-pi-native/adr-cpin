# Titre

Cet ADR traite de la **Gateway** API de Kubernetes en complément ou remplacement de l'API **Ingress**, de leurs différences et des cas d'usages dans le cadre de CPiN

## Statut

> Brouillon 

## Date

| STATUT    | Date       |
|-----------|------------|
| Brouillon | 07/08/2025 |
| Proposée  |            |
| Acceptée  |            |
| Rejetée   |            |
| Obsolète  |            |
| Remplacée |            |

## Participant

 - ServiceTeam

## Contexte et Problème

La [Gateway API](https://kubernetes.io/docs/concepts/services-networking/gateway/) de kubernetes est le successeur de l'API Ingress et permet de gérer les règles d'exposition d'une application en dehors d'un cluster Kubernetes / OpenShift, généralement pour l'exposition HTTPS d'une application.

L'API historique de Kubernetes pour faire cette opération est l'API Ingress. Une nouvelle API nommée Gateway permet d'aller plus loin la configuration. Cet ADR propose de décrire l'utilisation de l'API Gateway dans le cadre de l'offre CPiN. 

### Ingress

**Ingress** est l'API historique de Kubernetes pour gérer le trafic entrant HTTP/HTTPS :

- Configuration simple pour des cas basiques
- Limité aux protocoles HTTP/HTTPS
- Implementation spécifique à chaque contrôleur Ingress
- Pas de séparation claire entre les rôles (admin infrastructure vs développeurs)
- Configuration monolithique

Le schéma suivant présente une vue globale de l'utilisation d'un Ingress :

![Ingress](./img/ingress-basic-example.svg)

> L'API Ingress est une feature stable depuis la version 1.19 de Kubernetes mais que cette API est maintenant à un état *gelée* et ne prend donc plus de nouvelles fonctionnalités.

### Gateway


**Gateway** est l'API moderne qui succède à **Ingress** avec des capacités étendues :

- Support multi-protocoles (HTTP, TCP, UDP, gRPC, etc.)
- Architecture modulaire avec séparation des responsabilités :
  - `Gateway` : Configuration infrastructure (ports, TLS, etc.)
  - `HTTPRoute`/`TCPRoute`/etc. : Règles de routage applicatives
- Standardisation des ressources entre différentes implémentations
- Sécurité améliorée avec isolation des namespaces
- Validation native des configurations
- Support natif du cross-namespace routing
- Possibilité de déléguer des configurations aux équipes applicatives

> La famille d'API Gateway est une extension des API Kubernetes en version **gateway.networking.k8s.io/v1** 

Le schéma suivant présente une vue globale de l'utilisation de l'Gateway API :

![Ingress](./img/ingress-basic-example.svg)


## Options Considérées

### Rappel du contexte

L'API Ingress continue d'être supportée par Kubernetes et il est possible à date de continuer à les utiliser. Il n'existe pas d'obligation à date de migrer les Ingress vers des Gateway. Cependant, l'API Gateway est plus riche que l'API Ingress et il est donc intéressant que les projets commencent à les utiliser notamment pour les fonctionnalités avancées et non disponibles en standard avec les ingress comme la sécuriation par API-Key, le throttling (limitation du débit API), la séparation de la configuration entre gateway et route. La gestion des protocoles autres que HTTP(S) est considérée hors scope, en effet, l'exposition par CDS impose l'utilisation du protocole HTTPS.

> A date, seuls les environnements PAX sont configurés pour utiliser la Gateway API. Cette fonctionnalité sera implémentée sur CPiN prochainement.

### Mise en oeuvre des Gateway

La Gateway API est composées de 2 grandes parties :

 1. Le kind Gateway qui globalement correspond à l'ingressController et l'implémentation technique sous jacente (nginx, envoy, haproxy, etc.). Dans le contexte CPiN, ce composant n'est pas à la main des projets mais est provisionné par CPiN

Voici un exemple simple d'utilisation de la Gateway API

```yaml
# 1. Définition de la Gateway
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: default-gateway
  namespace: infra-envoy-gateway-system
spec:
  gatewayClassName: envoy # Nom de la classe d'implémentation de la Gateway
  listeners:
  - allowedRoutes:
      namespaces:
        from: All
    hostname: '*.formation-app-gateway.cpin.numerique-interieur.com'
    name: http
    port: 80
    protocol: HTTP
  - allowedRoutes:
      namespaces:
        from: All
    hostname: '*.formation-app-gateway.cpin.numerique-interieur.com'
    name: https
    port: 443
    protocol: HTTPS
    tls:
      certificateRefs:
      - group: ""
        kind: Secret
        name: global-tls-cert
      mode: Terminate
```
Cette Gateway écoute HTTP (80) et HTTPS (443) uniquement pour les hostnames compatible avec le nom : *.formation-app-gateway.cpin.numerique-interieur.com

L'implémentation mise en oeuvre sur PAX est envoy, voir la [documentation](https://gateway.envoyproxy.io/docs/api/gateway_api/) 

 2. Le kind HTTPRoute (et TCPRoute / UDPRoute mais non utilisé dans le cadre CPiN). Cet object est à la charge des projets.

Exemple d'une HTTPRoute
```yaml
# 2. Définition de la Route HTTP
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: my-route
  namespace: my-application
spec:
  hostnames: # Nom DNS de l'application
  - my-app.formation-app-gateway.cpin.numerique-interieur.com
  parentRefs: # Référence à la gateway
  - group: gateway.networking.k8s.io
    kind: Gateway
    name: default-gateway
    namespace: infra-envoy-gateway-system
  rules:
    - matches: # Règle de redirection
        - path:
            type: PathPrefix
            value: /api
      backendRefs: # Référence vers le service backend à utiliser
        - name: mon-api-service
          port: 8080
```

La définition ci-dessus de la route est assez proche d'un Ingress. L'exemple suivant présente les fonctionnalités propres à HTTPRoute notamment l'ajout de rate limit :

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: my-route
  namespace: my-application
spec:
  hostnames:
  - my-app.formation-app-gateway.cpin.numerique-interieur.com
  parentRefs:
  - name: default-gateway
    namespace: default
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /api
    filters:
    # Configuration du throttling
    - type: RateLimit
      rateLimit:
        type: Server
        config:
          # Limite à 100 requêtes par minute
          rate: "100"
          per: "1m"
          burst: "20"  # Autorise des pics courts jusqu'à 20 requêtes supplémentaires
    backendRefs:
    - name: mon-api-service
      port: 8080
```

L'exemple suivant ajoute en plus une authentification via API-KEY :

```yaml
apiVersion: v1
kind: Secret
type: Opaque
metadata:
  name: apikey-secret
stringData:
  client1: supersecret
---
apiVersion: gateway.envoyproxy.io/v1alpha1
kind: SecurityPolicy
metadata:
  name: apikey-auth-example
spec:
  targetRefs:
    - group: gateway.networking.k8s.io
      kind: HTTPRoute
      name: my-route
  apiKeyAuth:
    credentialRefs:
    - group: ""
      kind: Secret
      name: apikey-secret
    extractFrom:
    - headers:
      - x-api-key
```

La Gateway API permet également l'implémentation de plusieurs éléments de sécurité : basic authentification, CORS, IP whitelist, JWT, MTLS, OIDC et de gestion de traffic : circuit breakers, client traffic policy, failover, etc.

Voir la documentation officielle sur la [sécurité](https://gateway.envoyproxy.io/docs/tasks/security/) et la [gestion de traffic](https://gateway.envoyproxy.io/docs/tasks/traffic/)

## Décision

A instruire.

## Conséquences

Il est conseillé d'utiliser la Gateway API sur CPiN (pour l'instant uniquement sur PAX) pour l'exposition d'API nécessitant un degrès fin de configuration, notamment la limitation de débit (rate limit) et l'authentification. 


## Liens et Références

Liste des liens de référence :
 - [API Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/)
 - [API Gateway](https://kubernetes.io/docs/concepts/services-networking/gateway/)
 - [Migration Ingress vers Gatewayy](https://gateway-api.sigs.k8s.io/guides/migrating-from-ingress/)
 - [Envoy](https://gateway.envoyproxy.io/docs/api/gateway_api/)
  - [Tuto d'exemples CPiN](https://github.com/cloud-pi-native/tuto-java-infra-helm/tree/feat/gateway)