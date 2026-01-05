# 🔧 Take-Home Test – DevOps Engineer

> **Niveau cible** : Engineer 2 / Senior 1 (3-6 ans d'expérience)  
> **Durée estimée** : 4-6 heures (hors bonus)  
> **Délai de rendu** : 5 jours maximum  

---

## 🎯 Objectif

Ce test évalue votre capacité à concevoir et implémenter une **infrastructure de déploiement complète** pour une application web. Nous évaluons :

- La maîtrise de l'**Infrastructure as Code** (IaC)
- La conception de **pipelines CI/CD** robustes
- La **containerisation** et l'orchestration
- Les pratiques de **sécurité** et de gestion des secrets
- La mise en place d'**observabilité** (logs, métriques, alertes)
- La **documentation** et la reproductibilité

---

## 📋 Contexte

Vous rejoignez une startup fintech qui développe **PayFlow**, une API de paiement. L'équipe de développement vous fournit une application simple (voir section "Application Fournie") et vous demande de mettre en place toute l'infrastructure de déploiement.

### Environnements requis

| Environnement | Usage | Contraintes |
|---------------|-------|-------------|
| `development` | Tests des développeurs | Déploiement rapide, coût minimal |
| `staging` | Validation pré-production | Iso-production (architecture similaire) |
| `production` | Utilisateurs finaux | Haute disponibilité, sécurité maximale |

---

## 🐳 Application Fournie

Une API REST minimaliste est fournie. Vous pouvez utiliser celle-ci ou en créer une équivalente dans le langage de votre choix.

### Option 1 : Application Node.js fournie

```javascript
// app.js
const express = require('express');
const app = express();
const PORT = process.env.PORT || 3000;

// Health check
app.get('/health', (req, res) => {
  res.json({ 
    status: 'healthy', 
    timestamp: new Date().toISOString(),
    version: process.env.APP_VERSION || '1.0.0'
  });
});

// Ready check (pour Kubernetes)
app.get('/ready', (req, res) => {
  // Simuler une vérification de dépendances
  const dbConnected = process.env.DB_HOST ? true : false;
  if (dbConnected) {
    res.json({ status: 'ready' });
  } else {
    res.status(503).json({ status: 'not ready', reason: 'database not configured' });
  }
});

// Endpoint métier simulé
app.post('/api/v1/payments', express.json(), (req, res) => {
  const { amount, currency } = req.body;
  if (!amount || amount <= 0) {
    return res.status(400).json({ error: 'Invalid amount' });
  }
  res.status(201).json({
    id: `pay_${Date.now()}`,
    amount,
    currency: currency || 'EUR',
    status: 'pending',
    created_at: new Date().toISOString()
  });
});

// Métriques Prometheus (basique)
app.get('/metrics', (req, res) => {
  res.set('Content-Type', 'text/plain');
  res.send(`
# HELP http_requests_total Total HTTP requests
# TYPE http_requests_total counter
http_requests_total{method="GET",path="/health"} 100
http_requests_total{method="POST",path="/api/v1/payments"} 42

# HELP app_info Application information
# TYPE app_info gauge
app_info{version="${process.env.APP_VERSION || '1.0.0'}"} 1
  `);
});

app.listen(PORT, () => {
  console.log(`PayFlow API running on port ${PORT}`);
});
```

```json
// package.json
{
  "name": "payflow-api",
  "version": "1.0.0",
  "main": "app.js",
  "scripts": {
    "start": "node app.js",
    "test": "echo 'No tests yet' && exit 0"
  },
  "dependencies": {
    "express": "^4.18.2"
  }
}
```

### Option 2 : Créez votre propre application

Vous pouvez implémenter une application équivalente en Go, Python, ou autre, à condition qu'elle expose :

- `GET /health` → Health check
- `GET /ready` → Readiness probe
- `GET /metrics` → Métriques Prometheus
- `POST /api/v1/payments` → Endpoint métier

---

## 📦 Livrables Obligatoires

### 1. Dockerfile (Multi-stage)

Créez un Dockerfile optimisé avec :

| Critère | Exigence |
|---------|----------|
| Build multi-stage | Séparer build et runtime |
| Image de base | Alpine ou distroless |
| Utilisateur non-root | Sécurité |
| Taille optimisée | < 150 MB pour Node.js, < 50 MB pour Go |
| Labels OCI | version, maintainer, description |
| Health check intégré | Instruction HEALTHCHECK |

### 2. Docker Compose (Environnement local)

Fournissez un `docker-compose.yml` permettant de lancer localement :

- L'application PayFlow
- Une base de données PostgreSQL
- Redis (cache/sessions)
- Un reverse proxy (Traefik ou Nginx)

**Exigences** :
- Variables d'environnement externalisées (fichier `.env.example`)
- Volumes persistants pour les données
- Réseau dédié
- Health checks configurés

### 3. Infrastructure as Code

Choisissez **Terraform** ou **Pulumi** pour provisionner l'infrastructure cloud.

#### Provider au choix

| Cloud | Services attendus |
|-------|-------------------|
| AWS | ECS/EKS, RDS, ElastiCache, ALB, VPC |
| GCP | Cloud Run/GKE, Cloud SQL, Memorystore, Load Balancer |
| Azure | AKS/Container Apps, Azure SQL, Redis Cache |

#### Ressources minimales à provisionner

```
├── Réseau
│   ├── VPC / Virtual Network
│   ├── Subnets (public/private)
│   ├── Security Groups / Firewall Rules
│   └── NAT Gateway
│
├── Compute
│   ├── Cluster Kubernetes (EKS/GKE/AKS)
│   │   OU Container Service (ECS/Cloud Run)
│   └── Node pools / Auto-scaling
│
├── Data
│   ├── Base de données managée (PostgreSQL)
│   └── Cache managé (Redis)
│
├── Networking
│   ├── Load Balancer
│   └── DNS (optionnel)
│
└── Sécurité
    ├── IAM Roles / Service Accounts
    └── Secrets Manager
```

#### Exigences IaC

| Critère | Détail |
|---------|--------|
| **Modules réutilisables** | Séparer VPC, compute, database en modules |
| **Variables par environnement** | `terraform.tfvars` ou workspaces |
| **State distant** | S3/GCS backend avec locking |
| **Outputs documentés** | Endpoints, IDs des ressources |
| **Pas de secrets en dur** | Utiliser des références (SSM, Secrets Manager) |

### 4. Pipeline CI/CD

Créez un pipeline complet avec **GitHub Actions**, **GitLab CI**, ou **CircleCI**.

#### Stages obligatoires

```yaml
pipeline:
  stages:
    - lint          # Validation du code et de l'IaC
    - test          # Tests unitaires et d'intégration
    - security      # Scan de vulnérabilités
    - build         # Build et push de l'image Docker
    - deploy-staging    # Déploiement automatique en staging
    - deploy-production # Déploiement manuel en production
```

#### Détail des stages

| Stage | Outils attendus | Critères |
|-------|-----------------|----------|
| **Lint** | hadolint, tflint, eslint | Fail si erreurs |
| **Test** | Jest/pytest/go test | Couverture > 0% |
| **Security** | Trivy, Snyk, ou Grype | Scan image + dépendances |
| **Build** | Docker buildx | Tag avec SHA + semver |
| **Deploy Staging** | kubectl/helm/terraform | Automatique sur `main` |
| **Deploy Prod** | Idem | Manuel avec approbation |

#### Exigences pipeline

- [ ] Secrets gérés via le secret store du CI (pas en clair)
- [ ] Cache des dépendances (node_modules, .terraform)
- [ ] Artifacts : rapport de tests, rapport de sécurité
- [ ] Notifications (Slack/email) en cas d'échec
- [ ] Déploiement production avec **approval gate**

### 5. Manifestes Kubernetes

Si vous choisissez Kubernetes, fournissez :

```
k8s/
├── base/
│   ├── deployment.yaml
│   ├── service.yaml
│   ├── configmap.yaml
│   ├── secret.yaml (référence externe)
│   ├── hpa.yaml
│   └── ingress.yaml
│
└── overlays/
    ├── staging/
    │   └── kustomization.yaml
    └── production/
        └── kustomization.yaml
```

#### Exigences Kubernetes

| Ressource | Critères |
|-----------|----------|
| **Deployment** | Probes (liveness, readiness, startup), resources limits, securityContext |
| **Service** | ClusterIP ou LoadBalancer selon contexte |
| **HPA** | Scale 2-10 pods, CPU > 70% |
| **Ingress** | TLS, annotations spécifiques au controller |
| **NetworkPolicy** | Restreindre le trafic inter-pods |
| **PodDisruptionBudget** | minAvailable: 1 |

### 6. Observabilité

Documentez ou implémentez une stack d'observabilité :

#### Logs
- Centralisation (Loki, CloudWatch, Datadog)
- Format structuré (JSON)
- Rétention définie par environnement

#### Métriques
- Collecte Prometheus ou équivalent cloud
- Dashboards Grafana (ou équivalent)
- Métriques clés : latence P99, taux d'erreur, throughput

#### Alertes
Définissez au minimum ces alertes :

| Alerte | Condition | Sévérité |
|--------|-----------|----------|
| HighErrorRate | 5xx > 5% sur 5min | Critical |
| HighLatency | P99 > 500ms sur 5min | Warning |
| PodCrashLooping | Restarts > 3 en 10min | Critical |
| DiskSpaceLow | Usage > 85% | Warning |
| CertificateExpiring | Expiration < 14 jours | Warning |

### 7. Documentation

#### README principal

```markdown
# PayFlow Infrastructure

## Architecture
[Diagramme ASCII ou lien vers image]

## Prérequis
- Outils nécessaires (versions)
- Accès cloud requis

## Quick Start
# Commandes pour démarrer en local

## Déploiement
# Procédure par environnement

## Runbooks
- Incident type 1 : procédure
- Rollback : procédure

## Décisions techniques
- Pourquoi ce choix de [X] ?
```

---

## 📊 Grille d'Évaluation

### Critères Éliminatoires

| Critère | Conséquence |
|---------|-------------|
| Secrets en clair dans le code | ❌ Élimination |
| Dockerfile qui ne build pas | ❌ Élimination |
| Aucun pipeline CI/CD | ❌ Élimination |
| Infrastructure non reproductible | ❌ Élimination |

### Grille de Notation (sur 100 points)

| Catégorie | Points | Détail |
|-----------|--------|--------|
| **Dockerfile** | 10 | Multi-stage, sécurité, optimisation |
| **Docker Compose** | 10 | Complétude, configuration, health checks |
| **Infrastructure as Code** | 25 | Modules, variables, state, sécurité |
| **Pipeline CI/CD** | 20 | Stages, sécurité, déploiement |
| **Kubernetes/Orchestration** | 15 | Manifestes, best practices, scaling |
| **Observabilité** | 10 | Logs, métriques, alertes |
| **Documentation** | 10 | Clarté, complétude, diagrammes |

### Barème de Niveau

| Score | Niveau |
|-------|--------|
| < 50 | Insuffisant |
| 50-64 | DevOps Junior confirmé |
| 65-79 | **Engineer 2** ✅ |
| 80-89 | **Senior 1** ✅ |
| ≥ 90 | Senior+ / Staff |

---

## ⭐ Bonus (Optionnels)

### Bonus Sécurité (+15 points max)

| Bonus | Points |
|-------|--------|
| Pod Security Standards (restricted) | +3 |
| Network Policies restrictives | +3 |
| SOPS ou Sealed Secrets pour les secrets | +4 |
| OIDC pour l'authentification CI → Cloud | +3 |
| Rapport SBOM (Software Bill of Materials) | +2 |

### Bonus GitOps (+10 points max)

| Bonus | Points |
|-------|--------|
| ArgoCD ou Flux configuré | +5 |
| ApplicationSet pour multi-env | +3 |
| Sync automatique staging, manuel prod | +2 |

### Bonus Reliability (+10 points max)

| Bonus | Points |
|-------|--------|
| Chaos Engineering (LitmusChaos, Chaos Monkey) | +4 |
| Disaster Recovery documenté | +3 |
| Blue/Green ou Canary deployment | +3 |

### Bonus Advanced (+10 points max)

| Bonus | Points |
|-------|--------|
| Service Mesh (Istio, Linkerd) | +4 |
| Policy as Code (OPA/Gatekeeper) | +3 |
| Cost optimization (Spot instances, right-sizing) | +3 |

---

## 🗂️ Structure de Projet Attendue

```
payflow-infra/
├── README.md
├── .gitignore
│
├── app/                          # Application
│   ├── Dockerfile
│   ├── app.js (ou équivalent)
│   └── package.json
│
├── docker/
│   ├── docker-compose.yml
│   ├── docker-compose.override.yml
│   └── .env.example
│
├── terraform/                    # Infrastructure as Code
│   ├── modules/
│   │   ├── vpc/
│   │   ├── eks/
│   │   ├── rds/
│   │   └── redis/
│   ├── environments/
│   │   ├── staging/
│   │   │   ├── main.tf
│   │   │   ├── variables.tf
│   │   │   └── terraform.tfvars
│   │   └── production/
│   │       └── ...
│   └── backend.tf
│
├── k8s/                          # Manifestes Kubernetes
│   ├── base/
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   ├── configmap.yaml
│   │   ├── hpa.yaml
│   │   ├── pdb.yaml
│   │   └── networkpolicy.yaml
│   └── overlays/
│       ├── staging/
│       └── production/
│
├── .github/                      # CI/CD
│   └── workflows/
│       ├── ci.yml
│       ├── deploy-staging.yml
│       └── deploy-production.yml
│
├── monitoring/                   # Observabilité
│   ├── dashboards/
│   │   └── payflow-dashboard.json
│   ├── alerts/
│   │   └── alerting-rules.yaml
│   └── prometheus/
│       └── scrape-config.yaml
│
└── docs/
    ├── architecture.md
    ├── runbooks/
    │   ├── incident-response.md
    │   └── rollback.md
    └── adr/                      # Architecture Decision Records
        └── 001-choose-eks.md
```

---

## 🎯 Conseils pour Réussir

### Ce que nous recherchons

✅ Infrastructure **reproductible** et **idempotente**  
✅ Séparation claire des **environnements**  
✅ **Sécurité** intégrée dès la conception (shift-left)  
✅ Pipeline **fiable** avec gates de qualité  
✅ Documentation **opérationnelle** (pas juste descriptive)  
✅ Choix techniques **justifiés**

### Ce que nous pénalisons

❌ Infrastructure "ClickOps" (non reproductible)  
❌ Secrets versionnés en clair  
❌ Dockerfile avec `latest` ou user root  
❌ Pas de health checks  
❌ Pipeline sans étape de sécurité  
❌ Sur-ingénierie (Service Mesh pour une app simple)

---

## ❓ FAQ

**Q : Dois-je réellement provisionner l'infrastructure cloud ?**  
R : Non. Le code IaC doit être **valide** (`terraform validate`), mais vous n'avez pas besoin de l'appliquer. Fournissez des captures ou des `terraform plan` si possible.

**Q : Puis-je utiliser Helm au lieu de Kustomize ?**  
R : Oui. Helm, Kustomize, ou même des manifestes bruts sont acceptés. Justifiez votre choix.

**Q : Quelle version de Kubernetes cibler ?**  
R : Kubernetes 1.28+ est recommandé. Évitez les API dépréciées.

**Q : Le monitoring doit-il être fonctionnel ?**  
R : La configuration doit être correcte. Vous pouvez fournir des fichiers de config sans les déployer réellement.

**Q : Puis-je utiliser Ansible/Pulumi/CDK au lieu de Terraform ?**  
R : Oui, tant que l'infrastructure est déclarative et reproductible.

---

## 📤 Soumission

1. Repository Git (GitHub/GitLab/Bitbucket)
2. Assurez-vous que le repository est **accessible**
3. Objet de l'email : `[Take-Home DevOps] Votre Nom - PayFlow`

---

## 📌 Checklist Finale

Avant de soumettre, vérifiez :

- [ ] `docker build` fonctionne sans erreur
- [ ] `docker-compose up` lance l'environnement local
- [ ] `terraform validate` passe sur tous les modules
- [ ] Le pipeline CI est syntaxiquement valide
- [ ] Aucun secret en clair dans le repository
- [ ] README avec instructions claires
- [ ] `.gitignore` approprié (pas de `.terraform/`, `node_modules/`)

---

*Bonne chance ! Montrez-nous comment vous construisez des systèmes fiables.* 🚀
