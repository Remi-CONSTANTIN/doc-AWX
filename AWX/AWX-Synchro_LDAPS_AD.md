# DISCLAIMER
- Testé sur Debian 13 mais devrais fonctionner de la même façon sur les autres distributions récentes

# Contexte
Après le déploiement d'une instance AWX, il est souvent souhaité de pouvoir centraliser la gestion des identités. L'intérêt de la synchronisation LDAPS est de pouvoir utiliser les comptes AD déjà existants est de leur donner des groupes automatiquement dans AWX. Pas de nouveaux comptes à créer et à gérer.

# Prérequis
- Une instance AWX opérationnelle
- Un serveur ADCS (PKI) du type `Entreprise` et non `autonome`
- Un serveur Active Directory (AD)

# Procédure

## Export du certificat d'autorité de l'ADCS 

### Cas 1 : Autorité de certification racine seule
Le premier cas ne concerne que ceux qui n'ont qu'une seule autorité de certification racine dans leur infrastructure.

1. Se rendre sur le serveur ADCS puis dans le gestionnaire des certificats (certsrv)
2. Clique droit sur votre serveur --> Sélectionner le certificat 0 --> Afficher le certificat --> Copier dans un fichier --> Sélectionner `X.509 encodé en base 64 (*.cer)` --> Nommez le `root-ca.cer`

<img width="1110" height="594" alt="ADCS_CA_export_AWX" src="https://github.com/user-attachments/assets/a93b0651-b6f6-4d22-8f7b-2fd5f86b8144" />


3. Envoyez le de la façon que vous voulez sur votre serveur Linux AWX (WinSCP par exemple ?)

### Cas 2 : 
Le deuxième cas concerne ceux qui ont une autorité de certification intermédiaire en plus de la racine.
Je n'ai pas expérimenté le processus d'export du certificat depuis l'ADCS par contre je peux vous parler de la préparation du certificat pour AWX. 

1. Fournissez vous le certificat de l'autorité de certification intermédiaire
2. Double cliquez dessus et allez dans l'onglet `Certification path`
3. Si le certificat provient bien de l'intermédiaire alors vous devriez avoir une chaine d'au moins deux maillons
4. Cliquez une fois sur chaque maillons, cliquez sur `View Certificate` et exportez au format `.cer` le certificat de chaque maillons (sauf celui dont proviens ce certificat)
5. Une fois que vous avez le certificat `.cer` de chaques maillons, ouvrez les dans un éditeur de texte
6. Créer le fichier final qui contiendra tout les certificats et nommez le par exemple `ldap-bundle.cer`
7. Copiez chaque certificats dans ce fichier dans un ordre bien précis. Partez du bas de la chaine (le premier certificat que vous avez eu) et remontez jusqu'à l'autorité de certification racine
8. Une fois que vous avez ce fichier, continuer à la prochaine étape

## Injection dans Kubernetes

Le certificat doit être stocké dans un **Secret** pour que l'AWX Operator puisse le monter dans les conteneurs.

Placez vous dans le dossier où vous avez mit votre certificat ou adaptez le chemin dans la commande puis exécutez la commande de création du secret `awx-custom-certs`
```
kubectl create secret generic awx-custom-certs --from-file=bundle-ca.crt=root-ca.cer -n awx
```

## Configuration de la ressource AWX (YAML)

Il faut modifier le déploiement pour permettre la résolution DNS de l'AD.

1. Éditez le fichier de déploiement AWX (`awx-instance` si vous avez suivis ma procédure) 
2. Ajouts dans le bloc `spec`:
```
    spec:
      host_aliases:
        - ip: "10.82.14.23"          <-- A changer par votre IP
          hostnames:
            - "ad.tadaron.local"
```

> [!warning]
> Ne pas mettre de majuscule dans la partie `hostnames` sinon l'operator ne mettra pas à jour les pods à cause d'une erreur de syntax !

3. Puis (aussi dans le bloc `spec`) :
```
# Injection du certificat
  bundle_cacert_secret: awx-custom-certs
```

4. Plus (dans le bloc `extra_settings` cette fois) :
```
    - setting: AUTH_LDAP_CONNECTION_OPTIONS
      value:
        OPT_REFERRALS: 0
        OPT_NETWORK_TIMEOUT: 30
        OPT_X_TLS_NEWCTX: 0
        OPT_X_TLS_REQUIRE_CERT: 2
```
Détail des options en annexes

Visuellement ça donne :  
<img width="576" height="604" alt="ldaps_awx_instance_yml" src="https://github.com/user-attachments/assets/8ed15a07-f2bb-48e3-a9a4-9490066c6055" />

etc...

5. Plus qu'à appliquer ! (Adaptez avec vos informations bien sûr)
```
kubectl apply -f awx-instance.yml
```

6. Et tester !
```
kubectl exec -it deployment/awx-web -n awx -c awx-web -- openssl s_client -connect ad.tadaron.local:636 -brief
``` 
Vous devriez avoir `Verification OK` :

<img width="1184" height="138" alt="openssl_ldap_verification_awx" src="https://github.com/user-attachments/assets/ab415fa0-ed45-459a-b773-e030df736048" />

## Configuration liaison LDAPS dans l'interface Web AWX
1. Retournez dans l'interface Web AWX avec votre compte admin à l'URL (à adapter bien sûr) `https://<hostname.domain>/#/settings/ldap/default/details`

2. Complétez les paramètres LDAPS en fonction de votre contexte.
Voici un exemple : 

- **URI du serveur LDAP** : `ldaps://ad.tadaron.local:636`

- **Mot de passe de liaison** : MDP de l'utilisateur que vous allez utiliser pour lire l'AD

- **Type de groupe LDAP** : `ActiveDirectoryGroupType`

- **LDAP - Lancer TLS** : Désactivé (car le protocole `ldaps://` s'occupe déjà du chiffrement)

- **ND de la liaison LDAP** : Le chemin vers l'utilisateur AD servant à la lecture de l'annuaire
```
CN=awx_user,OU=Comptes_service,OU=Utilisateurs,OU=tadaron_corp,DC=tadaron,DC=local
```

- (Optionnel) **Groupe LDAP obligatoire** : Le chemin vers le groupe de sécurité dans l'AD nécessaire pour qu'un utilisateur puisse accéder à AWX

- (Optionnel) **Groupe LDAP refusé** : L'inverse du groupe obligatoire. Le chemin vers le groupe AD exclue de l'accès à AWX

- **Recherche d'utilisateurs LDAP** : OU où se trouvent les comptes AD qui pourront se connecter à AWX. Si besoin, vous pouvez filtrez les utilisateurs de cette OU en ajoutant la condition `Groupe LDAP obligatoire` configurable au dessus
```
[
  "OU=Admins,OU=Utilisateurs,OU=tadaron_corp,DC=tadaron,DC=local",
  "SCOPE_SUBTREE",
  "(sAMAccountName=%(user)s)"
]
```

- **Recherche de groupes LDAP** : OU où se trouvent les groupes AD qui serviront à mapper les utilisateurs membres dans les équipes AWX. Le mappage est configuré dans la partie `Mappe d'équipes LDAP`
```
[
  "OU=Groupes,OU=tadaron_corp,DC=tadaron,DC=local",
  "SCOPE_SUBTREE",
  "(objectClass=group)"
]
```

- **Mappe des attributs d'utilisateurs LDAP** : Attribues des utilisateurs AD qui seront renseignés automatiquement dans AWX
```
{
  "email": "mail",
  "last_name": "sn",
  "first_name": "givenName"
}
```

- (Optionnel) **Paramètres de types de groupes LDAP** : Utile pour les annuaires non-Active-Directory

- (Optionnel) **Marqueurs d'utilisateur LDAP par groupe** : Permet d'assigner des groupes administrateurs/auditeurs AWX en fonction des groupes de sécurité de l'AD

- (Optionnel) **Mappe d'organisations LDAP** : Permet de mapper automatiquement des organisation AWX dans un contexte de multi tenant

- **Mappe d'équipes LDAP** : Permet d'assigner une équipe + une organisation AWX en fonction d'un groupe de sécurité AD
```
{
  "BDD": {
    "users": "cn=bdd,ou=groupes,ou=tadaron_corp,dc=tadaron,dc=local",
    "remove": true,
    "organization": "econocom"
  },
  "Unix": {
    "users": "cn=unix,ou=groupes,ou=tadaron_corp,dc=tadaron,dc=local",
    "remove": true,
    "organization": "econocom"
  },
  "Reseau": {
    "users": "cn=reseaux,ou=groupes,ou=tadaron_corp,dc=tadaron,dc=local",
    "remove": true,
    "organization": "econocom"
  }
}
```

> [!warning] Ne pas enlever
> L'option `remove: true` permet d'actualiser le mappage à chaque connexion de l'utilisateur

3. Une fois finis, cliquez simplement sur enregistrer puis déconnectez vous et reconnectez vous avec un utilisateur AD


# Sources
[Trusting a Custom Certificate Authority](https://github.com/ansible/awx-operator/blob/devel/docs/user-guide/advanced-configuration/trusting-a-custom-certificate-authority.md)


# Annexes

### Détails des options `extra_settings`

- **`OPT_REFERRALS: 0`** : Interdit au client de suivre les redirections vers d'autres contrôleurs de domaine. C'est **indispensable avec Active Directory** pour éviter les boucles infinies et les lenteurs d'authentification.

- **`OPT_NETWORK_TIMEOUT: 30`** : Définit un délai d'attente de 30 secondes pour la réponse du serveur. Cela évite de bloquer les processus AWX (workers) si l'AD ne répond pas ou si le réseau sature.

- **`OPT_X_TLS_NEWCTX: 0`** : Réutilise le contexte SSL/TLS existant au lieu d'en recréer un pour chaque requête. Cela améliore les performances et stabilise la validation des certificats avec la bibliothèque `python-ldap`.

- **`OPT_X_TLS_REQUIRE_CERT: 2` (Mode Strict)** : Exige que le certificat de l'AD soit valide et signé par l'autorité racine fournie. C'est le **rempart contre les attaques "Man-in-the-middle"** : AWX refusera toute connexion si l'identité de l'AD n'est pas certifiée par votre PKI
*Source : Gemini*
