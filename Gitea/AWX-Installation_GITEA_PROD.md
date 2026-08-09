# DISCLAIMER
- Testé sur Rocky Linux 10 et Debian 13 mais devrais fonctionner sur les autres distributions récentes

# Contexte
Dans le cadre du déploiement d'AWX via k3s, nous allons avoir besoin de stocker nos playbooks. Plusieurs choix sont proposés par AWX mais nous allons utiliser ici un dépôt Git.

# Outils
Ayant une problématique de légèreté (RAM et CPU essentiellement), nous ne nous orienterons pas vers Gitlab qui consomme aux environs de 8GB RAM au repos mais plutôt vers Gitea, une alternative beaucoup plus légère écrite en GO (environ 512MB RAM au repos).

Vous passerez par HELM pour télécharger et installer Gitea. Vous verrez que l'installation est rapide.
[HELM est le gestionnaire de paquet pour Kubernetes](https://helm.sh/).

# Prérequis
Pour que l'installation se passe bien, il vous faudra vérifier que les flux vers internet soient ouverts.

Voici la liste des URLs qui seront demandées : 
> [!warning] Attention
> La liste n'est probablement pas exhaustive et sera mise à jour si besoin
##### Installation HELM
- `https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-4 | bash`
	- `https://get.helm.sh/helm4-latest-version`
	- `https://get.helm.sh/helm-v3.14.0-linux-amd64.tar.gz`
	- `https://get.helm.sh/helm-v3.14.0-linux-amd64.tar.gz.sha256`
	- `https://raw.githubusercontent.com/helm/helm/main/KEYS`
	- `https://github.com/helm/helm/releases/download/v3.14.0/helm-v3.14.0-linux-amd64.tar.gz.asc`
	- `https://github.com/helm/helm/releases/download/v3.14.0/helm-v3.14.0-linux-amd64.tar.gz.sha256.asc`
##### Installation Gitea
- `https://dl.gitea.com/charts/`
	- `https://charts.bitnami.com/bitnami`
	- `https://docker.io`
	- `https://registry-1.docker.io`
	- `https://production.cloudflare.docker.com`
et probablement plus …

# Procédure

## Préparation cluster
Afin d'éviter d'avoir l'erreur suivante à l'étape 7 de l'installation de Gitea il nous faut faire une configuration de Kubernetes  
<img width="1265" height="76" alt="cluster_reachability_error" src="https://github.com/user-attachments/assets/7e92d366-0ed4-4cae-b561-901353e0d8e9" />  

1. Définissez la variable rapportant le chemin vers la configuration K3S
```
echo 'export KUBECONFIG=/etc/rancher/k3s/k3s.yaml' >> ~/.bashrc
```
2. Appliquez les changements du fichier 
```
source ~/.bashrc
```

> [!warning]
> Ce paramètre n'est appliqué que sur l'utilisateur courant, dans mon cas `root`

## Installation HELM
L'installation est extrêmement rapide et fait en une seule étape via un script qui installe la dernière version en date.
```
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-4 | bash
```

## Installation Gitea

1. Créez le namespace Gitea
```
kubectl create namespace gitea
```

2. Créez le fichier de configuration `gitea-middleware-https.yaml` du middleware HTTPS pour Gitea avec le contenu suivant : 
```
apiVersion: traefik.io/v1alpha1
kind: Middleware
metadata:
  name: gitea-redirect-https
  namespace: gitea
spec:
  redirectScheme:
    scheme: https
    permanent: true
```

3. Appliquez le
```
kubectl apply -f gitea-middleware-https.yaml
```

4. Créez le fichier de configuration global de la stack Gitea `gitea-values.yaml`
```
gitea:
  config:
    APP_NAME: "Gitea: Mon AWX Git"
    server:
      DOMAIN: git.rocky-dvd2.tadaron    # Remplacez par votre enregistrement DNS
      ROOT_URL: https://git.rocky-dvd2.tadaron/
      PROTOCOL: http       # Traefik gère le SSL, Gitea reçoit du HTTP en interne
      SSH_DOMAIN: 10.34.33.76   # IP de votre serveur k3s
      SSH_PORT: 30022
    database:
      DB_TYPE: postgres
    # Désactive l'inscription pour éviter les utilisateurs non autorisés
    service:
      DISABLE_REGISTRATION: true
      ALLOW_ONLY_EXTERNAL_REGISTRATION: false
      SHOW_REGISTRATION_BUTTON: false      
ingress:
  enabled: true
  className: traefik
  annotations:
    # Utilise l'émetteur de certificat auto-signé créé pour AWX
    cert-manager.io/cluster-issuer: selfsigned-issuer
    # Lie le middleware de redirection
    traefik.ingress.kubernetes.io/router.middlewares: gitea-gitea-redirect-https@kubernetescrd
  hosts:
    - host: git.rocky-dvd2.tadaron
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: gitea-tls-secret
      hosts:
        - git.rocky-dvd2.tadaron
service:
  ssh:
    type: NodePort
    nodePort: 30022
persistence:
  enabled: true
  size: 10Gi
postgresql:
  enabled: true
  persistence:
    enabled: true
    size: 5Gi
# CRUCIAL : Désactive le mode HA pour éviter les erreurs d'installation
postgresql-ha:
  enabled: false
```

> [!warning] Attention
> Veillez à adapter l'IP et le hostname de votre machine dans le fichier !

> [!NOTE]
> Vous remarquerez qu'on désactive la HA de postgres car cela est utile seulement si nous avons plusieurs noeud dans le cluster Kubernetes, ce qui n'est pas notre cas

5. Ajoutez le dépôt HELM de Gitea
```
helm repo add gitea-charts https://dl.gitea.com/charts/
```

6. Mettez le à jour
```
helm repo update
```

7. Lancez l'installation à partir du dépôt et du fichier de configuration `gitea-values.yaml`
```
helm install gitea gitea-charts/gitea -n gitea -f gitea-values.yaml
```

8. Vérifiez l'état des pods, leur création n'est pas instantanée
```
kubectl get pods -n gitea -w
```

Vous devriez avoir ces pods :  
<img width="415" height="96" alt="stack_gitea_awx_prod" src="https://github.com/user-attachments/assets/cc7226ab-ac37-486e-9829-e3a0badbf33d" />


## Configuration DNS
Afin d'avoir accès à votre Gitea depuis une machine distante (ce qui est souvent le cas), il vous faudra créer un enregistrement DNS (type A) pointant sur votre nœud K3S.

La méthode dépendra de votre matériel mais voici un exemple sur un Adguard :  
<img width="1197" height="697" alt="dns_record_gitea_awx 1" src="https://github.com/user-attachments/assets/217d2649-07a5-4b5c-8f84-ff47df629022" />


## Accès à Gitea
Une fois toutes ces étapes passées, vous devriez avoir accès à votre Gitea à l'URL :  `https://<git.hostname.domain>/`
Si besoin le ssh est accessible à l'adresse : 
`ssh://git@<IP-noeud-K3S>:30022`

Exemple : 
`https://git.rocky-dvd2.tadaron/`
`ssh://git@10.34.33.76:30022`

#### Identifiants par défaut
Les identifiants par défaut avec l'installation Helm sont :
`Nom d'utilisateur` : gitea_admin
`Mot de passe` : r8sA8CPHD9!bt6d

## (Optionnel) - Ajout configuration AWX pour configuration projet Gitea
Dans le cas où vous déployez Gitea pour l'utiliser dans AWX (pour héberger des projets par exemple), il vous faudra ajouter quelques configuration à votre AWX.

#### Ajout 1 : Résolution statique `git.hostname.domain`
Si vous tentez de créer un dépôt Gitea et de l'ajouter dans un "projet", AWX n'arrivera pas à s'y connecter car il n'arrivera pas à résoudre votre enregistrement DNS `<git.hostname.domain>` 
Il semblerait que même en ayant configuré des enregistrements DNS sur nos serveurs les pods AWX n'arrivent pas à résoudre `<git.hostname.domain>` car ceux-ci utilisent le "Core-DNS" de K3S.

 Afin de contourner ce problème, nous allons modifier notre AWX en ajoutant une résolution statique dans sa configuration.
 
1. Ouvrez votre fichier de déploiement AWX. Si vous avez suivis mon tutoriel il devrait s'appeler `awx-instance.yml`
2. Ajoutez le bloc suivant juste après `spec` : 
```
  host_aliases:
    - ip: "192.168.1.76"
      hostnames:
        - "git.<votre_hostname>.<votre_domaine>"
```
Visuellement ça donne : 
<img width="605" height="274" alt="static_resolution_git_awx 1" src="https://github.com/user-attachments/assets/9b0d9721-e8cd-4bc9-a745-b1600f8755b5" />


3. Plus qu'à appliquer et attendre une petite minute le temps que les pods redémarrent
```
kubectl apply -f awx-instance.yml -n awx
```

4. Comme d'habitude on peut surveiller l'activité des pods
```
kubectl get pods -n awx -w
```

5. Pour vérifier que la résolution fonctionne
```
kubectl exec -it deployment/awx-web -n awx -- getent hosts <git.hostname.domain>
```
Exemple :
`kubectl exec -it deployment/awx-web -n awx -- getent hosts git.rocky-dvd2.tadaron`  
<img width="727" height="31" alt="git_dns_resolution_awx" src="https://github.com/user-attachments/assets/bd42d508-02e7-4c3c-8b7f-e5f8ab382355" />



#### Ajout 2 : Ignorer vérification certificat Gitea
Si vous tentez de créer un projet dans AWX en utilisant le type "Git" vous aurez un problème de certificat : 
<img width="1501" height="445" alt="gitea_certificat_verification_awx" src="https://github.com/user-attachments/assets/323c9536-6495-4312-8daa-d76e6acdcc93" />


Sur la version `24.6.1` d'AWX, il ne semble pas avoir de boutons pour désactiver cette vérification.
Il nous faut donc encore modifier le fichier de configuration d'AWX afin de la désactiver.

1. Modifier notre fichier `awx-instance` (si vous avez suivis mon tutoriel) et ajouter ce bloc dans la partie `extra_settings:`:
```
    - setting: AWX_TASK_ENV
      value:
        GIT_SSL_NO_VERIFY: "true"
```
Visuellement ça donne :  
<img width="509" height="178" alt="ignore_gitea_certificat_awx" src="https://github.com/user-attachments/assets/3911e52b-8aa9-434b-977d-78ae3c251bea" />


2. Appliquez les changements, cela va recréer les pods automatiquement
```
kubectl apply -f awx-instance.yml -n awx
```

3. Surveillez le processus
```
kubectl get pods -n awx -w
```

Vous pouvez maintenant faire un projet de type "git" dans votre AWX en suivant la procédure [AWX - Création projet avec Git](https://github.com/tadaron/Mon-Home-Lab/blob/main/Infra/1_Services/AWX/Gitea/AWX-Cr%C3%A9ation_projet_avec_Git.md)
