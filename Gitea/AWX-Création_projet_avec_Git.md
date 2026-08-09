# DISCLAIMER
- Testé sur Rocky Linux 10 et Debian 13 mais devrais fonctionner sur les autres distributions récentes

# Contexte
Dans une optique de segmentation et notamment de facilité de la gestion des accès, AWX permet la création de "projets". Je dirais qu'un projet est un dépôt (pas au sens Git) hébergeant des ressources Ansible (notamment des playbooks).

Les projets doivent se baser sur une source qui peut être un répertoire local, un Git, du Red Hat Insight ou autres trucs obscures.

Il sera vu ici la synchronisation d'un Git provenant d'un gitea déployé dans le même k3s spécialement pour l'occasion. Voir [[AWX - Installation rapide GITEA pour LAB]]. 

# Prérequis
- Une instance Gitea (Gitlab fonctionne aussi)
- Un repo contenant à minima un playbook (sinon il y aura une erreur à la connexion dans AWX)
# Procédure

## HTTP
1. Se rendre dans l'onglet "Projets" sur votre AWX à l'URL : `http://<IP>:<port_awx>/#/projects` et cliquer sur "Ajouter"

2. Renseignez :
	1. Nom : Nom que vous voulez donner à ce projet dans AWX
	2. Description
	3. Organisation : Obligatoire, si vous n'en avez pas, créez en une
	4. Environnement d'exécution : A vous de voir
	5. Type de Contrôle de la source : Git dans notre cas
	6. Certificat de validation de la signature du contenu : Optionnel, pas vraiment obligatoire dans notre cas
	7. URL Contrôle de la source : URL au format `http://<ip_gitea>:<port_gitea>/<utilisateur_gitea>/<repo_gitea.git>`
	Exemple : http://10.10.10.147:30080/admin/ansible-awx-projects.git
	8. Branche/ Balise / Commit du Contrôle de la source : Si vous ne travaillez pas dans la branche `Main
	9. Refspec Contrôle de la source et Identifiant Contrôle de la source : Pas obligatoire dans notre cas

<img width="1013" height="589" alt="projet_awx" src="https://github.com/user-attachments/assets/841fb5db-7792-479d-97ab-af2764a7703e" />

3. Une fois que vous avez enregistrer une première tentative de connexion se lance, si vous n'avez pas fait d'erreur de configuration, vous devriez être dans le status "Réussis"

<img width="1003" height="405" alt="synchro_git_projet_awx" src="https://github.com/user-attachments/assets/03f544b7-0e12-4a58-8638-a716c6d230da" />


Vous avez maintenant accès à différents paramètres et notamment le gestion des accès à se projet dans "Accès" et à la planification des synchronisations automatiques dans "Programmations".


## HTTPS
La méthode pour un Gitea en HTTPS est sensiblement la même, à quelques détails près :

1. Git étant en HTTPS autosigné, AWX refusera de se connecter au dépôt, il nous faut donc modifier le déploiement d'AWX afin de désactiver la vérification.

Modifiez donc `awx-instance.yml` pour ajouter le bloc suivant dans la partie `extra_settings` :
```
    - setting: AWX_TASK_ENV
      value:
        GIT_SSL_NO_VERIFY: "true"
```

Visuellement ça donne :  
<img width="236" height="60" alt="git_ssl_no_verify_awx" src="https://github.com/user-attachments/assets/eb4d99e6-6e44-4493-913c-deefa3c8e749" />


2. Bien sûr, il faut appliquer les modifications
```
kubectl apply -f awx-instance.yml
```

3. Comme d'habitude, vérifiez que tout se passe bien
```
kubectl get pods -n awx -w
```

4. Une fois la modification faites, vous pouvez créez vos dépôts dans AWX en changeant juste un paramètre
URL Contrôle de la source : URL au format `https://<hostname>.<domaine>/<utilisateur_gitea>/<repo_gitea.git>`
	Exemple : https://git.k3s.tadaron/admin/ansible-awx-projects.git

> [!caution]
> Veillez à bien avoir fait les étapes de configuration optionnelles d'AWX dans la procédure [AWX - Installation GITEA PROD]([https://pages.github.com/](https://github.com/tadaron/Mon-Home-Lab/blob/main/Infra/1_Services/AWX/Gitea/AWX-Installation_GITEA_PROD.md))


# Troubleshooting

**Erreur typique**
Si vous avez une erreur de synchronisation à la première connexion., vérifiiez que vous avez déjà mis au moins un playbook dans votre repo Git.
