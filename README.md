# Mini-projet Kubernetes : Déployer WordPress avec des manifests YAML et Kustomize

## 1. Présentation du projet

Ce mini-projet consiste à déployer une application WordPress connectée à une base de données MySQL dans un cluster Kubernetes, en utilisant des manifests YAML écrits manuellement, sans Helm.

L'objectif est de comprendre les mécanismes fondamentaux de Kubernetes : Deployments, Services, Secrets, volumes persistants, PersistentVolumes (PV), PersistentVolumeClaims (PVC) et personnalisation des déploiements avec Kustomize.

Le projet a été réalisé en deux étapes :

1. **Déploiement initial** : création et application des manifests Kubernetes pour MySQL et WordPress.
2. **Personnalisation avec Kustomize** : organisation des manifests en une base commune et des overlays permettant d'adapter la configuration selon l'environnement.

## 2. Objectifs pédagogiques

* Comprendre le déploiement d'applications conteneurisées dans Kubernetes.
* Créer des Deployments pour MySQL et WordPress.
* Exposer MySQL à l'intérieur du cluster avec un Service `ClusterIP`.
* Exposer WordPress avec un Service `NodePort`.
* Configurer les variables d'environnement nécessaires à la connexion entre WordPress et MySQL.
* Utiliser des Secrets Kubernetes pour stocker les paramètres de connexion.
* Comprendre la différence entre un volume, un PersistentVolume et un PersistentVolumeClaim.
* Persister les données de MySQL et configurer le stockage de WordPress.
* Structurer les manifests avec Kustomize pour éviter leur duplication.
* Préparer plusieurs environnements : `dev`, `staging` et `prod`.

## 3. Architecture technique

### Technologies utilisées

| Technologie    | Rôle                                             |
| -------------- | ------------------------------------------------ |
| Kubernetes     | Orchestration des conteneurs                     |
| Minikube       | Cluster Kubernetes local                         |
| Docker Desktop | Exécution du nœud Minikube avec le driver Docker |
| MySQL 5.7      | Base de données                                  |
| WordPress      | Application web                                  |
| YAML           | Déclaration des ressources Kubernetes            |
| Kustomize      | Personnalisation et composition des manifests    |
| kubectl        | Administration du cluster                        |

### Architecture applicative

L'application repose sur deux composants principaux :

* **MySQL** : stocke les données applicatives dans un volume persistant.
* **WordPress** : fournit l'interface web et communique avec MySQL via le Service `mysql-svc`.

Le flux de communication est le suivant :

```text
Navigateur Web
      |
      | HTTP
      v
Service WordPress (NodePort : 30001)
      |
      v
Deployment WordPress
      |
      | Connexion MySQL : mysql-svc:3306
      v
Service MySQL (ClusterIP : 3306)
      |
      v
Deployment MySQL
      |
      v
PVC mysql-pvc
      |
      v
PV mysql-pv
      |
      v
Stockage du nœud Minikube
```

Le Service `ClusterIP` rend MySQL accessible à l'intérieur du cluster. Le Service `NodePort` permet d'exposer WordPress à travers le nœud Kubernetes, sous réserve de la configuration réseau de Minikube.

## 4. Prérequis

Les outils suivants doivent être installés :

* Docker Desktop
* Minikube
* kubectl
* Git

Vérifier les installations :

```powershell
docker --version
minikube version
kubectl version --client
git --version
```

Démarrer Minikube avec le driver Docker :

```powershell
minikube start --driver=docker
```

Vérifier l'état du cluster :

```powershell
minikube status
kubectl get nodes
kubectl cluster-info
```

Le nœud Minikube doit être opérationnel avant de déployer l'application.

## 5. Étape 1 : déploiement avec les manifests Kubernetes classiques

Le répertoire `k8s-manifest/` contient les manifests utilisés pour réaliser le premier déploiement.

### Arborescence

```text
k8s-manifest/
├── mysql/
│   ├── mysql-deploy.yml
│   ├── mysql-pv.yml
│   ├── mysql-pvc.yml
│   ├── mysql-secret.yml
│   └── mysql-svc.yml
└── wordpress/
    ├── wp-deploy.yml
    ├── wp-secret.yml
    └── wp-svc.yml
```

### MySQL

Le composant MySQL comprend :

* `mysql-deploy.yml` : Deployment avec une réplique.
* `mysql-svc.yml` : Service interne de type `ClusterIP`.
* `mysql-secret.yml` : paramètres de connexion à la base.
* `mysql-pv.yml` : définition du stockage persistant.
* `mysql-pvc.yml` : demande de stockage de 10 Gio.

Le Deployment monte le PVC dans le répertoire de données de MySQL, `/var/lib/mysql`.

Le PV utilise un `hostPath` pointant vers `/data-mysql` sur le nœud Kubernetes.

### WordPress

Le composant WordPress comprend :

* `wp-deploy.yml` : Deployment de l'application.
* `wp-secret.yml` : paramètres de connexion à MySQL.
* `wp-svc.yml` : Service de type `NodePort`.

Les variables d'environnement permettent à WordPress de retrouver le serveur MySQL et les identifiants nécessaires.

Le Service MySQL est référencé par son nom DNS Kubernetes :

```text
mysql-svc:3306
```

Le Deployment WordPress monte un volume `hostPath` à partir du chemin `/data/wordpress` du nœud, dans le répertoire `/var/www/html` du conteneur.

Contrairement à MySQL, cette configuration WordPress utilise un volume simple et non un PVC.

### Déployer les manifests

Depuis la racine du dépôt :

```powershell
kubectl apply -f k8s-manifest/
```

Vérifier les ressources :

```powershell
kubectl get deployments
kubectl get pods
kubectl get services
kubectl get pv,pvc
kubectl get secrets
```

Pour examiner les détails d'un Pod :

```powershell
kubectl describe pod <nom-du-pod>
```

Pour consulter les logs de WordPress :

```powershell
kubectl logs deployment/wp-deploy
```

Pour consulter les logs de MySQL :

```powershell
kubectl logs deployment/mysql-deployment
```

## 6. Étape 2 : organisation et personnalisation avec Kustomize

Après validation du déploiement initial, les manifests ont été repris dans le répertoire `kustomize-wordpress/`.

Kustomize permet de composer plusieurs ressources Kubernetes et de personnaliser leurs paramètres sans recopier l'ensemble des manifests pour chaque environnement.

### Arborescence

```text
kustomize-wordpress/
├── base/
│   ├── kustomization.yml
│   ├── mysql/
│   │   ├── kustomization.yml
│   │   ├── mysql-deploy.yml
│   │   ├── mysql-pv.yml
│   │   ├── mysql-pvc.yml
│   │   ├── mysql-secret.yml
│   │   └── mysql-svc.yml
│   └── wordpress/
│       ├── kustomization.yml
│       ├── wp-deploy.yml
│       ├── wp-secret.yml
│       └── wp-svc.yml
└── overlays/
    ├── dev/
    │   └── kustomization.yml
    ├── staging/
    │   └── kustomization.yml
    └── prod/
        ├── kustomization.yml
        ├── service-patch.yml
        └── wordpress/
            └── service-patch.yml
```

### La base commune

Le répertoire `base/` contient les ressources partagées entre les environnements.

Les fichiers `kustomization.yml` déclarent les ressources à assembler.

Exemple :

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - mysql/
  - wordpress/
```

Les sous-répertoires `mysql/` et `wordpress/` possèdent également leur propre fichier `kustomization.yml`, qui liste les manifests du composant correspondant.

### Les overlays

Les overlays représentent les personnalisations propres à chaque environnement.

* `overlays/dev/` : environnement de développement.
* `overlays/staging/` : environnement de préproduction ou de validation.
* `overlays/prod/` : environnement de production.

Un overlay référence la base commune, puis applique des patches pour modifier les paramètres souhaités.

Exemple de configuration :

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - ../../base
```

Les patches peuvent modifier le nombre de répliques d'un Deployment ou les paramètres d'un Service sans modifier les manifests de base.

Les valeurs exactes appliquées à chaque environnement sont définies dans les fichiers `kustomization.yml` correspondants.

## 7. Déployer avec Kustomize

Depuis le répertoire `kustomize-wordpress/` :

### Vérifier les manifests générés

```powershell
kubectl kustomize overlays/dev/
```

Cette commande assemble la base et les patches, puis affiche les manifests finaux sans les appliquer au cluster.

### Appliquer l'environnement de développement

```powershell
kubectl apply -k overlays/dev/
```

### Appliquer l'environnement staging

```powershell
kubectl apply -k overlays/staging/
```

### Appliquer l'environnement de production

```powershell
kubectl apply -k overlays/prod/
```

**Attention :** ces commandes utilisent les mêmes noms de ressources Kubernetes que les manifests classiques. Il est donc préférable de ne déployer qu'une seule variante de l'application dans un même namespace, sauf si les noms et les namespaces ont été adaptés pour permettre leur coexistence.

## 8. Accéder à WordPress depuis le navigateur

Le Service WordPress est de type `NodePort` et utilise le port `30001` dans la configuration de base.

Vérifier le Service :

```powershell
kubectl get svc wp-svc
```

Avec Minikube utilisant le driver Docker sous Windows, la méthode suivante permet d'obtenir une URL accessible :

```powershell
minikube service wp-svc --url
```

Ouvrir l'URL affichée dans le navigateur. Selon la configuration de Minikube, le terminal doit rester ouvert pour maintenir le tunnel actif.

Une autre possibilité consiste à utiliser le port-forward :

```powershell
kubectl port-forward service/wp-svc 8080:80
```

Puis ouvrir :

```text
http://localhost:8080
```

Le terminal doit rester ouvert pendant l'utilisation du port-forward.

## 9. Vérification du fonctionnement

Après le déploiement, vérifier que les ressources attendues existent :

```powershell
kubectl get deployments
kubectl get pods
kubectl get services
kubectl get pv,pvc
```

Les vérifications importantes sont :

* Le Deployment MySQL dispose d'un Pod prêt à fonctionner.
* Le PVC `mysql-pvc` est associé à un PV et passe à l'état `Bound`.
* Le Service `mysql-svc` expose le port 3306 dans le cluster.
* Le Deployment WordPress dispose d'un Pod prêt à fonctionner.
* Le Service `wp-svc` expose le port 80 via le NodePort configuré.
* WordPress peut se connecter à MySQL.
* L'interface WordPress est accessible depuis le navigateur.

Si un Pod reste en erreur :

```powershell
kubectl describe pod <nom-du-pod>
kubectl logs <nom-du-pod>
```

Pour un Pod dont le conteneur ne démarre pas, consulter en priorité la section `Events` de `kubectl describe pod`.

## 10. Comprendre les volumes et la persistance

Le projet met en œuvre plusieurs mécanismes de stockage.

### PersistentVolume (PV)

Le PV représente une ressource de stockage persistante connue de Kubernetes.

Dans ce projet, `mysql-pv.yml` déclare un volume de 10 Gio basé sur un `hostPath`.

### PersistentVolumeClaim (PVC)

Le PVC est la demande de stockage exprimée par MySQL.

Le fichier `mysql-pvc.yml` demande 10 Gio. Kubernetes associe cette demande au PV compatible.

### Volume du Pod

Le volume déclaré dans le Deployment est monté dans le conteneur à l'aide de `volumeMounts`.

Pour MySQL, le volume référence le PVC `mysql-pvc`.

Pour WordPress, le volume utilise directement `hostPath`.

### Limites de cette configuration

Le stockage `hostPath` utilise le système de fichiers du nœud Kubernetes. Avec Minikube et le driver Docker, les chemins `/data-mysql` et `/data/wordpress` se trouvent dans l'environnement Linux du nœud, et non automatiquement dans un dossier Windows.

Cette approche est adaptée à un exercice local, mais elle nécessite une configuration différente pour un environnement multi-nœuds ou de production. Il faut notamment prévoir un stockage approprié, des sauvegardes et une stratégie de récupération.

## 11. Gestion des Secrets

Les Secrets Kubernetes sont utilisés pour fournir les paramètres de connexion de MySQL et WordPress.

Les clés utilisées dans les Secrets doivent correspondre exactement à celles référencées dans les Deployments.

Exemple :

```yaml
secretKeyRef:
  name: mysql-secret
  key: password
```

Le champ `key` désigne le nom de la clé présente dans le Secret.

Pour un projet réel, il faut éviter de versionner des mots de passe en clair dans Git. Les Secrets Kubernetes ne constituent pas à eux seuls un système complet de gestion des secrets : il convient de prévoir des contrôles d'accès adaptés et, selon le contexte, un gestionnaire de secrets dédié.

## 12. Nettoyer les ressources

Pour supprimer les ressources déployées à partir des manifests classiques :

```powershell
kubectl delete -f k8s-manifest/
```

Pour supprimer les ressources gérées par Kustomize dans l'environnement dev :

```powershell
kubectl delete -k overlays/dev/
```

Exécuter la commande Kustomize depuis le répertoire `kustomize-wordpress/`.

La suppression d'un Deployment ou d'un Pod ne signifie pas nécessairement que les données persistantes sont supprimées. Le devenir des données dépend notamment du PV, de sa politique de récupération et du stockage sous-jacent.

Ne supprimer le cluster Minikube ou les volumes qu'après avoir vérifié que les données à conserver ont été sauvegardées.

## 13. Bonnes pratiques et améliorations possibles

Ce projet est un exercice pédagogique. Pour aller plus loin :

* Remplacer `hostPath` par une solution de stockage adaptée au cluster.
* Prévoir des sauvegardes régulières de MySQL.
* Utiliser une version d'image WordPress explicitement définie plutôt que `latest`.
* Utiliser des Secrets gérés de manière sécurisée.
* Ajouter des probes de disponibilité et de vivacité.
* Ajouter des ressources CPU et mémoire adaptées aux besoins réels.
* Configurer un Ingress si un accès HTTP plus réaliste est nécessaire.
* Étudier une architecture MySQL adaptée à la haute disponibilité si nécessaire.
* Valider les overlays avec `kubectl kustomize` avant de les appliquer.
* Envisager des tests automatisés des manifests et de la connectivité entre les composants.

## 14. Conclusion

Ce mini-projet permet de comprendre le déploiement manuel d'une application composée de WordPress et MySQL dans Kubernetes.

La première étape met en pratique les ressources fondamentales de Kubernetes à travers des manifests YAML écrits manuellement.

La seconde étape introduit Kustomize pour organiser les ressources en une base commune et des overlays adaptés aux environnements `dev`, `staging` et `prod`.

L'ensemble constitue une première approche de la gestion de configurations Kubernetes réutilisables, tout en mettant en évidence les enjeux de persistance des données, de connectivité entre services et de personnalisation des déploiements.

## 15. Dépôt GitHub

**Dépôt du projet :** ajouter ici l'URL du dépôt GitHub.

Ce projet a été réalisé dans le cadre d'un exercice de déploiement de WordPress sur Kubernetes, sans Helm, afin de mieux comprendre les manifests et les mécanismes de configuration de Kubernetes.
