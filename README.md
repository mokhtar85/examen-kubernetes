# Examen Kubernetes - Instructions

Cet examen évalue votre maîtrise des concepts fondamentaux de Kubernetes en déployant une application fullstack (backend FastAPI + frontend React + base de données) de manière progressive et professionnelle.

## Compétences évaluées

Cet examen évalue les compétences suivantes :
- Déploiement d'une application multi-services sur Kubernetes
- Gestion de la configuration et des secrets de manière sécurisée
- Exposition d'une application via Ingress avec HTTPS
- Observation et monitoring d'une application avec Prometheus et Grafana
- Logging à travers la stack OpenSearch, Fluentbit et OpeanSearch Dashboards

## Prérequis

- Cluster Kubernetes fonctionnel (Minikube recommandé)
- `kubectl` installé et configuré
- `helm` installé
- Docker installé pour builder les images
- **Chargement des images dans Minikube** : 
  - Si vous utilisez Minikube, vous pouvez charger les images directement dans le cluster avec `minikube image load <image-name>` ou sinon les pousser vers un registry Docker
  - Exemple : `minikube image load exam-kubernetes-backend:latest`
  - Alternative : Utiliser un registry Docker (Docker Hub, ou registry local avec Minikube)

### Optionnel (Point bonus)

**Déploiement via Terraform/OpenTofu** : Des points supplémentaires seront attribués si tout le déploiement (ou une partie significative) est réalisé via Terraform ou OpenTofu au lieu de manifests YAML bruts ou Helm. Vous devrez fournir le code Terraform/OpenTofu et expliquer votre approche.

## Préparation des images Docker

Avant de déployer l'application sur Kubernetes, vous devez :

1. **Builder les images Docker** du backend et du frontend
   - **Important pour le frontend** : Builder avec `VITE_API_BASE_URL=""` (vide) pour que les redirections fonctionnent sur Kubernetes :
     ```bash
     docker build -t exam-kubernetes-frontend:latest \
       --build-arg VITE_API_BASE_URL="" ./frontend
     ```
   - Pour le backend, builder normalement :
     ```bash
     docker build -t exam-kubernetes-backend:latest ./backend
     ```

2. **Charger les images dans votre cluster Kubernetes**
   
   **Option A : Avec Minikube (recommandé pour le TP)**
   - Utiliser `minikube image load` pour charger les images directement dans Minikube :
     ```bash
     minikube image load exam-kubernetes-backend:latest
     minikube image load exam-kubernetes-frontend:latest
     ```
   - Dans vos manifests, utiliser directement le nom de l'image (sans registry) :
     ```yaml
     image: exam-kubernetes-backend:latest
     image: exam-kubernetes-frontend:latest
     ```
   
   **Option B : Avec un registry Docker**
   - Pousser les images vers un registry Docker de votre choix (ex: Docker Hub) :
     ```bash
     docker tag exam-kubernetes-backend:latest <votre-registry>/exam-kubernetes-backend:latest
     docker push <votre-registry>/exam-kubernetes-backend:latest
     ```
   - Dans vos manifests, utiliser le nom complet avec le registry :
     ```yaml
     image: <votre-registry>/exam-kubernetes-backend:latest
     ```

3. **Utiliser ces images dans vos manifests** : Lors de la création de vos Deployments backend et frontend, vous devrez spécifier les noms d'images correspondant à la méthode choisie (Minikube ou registry Docker)

---

## PHASE 1 — Kubernetes Core (fondamentaux)

### Objectif
Rendre l'application fonctionnelle et stable

### 📝 Travaux demandés

1. **Créer un namespace dédié**
   - Créer un namespace **todolist** pour isoler l'application
   - Utiliser ce namespace pour tous les déploiements suivants

2. **Déployer les composants**
   - **Backend** : Créer un Deployment pour le backend FastAPI
   - **Frontend** : Créer un Deployment pour le frontend React
   - **Note** : Le backend affichera des erreurs de connexion à la base de données car celle-ci n'est pas encore déployée (étape 8). C'est normal et attendu. Le backend deviendra fonctionnel une fois la base de données déployée. 

3. **Exposer les services**
   - **Backend** : Exposer via Service ClusterIP
   - **Frontend** : Exposer via Service ClusterIP

4. **Ajouter les probes au backend**
   - `livenessProbe` sur `/health`
   - `readinessProbe` sur `/ready`

5. **Configurer les ressources**
   - Ajouter `resources.requests` et `resources.limits` pour tous les conteneurs :
     - **Backend** : requests (CPU: 100m, Memory: 128Mi), limits (CPU: 500m, Memory: 512Mi)
     - **Frontend** : requests (CPU: 50m, Memory: 64Mi), limits (CPU: 200m, Memory: 256Mi)

6. **Externaliser la configuration**
   - Créer une **ConfigMap** pour la configuration non sensible :
     - **Backend** : `APP_NAME`, `APP_VERSION`, `DB_HOST` (utiliser le FQDN du service : `<service-name>.<namespace>.svc.cluster.local`), `DB_PORT`, `DB_NAME`
     - **Frontend** : `VITE_API_BASE_URL` (vide), `BACKEND_URL` (URL du backend pour Nginx, ex: `http://backend:8000`)
   - Créer un **Secret** pour les informations sensibles :
     - **Backend** : `DB_USER`, `DB_PASSWORD`
   - **Optionnel (Point bonus)** : Utiliser des secrets chiffrés avec **Sealed Secrets** ou **SOPS** au lieu de secrets en clair dans les YAML. Les secrets doivent être chiffrés dans le code source et déchiffrés au déploiement. **Important** : Vous devrez me fournir la clé de déchiffrement pour permettre la validation et le déchiffrement des secrets.

7. **Utiliser un ServiceAccount dédié avec RBAC**
   - Créer un ServiceAccount spécifique à l'application
   - L'associer aux Deployments
   - Créer un **Role** permettant de créer des pods dans le namespace `todolist`
   - Créer un **RoleBinding** associant le Role au ServiceAccount
   - **Utilité** : Le backend expose un endpoint `/api/test-pod` qui crée un pod de test. Cela permet de vérifier que le ServiceAccount a les bonnes permissions RBAC.

8. **Déployer la base de données avec volume persistant**
   - **Optionnel (Point bonus)** : Utiliser le CRD CockroachDB au lieu de PostgreSQL (plus complexe mais distribué)
   - Créer un namespace **db** pour la base de données
   - Créer un **PersistentVolumeClaim (PVC)** pour la base de données dans le namespace `db`
   - Créer un **Secret** pour stocker les informations sensibles de la base de données :
     - Le nom de la base de données
     - L'utilisateur
     - Le mot de passe
     - **Notes** : Consultez la documentation officielle de l'image PostgreSQL ou CockraochDB pour connaître les noms exacts des variables d'environnement à utiliser et les valeurs par défaut
   - Déployer la base de données dans le namespace `db` avec le volume attaché et le Secret configuré
   - **Important** : Le backend dans le namespace `todolist` devra accéder à la base de données dans le namespace `db` via le FQDN du service : `<service-name>.<namespace>.svc.cluster.local` (ex: `postgres.db.svc.cluster.local`). De plus, regardez bien la documentation de postgres sur DockerHub pour voir quel volume monté.

9. **Initialiser la base de données**
   - Une fois la base de données déployée dans le namespace `db`, initialiser la table `items` :
     - **Note** : Le script `init_db.py` se trouve dans le dossier `resources/` du projet et n'est pas inclus dans l'image Docker. Vous devrez le rendre accessible au pod qui l'exécutera.
     ```bash
     # Via kubectl cp (copier le script dans le pod backend)
     kubectl cp resources/init_db.py <pod-backend>:/tmp/init_db.py -n todolist
     kubectl exec -it <pod-backend> -n todolist -- python3 /tmp/init_db.py
     ```

### ✅ Validation attendue

- [ ] Application accessible (frontend et backend fonctionnels)
- [ ] `/ready` retourne 503 si DB indisponible
- [ ] Aucun secret en clair dans les YAML (utilisation de Secret Kubernetes)
- [ ] Les pods démarrent correctement avec les probes configurées
- [ ] Les ressources sont limitées et les requests définies
- [ ] Le bouton "Créer un pod de test" dans le frontend fonctionne (teste les permissions RBAC du ServiceAccount)
- [ ] **Bonus** : Secrets chiffrés avec Sealed Secrets ou SOPS (si implémenté)
- [ ] **Bonus** : Utiliser le CRD CockroachDB au lieu de PostgreSQL (plus complexe mais distribué)

---

## PHASE 2 — Observabilité (Prometheus & Grafana)

### Objectif
Observer l'application comme un SRE

### 📝 Travaux demandés

1. **Installer kube-prometheus-stack via Helm**
   - Ajouter le repo Helm prometheus-community
   - Installer kube-prometheus-stack dans le namespace `monitoring`
   - **Vérifier le résultat de l'installation** : Vérifier que tous les pods sont en cours d'exécution avec `kubectl get pods -n monitoring`
   - **Récupérer le mot de passe Grafana** : Le mot de passe admin de Grafana est stocké dans un Secret Kubernetes. Pour le récupérer :
     ```bash
     kubectl get secret prometheus-grafana -n monitoring -o jsonpath="{.data.admin-password}" | base64 -d
     ```
     Notez ce mot de passe, vous en aurez besoin pour vous connecter à Grafana.
   - Accéder à Prometheus et Grafana via port-forward 

2. **Créer un ServiceMonitor afin de scraper le backend**
   
   **Note** : Notre backend FastAPI expose un endpoint `/metrics` qui retourne les métriques au format Prometheus (Counter, Histogram, Gauge). Vous pouvez vérifier le code pour vérifier.

   **Qu'est-ce qu'un ServiceMonitor ?**
   
   Un ServiceMonitor est une Custom Resource Definition (CRD) fournie par le Prometheus Operator. Il permet de déclarer à Prometheus comment scraper (collecter) les métriques d'un service Kubernetes, sans avoir à modifier la configuration de Prometheus manuellement.
   
   **Fonctionnement** :
   - Le ServiceMonitor décrit quel Service Kubernetes scraper
   - Il spécifie le port, le chemin (`/metrics`), et l'intervalle de scraping
   - Prometheus Operator découvre automatiquement les ServiceMonitors et configure Prometheus pour les scraper
   - Cela permet une gestion déclarative des targets Prometheus via des ressources Kubernetes natives
   
   **Ce qu'il faut faire** :
   - Créer un ServiceMonitor qui scrape l'endpoint `/metrics` de notre backend
   - Le ServiceMonitor doit sélectionner le Service backend via les labels
   - Spécifier le port `http` (le Service backend doit avoir un port nommé `http`)
   - Spécifier le chemin `/metrics` dans les endpoints
   - Utiliser un relabeling pour ajouter des labels personnalisés :
     - Ajouter le label `pod_name` à partir de `__meta_kubernetes_pod_name`
     - Ajouter le label `namespace` à partir de `__meta_kubernetes_namespace`
   - Ces labels permettront d'identifier plus facilement les métriques dans Prometheus et Grafana   

3. **Vérifier l'exposition des métriques**
   - Vérifier que le endpoint `/metrics` du backend est accessible
   - Vérifier que Prometheus scrape les métriques en cliquant sur "Status" puis "Target Health" et rendez-vous tout en bas afin de voir votre serviceMonitor
   ![Operator](images/service-monitor.png)

4. **Créer une PrometheusRule**
   - Utilisez ce manifest YAML
```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: backend-alerts
  namespace: todolist
  labels:
    app: backend
    # Ces labels sont importants pour que Prometheus découvre la PrometheusRule
    # Ils doivent correspondre aux labels du RuleSelector dans la config Prometheus
spec:
  groups:
  - name: backend.rules
    interval: 30s
    rules:
    # Alerte : Backend down (pas de métriques depuis 2 minutes)
    # Note: Le job name peut varier selon la configuration Prometheus
    # Vérifier le nom réel avec: up{namespace="todolist",service="backend"}
    - alert: BackendDown
      expr: |
        up{namespace="todolist",service="backend"} == 0
        or
        (absent(up{namespace="todolist",service="backend"}) == 1)
      for: 2m
      labels:
        severity: critical
        component: backend
      annotations:
        summary: "Backend is down"
        description: "Le backend n'expose plus de métriques depuis 2 minutes. Le service est probablement down."

    # Alerte : Taux d'erreurs 5xx élevé (> 10% sur 5 minutes)
    - alert: HighErrorRate
      expr: |
        (
          sum(rate(http_requests_total{namespace="todolist",status=~"5.."}[5m])) by (route)
          /
          sum(rate(http_requests_total{namespace="todolist"}[5m])) by (route)
        ) * 100 > 10
      for: 5m
      labels:
        severity: warning
        component: backend
      annotations:
        summary: "Taux d'erreurs 5xx élevé sur {{ $labels.route }}"
        description: "Le taux d'erreurs 5xx est de {{ $value | humanizePercentage }}% sur la route {{ $labels.route }} (seuil: 10%)"

    # Alerte : Latence p95 élevée (> 1 seconde)
    - alert: HighLatency
      expr: |
        histogram_quantile(0.95,
          sum(rate(http_request_duration_seconds_bucket{namespace="todolist"}[5m])) by (le, route)
        ) > 1
      for: 5m
      labels:
        severity: warning
        component: backend
      annotations:
        summary: "Latence p95 élevée sur {{ $labels.route }}"
        description: "La latence p95 est de {{ $value | humanize }}s sur la route {{ $labels.route }} (seuil: 1s)"

    # Alerte : Readiness probe échoue
    - alert: BackendNotReady
      expr: |
        up{namespace="todolist",service="backend"} == 1 
        and 
        rate(http_requests_total{namespace="todolist",route="/ready",status="503"}[1m]) > 0
      for: 2m
      labels:
        severity: warning
        component: backend
      annotations:
        summary: "Backend not ready"
        description: "Le backend retourne 503 sur /ready, la base de données est probablement inaccessible."

    # Alerte : Trop de requêtes en cours (> 100)
    - alert: HighInflightRequests
      expr: sum(http_inflight_requests{namespace="todolist"}) > 100
      for: 2m
      labels:
        severity: warning
        component: backend
      annotations:
        summary: "Trop de requêtes en cours"
        description: "Il y a {{ $value }} requêtes en cours de traitement (seuil: 100)"
```

- Rendez vous dans "Alerts" et vérifiez dans votre groupe backend les différentes alertes. Vous pouvez supprimer de déployement **backend** afin de voir que les alertes se déclenchent.

5. **Importer un dashboard Grafana**
   - Accéder à Grafana (port-forward : `kubectl port-forward svc/prometheus-grafana 3000 -n monitoring`)
   - Se connecter avec les identifiants admin (mot de passe récupéré à l'étape 1)
   - Aller dans **Dashboards → Import**
   - Importer le dashboard JSON fourni dans le projet (fichier `resources/dashboard.json` à la racine du projet)
   - Le dashboard contient les panneaux suivants :
     - **Total Requêtes (COUNT)** : nombre total de requêtes sur la période sélectionnée
     - **RPS par méthode HTTP** : taux de requêtes par seconde groupé par méthode (GET, POST, DELETE, etc.)
     - **Erreurs 500 dans le temps** : graphique temporel des erreurs 500 groupées par route et méthode
     - **Latence p95** : 95ème percentile de la latence par route
     - **Routes et méthodes HTTP** : tableau avec le nombre total de requêtes par route et méthode
     - **Nombre de requêtes dans le temps** : graphique temporel du nombre de requêtes groupées par méthode, route et code de retour
   - Le dashboard inclut également une variable **"Exclure /health et /ready"** (Oui/Non) permettant d'exclure ou d'inclure ces endpoints système des métriques
   - Vérifier que la datasource Prometheus est correctement configurée dans le dashboard
   - Vérifier que les panneaux affichent des données (générer du trafic si nécessaire)

![Grafana](images/grafana.png)

### ✅ Validation attendue

- [ ] Métriques visibles dans Prometheus (capture d'écran)
- [ ] ServiceMonitor fonctionnel et découvert par Prometheus (capture d'écran)
- [ ] Dashboard Grafana fonctionnel avec tous les panneaux demandés (capture d'écran)
- [ ] Au moins une PrometheusRule créée et visible (capture d'écran)

---

## PHASE 3 — Hashicorp Vault (mots de passe dynamiques)

### Objectif
Utiliser Vault pour générer des mots de passe dynamiques pour PostgreSQL/CockRoachDB au lieu d'utiliser des secrets statiques

### Contexte

Au lieu d'utiliser un Secret Kubernetes avec un mot de passe statique pour PostgreSQL, nous allons utiliser le **Database Secrets Engine** de Vault pour générer des credentials dynamiques à la demande. Cela permet :
- ✅ **Rotation automatique** des mots de passe
- ✅ **Création de credentials temporaires** (TTL)
- ✅ **Traçabilité** via les audit logs de Vault
- ✅ **Sécurité renforcée** : chaque pod peut avoir son propre credential

### Bonus

Vous pouvez utilisez le provider Vault sur OpenTofu/Terraform une fois celui-ci déployer afin de créer les secrets engine.

### 📝 Travaux demandés

1. **Installer Hashicorp Vault dans Kubernetes**
   - Installer Vault via Helm (repo officiel HashiCorp)
   - Déployer Vault dans un namespace dédié (ex: `vault`)
   - **Important** : Ajouter l'option `--set server.dev.enabled=true` lors de l'installation pour activer le mode dev
   - Vérifier que les pods sont en cours d'exécution
   - **Note** : En mode dev, Vault est automatiquement initialisé et déverrouillé. Le token root est `root`.

2. **Se connecter à Vault (mode dev)**
   - Faire un port-forward vers Vault
   - Se connecter à Vault avec le token root sur la console UI ou via CLI avec la commande `vault login root`

3. **Configurer le Database Secrets Engine**
   - Activer le Database Secrets Engine dans Vault
   - Configurer la connexion PostgreSQL :
     - Utiliser la connection URL (les éléments peuvent changer selon votre configuration): `postgresql://{{username}}:{{password}}@postgres.db.svc.cluster.local:5432/tpkubernetes`
     - Configurer la connexion dans Vault avec les credentials PostgreSQL existants
     - **Important** : Ne pas activer la rotation automatique du mot de passe (désactiver la rotation)
   - Créer un rôle Vault `ro` pour générer des credentials **dynamiques** :
     - Définir la TTL (ex: 1h) et le TTL max (ex: 24h)
     - **Creation statements** :
       ```sql
       CREATE ROLE "{{name}}" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}';
       GRANT CONNECT ON DATABASE tpkubernetes TO "{{name}}";
       GRANT USAGE ON SCHEMA public TO "{{name}}";
       GRANT SELECT ON ALL TABLES IN SCHEMA public TO "{{name}}";
       ALTER DEFAULT PRIVILEGES IN SCHEMA public
       GRANT SELECT ON TABLES TO "{{name}}";
       ```
     - **Revocation statements** :
       ```sql
       REVOKE ALL PRIVILEGES ON ALL TABLES IN SCHEMA public FROM "{{name}}";
       REVOKE USAGE ON SCHEMA public FROM "{{name}}";
       REVOKE CONNECT ON DATABASE tpkubernetes FROM "{{name}}";
       DROP ROLE IF EXISTS "{{name}}";
       ```
     - **Rollback statements** :
       ```sql
       DROP ROLE IF EXISTS "{{name}}";
       ```

4. **Tester la génération de credentials dynamiques**
   - Générer un mot de passe dynamique depuis Vault :
     ```bash
     vault read database/creds/ro
     ```
   - Noter le `username` et le `password` générés
   - Démarrer un pod avec l'image `alpine/psql` pour tester la connexion :
     ```bash
     kubectl run psql-test --image=alpine/psql --rm -it --restart=Never \
       --env="<password-vault>" \
       -- psql \
       -h <service-db>.db.svc.cluster.local \
       -p 5432 \
       -U <username-vault> \
       -d <database-name>
     ```
   - Lorsque vous êtes connecté, tester les permissions read-only :
     ```sql
     -- Vérifier que la lecture fonctionne
     SELECT * FROM items;
     
     -- Essayer de créer un item (devrait échouer avec permissions read-only)
     INSERT INTO items (title) VALUES ('Test item');
     
     -- Essayer de supprimer l'item créé (devrait échouer avec permissions read-only)
     DELETE FROM items WHERE title = 'Test item';
     ```
   - Vérifier que les opérations d'écriture (INSERT, DELETE) échouent avec une erreur de permissions, confirmant que le rôle est bien en read-only

5. **Configurer l'authentification Kubernetes (Optionnel - Point bonus)**
   - Cette étape est optionnelle et permet d'aller plus loin dans l'utilisation de Vault
   - Activer le backend d'authentification Kubernetes dans Vault
   - Configurer le ServiceAccount de Vault pour s'authentifier auprès de Kubernetes
   - Créer un rôle Vault pour s'authentifier au cluster Kubernetes de manière temporaire
   - **Note** : Cette configuration permet de créer des rôles Vault qui peuvent s'authentifier au cluster Kubernetes de manière temporaire, permettant ainsi aux applications de s'authentifier automatiquement auprès de Vault via leur ServiceAccount Kubernetes sans avoir besoin de tokens statiques. C'est une pratique avancée pour la gestion des secrets dynamiques.

### ✅ Validation attendue

- [ ] Vault installé et fonctionnel dans Kubernetes
- [ ] Database Secrets Engine configuré pour PostgreSQL/CockroachDB
- [ ] Credentials dynamiques générés et testés avec le pod `alpine/psql` (permissions read-only vérifiées)
- [ ] **Bonus** : Authentification Kubernetes configurée (optionnel)

---

## PHASE 4 — Ingress & HTTPS (cert-manager)

### Objectif
Exposer l'application de manière sécurisée via HTTPS avec gestion automatique des certificats TLS

### 🔐 À quoi sert cert-manager ?

**cert-manager** est un opérateur Kubernetes qui automatise la gestion des certificats TLS/SSL. Dans un environnement Kubernetes, lorsque vous exposez une application via Ingress avec HTTPS, vous avez besoin de certificats TLS pour sécuriser les communications.

**Sans cert-manager** :
- Vous devriez créer manuellement les certificats TLS
- Les renouveler manuellement avant expiration
- Gérer les Secrets Kubernetes contenant les certificats
- Risque d'interruption de service si un certificat expire

**Avec cert-manager** :
- ✅ **Génération automatique** des certificats TLS
- ✅ **Renouvellement automatique** avant expiration
- ✅ **Gestion des Secrets** Kubernetes automatique
- ✅ **Support de plusieurs sources** : Let's Encrypt (production), auto-signé (dev/test), autres CAs
- ✅ **Déclaration via CRD** : vous déclarez ce que vous voulez (Certificate), cert-manager s'occupe du reste

### 📚 Concepts clés

**Issuer / ClusterIssuer** :
- Définit **d'où** obtenir les certificats (source d'autorité)
- **Issuer** : valable pour un namespace spécifique
- **ClusterIssuer** : valable pour tout le cluster
- Types : `selfSigned` (auto-signé), `acme` (Let's Encrypt), `ca` (autorité de certification interne), etc.

**Certificate** :
- Ressource Kubernetes (CRD) qui déclare **quel certificat** vous voulez
- Spécifie : le domaine, la durée de validité, l'Issuer à utiliser
- cert-manager génère automatiquement un **Secret TLS** avec le certificat

**Secret TLS** :
- Secret Kubernetes contenant le certificat et la clé privée
- Généré automatiquement par cert-manager
- Utilisé par l'Ingress pour activer HTTPS

### Ce que vous devez faire

1. **Installer cert-manager** via Helm pour bénéficier de la gestion automatique des certificats dans le namespace **cert-manager**

2. **Comprendre les CRD** installées par cert-manager (Issuer, Certificate, etc.) pour maîtriser les ressources disponibles

3. **Créer un Issuer self-signed** :
   - Pour l'examen, vous utiliserez un certificat auto-signé (pas de validation externe)
   - **Notes** : En production, on utiliserait un ClusterIssuer avec Let's Encrypt pour des certificats reconnus

4. **Créer un Certificate** pour `app.localhost` :
   - Déclarer le certificat souhaité
   - cert-manager générera automatiquement le Secret TLS

5. **Installer un Ingress Controller** (nginx-ingress) :
   - Kubernetes ne fournit pas d'Ingress Controller par défaut
   - nginx-ingress est le plus populaire et le plus utilisé

6. **Configurer l'Ingress avec TLS** :
   - Utiliser le Secret TLS généré par cert-manager
   - Configurer le routage path-based : `/` → frontend, `/api` → backend
   - Accéder à l'application via `https://app.localhost`

### Travaux demandés

1. **Installer cert-manager via Helm**
   - Ajouter le repo Helm cert-manager (Faites une recherche sur la documentation officielle de [cert-manager](https://cert-manager.io))
   - Installer cert-manager dans le namespace `cert-manager`
   - Vérifier l'installation avec `kubectl get pods -n cert-manager`

2. **Identifier les CRD installés**
   - Lister les Custom Resource Definitions (CRD) installées par cert-manager :
     ```bash
     kubectl get crd | grep cert-manager
     ```
   - Comprendre leur rôle :
     - `issuers.cert-manager.io` / `clusterissuers.cert-manager.io` : Sources d'autorité pour les certificats
     - `certificates.cert-manager.io` : Déclaration des certificats souhaités
     - `certificaterequests.cert-manager.io` : Requêtes internes de certificats (gérées automatiquement)

3. **Créer les ressources cert-manager**
   - Créer un **Issuer** self-signed dans le namespace `todolist`
     - Type : `selfSigned` (certificat auto-signé, pas de validation externe)
     - Utile pour le développement et les tests
   - Créer un **Certificate** pour `app.localhost` dans le namespace `todolist`
     - Référencer l'Issuer créé précédemment
     - Spécifier le domaine : `app.localhost`
     - cert-manager générera automatiquement un Secret TLS nommé `app-localhost-tls`

4. **Vérifier la génération du certificat**
   - Vérifier que le Certificate est prêt : `kubectl get certificate -n todolist`
   - Vérifier que le Secret TLS a été créé : `kubectl get secret app-localhost-tls -n todolist`
   - Examiner les détails : `kubectl describe certificate app-localhost-tls -n todolist`

5. **Installer un Ingress Controller** (si pas déjà fait)
   - **Avec Minikube** : `minikube addons enable ingress`
   - **Avec un cluster standard** : Installer nginx-ingress via Helm ou kubectl
   - Vérifier que l'Ingress Controller est actif : `kubectl get pods -n ingress-nginx`

6. **Configurer un Ingress HTTPS**
   - Créer un Ingress avec TLS dans le namespace `todolist`
   - Utiliser le Secret TLS `app-localhost-tls` généré par cert-manager
   - Configurer le routage path-based :
     - `/api` → Service backend (port 8000)
     - `/` → Service frontend (port 80)
   - Spécifier l'IngressClass : `nginx`
   - Accéder à l'application via `https://app.localhost`

### Notes importantes

- **Certificat auto-signé** : Le certificat étant auto-signé, un warning de sécurité s'affichera dans le navigateur. C'est normal et attendu pour un environnement de développement/test. En production, on utiliserait Let's Encrypt pour des certificats reconnus.

- **minikube tunnel** : Avec Minikube, vous devrez lancer `sudo -E minikube tunnel` dans un terminal séparé pour exposer l'Ingress Controller (type LoadBalancer) puis y accéder depuis votre navigateur web.

### ✅ Validation attendue

- [ ] cert-manager installé et pods en cours d'exécution
- [ ] Issuer créé et prêt
- [ ] Certificate créé et dans l'état "Ready"
- [ ] Secret TLS `app-localhost-tls` généré automatiquement par cert-manager
- [ ] Ingress Controller installé et actif
- [ ] Ingress créé avec une adresse IP
- [ ] HTTPS fonctionnel (accès via `https://app.localhost`)
- [ ] Routage correct : `/` → frontend, `/api` → backend
- [ ] Le certificat est visible dans les secrets du namespace

### 🔗 Ressources utiles

- [Documentation cert-manager](https://cert-manager.io/docs/)
- [Guide d'accès Ingress](./kubernetes/ingress/ACCESS.md)
- [nginx-ingress Documentation](https://kubernetes.github.io/ingress-nginx/)

---

## PHASE 5 — Collecte de logs (OpenSearch & Fluent Bit)

### Objectif
Centraliser et visualiser les logs du backend et du frontend avec OpenSearch et Fluent Bit

### Travaux demandés

1. **Déployer OpenSearch via Helm**
   - Ajouter le repo Helm OpenSearch
   - Installer OpenSearch dans le namespace `logging` avec Helm et utiliser le fichier de configuration `resources/values-opensearch.yaml`
   - Vérifier que le pod OpenSearch sont en cours d'exécution (**Notes** : OpenSearch met du temps à démarrer)

2. **Déployer OpenSearch Dashboards via Helm**
   - Installer OpenSearch Dashboards via Helm dans le namespace `logging` en utilisant le fichier de configuration `resources/values-opensearch-dashboards.yaml`
   - Accéder à l'interface via port-forward : `kubectl -n logging port-forward svc/opensearch-dashboards 5601:5601`

3. **Déployer Fluent Bit via Helm**
   - **Créer un ConfigMap pour les parsers personnalisés** : Les logs du backend (Uvicorn) et du frontend (Nginx) ont des formats spécifiques. Pour les parser correctement et extraire les champs (méthode HTTP, chemin, code de statut, etc.), il faut créer des parsers personnalisés :
     ```bash
     kubectl create configmap fluent-bit-custom-parsers --from-literal=custom_parsers.conf='[PARSER]
         Name        uvicorn_access
         Format      regex
         Regex       ^(?<time>\d{4}-\d{2}-\d{2} \d{2}:\d{2}:\d{2}) (?<level>[A-Z]+):\s+(?<client_ip>[^:]+):(?<client_port>\d+)\s+-\s+"(?<method>[A-Z]+)\s+(?<path>[^ ]+)\s+(?<protocol>[^"]+)"\s+(?<status>\d{3})
         Time_Key    time
         Time_Format %Y-%m-%d %H:%M:%S

     [PARSER]
         Name        nginx_access
         Format      regex
         Regex       ^(?<remote>[^ ]+) [^ ]+ [^ ]+ \[(?<time>[^\]]+)\] "(?<method>[A-Z]+) (?<path>[^ ]+) (?<protocol>[^"]+)" (?<status>\d{3}) (?<bytes>\d+) "(?<referer>[^"]*)" "(?<agent>[^"]*)" "(?<forwarded_for>[^"]*)"
         Time_Key    time
         Time_Format %d/%b/%Y:%H:%M:%S %z' -n logging
     ```
   - Ajouter le repo Helm Fluent
   - Installer Fluent Bit via Helm dans le namespace `logging`
   - Configurer Fluent Bit en utilisant le fichier `kubernetes/logging/values-fluentbit.yaml` pour :
     - Collecter les logs des pods backend et frontend (namespace `todolist`)
     - Parser les logs avec les parsers personnalisés (uvicorn_access, nginx_access)
     - Envoyer les logs vers OpenSearch
   - Vérifier que le DaemonSet Fluent Bit est déployé sur tous les nœuds

4. **Créer un index pattern dans OpenSearch Dashboards**
   - Se connecter à OpenSearch Dashboards via port-forward : `kubectl port-forward svc/opensearch-dashboards 5601:5601 -n logging`
   - Accéder à l'interface dans votre navigateur : http://localhost:5601
   - Se rendre dans **"Dashboards Management"** puis **"Index patterns"**
   - Cliquer sur **"Create index pattern"**

   ![OpenSearch](images/indexpattern.png)

   - Appeler `todolist*` pour le nom de l'index pattern (avec le wildcard `*` pour inclure tous les indices commençant par `todolist`)
   - Choisir `@timestamp` pour le champ de temps (Time field)
   - Cliquer sur **"Create index pattern"**

4. **Vérifier la collecte de logs**
   - Générer du trafic sur le backend et le frontend
   - Vérifier que les logs sont bien collectés et indexés dans OpenSearch
   - Accéder à l'interface OpenSearch Dashboards puis "Discover" pour visualiser les logs
   - Filtrer les logs par namespace (`todolist`), par pod (`backend-*`, `frontend-*`)

   ![Dashbord](images/opensearch.png)

### ✅ Validation attendue

- [ ] OpenSearch déployé et fonctionnel dans le namespace `logging`
- [ ] OpenSearch Dashboards accessible et fonctionnel
- [ ] Fluent Bit déployé et collectant les logs
- [ ] Logs du backend et du frontend visibles dans OpenSearch
- [ ] Filtrage des logs par namespace et pod fonctionnel
- [ ] Capture d'écran des logs dans OpenSearch Dashboards (Discover) montrant les logs du backend et du frontend

---

## Livrables attendus

### Repository Git contenant :

- [ ] Manifests Kubernetes 
- [ ] Application fonctionnelle en HTTPS 
- [ ] Monitoring opérationnel (Prometheus + Grafana)
- [ ] Dashboard Grafana exporté (JSON)
- [ ] Configuration OpenSearch et Fluent Bit

### Rapport expliquant :

- [ ] Chaque décision technique prise
- [ ] Chaque CRD utilisé (cert-manager, Prometheus Operator)
- [ ] Limites liées à Minikube (si applicable)
- [ ] Architecture finale de l'application
- [ ] Choix de la base de données (PostgreSQL ou CockroachDB) et justification

### Captures d'écran :

Merci de prendre un maximum de captures d’écran des éléments que vous estimez utiles afin de faciliter ma correction. Voici quelques exemples :

- [ ] Interface Grafana avec le dashboard importé
- [ ] Prometheus avec les métriques du backend
- [ ] OpenSearch Dashboards avec les logs du backend et frontend
- [ ] Application accessible via HTTPS sur https://app.local
- [ ] Vault avec les credentials dynamiques générés
- [ ] Vault avec les secrets Engine PostgreSQL et Kubernetes configurés
- [ ] Pod alpine/psql connecté à PostgreSQL avec identifiants dynamique avec tentative de suppression d'un élément d'une table (montrant l'échec avec permissions read-only)
- [ ] Capture d'écran des logs dans OpenSearch Dashboards (Discover) montrant les logs du backend et du frontend

---

## Conseils et bonnes pratiques

### Structure recommandée pour les manifests

```
manifests/
├── database/               # PostgreSQL ou CockroachDB (dans namespace db)
├── backend/
├── frontend/
├── ingress/
│   └── ingress.yaml
├── cert-manager/
│   ├── issuer.yaml
│   └── certificate.yaml
├── monitoring/   
├── logging/
```

### Commandes utiles

```bash
# Vérifier l'état des pods
kubectl get pods -n <namespace>

# Voir les logs
kubectl logs -f <pod-name> -n <namespace>

# Décrire une ressource
kubectl describe <resource-type> <resource-name> -n <namespace>

# Vérifier les événements
kubectl get events -n <namespace> --sort-by='.lastTimestamp'

# Port-forward pour tester
kubectl port-forward svc/<service-name> <local-port>:<service-port> -n <namespace>
```
---

## Ressources

- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [Helm Documentation](https://helm.sh/docs/)
- [Prometheus Operator](https://prometheus-operator.dev/)
- [cert-manager Documentation](https://cert-manager.io/docs/)
- [OpenSearch Documentation](https://opensearch.org/docs/)
- [OpenSearch Helm Charts](https://github.com/opensearch-project/helm-charts)
- [Fluent Bit Documentation](https://docs.fluentbit.io/)
- [Fluent Bit Helm Charts](https://github.com/fluent/helm-charts)
- [Hashicorp Vault Documentation](https://developer.hashicorp.com/vault/docs)
- [Vault Helm Charts](https://github.com/hashicorp/vault-helm)
- [CockroachDB Documentation](https://www.cockroachlabs.com/docs/) (optionnel)

---

Bon courage et n'hésitez pas à me contacter si besoin !
Pour les intéréssés, une session de correction de l'examen sera organisée.

---

BOUNACEUR Mehdi

