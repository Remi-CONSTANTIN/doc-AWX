# DISCLAIMER
- Testé sur Rocky Linux 10 et Debian 13 mais devrais fonctionner sur les autres distributions récentes
  
# Contexte
Nous venons d'installer un nouveau serveur AWX, sa configuration est en cours. Nous sommes donc arrivé à l'étape de création des "Projets" (Dépots de playbooks) pour chaque équipes. Il a été décidé de d'utiliser le type de contrôle à la source dit "Manuel" qui a comme comportement pas défaut d'aller chercher les données dans /var/lib/awx/Projects situé dans le pod "awx-task".
  
# Objectif
Modifier le déploiement d'AWX afin de faire persister les données du dossier /var/lib/awx/Projects du pod awx-task. Cela permeterait de pouvoir utiliser le type de contrôle à la source "Manuel" sans perdre les données à chaque reboot.
Cela permetterait aussi d'utiliser un dossier de l'hôte comme repertoire de travail.
 
# Prérequis
Avoir un AWX déjà installé
 
# Procédure
## Préparation du répertoire sur l'hôte
Créer le dossier sur le nœud hôte dans le répertoire de travail que vous souhaitez. Dans mon cas :
```
mkdir -p /opt/awx-operator/pv-projects
```
 
 
## Créer le PV et le PVC
 
1. Créer le fichier `pv-awx-projects.yaml` :
```
cat > pv-awx-projects.yaml << 'EOF'
apiVersion: v1
kind: PersistentVolume
metadata:
 name: awx-projects-pv
spec:
 capacity:
   storage: 10Gi
 accessModes:
   - ReadWriteOnce
 persistentVolumeReclaimPolicy: Retain
 storageClassName: manual
 hostPath:
   path: /opt/awx-operator/pv-projects   # point de montage sur l'hôte AWX. A modifier à votre convenance
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
 name: awx-projects-pvc
 namespace: awx
spec:
 accessModes:
   - ReadWriteOnce
 storageClassName: manual
 resources:
   requests:
     storage: 10Gi
EOF
```
 
2. Appliquer la configuration :
```
kubectl apply -f pv-awx-projects.yaml
```
 
3. Vérifier que le PV et le PVC sont en `bound` avec les deux commandes :
```
kubectl get pv
kubectl get pvc -n awx
```
 
 
## Modifier le déploiement d'AWX
 
1. Ajouter les deux lignes suivantes dans awx-instance.yml, dans la partie "spec: " en veillant à respecter l'indentation (voir capture d'écran en dessous) :
 ```
 # --- Persistance des donnees ---
 projects_persistence: true
 projects_existing_claim: awx-projects-pvc
 ```
 
> [!warning]
> Le paramètre "projects_existing_claim" est obligatoire. Sans lui, l'Operator crée son propre PVC automatiquement avec la StorageClass par défaut (local-path) et ignore totalement votre PVC manuel
 
2. Appliquer le fichier awx-instance.yml après l'avoir modifié
```
kubectl apply -f awx-instance.yml
```
 
3. Vérifier que les pods passent en "running"
```
kubectl get pods -n awx
```
 
 
## Test
Pour tester nos manipulations nous allons créer du contenu dans le point de montage sur l'hôte et voir si les modifications sont repercutées dans le pod "awx-task"
 
1. Se rendre dans le point de montage que vous avez choisis dans votre "pv-awx-projects.yaml"
```
cd /opt/awx-operator/pv-projects
```
 
2. Créer un dossier "WINDOWS-depot"
```
mkdir WINDOWS-depot
```
 
3. Créer un playbook YML bidon dont vous savez que la syntax est bonne. Dans l'idéal mettez en un que vous pourrez exécuter en local sur l'AWX.
 
4. Affichez la liste de vos pods awx afin d'identifier le nom complet de votre pod "awx-task"
```
kubectl get pods -n awx
```
 
5. Une fois le nom copié, exécutez la commande suivante en l'adaptant afin d'ouvrir un shell dans le pod
```
kubectl exec -it awx-task-6c5749d8c5-pq4vz -n awx -- /bin/bash
```
Puis
```
ls /var/lib/awx/projects/
```
Vous devriez voir exactement le même contenu que sur votre hôte
 
6. Dernier test : redémarrer votre nœud k3s pour voir si vous perdez tout votre travail dans votre pod
 
Si vous avez tout testé et que tout est resté tel quel, BRAVO vous avez réussis !!
 
 
## Créer un projet type "Manuel"
 
1. Se rendre dans l'interface web AWX, dans l'onglet "Projets"
2. Renseigner les informations sur le nouveau projet (Nom, Organisation et type de Contrôle de la source au minimum)
3. Choisissez le répertoire sur lequel votre projet va s'appuyer, nous avons créé WINDOWS-depot plus tôt

> [!Remarque]
> Vous remarquerez que le chemin de base du projet n'est pas modifiable, cela est normal. Il correspond au répertoire dans le pod "awx-Task", c'est pour cela que nous avons rendu ce pod persistant afin de ne pas perdre les playbooks que nous allons y mettre et que nous avons monté ce répertoire sur l'hôte directement ( /opt/awx-operator/pv-projects)

4. Plus qu'à enregistrer et ne pas oublier de configurer les droits (onglet "accès") sur toutes les ressource que vous créez (que ce soit Projet, Inventaire ou Modèles) !
 
