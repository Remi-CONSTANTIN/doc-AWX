#kubernetes #k3s #checkmk #checmk23 

# DISCLAIMER
- Testé sur Rocky Linux 10 et Debian 13 mais devrais fonctionner sur les autres distributions récentes

# Contexte
Procédure testée avec un serveur Checkmk RAW 2.4 et un cluster mono-nœud K3S.

# Objectif
Nous objectif ici sera de mettre en place la supervision du cluster Kubernetes mono-nœud allant du système (agent Checkmk) à l'API Kubernetes.

# Prérequis
- Un serveur Checkmk pas trop vieux (pas 1.6 par exemple)
- Un cluster Kubernetes (K3S fonctionne, pas testé sur K8S et autres...)

# Procédure

### Agent Checkmk
L'agent ne supervise pas la partie Kubernetes mais reste utile car il faut tout de même superviser les nœuds.

Plusieurs options s'offrent à vous en fonction de votre déploiement du serveur Checkmk.
Il vous faudra adapter à votre gestionnaire de paquet et à votre interface WEB (HTTP/HTTPS)

1. Téléchargez et installez l'agent
```
wget --no-check-certificate https://192.168.1.223/supervision1/check_mk/agents/check-mk-agent_2.4.0p5-1_all.deb && rpm -Uvh ./check-mk-agent-2.4.0p5-1.noarch.rpm
```

2. Créez votre hôte via l'interface Web du serveur Checkmk (spécifiez les informations habituelles, IP, hostname etc... )

3. Activez votre agent
```
cmk-agent-ctl register --server <IP-serveur(checkmk) --site supervision1 --user agent_registration --hostname <hostname>
```

4. Retournez dans Checkmk et déclenchez un rescan pour obtenir vos sondes. Libre à vous de les adapter.

### Installation "agent-collector" checkmk

 1. Si vous ne l'avez pas déjà, installez `Helm`, le gestionnaire de paquet de Kubernetes 
```
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-4 | bash
```

 2. Ajouter le [dépôt du collector Checkmk](https://checkmk.github.io/checkmk_kube_agent/)
```
helm repo add checkmk_kube_agent https://checkmk.github.io/checkmk_kube_agent
```

3. Mettez à jour les sources avec le nouveau dépot
```
helm repo update
```

4. Installer le collector dans un namespace dédié
```
helm upgrade --install --create-namespace -n checkmk-monitoring checkmk checkmk_kube_agent/checkmk
```

5. Vérifiez que les pods sont bien en cours d'exécution :
```
kubectl get pods -n checkmk-monitoring
``` 
![[checkmk-collector-k3s.png]]

6. Transformez le service en `NodePort`
Par défaut, le collecteur n'est pas accessible depuis l'extérieur du cluster. Nous allons l'exposer sur un port fixe de la machine physique (NodePort) pour que le serveur Checkmk puisse l'interroger.

```
kubectl patch svc checkmk-cluster-collector -n checkmk-monitoring -p '{"spec": {"type": "NodePort", "ports": [{"port": 8080, "nodePort": 30030}]}}'
```

7. Vérifiiez que le port a bien été fixé
```
kubectl get svc -n checkmk-monitoring checkmk-cluster-collector
```
![[static-nodeport-checkmk-collector.png]]
On voit que le port est bien `30030`, celui que nous avons choisis

8. Extraire le token et l'afficher car Checkmk a besoin de celui-ci pour avoir le droit de lire les données
```
export TOKEN=$(kubectl get secret checkmk-checkmk -n checkmk-monitoring -o=jsonpath='{.data.token}' | base64 --decode)
echo "$TOKEN"
```
Faites en sorte de garder votre token dans un endroit sécurisé car nous en aurons besoin juste après, pour le paramétrage de la supervision Checkmk

### Configuration supervision Kubernetes dans Checkmk
Vous allez maintenant configurer la supervision via l'API Kubernetes et le collector dans l'interface Checkmk

1. Rendez-vous dans `Setup --> Agents --> VM, cloud, container --> Kubernetes`
2. Configurez comme suit votre règle :  
<img width="427" height="577" alt="kubernetes-rule-checkmk" src="https://github.com/user-attachments/assets/d07fa619-df01-46b0-8038-67152d1f9be6" />


**Détail :** 
- `Cluster-name` : Le nom de votre choix
- `Token` : Le token que vous avez mis de côté tout à l'heure
- `API serveur connection`
	- `Endpoint` : L'URL de votre kubernetes, vous avez juste à adapter l'IP, le port reste le même
	- `Verify the certificate` :  Vous pouvez l'activer ou la désactiver, cela importe marche dans les deux cas
- `HTTP Proxy`: pas besoin dans une installation basique comme la notre
- `TCP Timeout` : Pas besoin dans notre cas, à vous de voir si vous avez des problèmes
- `Collect informations about...` :  Vous pouvez laissez les éléments supervisés par défaut, ou adapter à votre besoin (Voir annexe `Types de ressources Kubernetes`)
- `Monitor namespaces` : Vous pouvez exclure ou limiter à certains namespaces
- `Import annotations as host labels` : Séléctionnez "Import all valid annotations"

3. N'oubliez pas d'associer cette règle à notre hôte afin que certaines des sondes lui remontent  
<img width="564" height="163" alt="kubernetes-rule-checkmk-2" src="https://github.com/user-attachments/assets/028a0945-32e2-4530-b382-30d9f7e328f3" />


# Annexes

### Types de ressources Kubernetes

| **Ressource**    | **Description courte**               | **Utilité dans Checkmk**              |
| ---------------- | ------------------------------------ | ------------------------------------- |
| **Namespaces**   | **Quartiers** logiques (ex: `awx`)   | Santé globale d'un projet             |
| **Nodes**        | **Serveurs** (ex: VM Rocky)          | Charge CPU/RAM du matériel            |
| **Deployments**  | **Apps stables** (ex: AWX Web, Task) | **Top priorité** pour surveiller AWX  |
| **StatefulSets** | **Bases de données** (ex: Postgres)  | Suivi de l'état de la BDD             |
| **DaemonSets**   | **Agents de fond** (ex: Monitoring)  | Vérifier que l'agent tourne partout   |
| **Pods**         | **Ouvriers** éphémères               | Trop instables pour la version Raw    |
| **PVC**          | **Disques** virtuels                 | Alertes sur le stockage plein         |
| **CronJobs**     | **Planning** (Backups)               | Vérifier que les tâches sont prévues  |
| **Pods of CJ**   | **Exécution** temporaire             | Inutile à surveiller individuellement |

    Source tableau : Gemini
