# deploy

Dépôt "app-of-apps" ArgoCD pour les déploiements Open eIDAS — actuellement
uniquement l'environnement de staging public
(`staging-api.open-eidas.eu` / `staging-pki.open-eidas.eu` /
`staging-ocsp.open-eidas.eu`).

Le code applicatif et le chart Helm restent dans
[open-eidas/open-eidas](https://github.com/open-eidas/open-eidas). Ce dépôt
ne contient que des manifestes ArgoCD `Application`.

## Structure

- `root-app.yaml` — Application racine à appliquer manuellement une fois sur
  le cluster (`kubectl apply -f root-app.yaml`). Elle synchronise ensuite le
  contenu de `apps/`.
- `apps/` — une Application ArgoCD par environnement/composant, gérée
  automatiquement par la racine. Actuellement :
  - `open-eidas-staging.yaml` — environnement de staging public.

## Ajouter un nouvel environnement

Ajouter un fichier dans `apps/`, le fusionner sur `main` : ArgoCD crée
l'Application correspondante à la prochaine synchronisation, sans action
manuelle sur le cluster.
