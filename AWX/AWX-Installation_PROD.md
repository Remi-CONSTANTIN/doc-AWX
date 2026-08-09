# DISCLAIMER
- Testé sur Rocky Linux 10 et Debian 13 mais devrait fonctionner sur les autres distributions récentes

# Fonctionnalités
Contrairement à la procédure de déploiement d'AWX pour un LAB, cette documentation inclue les fonctionnalités suivantes : 
- Interface Web HTTPS (certificat autosigné)
- Limitation des ressources des pods AWX (afin d'éviter de faire crash la machine hôte)

# Prérequis
Afin de facilité l'installation, il vous faudra désactiver certaines fonctionnalités de sécurité. Vous pourrez les réactiver après.

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
mkdir -p awx-operator && cd awx-operator
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

## Installation de cert-manager (sans Helm)

1. L'installation de la stack cert-manager se fait en une bête commande
```
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.17.0/cert-manager.yaml
```

2. Vérifiez que les pods sont en état Running avant de continuer
```
kubectl get pods -n cert-manager
```
Le déploiement des pods devrait prendre quelques secondes tout au plus  

<img width="545" height="70" alt="cert-manager" src="https://github.com/user-attachments/assets/a75dd12e-2d62-4d2d-86b8-8d266b441e6e" />  

## Création de notre propre autorité de certification
Création de notre propre autorité de certification car nous ne passerons pas par lets-encrypt.
C'est une machine locale, non exposé au web.

1. Créez le fichier `cluster-issuer.yaml` pour le déploiement
```
# cluster-issuer.yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: selfsigned-issuer
spec:
  selfSigned: {}
```

2. Comme d'habitude appliquez le fichier pour créer les ressources
```
kubectl apply -f cluster-issuer.yaml
```

## Configurer la redirection HTTPS Traefik
Créez un middleware pour forcer la redirection automatique du trafic HTTP vers HTTPS.

1. Créez le fichier `middleware-https.yaml`
```
# middleware-https.yaml
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: redirect-https
  namespace: awx
spec:
  redirectScheme:
    scheme: https
    permanent: true
```

2. Appliquez le
```
kubectl apply -f middleware-https.yaml
```

3. Vérifiez que le middleware soit bien déployée
```
kubectl get middleware -n awx
```
<img width="445" height="43" alt="middleware-traefik-https-awx" src="https://github.com/user-attachments/assets/c6b36707-1888-4601-8f4e-c76e3a316da3" />


## Installation d'AWX (HTTPS)
Comme expliqué précédemment, cette version du fichier de déploiement d'AWX comporte la mise en place du HTTPS via Ingress.

> [!warning]
> Notre AWX sera accessible via un enregistrement DNS (hostname.domain), donnant l'URL : `https://hostname.domain`

1. Créez un enregistrement DNS pour votre machine dans votre serveur/appareil réseau
Dans mon cas je créé mon enregistrement dans mon routeur

<img width="282" height="605" alt="enregistrement_dns_awx" src="https://github.com/user-attachments/assets/19d1191b-ba68-4ea2-8955-3229d8410e58" />


Vérifiez que celui-ci fonctionne avec les commandes nslookup et/ou ping.
Si vous êtes là, je ne vais pas vous apprendre à les utiliser...

2. Créez le fichier de définition `awx-instance.yml`
```
apiVersion: awx.ansible.com/v1beta1
kind: AWX
metadata:
  name: awx
  namespace: awx
spec:
  # --- Configuration de l'accès réseau (HTTPS via Ingress) ---
  ingress_type: ingress
  ingress_hosts:
    - hostname: rocky-dvd2.tadaron # Ton enregistrement DNS
      tls_secret: awx-tls-secret   # Géré par cert-manager
  
  # Annotations pour lier cert-manager et le middleware Traefik
  ingress_annotations: |
    cert-manager.io/cluster-issuer: selfsigned-issuer
    traefik.ingress.kubernetes.io/router.middlewares: awx-redirect-https@kubernetescrd

  # --- Correctifs de sécurité pour Django (essentiel en HTTPS) ---
  extra_settings:
    - setting: CSRF_TRUSTED_ORIGINS
      value:
        - https://rocky-dvd2.tadaron
    - setting: SECURE_PROXY_SSL_HEADER
      value:
        - HTTP_X_FORWARDED_PROTO
        - https

  # --- Optimisation des Ressources (Spécifique 8Go RAM / 4vCPU) ---
  
  # AWX Web (Interface et API)
  web_resource_requirements:
    requests:
      cpu: "250m"
      memory: "1300Mi"
    limits:
      cpu: "1000m"
      memory: "2Gi"

  # AWX Task (Moteur Ansible)
  task_resource_requirements:
    requests:
      cpu: "500m"
      memory: "900Mi"
    limits:
      cpu: "1500m"
      memory: "3Gi"

  # PostgreSQL (Base de données)
  postgres_resource_requirements:
    requests:
      cpu: "250m"
      memory: "512Mi"
    limits:
      cpu: "1000m"
      memory: "2Gi"

  # Redis (Cache et Queues)
  redis_resource_requirements:
    requests:
      cpu: "100m"
      memory: "128Mi"
    limits:
      cpu: "200m"
      memory: "512Mi"
```

> [!tip]
Le type `ingress` permet d’exposer une application via **une URL ou un nom de domaine** plutôt qu’un port spécifique.  
Il agit comme un **point d’entrée HTTP/HTTPS** vers les services du cluster.  
Cela permet par exemple d’accéder à l’interface Web d’AWX avec une adresse comme `awx.mondomaine.local`.

2. Appliquez la configuration à partir du fichier `awx-demo.yml`
```
kubectl apply -f awx-instance.yml -n awx
```

> [!NOTE] Pause café
> Ne soyez pas pressé. La création des pods (base de données, redis, web, task) peut prendre plusieurs minutes.

Vous devriez avoir ces pods dans l'état `running`.
(Ne faites pas attention à leur âge, j'ai fais ce tutoriel en plusieurs fois)

3. Encore et toujours la même commande
```
kubectl get pods -n awx -w
```
Petite variante, le `-w` permet une sorte de live-view (en réalité ce n'est pas si pratique mais je vous laisse tester)
<img width="634" height="100" alt="awx-instance-pods-https" src="https://github.com/user-attachments/assets/84052950-88fe-477b-b134-c879ad0e292a" />

5. Une fois créé, il vous faut trouver les identifiants et le port de l'interface Web AWX, pour ce faire exécutez cette commande pour les identifiants
```
kubectl get secret awx-admin-password -n awx -o jsonpath="{.data.password}" | base64 --decode; echo
```

> [!warning]
> Pensez à le changer dès lors que vous vous serez connecté à l'interface Web !

4. Vous avez maintenant tout ce qu'il faut pour utiliser AWX à l'URL suivante : `https://<hostname.domain>`
<img width="1278" height="676" alt="awx-web-interface-https" src="https://github.com/user-attachments/assets/d5b8adfe-363e-499a-a8fe-d2715ace54e2" />

**Amusez vous bien !**

---
# Sources
[Basic Install AWX](https://docs.ansible.com/projects/awx-operator/en/latest/installation/basic-install.html)

---
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

