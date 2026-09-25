# deploy

Dépôt "app-of-apps" ArgoCD pour les déploiements Open eIDAS — actuellement
uniquement l'environnement de staging public
(`api.staging.open-eidas.eu` / `pki.staging.open-eidas.eu` /
`ocsp.staging.open-eidas.eu`).

Le code applicatif et le chart Helm restent dans
[otspi/open-eidas](https://github.com/otspi/open-eidas). Ce dépôt
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

Voir [docs/STAGING.md](https://github.com/otspi/open-eidas/blob/main/docs/STAGING.md)
dans le dépôt principal pour le guide pas-à-pas complet.

## Structure

- `root-app.yaml` — Application racine à appliquer manuellement une fois sur
  le cluster (`kubectl apply -f root-app.yaml`). Elle synchronise ensuite le
  contenu de `apps/`.
- `apps/` — une Application ArgoCD par environnement/composant, gérée
  automatiquement par la racine :
  - `open-eidas-staging-postgres.yaml` — Secret scellé et Cluster
    CloudNativePG du staging (sync-wave `-1`, avant le chart applicatif).
  - `open-eidas-staging.yaml` — le chart open-eidas lui-même, en
    environnement de staging public. Les tags d'image (`ca`/`tsa`/`ocsp`)
    sont épinglés en valeurs Helm inline sur un SHA de commit précis de
    `otspi/open-eidas`, mis à jour automatiquement par sa CI (voir
    « Épinglage des images » ci-dessous) — jamais sur `latest` :
    `imagePullPolicy: IfNotPresent` ne se re-pull jamais tout seul sur un
    tag flottant, et le self-heal ArgoCD annulerait toute correction faite
    directement sur le cluster.

Le listener HTTPS partagé `shared-gateway` (namespace `ingress`) pour les
domaines de staging est géré manuellement, hors ArgoCD, en dehors de ce
dépôt.
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

## Épinglage des images de staging

`apps/open-eidas-staging.yaml` embarque un tag d'image précis
(`ca.image.tag`/`tsa.image.tag`/`ocsp.image.tag`, en valeurs Helm inline)
plutôt que `latest`. Après chaque publication réussie sur `dev` dans
[otspi/open-eidas](https://github.com/otspi/open-eidas), son job
CI `pin-staging` committe ici le nouveau SHA — ce dépôt reste la seule
source de vérité pour ArgoCD, qui applique le changement via son
`selfHeal` déjà configuré. Cette CI n'a et n'aura jamais accès ni au
cluster ni à ArgoCD directement : seulement à ce dépôt git, via une
GitHub App dédiée.

**Mise en place de la GitHub App** (à faire une fois, manuellement) :

1. Organisation `open-eidas` → **Settings → Developer settings → GitHub
   Apps → New GitHub App**.
2. Nom libre (ex. `open-eidas-deploy-bot`), pas de webhook actif.
3. Permissions : **Repository permissions → Contents: Read and write**
   uniquement (rien d'autre — surtout pas d'accès à un quelconque secret
   ou déploiement).
4. Une fois créée : **Generate a private key** (télécharge un `.pem`), et
   noter l'**App ID**.
5. **Install App** → sélectionner uniquement le dépôt `open-eidas/deploy`.
6. Dans les secrets du dépôt `otspi/open-eidas` (Settings → Secrets
   and variables → Actions) : `OPENEIDAS_DEPLOY_APP_ID` (l'App ID) et
   `OPENEIDAS_DEPLOY_APP_PRIVATE_KEY` (contenu du `.pem`).

Tant que ces secrets n'existent pas, le job `pin-staging` s'exécute mais
ne fait rien (avertissement, pas d'échec) — le pin reste alors manuel
(éditer `apps/open-eidas-staging.yaml`, voir son commentaire).

## Ajouter un nouvel environnement

Ajouter un fichier dans `apps/`, le fusionner sur `main` : ArgoCD crée
l'Application correspondante à la prochaine synchronisation, sans action
manuelle sur le cluster.

## Licence

Ce dépôt est publié sous la licence publique de l'Union européenne (EUPL) v1.2 — voir [LICENSE](LICENSE).
