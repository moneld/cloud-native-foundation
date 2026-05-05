# ADR-001 — Environnement de développement local optionnel basé sur k3s

- **Statut** : Accepté
- **Date** : 2026-05-05

## Contexte

La plateforme cible est AWS EKS. Cependant, itérer directement sur EKS pose
trois problèmes pour le développement quotidien :

1. **Coût** : un cluster EKS minimal avec un node group, un NAT gateway et un
   contrôleur ALB représente environ 150 €/mois en continu. Pour un projet
   open source, ce coût est rédhibitoire.
2. **Latence d'itération** : provisionner un node group, attendre qu'un ALB
   se crée ou qu'un certificat ACME soit émis prend plusieurs minutes. Pour
   le développement de manifests Helm et Kubernetes, on a besoin d'un cycle
   "modifier → appliquer → vérifier" en quelques secondes.
3. **Accessibilité** : un contributeur potentiel doit pouvoir tester le projet
   sans créer de compte AWS ni risquer de facturation imprévue.

Une approche purement cloud sacrifie le coût et l'accessibilité. Une approche
purement locale sacrifie le réalisme cloud et empêche de valider les
intégrations spécifiques à AWS (IRSA, ALB, RDS).

## Décision

Maintenir deux plateformes en parallèle dans le même dépôt :

- **`platforms/eks-aws/`** : la plateforme cible, AWS EKS, provisionnée par
  Terraform. C'est la plateforme officielle et le focus principal du projet.
- **`platforms/k3s-local/`** : une plateforme de développement optionnelle,
  cluster k3s mono-nœud sur une VM Linux. Elle permet d'itérer sur la majorité
  des manifests Kubernetes sans dépendance cloud.

Les manifests applicatifs (`apps/`) sont communs aux deux plateformes via des
overlays Kustomize. Les composants spécifiques à chaque plateforme (load
balancing, certificats, secrets) sont configurés différemment :

| Composant            | k3s-local                  | eks-aws                          |
|----------------------|----------------------------|----------------------------------|
| Load balancer        | MetalLB (mode L2)          | AWS Load Balancer Controller     |
| Certificats          | ClusterIssuer auto-signé   | Let's Encrypt via cert-manager   |
| Secrets              | Non testé en local         | External Secrets Operator + AWS Secrets Manager |
| Autoscaling nœuds    | N/A (mono-nœud)            | Karpenter                        |
| Stockage             | local-path-provisioner     | EBS CSI driver                   |

Certains composants (RDS, External Secrets Operator, Karpenter, IRSA) ne sont
pas répliqués en local. Leur intérêt tient à leur intégration AWS native, et
toute simulation locale (PostgreSQL en pod, secrets en clair, etc.) en perdrait
la valeur pédagogique. Le développement local couvre donc les manifests
Kubernetes portables ; les intégrations cloud sont validées exclusivement sur
EKS.

## Pourquoi k3s plutôt que kind, minikube ou kubeadm

| Distribution | Avantages                                                       | Inconvénients                                          |
|--------------|-----------------------------------------------------------------|--------------------------------------------------------|
| **k3s**      | Léger (~500 Mo RAM idle), démarrage <30s, supporté en production par Rancher | Quelques bundles à désactiver pour cohérence avec EKS |
| kind         | Très rapide à provisionner, scriptable                          | Tourne dans Docker, networking peu représentatif       |
| minikube     | Mature, bonne UX                                                | Plus lourd, moins fidèle à un cluster réel             |
| kubeadm      | 100% upstream                                                   | Setup long, surdimensionné pour un mono-nœud lab       |

k3s gagne sur l'empreinte mémoire et la fidélité comportementale : ce qui
fonctionne sur k3s fonctionne en général sur EKS, modulo les composants
cloud-spécifiques. Les bundles k3s par défaut (Traefik v2, ServiceLB, local
storage) sont désactivés via `--disable traefik --disable servicelb` pour
installer les **mêmes versions upstream** des composants en local et sur EKS.

## Conséquences

### Positives

- Le projet est utilisable sans compte AWS pour la majorité des composants.
- Les contributeurs externes peuvent tester localement avant d'ouvrir une PR.
- Les manifests sont **portables par construction** : si un changement
  fonctionne en local mais casse sur EKS, c'est le signe d'une dépendance
  implicite à AWS qu'il faut documenter ou refactoriser.
- Le coût d'expérimentation est nul ou minimal.

### Négatives

- **Maintenance double** : certains composants ont une configuration
  différente entre les deux plateformes. Mitigé par l'usage de Kustomize
  overlays pour `apps/` et de `values.yaml` distincts pour les composants
  Helm.
- **Couverture limitée en local** : les NetworkPolicies inter-nœuds, les
  intégrations IRSA, le comportement de Karpenter et les ALB ne peuvent être
  validés que sur EKS. Cette limite est documentée dans le README de
  `k3s-local/`.
- **Risque de drift** entre les deux plateformes si un composant est ajouté
  d'un côté et oublié de l'autre. Mitigé par une CI qui lint les deux
  arborescences.

### Neutres

- La VM utilisée pour k3s doit avoir une IP fixe sur son réseau hôte pour
  que MetalLB reste stable entre redémarrages.
- Le cluster k3s mono-nœud limite les workloads testables localement à ceux
  qui tiennent dans la RAM allouée à la VM (typiquement 8 Go).

## Alternatives écartées

### Plateforme unique sur EKS, sans environnement local

Rejetée : barrière à l'entrée trop élevée pour un projet open source. Tout
contributeur devrait créer un compte AWS, configurer un budget, et risquer
des coûts imprévus pour tester une simple modification de chart Helm.

### Plateforme unique en local, sans environnement AWS

Rejetée : sans validation sur EKS, le projet ne tient pas sa promesse de
plateforme production-grade. Les composants spécifiques au cloud (IRSA, ALB,
RDS, Karpenter) sont essentiels.

### Cluster k3s sur EC2 spot

Considérée puis rejetée : ajoute la gestion du cycle de vie d'une instance
EC2 (patches OS, monitoring, sécurité) sans apporter de bénéfice par rapport
à un cluster EKS provisionné à la demande.

### Multi-nœuds k3s en local via plusieurs VMs

Rejetée : empreinte mémoire trop élevée sur une station de travail standard
(16 Go). La validation multi-nœuds se fait sur EKS où elle est gratuite tant
que le cluster est éteint en dehors des sessions de travail.

## Validation de la décision

Cette décision sera reconsidérée si :

- Un composant requis par le projet ne peut pas tourner sur k3s mono-nœud.
- La maintenance des deux plateformes devient disproportionnée par rapport au
  bénéfice (par exemple, plus de 20% du temps de contribution passé à
  réconcilier les deux).
- Un contributeur fournit une alternative locale plus convaincante (par
  exemple, une intégration LocalStack solide pour simuler IRSA et ALB).

## Références

- [k3s documentation](https://docs.k3s.io/)
- [Kustomize overlays pattern](https://kubectl.docs.kubernetes.io/guides/config_management/components/)
- ADR-002 — Traefik + Gateway API
- ADR-004 — Helm partout
