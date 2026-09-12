# deploy

Dépôt "app-of-apps" ArgoCD pour les déploiements Open eIDAS — actuellement
uniquement l'environnement de staging public
(`api.staging.open-eidas.eu` / `pki.staging.open-eidas.eu` /
`ocsp.staging.open-eidas.eu`).

Le code applicatif et le chart Helm restent dans
[open-eidas/open-eidas](https://github.com/open-eidas/open-eidas). Ce dépôt
contient les manifestes ArgoCD `Application`, ainsi que les quelques
ressources gérées hors de ce chart (Secret scellé, Cluster CloudNativePG).

## Prérequis sur le cluster cible

Non gérés par ce dépôt, doivent déjà être en place :

- [sealed-secrets](https://github.com/bitnami-labs/sealed-secrets) (le
  contrôleur, pour déchiffrer les `SealedSecret` de `manifests/`) ;
- [CloudNativePG](https://cloudnative-pg.io/) (les CRD et l'opérateur) ;
- une Gateway API (`gateway.networking.k8s.io`) nommée `shared-gateway` dans
  le namespace `ingress`, avec un listener HTTPS par hôte de staging
  (`api`/`pki`/`ocsp.staging.open-eidas.eu`) et son
  `Certificate` cert-manager — ressource partagée, gérée ailleurs.

Voir [docs/STAGING.md](https://github.com/open-eidas/open-eidas/blob/main/docs/STAGING.md)
dans le dépôt principal pour le guide pas-à-pas complet.

## Structure

- `root-app.yaml` — Application racine à appliquer manuellement une fois sur
  le cluster (`kubectl apply -f root-app.yaml`). Elle synchronise ensuite le
  contenu de `apps/`.
- `apps/` — une Application ArgoCD par environnement/composant, gérée
  automatiquement par la racine :
  - `open-eidas-staging-gateway.yaml` — trois listeners HTTP dédiés
    (un par domaine de staging) ajoutés au Gateway partagé `shared-gateway`
    (namespace `ingress`), possédé par une autre Application ArgoCD hors de
    ce dépôt. Synchronise avec Server-Side Apply, sans prune (voir le
    commentaire du fichier) : c'est le seul moyen de cohabiter proprement
    avec l'autre gestionnaire de cette ressource partagée. HTTP seul pour
    l'instant — le TLS par domaine est une étape ultérieure distincte.
  - `open-eidas-staging-postgres.yaml` — Secret scellé et Cluster
    CloudNativePG du staging (sync-wave `-1`, avant le chart applicatif).
  - `open-eidas-staging.yaml` — le chart open-eidas lui-même, en
    environnement de staging public.
- `manifests/staging-postgres/` — ressources brutes de l'Application
  `open-eidas-staging-postgres` :
  - `sealed-secret.yaml` — `SealedSecret` "open-eidas-generated" : mot de
    passe PostgreSQL, PIN des tokens PKCS#11, secret HMAC d'enrôlement.
    Chiffré contre la clé publique du contrôleur sealed-secrets du cluster
    de staging — illisible sans la clé privée correspondante, sûr à
    committer.
  - `cluster.yaml` — Cluster CloudNativePG amorcé avec les identifiants du
    Secret ci-dessus.

## Régénérer le secret scellé

Si les valeurs sensibles doivent être renouvelées (rotation, compromission
suspectée), régénérer entièrement `sealed-secret.yaml` — un `SealedSecret`
scellé pour une clé de contrôleur donnée ne se "met pas à jour" par simple
édition :

```bash
kubectl create secret generic open-eidas-generated \
    --namespace open-eidas-staging --dry-run=client -o yaml \
    --from-literal=postgres-password=... \
    --from-literal=username=openeidas \
    --from-literal=password=...  \
    --from-literal=tsa-pin=... --from-literal=ocsp-pin=... \
    --from-literal=ca-root-pin=... --from-literal=ca-issuing-pin=... \
    --from-literal=webdav-password=... --from-literal=enroll-hmac-key=... \
    > /tmp/open-eidas-generated.yaml
kubeseal --controller-namespace sealed-secrets -f /tmp/open-eidas-generated.yaml \
    -w manifests/staging-postgres/sealed-secret.yaml
shred -u /tmp/open-eidas-generated.yaml
```

`password` doit toujours reprendre la même valeur que `postgres-password` :
c'est ce que `cluster.yaml` utilise pour amorcer l'utilisateur applicatif
PostgreSQL.

## Ajouter un nouvel environnement

Ajouter un fichier dans `apps/`, le fusionner sur `main` : ArgoCD crée
l'Application correspondante à la prochaine synchronisation, sans action
manuelle sur le cluster.
