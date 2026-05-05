# Traefik + Gateway API

## Rôle

Traefik est l'ingress controller du cluster. Il implémente l'API standard
Gateway API (sigs.k8s.io/gateway-api), pas l'API Ingress legacy ni les
CRDs propres à Traefik (IngressRoute, Middleware).

Concrètement, Traefik :

- Provisionne une GatewayClass nommée `traefik`.
- Watch les objets `Gateway` et `HTTPRoute` du cluster.
- Reçoit le trafic externe via un Service LoadBalancer (IP attribuée par MetalLB).
- Route vers les Services backend selon les HTTPRoutes.

## Architecture

Client
│
│ curl -H "Host: demo.local"
▼
IP MetalLB (192.168.122.200)
│
▼
Service Traefik (LoadBalancer)
│
▼
Pod Traefik (proxy)
│
│ matche le hostname et le path
▼
HTTPRoute "nginx-test" dans namespace "demo"
│
▼
Service "nginx" dans namespace "demo"
│
▼
Pod nginx

## Configuration

Les providers `kubernetesIngress` et `kubernetesCRD` sont **désactivés**.
Seul `kubernetesGateway` est actif. Discipline : tout le routing passe par
l'API standard Gateway API, pas par les extensions Traefik propriétaires.

Les CRDs Gateway API sont installées **séparément** depuis upstream
(voir `platforms/k3s-local/gateway-api/`). Le chart Traefik embarque sa
propre copie des CRDs, mais elle est ignorée via le flag `--skip-crds`.
Cela évite les conflits de field manager entre `kubectl apply` et Helm,
et découple le cycle de vie des CRDs (standard Kubernetes) de celui du
controller (releases Traefik).

## Gateway "main"

Un objet `Gateway` nommé `main` est créé dans le namespace `traefik`. Il
écoute en HTTP sur le port 80 et accepte les HTTPRoutes de tous les
namespaces (`from: All`).

Pour un usage multi-tenant en production, restreindre à `Same` (même
namespace que le Gateway) ou `Selector` (avec un label sur les namespaces
autorisés).

Un listener HTTPS sur le port 443 sera ajouté à la prochaine étape, quand
cert-manager pourra émettre les certificats.

## Installation

```bash
# 1. Installer les CRDs Gateway API d'abord (voir gateway-api/)

# 2. Installer Traefik
helm repo add traefik https://traefik.github.io/charts
helm repo update
kubectl create namespace traefik

helm install traefik traefik/traefik \
  --namespace traefik \
  --values values.yaml \
  --skip-crds \
  --wait

# 3. Créer le Gateway
kubectl apply -f gateway.yaml
```

## Test rapide

```bash
kubectl create namespace demo
kubectl create deployment nginx --image=nginx -n demo
kubectl expose deployment nginx --port=80 -n demo

cat <<EOF | kubectl apply -f -
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: nginx-test
  namespace: demo
spec:
  parentRefs:
    - name: main
      namespace: traefik
  hostnames:
    - "demo.local"
  rules:
    - matches:
        - path:
            type: PathPrefix
            value: /
      backendRefs:
        - name: nginx
          port: 80
EOF

GATEWAY_IP=$(kubectl get svc traefik -n traefik -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
curl -H "Host: demo.local" http://$GATEWAY_IP

kubectl delete namespace demo
```

## Pourquoi le Gateway écoute sur 8000 et pas 80

Traefik distingue les **entryPoints** (ports d'écoute internes du proxy,
configurés dans le chart Helm) et les **Gateway listeners** (déclarations
de routing dans l'API Gateway).

Pour qu'un Gateway listener fonctionne, son `port` doit correspondre au
port interne d'un entryPoint Traefik, **pas** au port exposé par le
Service.

Configuration adoptée :

- entryPoint `web` : port interne `8000`, exposé en `80` via le Service
- entryPoint `websecure` : port interne `8443`, exposé en `443` via le Service
- Gateway listener HTTP : `port: 8000` (matche l'entryPoint web)
- Gateway listener HTTPS : `port: 8443` (matche l'entryPoint websecure)

L'expérience utilisateur n'est pas affectée : on accède toujours à
`http://IP` (port 80 par défaut), c'est le Service LoadBalancer qui
fait le mapping vers le port interne du pod Traefik.

Cette séparation est cohérente avec ce qui se fait sur EKS : le NLB AWS
fait le même mapping `80:8000` vers les pods Traefik. Elle évite aussi
de devoir donner au pod Traefik la capability `NET_BIND_SERVICE` pour
binder sur des ports < 1024.

## HTTPS

Le Gateway expose un listener HTTPS sur le port externe 443 (interne 8443).
Le certificat utilisé est `demo-tls-secret`, émis par cert-manager via le
ClusterIssuer `lab-ca`.

Tous les certs applicatifs partagent la même CA racine. Un seul import
dans le trust store du système permet de valider tous les services
HTTPS du lab sans `curl -k`.

## Sur EKS

Sur EKS :

- Le Service `traefik` reste en `LoadBalancer` mais c'est l'AWS Load
  Balancer Controller qui provisionne un NLB AWS au lieu de MetalLB.
- Les CRDs Gateway API et la GatewayClass `traefik` sont identiques.
- Les HTTPRoutes des apps sont **rigoureusement identiques** entre
  k3s-local et EKS — c'est tout l'intérêt de Gateway API.
