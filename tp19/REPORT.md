# TP19 : GitOps avec Argo CD

**Étudiante :** Meryem El Heni, Master DevOps & Cloud

## 1. Objectif
Déployer l'application `whoami` avec Argo CD selon le principe GitOps : Git est la source de vérité, Argo CD réconcilie le cluster K3s.

## 2. Environnement
- Cluster K3s sur VM Ubuntu (3,3 Go de RAM)
- Argo CD v3.6.3, Kustomize
- Dépôt : https://github.com/meryemelheni/partie-7-tp19-20 (dossier `tp19/`)
- Structure : `base/`, `envs/dev|staging|prod`, `argocd/`

## 3. Réalisation

### 3.1 Application dev (synchronisation automatique)
Application `whoami-dev` avec `automated: prune + selfHeal`. Résultat : Synced / Healthy.
Capture : `captures/argocd-synced.png`

### 3.2 Self-healing
`kubectl scale deploy whoami -n whoami-dev --replicas=5` : Argo CD a remis 1 replica (valeur de Git).
Capture : `captures/self-heal.png`

### 3.3 Rollback par Git
Push du tag inexistant `v9.9.9` : Pod en `ImagePullBackOff`, application non saine. Puis `git revert HEAD && git push` : retour à Healthy.
Captures : `captures/git-revert.png`, `captures/git-revert-ok.png`

### 3.4 ApplicationSet
Un générateur Git sur `tp19/envs/*` crée `whoami-dev`, `whoami-staging`, `whoami-prod`. Dev et staging sont en sync automatique, prod en manuel (via `templatePatch`).
Capture : `captures/applicationset.png`

### 3.5 Promotion sans reconstruction
PR #1 : staging `v1.10.0` → `v1.10.1`. PR #2 : prod `v1.10.0` → `v1.10.1`. La prod a été synchronisée manuellement.
Captures : `captures/promotion-pr.png`, `captures/same-digest.png`
Les trois environnements ont le même digest `sha256:6956a30c...2fe9` : l'image n'est pas reconstruite, c'est la même qui est promue.

### 3.6 Protection de `main`
Ruleset : Pull Request obligatoire, 1 approbation. La revue de la PR devient la revue du changement en production : chaque modification est relue, tracée et réversible par `git revert`.

## 4. Difficultés rencontrées
- **Sync Unknown** : l'API server K3s était saturé (RAM, swap). `kubectl get crd` prenait 12 s. Après avoir arrêté Keycloak, trivy-operator et Falco, il est passé à 0,4 s, puis le contrôleur Argo CD a été redémarré.
- **CRD ApplicationSet absent** : réinstallation des manifests Argo CD avec `--server-side --force-conflicts`.
- **Erreur YAML** : `directories` doit être une liste (`- path:`).
- **Namespace prod manquant** : sync en échec (`namespaces "whoami-prod" not found`) ; création manuelle du namespace.

## 5. Conclusion
Le GitOps rend les déploiements reproductibles, auditables et réversibles : toute dérive manuelle est annulée et tout changement passe par une Pull Request.
