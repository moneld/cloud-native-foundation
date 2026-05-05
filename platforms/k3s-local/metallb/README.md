# MetalLB

## Rôle

MetalLB fournit un load balancer logiciel pour les Services Kubernetes de
type `LoadBalancer` sur le cluster k3s local. Il joue le rôle qu'un
cloud-controller-manager joue dans un cluster managé (EKS, GKE), où
l'allocation d'IP externe est déléguée au cloud provider.

Sans MetalLB, tout Service `type: LoadBalancer` resterait en `<pending>`.

## Mode

**Layer 2 (ARP)**. Un nœud du cluster annonce les IPs via ARP sur le LAN.
Simple, ne nécessite aucune configuration réseau hors cluster, et suffit
pour un cluster mono-nœud sur un réseau libvirt.

Le mode BGP n'est pas pertinent ici : il nécessiterait un routeur capable
de peering BGP, ce qui n'est pas le cas du réseau virtuel libvirt.

## Pool d'IPs

`192.168.122.200-192.168.122.220` (21 IPs disponibles).

## Dette technique connue

Le réseau libvirt `default` a par défaut une plage DHCP qui couvre
`.2-.254`, ce qui chevauche notre pool MetalLB. En pratique, ce lab n'a
qu'une VM et ce risque est accepté.

Pour résoudre proprement : éditer `virsh net-edit default` pour réduire la
plage DHCP à `.2-.199`, redémarrer le réseau libvirt.

## Vérification

```bash
kubectl get pods -n metallb-system
kubectl get ipaddresspools.metallb.io -n metallb-system
kubectl get l2advertisements.metallb.io -n metallb-system
```

## Test rapide

```bash
kubectl create deployment nginx-test --image=nginx
kubectl expose deployment nginx-test --port=80 --type=LoadBalancer
kubectl get svc nginx-test  # doit afficher une EXTERNAL-IP dans le pool
curl http://<EXTERNAL-IP>   # doit renvoyer la page nginx

kubectl delete deployment nginx-test
kubectl delete svc nginx-test
```

## Sur EKS

Aucune. MetalLB est remplacé par AWS Load Balancer Controller, qui
provisionne de vrais ALB/NLB au lieu d'annoncer des IPs en L2.
