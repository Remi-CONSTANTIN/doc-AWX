## 1. L'exécution : Local vs Conteneurisé
- **Ansible Pure :** Tu lances la commande `ansible-playbook` sur ta machine. Ansible utilise les bibliothèques et les collections installées localement sur ton OS.
- **AWX :** Rien n'est installé "en local" sur le serveur AWX. Chaque job est lancé dans un **Execution Environment (EE)**. C'est un conteneur isolé qui contient une version précise d'Ansible, de Python et des collections.

## 2. L'arborescence : Statique vs Dynamique
C'est là que ça change le plus pour nous

#### En Ansible Pure (CLI)
Ton arborescence est **fixe** sur ton disque dur. Quand tu lances un playbook, Ansible regarde autour de lui (dossier courant) pour trouver les rôles et les variables.

#### Dans AWX (Projets)
AWX ne "stocke" pas tes fichiers de manière permanente pour l'exécution.
1. **Le Projet :** AWX crée un clone temporaire de ton dépôt Git (dans `/runner/project` à l'intérieur du conteneur).
2. **L'isolation :** Chaque exécution de job se fait dans son propre espace. Si tu as 10 jobs en même temps, AWX crée 10 copies temporaires.
3. **La priorité :** Contrairement au CLI, AWX force souvent certains chemins. Il va chercher le `ansible.cfg` à la racine pour savoir comment se comporter.

## 3. Gestion des Dépendances (Rôles & Collections)
C'est l'un des plus gros avantages d'AWX.
- **CLI :** Tu dois lancer manuellement `ansible-galaxy install -r requirements.yml`.
- **AWX :** Dès qu'il voit un fichier `roles/requirements.yml` ou `collections/requirements.yml` lors de la synchronisation du projet, **il télécharge tout seul les dépendances**. Elles sont ensuite stockées dans un dossier caché pour être disponibles lors de tes exécutions.

## 4. Inventaire : Fichier vs Base de données

| Caractéristique | Ansible Pure (CLI)                           | AWX                                                   |
| --------------- | -------------------------------------------- | ----------------------------------------------------- |
| **Source**      | Fichier texte (`hosts`, `inventory.ini/yml`) | Base de données PostgreSQL                            |
| **Gestion**     | Manuelle (éditeur de texte)                  | Interface Web ou API                                  |
| **Variables**   | `group_vars/` et `host_vars/` dans le dépôt  | Définies dans l'UI d'AWX (et fusionnées avec le code) |
| **Dynamisme**   | Scripts d'inventaire complexes               | Connecteurs natifs (AWS, Azure, VMware, NetBox...)    |

> [!NOTE] 
> Même si AWX gère l'inventaire dans sa base de données, il sait toujours lire les dossiers `group_vars/` et `host_vars/` situés à la racine de ton projet Git pour compléter les informations.

## 5. Comparaison du Workflow

### Workflow Ansible Pure

1. Écrire le code.
2. Gérer l'inventaire (`hosts`).
3. Installer les dépendances (`ansible-galaxy`).
4. Lancer : `ansible-playbook -i hosts playbook.yml`.
### Workflow AWX

1. **Git Push :** Tu envoies ton code (playbooks + rôles + `ansible.cfg`) sur ton dépôt.
2. **Project Sync :** AWX détecte le changement (ou tu le forces) et télécharge les rôles Galaxy.
3. **Inventory Management :** AWX pioche dans sa propre liste de serveurs (souvent synchronisée avec ton Cloud).
4. **Job Template :** Tu cliques sur "Launch". AWX crée le conteneur, injecte les **Credentials** (clés SSH/mots de passe) de manière sécurisée et lance le playbook.

Source : Gemini
