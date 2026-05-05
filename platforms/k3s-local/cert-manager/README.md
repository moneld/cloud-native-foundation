# cert-manager

## Rôle

cert-manager émet et renouvelle automatiquement les certificats TLS du
cluster. Il s'intègre nativement à Gateway API : un Gateway peut référencer
un Secret TLS, cert-manager le crée et le maintient à jour.

## Stratégie d'émission

### Sur k3s-local : CA auto-signée locale (pattern bootstrap)

Deux ClusterIssuer en cascade :

1. **`selfsigned-bootstrap`** : ClusterIssuer auto-signé minimal, n'émet
   qu'un seul certificat (la CA racine du lab).
2. **`lab-ca`** : ClusterIssuer adossé à la CA racine émise par le
   bootstrap. C'est lui qui signe tous les certificats applicatifs.

Avantage : tous les certificats applicatifs partagent la même CA racine.
On peut importer une seule CA (`lab-ca-secret`) dans le trust store du
système ou du navigateur pour valider tous les certs du lab.

### Sur eks-aws : Let's Encrypt via DNS-01

Le ClusterIssuer `lab-ca` sera remplacé par un ClusterIssuer ACME
configuré pour Let's Encrypt avec validation DNS-01 via Route53. Le
reste de la stack (Certificate, Gateway, HTTPRoute) ne change pas.

## Gateway API integration

Cert-manager doit être configuré avec `enableGatewayAPI: true` dans son
controller config (cf. values.yaml). Sans cela, il ignore les Gateway
et n'agit que sur les Ingress legacy.

## Importer la CA dans le système (optionnel)

```bash
kubectl get secret lab-ca-secret -n cert-manager \
  -o jsonpath='{.data.ca\.crt}' | base64 -d > /tmp/lab-ca.crt

# Ubuntu/Debian
sudo cp /tmp/lab-ca.crt /usr/local/share/ca-certificates/
sudo update-ca-certificates
```

Après cette opération, `curl https://demo.local` (avec `/etc/hosts`
configuré) marche sans `-k`.

## Test rapide

```bash
# Vérifier les ClusterIssuer
kubectl get clusterissuer

# Doit afficher selfsigned-bootstrap et lab-ca, tous deux READY=True
```

## ClusterIssuer

### Pattern bootstrap CA

Deux ClusterIssuer en cascade :

1. **`selfsigned-bootstrap`** — ClusterIssuer auto-signé minimal, utilisé
   uniquement pour émettre la CA racine du lab.
2. **`lab-ca`** — ClusterIssuer adossé à la CA racine émise par le
   bootstrap. Signe tous les certificats applicatifs.

Avantage : tous les certificats applicatifs partagent la même CA. On peut
importer une seule fois `lab-ca-secret` dans le trust store du système ou
du navigateur pour valider tous les services HTTPS du lab sans warning.

### Comparaison ClusterIssuer

| ClusterIssuer          | Type        | Usage                           |
| ---------------------- | ----------- | ------------------------------- |
| `selfsigned-bootstrap` | Self-signed | Émet la CA racine, rien d'autre |
| `lab-ca`               | CA          | Émet les certs applicatifs      |

### Sur EKS

Sur EKS, le ClusterIssuer `lab-ca` sera remplacé par un ClusterIssuer ACME
configuré pour Let's Encrypt avec validation DNS-01 via Route53. Les
Certificate des apps ne changent pas (juste leur `issuerRef` est modifié).
