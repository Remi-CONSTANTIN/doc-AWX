# Présentation
Vous trouverez ici ma documentation sur la découverte/installation/configuration de la solution AWX.

# Pourquoi AWX ?
Ansible en CLI ne permet pas la collaboration entre équipe et la segmentation des droits.
AWX permet d'avoir une interface graphique pour exécuter et utiliser ansible efficacement.

# Méthode de déploiement
AWX se déploie via K3s. Un memo des commandes utiles est disponible dans [[K3S - Memo commandes]].

# Recommandations système
Il faut prévoir assez de ressources pour le bon fonctionnement du système et prévoir le fait qu'AWX est plutôt gourmand même au repos.

**Consommation RAM au repos :**
- OS : 1 Go RAM
- Pods AWX : 2,5 Go RAM
Pour une machine avec 8Go, on consomme donc pas loin de la moitié de la RAM rien qu'au repos.

**Pour de la PROD, prévoyez dans votre machine hôte :** 
- 4VCPU
- 8 Go RAM 
- 70 Go de disque dont 40Go pour le système (partitionnement au choix) et /var/lib/rancher de 30Go (prends 6Go rien qu'à l'installation et augmente vite)


# Politique IA
Je ne suis pas un puriste totalement contre l'utilisation de l'IA mais seulement quelqu'un de modéré ayant pesé le pour et le contre.
Vous trouverez donc dans ce dépôt, de la documentation rédigée avec l'aide de Gemini mais rien ici n'a été bêtement copié sans être relu et testé par une intelligence humaine.
Si quelque chose a été rédigée par une IA vous retrouverez sa source en dessous.

# Ressources utiles
- **Github AWX** : https://github.com/ansible/awx
- **Documentation installation basique** : https://docs.ansible.com/projects/awx-operator/en/latest/installation/basic-install.html
- **Github AWX Operato**r : https://github.com/ansible/awx-operator/tree/devel
