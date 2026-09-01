# Présentation
Vous trouverez ici ma documentation sur la découverte/installation/configuration de la solution AWX.

# Pourquoi AWX ?
Ansible en CLI ne permet pas la collaboration entre équipe et la segmentation des droits.
AWX permet d'avoir une interface graphique pour exécuter et utiliser ansible efficacement.

# Parcours recommandé
0. Commencez par comprendre l’intérêt d'AWX  
[AWX_VS_Ansible](AWX_VS_Ansible.md)
1. Chose faite, le premier contact avec la technologie peut se faire en déployant une instance de test    
[AWX/AWX-Installation_rapide_LAB](AWX/AWX-Installation_rapide_LAB.md)
2. Complétez votre expérience en déployant un Gitea afin d’héberger vos YAML  
[Gitea/AWX-Installation_rapide_GITEA_LAB](Gitea/AWX-Installation_rapide_GITEA_LAB.md)
3. Une fois la technologie testée et appréhendée, déployez le en PROD  
[AWX/AWX-Installation_PROD](AWX/AWX-Installation_PROD.md)
4. Déployez Gitea avec un tutoriel plus adapté à la PROD  
[Gitea/AWX-Installation_GITEA_PROD](Gitea/AWX-Installation_GITEA_PROD.md)

**(Optionnel)**
1. Mettez en supervision votre nœud Kubernetes si vous possédez un serveur Checkmk  
[K3S/K3S-Supervision_via_Checkmk](K3S/K3S-Supervision_via_Checkmk.md)
3. Connectez votre AWX à l'AD afin de faciliter/sécuriser la gestion des droits  
[AWX/Synchro_LDAPS_AD](AWX/Synchro_LDAPS_AD.md)

# Recommandations système
Il faut prévoir assez de ressources pour le bon fonctionnement du système et prévoir le fait qu'AWX est plutôt gourmand même au repos.

**Consommation RAM au repos :**
- OS : 1 Go RAM
- Pods AWX : 3 Go RAM
Pour une machine avec 8Go, on consomme donc environ la moitié de la RAM rien qu'au repos

**Pour de la PROD, prévoyez dans votre machine hôte :** 
- 4VCPU
- 8 Go RAM 
- 70 Go de disque dont 40Go pour le système (partitionnement au choix) et `/var/lib/rancher` de 30Go (prends 6Go rien qu'à l'installation et augmente vite)

# Politique IA
Vous trouverez donc dans ce dépôt, de la documentation rédigée avec l'aide de Gemini mais rien ici n'a été bêtement copié sans être relu et testé par une intelligence humaine.
Si quelque chose a été rédigée par une IA vous retrouverez sa source en dessous.

# Ressources utiles
- **Github AWX** : https://github.com/ansible/awx
- **Documentation installation basique** : https://docs.ansible.com/projects/awx-operator/en/latest/installation/basic-install.html
- **Github AWX Operato**r : https://github.com/ansible/awx-operator/tree/devel
