# Architecture - Cloud-Native Foundation

## Objectif

Cloud-Native Foundation est une plateforme Kubernetes production-grade sur AWS,
déployable en une commande via Terraform et ArgoCD. Le projet vise à fournir
un socle réutilisable pour héberger des applications conteneurisées avec
observabilité, sécurité et GitOps intégrés dès le départ.

## Pour qui

- Équipes plateforme qui ont besoin d'un point de départ sérieux pour monter
  un cluster EKS sans repartir de zéro.
- Développeurs qui veulent un environnement représentatif d'un cluster
  production pour tester leurs workloads.
- Toute personne qui souhaite étudier ou réutiliser une architecture
  cloud-native moderne, documentée et reproductible.

## Principes directeurs

1. **Reproductibilité.** Tout est en Git. Aucun clic console, aucun setup
   manuel. Un `make deploy` provisionne l'ensemble.
2. **Helm partout.** Une seule façon d'installer un composant Kubernetes.
   Les charts upstream officiels sont préférés aux manifests custom dès qu'ils
   existent.
3. **GitOps comme source de vérité.** Une fois ArgoCD bootstrapé, plus aucune
   modification cluster ne se fait sans passer par Git.
4. **Observabilité intégrée.** Métriques, logs et alertes sont des prérequis
   du jour 1, pas des ajouts a posteriori.
5. **Sécurité par défaut.** IAM least-privilege via IRSA, secrets gérés
   externe au cluster, détection runtime, NetworkPolicies restrictives,
   scans CI/CD systématiques.
6. **Coûts maîtrisés.** Auto-shutdown des ressources non-critiques,
   budgets AWS configurés avec alertes, documentation des coûts par composant.

## Vue d'ensemble

┌──────────────────────────────────────────────────────────────────┐
│                            AWS Account                           │
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │                       VPC (multi-AZ)                       │  │
│  │                                                            │  │
│  │   ┌──────────────┐    ┌──────────────┐    ┌─────────────┐  │  │
│  │   │ Public subnet│    │Private subnet│    │ DB subnet   │  │  │
│  │   │   ALB / NAT  │    │ EKS workers  │    │ RDS Postgres│  │  │
│  │   └──────────────┘    └──────────────┘    └─────────────┘  │  │
│  │                                                            │  │
│  └────────────────────────────────────────────────────────────┘  │
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐  │
│  │                       EKS Cluster                          │  │
│  │                                                            │  │
│  │   Platform addons:                                         │  │
│  │   - Karpenter (autoscaling nœuds)                          │  │
│  │   - AWS Load Balancer Controller                           │  │
│  │   - cert-manager + Let's Encrypt                           │  │
│  │   - External Secrets Operator → AWS Secrets Manager        │  │
│  │   - Traefik + Gateway API                                  │  │
│  │   - ArgoCD (GitOps)                                        │  │
│  │   - kube-prometheus-stack (Prometheus, Grafana, AM)        │  │
│  │   - Loki + Grafana Alloy (logs, métriques, traces)         │  │
│  │   - Falco (détection runtime)                              │  │
│  │                                                            │  │
│  │   Workloads:                                               │  │
│  │   - demo-go-service (app exemple, /health, /metrics)       │  │
│  └────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────┘

## Stack technique

### Infrastructure (Terraform)

- **VPC** multi-AZ avec subnets publics, privés et database.
- **EKS** managé, version stable récente, control plane chiffré.
- **RDS PostgreSQL** dans des subnets dédiés, accès restreint au security
  group des workers EKS.
- **IAM** : rôles distincts par composant, policies en least-privilege, IRSA
  pour les addons cluster.
- **State distant** : S3 + DynamoDB lock, provisionné par le bloc `bootstrap/`.

### Plateforme Kubernetes

| Composant                      | Rôle                                             |
|--------------------------------|--------------------------------------------------|
| Karpenter                      | Autoscaling des nœuds, plus rapide et flexible que cluster-autoscaler |
| AWS Load Balancer Controller   | Provisionne les ALB/NLB pour les Services et Ingress |
| Traefik + Gateway API          | Ingress controller moderne, basé sur l'API Gateway K8s |
| cert-manager                   | Émission automatique de certificats via Let's Encrypt |
| External Secrets Operator      | Synchronise les secrets depuis AWS Secrets Manager |
| ArgoCD                         | GitOps, pattern app-of-apps                      |

### Observabilité

- **Métriques** : kube-prometheus-stack (Prometheus, Grafana, Alertmanager).
- **Logs** : Loki, alimenté par Grafana Alloy.
- **Collecteur unifié** : Grafana Alloy en DaemonSet, qui remplace Promtail et
  prépare le terrain pour la collecte de traces OpenTelemetry et de profils.
- **Détection runtime** : Falco avec règles par défaut, alertes routées vers
  Alertmanager.

### CI/CD

- **GitHub Actions** :
  - `terraform-plan` sur PR avec commentaire automatique.
  - `terraform-apply` sur merge vers `main`.
  - `helm-lint` pour valider les charts.
  - `security-scan` : Trivy (images), tfsec / checkov (Terraform).
- **Pre-commit hooks** : `terraform fmt`, `tflint`, `yamllint`, secrets
  detection (gitleaks).

### Application de démonstration

Service Go minimaliste avec endpoints `/health` et `/metrics`, packagé en
image distroless multi-stage, déployé via Kustomize avec overlays par
environnement.

## Environnements

Le projet est structuré pour supporter trois environnements (`dev`, `staging`,
`prod`) via des répertoires Terraform et des overlays Kustomize séparés.
Chaque environnement a son propre `terraform.tfvars` (taille des nœuds,
plage CIDR, instance RDS) et son propre overlay applicatif.

À ce jour, seul `dev` est déployé par défaut. Les répertoires `staging/` et
`prod/` sont prêts mais non provisionnés, pour limiter les coûts d'un projet
de démonstration. Les ajouter consiste à exécuter `terraform apply` dans le
répertoire correspondant.

## Développement local

Pour itérer rapidement sur les manifests Helm et Kubernetes sans payer un EKS,
le projet inclut une plateforme `k3s-local/` : un cluster k3s mono-nœud
déployable sur une VM Linux. Les composants Kubernetes-natifs (Traefik,
cert-manager, ArgoCD, observabilité) y sont identiques à ceux d'EKS, mais
leurs intégrations cloud-spécifiques sont remplacées :

| Composant            | k3s-local                    | eks-aws                              |
|----------------------|------------------------------|--------------------------------------|
| Load balancer        | MetalLB (mode L2)            | AWS Load Balancer Controller         |
| Certificats          | ClusterIssuer auto-signé     | Let's Encrypt via cert-manager       |
| Stockage             | local-path-provisioner       | EBS CSI driver                       |

Les composants exclusivement cloud (RDS, External Secrets Operator, Karpenter,
IRSA) ne sont **pas répliqués** en local : leur valeur pédagogique tient à
leur intégration AWS native, qu'aucune simulation locale ne reproduit
fidèlement. Le développement local cible donc les manifests Kubernetes,
pas l'intégration cloud.

Cet environnement local est **optionnel**. Le projet est utilisable
directement sur AWS sans passer par k3s.

## Choix qu'on n'a pas faits

- **ECS / Fargate sans Kubernetes** : non-portable, écosystème AWS-only.
  EKS donne des compétences cloud-native standards et permet le développement
  local sur k3s.
- **ingress-nginx** : end of life en mars 2026, l'écosystème migre vers
  Gateway API. Voir ADR-002.
- **cluster-autoscaler** : Karpenter est plus rapide, plus flexible et
  consolide automatiquement les nœuds. Voir ADR-005.
- **Flux** : ArgoCD a une UI plus mature et est plus répandu en entreprise.
  Voir ADR-006.
- **Trois environnements actifs simultanément** : surcoût AWS injustifié pour
  un projet open source. La structure les supporte, leur activation est laissée
  à l'utilisateur. Voir ADR-001.
- **Promtail** : en maintenance mode chez Grafana Labs depuis 2025, remplacé
  par Alloy qui unifie la collecte logs/métriques/traces. Voir ADR-009.

## Démarrage rapide

```bash
# Prérequis : AWS CLI configuré, Terraform >= 1.6, kubectl, helm
git clone https://github.com/moneld/cloud-native-foundation
cd cloud-native-foundation

# 1. Provisionner le backend Terraform (une seule fois)
make bootstrap

# 2. Déployer l'environnement dev
make deploy ENV=dev

# 3. Récupérer le kubeconfig
make kubeconfig ENV=dev

# 4. Bootstrap ArgoCD et les apps
make argocd-bootstrap
```

Détail dans `platforms/eks-aws/README.md`.

## Documentation

- `docs/decisions/` - Architecture Decision Records, justifiant chaque choix
  technique.
- `docs/runbooks/` - Procédures opérationnelles (troubleshooting, DR).
- `platforms/k3s-local/README.md` - Setup de l'environnement local.
- `platforms/eks-aws/README.md` - Setup de l'environnement AWS.

## Licence

MIT. Voir `LICENSE`.
