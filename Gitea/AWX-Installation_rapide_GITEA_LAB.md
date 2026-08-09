# DISCLAIMER
- Testé sur Rocky Linux 10 et Debian 13 mais devrais fonctionner sur les autres distributions récentes

# Contexte
Dans le cadre du déploiement d'AWX via k3s, nous allons avoir besoin de stocker nos playbooks. Plusieurs choix sont proposés par AWX mais nous allons utiliser ici un dépôt Git.

# Outils
Ayant une problématique de légèreté (RAM et CPU essentiellement), nous ne nous orienterons pas vers Gitlab qui consomme aux environs de 8GB RAM au repos mais plutôt vers Gitea, une alternative beaucoup plus légère écrite en GO (environ 512MB RAM au repos).

Vous passerez par HELM pour télécharger et installer Gitea. Vous verrez que l'installation est rapide.
[HELM est le gestionnaire de paquet pour Kubernetes](https://helm.sh/).

# Procédure

## Installation HELM
L'installation est extrêmement rapide et fait en une seule étape via un script qui installe la dernière version en date.
```
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-4 | bash`
```

## Installation Gitea

1. Ajoutez le dépôt de Gitea via HELM et mettez à jour les sources
```
helm repo add gitea-charts https://dl.gitea.com/charts/
helm repo update
```

2. Créez le namespace dédié à Gitea
```
kubectl create namespace gitea
```

3. Créez le fichier `gitea-values.yaml` qui va définir les paramètres de notre pod

> [!warning]
> Cette version part du principe que vous n'allez pas utiliser de nom de domaine pour accéder à votre Git et que vous allez passer par son IP

**Fais attention à remplacer les IP par les tiennes !**
```
gitea:
  config:
    APP_NAME: "Gitea: Mon AWX Git"
    server:
      # Remplace par l'IP de ton serveur K3s
      DOMAIN: 192.168.1.50
      ROOT_URL: http://192.168.1.50:30080/
      # Port SSH exposé via NodePort
      SSH_DOMAIN: 192.168.1.50
      SSH_PORT: 30022
      # On désactive le HTTPS interne pour simplifier (vu qu'on est en local)
      PROTOCOL: http
    database:
      DB_TYPE: postgres

# On désactive l'Ingress car on passe par l'IP directe
ingress:
  enabled: false

# On configure le Service en NodePort
service:
  http:
    type: NodePort
    nodePort: 30080
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
```

4. Plus qu'à lancer le déploiement à partir du fichier
```
helm install gitea gitea-charts/gitea -n gitea -f gitea-values.yaml
```

5. Pour surveiller l'avancée de la création du pod, une commande : `kubectl get pods -n gitea -w`

**Vous devriez avoir ces pods :**
(Ne faites pas attention à leur âge, j'ai fais ce tutoriel en plusieurs fois)  
<img width="454" height="117" alt="stack_gitea_awx" src="https://github.com/user-attachments/assets/c442919c-977f-4979-8f0b-1e77eaa020f4" />  

6. Le Gitea est maintenant accessible à l'URL : `http://<votre-IP>:3080`
Pour y accéder en SSH, connectez vous au port `3022`.

> [!NOTE]
> A la première connexion web, il vous faudra créer le compte administrateur

### Identifiants par défaut
Les identifiants par défaut avec l'installation Helm sont :
`Nom d'utilisateur : gitea_admin`
Mot de passe : `r8sA8CPHD9!bt6d`
