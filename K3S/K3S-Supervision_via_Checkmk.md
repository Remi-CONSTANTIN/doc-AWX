# DISCLAIMER
- Testé sur Rocky Linux 10 et Debian 13 mais devrais fonctionner sur les autres distributions récentes

# Contexte
Procédure testée avec un serveur Checkmk RAW 2.5 et un cluster mono-nœud K3S.

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

1. Téléchargez et installez l'agent provenant de votre serveur checkmk. Voici un exemple à adapter à votre contexte :
```
wget https://10.10.10.125/supervision1/check_mk/agents/check-mk-agent_2.5.0p11-1_all.deb && apt install ./check-mk-agent_2.5.0p11-1_all.deb
```

2. Créez votre hôte via l'interface Web du serveur Checkmk (spécifiez les informations habituelles, IP, hostname etc... )

3. Activez votre agent
```
cmk-agent-ctl register --server <IP-serveur(checkmk) --site supervision1 --user agent_registration --hostname <hostname>
```

4. Retournez dans Checkmk et déclenchez un rescan pour obtenir vos sondes. Libre à vous de les adapter.
</br>

## Installation "agent-collector" checkmk

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
helm upgrade --install --create-namespace -n checkmk-monitoring checkmk checkmk_kube_agent/checkmk \
  --set containerdOverride="/run/k3s/containerd/containerd.sock" \
  --set clusterCollector.service.type=NodePort \
  --set clusterCollector.service.nodePort=30030
```
- `--set clusterCollector.service.type=NodePort` et `--set clusterCollector.service.nodePort=30030` : Par défaut, le collecteur n'est pas accessible depuis l'extérieur du cluster. Nous avons l'exposer sur un port fixe de la machine physique (NodePort) pour que le serveur Checkmk puisse l'interroger.
- `--set containerdOverride="..."` : Indique au collecteur l'emplacement spécifique du socket `containerd` de K3s (`/run/k3s/containerd/containerd.sock`) au lieu du chemin Kubernetes standard, permettant à `cAdvisor` d'identifier correctement les métriques de chaque conteneur

5. Vérifiez que les pods sont bien en cours d'exécution :
```
kubectl get pods -n checkmk-monitoring
```
<img width="553" height="86" alt="cluster_collector_checkmk" src="https://github.com/user-attachments/assets/d1c38715-a0ec-42ed-9362-286385f14e1f" />

ET  
```
kubectl get svc -n checkmk-monitoring checkmk-cluster-collector
```
<img width="593" height="59" alt="cluster_collector_service_checkmk" src="https://github.com/user-attachments/assets/ba722cb1-7e30-462b-a334-319a0b79a6b5" />

Tous les pods doivent être au statut Running et le port 8080:30030/TCP doit être visible

6. Extraire le token et l'afficher car Checkmk a besoin de celui-ci pour avoir le droit de lire les données
```
export TOKEN=$(kubectl get secret checkmk-checkmk -n checkmk-monitoring -o=jsonpath='{.data.token}' | base64 --decode)
echo "$TOKEN"
```
Faites en sorte de garder votre token dans un endroit sécurisé car nous en aurons besoin juste après, pour le paramétrage de la supervision Checkmk

</br>

## Configuration supervision Kubernetes dans Checkmk
Vous allez maintenant configurer la supervision via l'API Kubernetes et le collector dans l'interface Checkmk

1. Rendez-vous dans `Setup --> Agents --> VM, cloud, container --> Kubernetes`
2. Configurez comme suit votre règle :  
<img width="427" height="577" alt="kubernetes-rule-checkmk" src="https://github.com/user-attachments/assets/d07fa619-df01-46b0-8038-67152d1f9be6" />


**Détail :** 
- `Cluster-name` : Le nom de votre choix
- `Token` : Le token que vous avez mis de côté tout à l'heure
- `API serveur connection`
	- `Endpoint` : L'URL de votre kubernetes, vous avez juste à adapter l'IP, le port reste le même
	- `Verify the certificate` :  Vous pouvez l'activer ou la désactiver, si vous avez suivis mes tutoriels, désactivez le
- `HTTP Proxy`: pas besoin dans une installation basique comme la notre
- `TCP Timeout` : Pas besoin dans notre cas, à vous de voir si vous avez des problèmes
- `Collect informations about...` :  Vous pouvez laissez les éléments supervisés par défaut, ou adapter à votre besoin (Voir annexe `Types de ressources Kubernetes`)
- `Monitor namespaces` : Vous pouvez exclure ou limiter à certains namespaces
- `Import annotations as host labels` : Séléctionnez "Import all valid annotations"

3. Associer cette règle à notre hôte afin que certaines des sondes lui remontent  
<img width="564" height="163" alt="kubernetes-rule-checkmk-2" src="https://github.com/user-attachments/assets/028a0945-32e2-4530-b382-30d9f7e328f3" />  

4. Après avoir configuré la règles, n'oubliez pas de modifier la configuration de votre hôte k3s afin qu'il accepte un retour l'API **ET** de l'agent
<img width="547" height="81" alt="monitoring_settings_k3s_checkmk" src="https://github.com/user-attachments/assets/845d93d2-904a-472c-a28b-44e504af8d30" />  

Coté hôte cela nous donnes donc plusieurs nouveaux services :
<img width="2371" height="561" alt="k3s_checkmk_sondes" src="https://github.com/user-attachments/assets/ada87237-15cf-48a4-81fc-a195129861fc" />  
</br>

## Supervision ressources K3S
Après la supervision de l'hôte K3S, il est possible de superviser certaines ressources à l'intérieur de K3S afin, et notamment, d'avoir de la visibilité sur les services qui y tournent.  
Nous allons profitez du mécanisme de `piggyBack` fournis par Checkmk et déjà activé dès l'activation de la supervision sur l'hôte K3S.  
Cela fonctionne pour celles du type : `statefulset`, `deployment`, `namespace`, `node`   

Pour ce faire, il faut tout simplement créer un hôte dans checkmk avec le nom du conteneur dans k3s.  

**--> Par exemple la supervision du namespace `awx` de k3s :**  

1. On prends le nom du namespace
<img width="241" height="152" alt="awx_namespace" src="https://github.com/user-attachments/assets/f623dc36-882a-4cfb-8f9e-8ee86ef5ade2" />  

2. On créer l'hôte dans checkmk en suivant le pattern `<type>_<nom-cluster>_<nom-ressource>`

<img width="489" height="233" alt="awx_namespace_checkmk" src="https://github.com/user-attachments/assets/cfbd864e-c7c9-4458-95f7-7b2850f6d234" />  

**Détails :**
- `Type` : Pour rappel, `statefulset`, `deployment`, `namespace`, `node`
- `nom-cluster` : Relatif au champs `Cluster-name` de votre règle crée plus tôt
- `nom-namespace` : Nom du namespace AWX visible dans la commande `kubectl get namespaces -A`
</br><br/>

3. On valide et on déclenche un scan

<img width="2313" height="809" alt="awx_namespace_sondes_checkmk" src="https://github.com/user-attachments/assets/385a08f5-63cc-4688-919d-387e6366b532" />  

**Détails :**  
- `CPU resources` :  
`Rôle` : Affiche la somme des réservations minimales (requests : 1,55 vCPU) et des plafonds maximaux autorisés (limits : 5,90 vCPU) configurés pour les conteneurs d'AWX  
`Intérêt` : Vérifier que les playbooks Ansible ne risquent pas d'étrangler le processeur du serveur ou de dépasser les quotas du cluster  

- `Memory resources` :  
`Rôle` : Mesure la consommation RAM réelle du projet (2,38 Go) par rapport aux requests (3,30 Go, soit 71,94 %) et aux limits (9,06 Go, soit 26,23 %)  
`Intérêt` : C'est la sonde anti-crash. Si l'usage s'approche des 100 % des limits, le noyau Linux déclenche l'OOM Killer (tueur de conteneurs), ce qui ferait tomber la base Postgres ou l'API AWX  

- `Pod resources` :  
`Rôle` : Comptabilise l'état des pods (Running: 4, Pending: 0, Failed: 0...)  
`Intérêt` : Détecter immédiatement un blocage global. Si un pod passe en Pending (manque de RAM/CPU sur le nœud) ou Failed (crash au démarrage), la sonde passe en alerte  

- `Info` :  
`Rôle` : Identifie le namespace et son ancienneté (Name: awx, Age: 179 days...)  
`Intérêt` : Traceur d'inventaire permettant de vérifier qu'aucun re-déploiement complet ou suppression accidentelle du namespace n'a eu lieu

*Détails issues de gemini*

---

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

*Source tableau : Gemini*
