# DISCLAIMER
- Testé sur Rocky Linux 10 et Debian 13 mais devrais fonctionner sur les autres distributions récentes

# Prérequis

1. Désactivez votre pare-feu local (ufw, firewalld ou autres...)
```
systemctl disable --now <votre-firewall>
```

2. Désactivez SE-linux (pour Rocky Linux notamment)
Pour ce faire éditez le fichier de config
```
nano /etc/selinux/config
```
et repérez la ligne `SELINUX=` pour y mettre la valeur `disabled`
```
SELINUX=disabled
```

3. Prévoyez au moins 20Go dans votre `/var` car k3s stocke énormément de données dans `/var/lib/rancher` (les volumes notamment)

# Procédure
## Installation K3S
Rancher fournis un script d'installation pour installer rapidement K3S

1. Exécutez le script
```
curl -sfL https://get.k3s.io | sh -
```

2. Vérifiez que votre machine héberge maintenant un nœud
```
kubectl get nodes
```

## Installation AWX Operator
L'opérateur est un gestionnaire qui automatise le déploiement, la configuration et la mise à jour d'AWX.

1. Créez le répertoire de travail et placez-vous dedans
```
mkdir -p awx-operator
cd awx-operator
```

2. Créez le fichier ``kustomization.yml`` avec le contenu suivant :
```
# kustomization.yml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - github.com/ansible/awx-operator/config/default?ref=2.19.1

namespace: awx

images:
  - name: quay.io/ansible/awx-operator
    newTag: 2.19.1
  - name: gcr.io/kubebuilder/kube-rbac-proxy
    newName: quay.io/brancz/kube-rbac-proxy
    newTag: v0.21.0
```

> [!NOTE]
> Vous remarquerez que nous remplaçons ici la source du pod `kube-rbac-proxy`, en passant du dépôt Google `gcr.io/kubebuilder/kube-rbac-proxy` à `quay.io/brancz/kube-rbac-proxy` car la source n'existe plus à l'heure où j'écris cette procédure

6. Installez git
```
apt install git -y
```

7. Appliquer la configuration à partie du fichier ``kustomization.yml``
```
kubectl apply -k .
```

7. Vérifier que l'Operator soit créé ou en cours de création
```
kubectl get pods -n awx
```
Attendez que le pod `awx-operator-controller-manager` soit en état `Running`

> [!tip]
> Si l'awx-operator ne démarre pas et reste en running, allez dans la partie TroubleShooting

## Installation d'AWX (enfin)

1. Créez le fichier de définition `awx-demo.yml`
```
---
apiVersion: awx.ansible.com/v1beta1
kind: AWX
metadata:
  name: awx-demo
spec:
  service_type: nodeport
```

> [!tip]
> Le type `nodeport` permet d'accéder à AWX via l'IP de votre serveur sur un port spécifique.
> Cela nous permettra après de demander à K3S sur quel port nous pouvons accéder à l'interface Web AWX.
> /!\ Il est possible de le définir manuellement

2. Appliquez la configuration à partir du fichier `awx-demo.yml`
```
kubectl apply -f awx-demo.yml -n awx
```

> [!NOTE] Pause café
> Ne soyez pas pressé. La création des pods (base de données, redis, web, task) peut prendre plusieurs minutes.

Vous devriez avoir ces pods dans l'état `running`.
(Ne faites pas attention à leur âge, j'ai fais ce tutoriel en plusieurs fois)

<img width="656" height="117" alt="stack_awx" src="https://github.com/user-attachments/assets/e25182a5-7828-4e4d-9364-d89ef9509391" />


3. Une fois créé, il vous faut trouver les identifiants et le port de l'interface Web AWX, pour ce faire exécutez cette commande pour les identifiants
```
kubectl get secret awx-demo-admin-password -n awx -o jsonpath="{.data.password}" | base64 --decode; echo
```
Et cette commande pour le port
```
kubectl get svc -n awx awx-demo-service
```

4. Vous avez maintenant tout ce qu'il faut pour utiliser AWX à l'URL suivante : `http://<IP_NODE>:<PORT_NODEPORT>`

<img width="1500" height="852" alt="AWX_login_web" src="https://github.com/user-attachments/assets/be6b2c3c-80d4-417c-99d7-cf4423a06508" />


**Amusez vous bien !**

---

# Sources
[Basic Install AWX](https://docs.ansible.com/projects/awx-operator/en/latest/installation/basic-install.html)

# Annexes

## Explication du ``kustomization.yml``

```
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
```
Indiquent à `kubectl` que ce n'est pas un pod ou un service classique, mais un fichier de configuration **Kustomize**. C'est ce qui permet d'utiliser la commande `kubectl apply -k` (le `-k` signifiant kustomize).

```
resources:
  - github.com/ansible/awx-operator/config/default?ref=2.19.1
```
C'est ici que la magie opère. Au lieu d'avoir 50 fichiers YAML sur votre disque, vous dites à Kustomize :
- **Va chercher** les fichiers de configuration directement sur le GitHub officiel de l'AWX Operator.
- **Dossier** : Il regarde dans `/config/default`.
- **Version (`?ref=2.19.1`)** : C'est crucial. Vous verrouillez l'installation sur une version précise pour éviter que votre cluster ne change tout seul si une version 3.0 sort demain.

```
images:
  - name: quay.io/ansible/awx-operator
    newTag: 2.19.1
```
Par défaut, le fichier téléchargé sur GitHub pointe peut-être vers une image générique ou une version "latest".
- **name** : Kustomize scanne tous les fichiers récupérés sur GitHub à la recherche de cette image précise.
- **newTag** : Dès qu'il la trouve, il remplace le tag par `2.19.1`. Cela garantit que l'opérateur qui tourne dans votre K3s est exactement de la même version que les fichiers de configuration que vous avez téléchargés juste au-dessus.

```
namespace: awx
```
C'est une sécurité très puissante. Cette ligne dit à Kustomize : **"Peu importe ce qui est écrit dans les fichiers sur GitHub, force le déploiement de TOUTES les ressources dans le namespace `awx`."** Cela vous évite de polluer le reste de votre cluster K3s et permet de tout supprimer d'un coup avec un `kubectl delete namespace awx` si besoin.

---

