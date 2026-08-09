# Cluster K3S

**Installation**
`curl -sfL https://get.k3s.io | sh -`

**Afficher les nœuds K3S**
`kubectl get nodes`

# COMMANDES K3S

### Pods

**Voir tout le pods de tout les Namespaces**
`kubectl get pods -A`

**Afficher pods d'un namespace**
`kubectl get pods -n <NomDuNamespace>`

### Kustomization.yml

**Appliquer un kustomization.yml provenant du répertoire courant**
`kubectl apply -k .`

**Suppression projet à partir du kustomization.yml provenant du répertoire courant**
`kubectl delete -k .`

### Namespace

**Créer namespace**
`kubectl create namespace <NomDuNamespace>`

**Suppression namespace**
`kubectl delete namespace <NomDuNamespace>`

### Service

**Voir port assigné à un service**
`kubectl get svc <NomDuService> -n <NomDuNamespace>`

## Débogage & Inspection

**Inspecter les détails d'une ressource (très utile pour voir les erreurs)**
`kubectl describe pod <NomDuPod> -n <Namespace>`

**Exécuter une commande dans un Pod (Shell)**
`kubectl exec -it <NomDuPod> -n <Namespace> -- /bin/bash`

**Voir logs**
`kubectl logs -f deployments/awx-operator-controller-manager -c awx-manager -n <NomDuNamespace>`
