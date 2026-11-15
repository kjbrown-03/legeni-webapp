# Projet final DevOps : chaîne CI/CD « du commit à la production »

Baptiste Kaldjob, B3. Dépôt : `kjbrown-03/legeni-webapp`. Image : `kaldjobbaptiste/legeni-webapp`.

## 1. Architecture

J'ai pris l'**option A** : une seule machine porte Jenkins et Minikube. C'est mon PC Windows 11, avec Docker Desktop et Ubuntu 24.04 sous WSL2. Ansible tourne dans Ubuntu (provisionnement) et dans le conteneur Jenkins (déploiements).

```
 développeur ──git push / PR──▶ GitHub ──webhook (smee.io)──▶ Jenkins (conteneur tp3-jenkins)
                                                                 │
              ┌──────────────── CI : toutes les branches et PR ──┤
              │  contrôles qualité en parallèle                  │
              │  (pytest + couverture, flake8, hadolint, syntaxe Ansible)
              │  → build image <VERSION>-<build> → scan Trivy     │
              └──────────────────────────────────────────────────┤
              ┌──────────────── CD : branche main uniquement ────┤
              │  push Docker Hub → ansible deploy.yml staging    │
              │  → validation humaine → ansible deploy.yml prod   │
              │  → tag Git v<VERSION>                             │
              └──────────────────────────────────────────────────┘
                     │ kubernetes.core (API https://192.168.49.2:8443, réseau Docker "minikube")
                     ▼
        Minikube : namespace legeni-staging (2 réplicas, orange, NodePort 30081)
                   namespace legeni-prod    (3 réplicas, vert,   NodePort 30082)
                   Redis en StatefulSet + PVC dans chaque namespace
```

Structure du dépôt :

| Chemin | Rôle |
|---|---|
| `app.py`, `tests/`, `requirements*.txt` | l'application Flask et ses tests |
| `Dockerfile`, `.dockerignore` | image multi-stage : `test` (flake8 + pytest), puis image finale non-root avec HEALTHCHECK |
| `VERSION` | version sémantique, lue par Jenkins |
| `Jenkinsfile` | le pipeline CI/CD |
| `ansible/provision.yml` + `roles/` | infrastructure : rôles `docker`, `minikube`, `jenkins` |
| `ansible/deploy.yml` + `templates/k8s/webapp.yml.j2` | déploiement Kubernetes avec smoke test et rollback |
| `ansible/vars/{staging,prod}.yml` | ce qui change entre les deux environnements |
| `ansible/vars/vault.yml` | mots de passe Redis, **chiffrés** avec Ansible Vault |

## 2. Choix techniques et pourquoi

- **Une image, un tag traçable.** L'image est construite une seule fois par build, taguée `<VERSION>-<numéro de build>` (par exemple `1.1.0-4`), et c'est cette image qui va en staging puis en production. `deploy.yml` refuse `latest` par un `assert`.
- **Manifests en Jinja2 générés par Ansible** (repris du TP 5.6) plutôt qu'en Kustomize. Le même template sert aux deux environnements, et une annotation `checksum` force un rolling update quand la configuration ou le mot de passe Redis changent.
- **Zéro coupure.** `RollingUpdate` avec `maxUnavailable: 0`, une `readinessProbe` sur `/ready` et un `preStop: sleep 5`. Au TP 4.8, j'ai mesuré qu'**avec seulement `maxUnavailable: 0`, il y avait encore 13 à 16 erreurs** pendant la mise à jour, et **0 erreur** une fois la readiness probe et le `preStop` ajoutés.
- **`startupProbe`** : au TP 4.12, des pods lents au démarrage étaient tués par la liveness. La startup probe leur laisse 60 s.
- **`enableServiceLinks: false`** : au TP 4.6, le Service `redis` injectait `REDIS_PORT=tcp://…` dans les pods, ce qui faisait planter l'application.
- **Rollback automatique** : `deploy.yml` attend la fin du rolling update (`wait: true`), lance un smoke test **dans** le cluster qui vérifie la version servie, et en cas d'échec exécute `k8s_rollback` dans un bloc `rescue`, puis échoue volontairement pour que Jenkins passe au rouge.
- **Secrets** : Docker Hub, kubeconfig et mot de passe Vault sont des credentials Jenkins (`registry-credentials`, `kubeconfig`, `ansible-vault-pass`). Les mots de passe Redis sont dans `vault.yml` chiffré. Le Jenkinsfile ne contient aucun secret.
- **Webhook par smee.io** : GitHub ne peut pas joindre `localhost:8080`. Le relais smee tourne dans un conteneur (`tp3.6-smee`).

## 3. Le pipeline, étape par étape

| Stage | Branches | Ce qu'il fait | Ce qui l'arrête |
|---|---|---|---|
| Contrôles qualité (parallèle) | toutes | pytest avec rapport JUnit et couverture ≥ 80 %, flake8, hadolint, `ansible-playbook --syntax-check` | un test rouge, couverture < 80 %, une erreur de lint |
| Build image | toutes | `docker build --build-arg APP_VERSION=<tag>` ; l'étape `test` du Dockerfile relance les tests | échec de build |
| Scan Trivy | toutes | vulnérabilités CRITICAL corrigeables | `--exit-code 1` dès la première |
| Push registre | main | `docker push kaldjobbaptiste/legeni-webapp:<tag>` | |
| Déploiement staging | main | `ansible-playbook deploy.yml -e target_env=staging` | smoke test KO → rollback, puis build rouge |
| Validation production | main | `input` : une personne clique sur « Déployer » | refus ou délai de 30 min dépassé |
| Déploiement production | main | idem, en prod | smoke test KO → rollback |
| Tag Git | main | `git tag v<VERSION>` poussé avec le credential `github` | |
| post failure | toutes | e-mail « Échec … » (serveur SMTP local Mailpit) | |

Les branches `feature/*` et les pull requests s'arrêtent après le scan Trivy : elles ne poussent ni ne déploient jamais. `main` est protégée : merge uniquement par PR, avec le check Jenkins `continuous-integration/jenkins/pr-merge` obligatoire.

## 4. Infrastructure as Code (`provision.yml`)

- **`docker`** : détecte un Docker déjà opérationnel (Docker Desktop avec l'intégration WSL). Sinon, il l'installe depuis le dépôt officiel, ou depuis des binaires locaux (variante écrite au TP 5.4 pour ma connexion lente).
- **`minikube`** : si le cluster ne répond pas, installe kubectl et minikube, puis `minikube start --driver=docker`. Il active ensuite les addons `metrics-server` et `ingress` s'ils manquent.
- **`jenkins`** : construit l'image du TP 3.1 si elle n'existe pas, lit le groupe du socket Docker, génère le `docker-compose.yml` et démarre Jenkins relié au réseau `minikube`, avec un volume de données externe conservé.

Sur ma machine, le 1er passage a donné `changed=4` : il a repris en main le Jenkins existant (même volume, jobs conservés) et l'a relié au réseau du cluster. Le 2e passage donne **`changed=0`**.

## 5. Résultats

### Jalons

| Jalon | Preuve |
|---|---|
| S1 : `provision.yml` + image publiée | 1er passage `changed=4`, 2e passage `changed=0` ; `kaldjobbaptiste/legeni-webapp:1.0.0-7` puis `1.0.0-8` sur Docker Hub |
| S2 : `deploy.yml` à la main + CI verte sur une PR | `deploy.yml -e target_env=staging -e app_version=2.0.0` : `staging OK`, 2e passage `changed=0` ; `deploy.yml -e app_version=latest` refusé par l'`assert` ; PR #10 : CI verte, stages de déploiement *skipped* |
| S3 : scénario complet | les 3 scénarios ci-dessous |

### Scénario 1 : livraison nominale (PR #11, version 1.1.0)

Nouvelle couleur par défaut (`#0969da`) et `VERSION` à `1.1.0`.

- **PR #11** : contrôles qualité, build et Trivy verts, aucun déploiement.
- **Merge, puis build `main` #10** : image `1.1.0-10` poussée, staging `staging OK`, validation **Approved by Baptiste Kaldjob**, production `prod OK`, tag Git `v1.1.0` créé (`[new tag] v1.1.0 -> v1.1.0`).
- **Boucle `curl` sur la production** pendant la mise à jour (une requête toutes les 0,5 s, depuis le nœud) : 59 réponses en `1.0.0-8`, puis 74 en `1.1.0-10`, **0 erreur**.

Même mesure lors de la première mise en production (`main` #8, 2.0.0 → 1.0.0-8) : 108 + 74 réponses, 0 erreur.

### Scénario 2 : code défectueux (PR #12)

Un test faux est ajouté (`assert 1 + 1 == 3`).

- Le build est rouge à **Tests + couverture** : `FAILED tests/test_app.py::test_scenario_2_soutenance`, rapport JUnit avec 1 échec et 6 réussites.
- Build image, Trivy, push et déploiements sont tous *skipped* : **aucune image produite**.
- La production reste en `1.1.0-10`.
- GitHub refuse le merge (check `pr-merge` obligatoire).
- L'e-mail « Échec legeni-webapp-mb/PR-12 #1 » arrive dans la boîte locale.

### Scénario 3 : déploiement raté (PR #13)

La `readinessProbe` pointe sur `/pas-pret`, un chemin qui n'existe pas. Le code étant correct, la CI est verte et la PR est mergée.

- Build `main` #11 : image `1.1.0-11` poussée. En staging, les nouveaux pods ne deviennent jamais prêts, et `"Deployment" "webapp": Timed out waiting on resource`.
- Le `rescue` exécute « Retour à la version précédente », puis `Déploiement de 1.1.0-11 en staging échoué : rollback effectué`. Récapitulatif : `failed=1 rescued=1`.
- Validation et production *skipped*. `kubectl get deploy webapp -n legeni-staging` montre l'image `1.1.0-10`, en 2/2. La production n'a jamais été touchée et répond en `1.1.0-10`.
- La correction (retour à `/ready`) repasse par une PR, avec ce rapport.

## 6. Difficultés rencontrées et solutions

| Difficulté | Solution |
|---|---|
| Connexion de 2 à 400 Ko/s, coupures fréquentes (pip, Docker Hub, GitHub) | tests et lint exécutés dans l'image de test du Dockerfile (dépendances en cache Docker, plus de PyPI à chaque build) ; délai d'attente du rolling update porté à 600 s |
| Premier déploiement staging en échec : l'image mettait 2 min 27 s à arriver dans Minikube (couches de base absentes) | le rollback a protégé l'environnement ; l'image est ensuite en cache, et les versions suivantes ne changent que la configuration de l'image, qu'elles tirent en quelques secondes |
| Docker Desktop redémarré par manque de mémoire pendant un build (14 Go de RAM pour Jenkins, Minikube et les builds) | Jenkins limité à 1 exécuteur (builds en file au lieu d'en parallèle) |
| Smoke test lancé sur un ancien pod en cours d'arrêt (404) qui déclenchait un rollback à tort | sélection d'un pod de la nouvelle image, non marqué pour suppression |
| Changement du mot de passe Redis non pris en compte (Redis ne relit le Secret qu'au démarrage) | annotation `checksum/secret` sur Redis et sur l'application |
| `post { failure }` en erreur quand l'échec arrive avant le calcul de `IMAGE_TAG` | `env.IMAGE_TAG ?: '…'` |
| GitHub ne peut pas joindre Jenkins (`localhost`) | webhook relayé par smee.io (conteneur `tp3.6-smee`) |

## 7. Pistes d'amélioration

- **Registre et cache plus près du cluster** : un registre local ou un miroir Docker Hub éviterait le premier tirage lent.
- **Ingress avec TLS** (`staging.legeni.local` / `legeni.local`), déjà réalisé au TP 4.11, à ajouter dans les manifests du projet (bonus).
- **HPA en production** (TP 4.8) et **Job de migration** lancé avant le rolling update (TP 4.6), deux bonus du sujet.
- **GitOps avec Argo CD** : Jenkins ne ferait plus que modifier le tag dans Git, et le cluster se synchroniserait seul.
- **Machine dédiée** : sur une vraie VM Ubuntu, `provision.yml` installerait Docker, Minikube et Jenkins depuis zéro. Sur mon PC, Docker est fourni par Docker Desktop et le rôle le détecte.
