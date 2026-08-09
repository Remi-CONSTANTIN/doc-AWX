# DISCLAIMER
- Testé sur Debian 13 mais devrais fonctionner sur les autres distributions récentes
- Il ne sera pas abordé ici la construction de zero de l'architecture d'un EE mais simplement comment le builder et l'importer sur son instance AWX

# Contexte
Il arrive que certains playbooks/roles nécessitent des modules spécifiques mais que les EE par défaut d'AWX n'incluent pas les bons paquets/dépendances/modules. Il est donc important de comprendre comment utiliser les EE.
  
# Objectif
Comme précisé dans le disclaimer, l'objectif ici est de fournir un exemple du processus de build et d'import d'un EE sur une instance AWX. Nous verrons le cas de l'utilisation d'un service de dépôt externe. Cela devrait être plus ou moins la même chose pour un dépôt interne.

# Prérequis
- Une Instance AWX opérationnelle
- L'accès SSH sur un machine Linux possédant Docker et Ansible

# Procédure
## Builder son image
1. Se connecter à un serveur qui possède ansible et docker
2. Nous allons utiliser la commande "ansible-builder" nécessitant la bibliothèque python `ansible-builder`
```
pip install ansible-builder
```
La commande est issue du paquet `apt install pip`

> [!warning]
> Il se peut que vous ayez un échec vous signalant qu'il vaut mieux travailler dans un environnement python virtuel. Si vous voulez simplement faire des tests sur une machine de test, alors forcez l'installation avec `pip install ansible-builder --break-system-packages`

3. Vérifier la version de l'outil
```
ansible-builder --version
```

4. Créez un dossier de travail où vous allez placer vos fichiers de configuration pour l'EE

5. Si vous avez déjà vos ressources alors importer vos 4 fichiers sur votre serveur via l'outil que vous désirez :
- `bindep.txt`
- `execution-environment.yml`
- `requirements.txt`
- `requirements.yml`
ou rédigez les maintenant

5. Pour lancer le build de l'image à partir de vos fichiers
```
ansible-builder build -t mon-awx-ee:latest
```
Cela va prendre plusieurs minutes en fonction de votre réseau et du hardware de votre machine

7. Une fois le processus terminé, vous avez un nouveau dossier `context` contenant votre image buildée
Vous pouvez aussi vérifier avec la commande :
```
docker images
```
Vous devriez avoir votre image dans la liste

## Envoyer son image sur `docker.com`
Nous verrons ici comment envoyer gratuitement son image sur `docker.com`

1. Se rendre sur `https://hub.docker.com/` et se créer un compte gratuit

2. Rendez-vous dans la partie `Repositories` de votre compte et créez le dépôt `awx-ee`

> [!NOTE]
> Vous pouvez choisir de le mettre en privé, cela ajoutera juste quelques étapes lors de la configuration du pull par AWX. Recommandé en PROD

3. Ensuite loguez vous sur la machine où vous avez buildé l'image
```
docker login
```
Vous allez devoir vous authentifier via connexion web

3. Docker a besoin que l'image soit "étiquetée" (tagguée) avec l'adresse du registre. 
Dans mon cas ça donne (adaptez à votre nom de compte) :
```
docker tag mon-ee-proxmox:latest tadaron/awx-ee:latest
```

4. Envoyez l'image sur votre dépôt (adaptez à votre nom de compte)
```
docker push tadaron/awx-ee:latest
```  
<img width="1313" height="735" alt="awx-ee_push_on_dockerhub" src="https://github.com/user-attachments/assets/cf46cb7e-470d-4f92-b848-05a73f1a223a" />


Vous n'avez plus qu'à attendre

5. Vous devez maintenant avoir votre image sur votre dépôt ! 
<img width="598" height="444" alt="awx-ee_repo_docker" src="https://github.com/user-attachments/assets/493ba49c-6a7d-4900-9f20-25285b8db147" />



## Pull l'image sur son AWX
1. Si vous avez configuré votre dépôt en `privé`, restez sur `docker.com` et créez un token d'accès à votre compte pour AWX dans `https://app.docker.com/accounts/<votre-username>/settings/personal-access-tokens`.
Sauvegardez votre token dans un endroit sûr, choisissez la date d'expiration que vous souhaitez.

2. Dans le cas où vous avez créé un token, rendez-vous dans votre AWX, dans la partie `Informations d'identification` afin de créer une entrée pour vos identifiants `docker.com`

Il vous faudra compléter à minima, ces champs :
- **Nom**
- **Type d'informations d'identification** : Registre de conteneurs
- **URL d'authentification** : `https://index.docker.io/v1/`
- **Nom d'utilisateur** : Nom d'utilisateur de votre compte docker
- **Mot de passe ou Jeton** : Votre token
- **Option** : Laissez cocher la vérification SSL (utile dans le cas où vous autohébergez votre registre)

3. Allez maintenant dans votre AWX, dans la partie `Environnement d'execution` et créez un nouvel environnement

Encore une fois, plusieurs informations vous sont demandées :
- **Nom**
- **Image** : `docker.io/<votre-username>/<votre-depot>:<votre-tag>`
	- (Ex :`docker.io/tadaron/awx-ee:latest`)
- **Extraire** : Extraire le conteneur avant tout exécution (conseillé si vous voulez vous assurer de toujours avoir la dernière version)
- **Information d’identification au registre** : Choisissez le token que vous venez de créez si votre dépôt est privé

Plus qu'à tester dans un modèle ! Il vous suffit de sélectionner le nouvel environnement dans la configuration de celui-ci !
