# Spring PetClinic sur Kubernetes

Déploiement d'une application Spring Boot sur un cluster Kubernetes multi-nœuds avec haute disponibilité et persistance des données.

---

## 📋 À Propos

Ce projet démontre le déploiement d'une application **Spring Boot** (PetClinic) sur Kubernetes avec :
- Architecture multi-tiers (application + base de données)
- Haute disponibilité (2 réplicas)
- Persistance des données (MySQL avec volume)
- Gestion sécurisée des credentials (Secrets K8s)
- Monitoring des ressources

Le déploiement s'effectue sur un **cluster Kubernetes local** provisionné avec Vagrant (3 VMs : 1 master + 2 workers), simulant un environnement de production.

---

## 🎯 Objectifs

- ✅ Conteneuriser une application Java/Spring Boot
- ✅ Déployer une stack complète sur Kubernetes
- ✅ Implémenter la persistance des données
- ✅ Configurer la haute disponibilité
- ✅ Sécuriser les credentials avec Secrets
- ✅ Mettre en place du monitoring basique

---

## 🛠️ Stack Technique

| Composant | Technologie | Version |
|-----------|-------------|---------|
| **Application** | Spring PetClinic (Java) | Latest |
| **Base de données** | MySQL | 8.0 |
| **Conteneurisation** | Docker | 20.10+ |
| **Orchestration** | Kubernetes | 1.28+ |
| **Provisionnement** | Vagrant + VirtualBox | 2.3+ / 7.0+ |
| **OS** | Ubuntu | 22.04 |

---

## ⚙️ Prérequis

Avant de commencer, installez les outils suivants sur votre machine :

- **Vagrant** 2.3+ → [Télécharger](https://www.vagrantup.com/downloads)
- **VirtualBox** 7.0+ → [Télécharger](https://www.virtualbox.org/wiki/Downloads)
- **Git** 2.30+ → [Télécharger](https://git-scm.com/downloads)

**Configuration minimale :**
- RAM : 8 GB
- CPU : 4 cœurs
- Stockage : 20 GB libre

---

## 🚀 Installation et Déploiement

### Étape 1 : Cloner le Repository

```bash
git clone https://github.com/votre-username/petclinic-k8s.git
cd petclinic-k8s
```

### Étape 2 : Provisionner l'Infrastructure

Le [`Vagrantfile`](./Vagrantfile) crée automatiquement 3 VMs :
- **k8s-master** (192.168.56.10) - Control Plane
- **k8s-worker1** (192.168.56.11) - Worker Node
- **k8s-worker2** (192.168.56.12) - Worker Node

```bash
# Démarrer toutes les VMs (prend 5-10 minutes)
vagrant up

# Vérifier l'état
vagrant status
```

### Étape 3 : Initialiser le Cluster Kubernetes

Se connecter au master et suivre le [Guide de Déploiement](./docs/deployment-guide.md) :

```bash
# Connexion au master
vagrant ssh k8s-master

# Le cluster est initialisé automatiquement par Vagrant
# Vérifier que tous les nœuds sont Ready
kubectl get nodes
```

**Résultat attendu :**
```
NAME           STATUS   ROLES           AGE
k8s-master     Ready    control-plane   5m
k8s-worker1    Ready    <none>          3m
k8s-worker2    Ready    <none>          3m
```

### Étape 4 : Builder l'Image Docker

Le [`Dockerfile`](./docker/dockerfile) utilise un build multi-stage pour optimiser la taille de l'image.

```bash
# Exécuter le script de build
./scripts/build.sh
```

**Ce script va :**
1. Cloner le code source de Spring PetClinic
2. Compiler l'application avec Maven
3. Créer une image Docker optimisée
4. Taguer l'image : `petclinic:1.0`

**Durée estimée :** 5-10 minutes

### Étape 5 : Déployer sur Kubernetes

Le script [`deploy.sh`](./scripts/deploy.sh) applique tous les manifests Kubernetes dans le bon ordre.

```bash
# Lancer le déploiement complet
./scripts/deploy.sh
```

**Ce script va :**
1. Créer le namespace `petclinic-dev`
2. Créer les Secrets (credentials MySQL)
3. Déployer MySQL avec PersistentVolumeClaim (10Gi)
4. Déployer PetClinic (2 réplicas)
5. Exposer l'application via Service NodePort

**Surveiller le déploiement :**
```bash
# Suivre en temps réel
kubectl get pods -n petclinic-dev -w

# Vérifier le statut final
kubectl get all -n petclinic-dev
```

**Résultat attendu :**
```
NAME                         READY   STATUS    RESTARTS   AGE
pod/mysql-xxx                1/1     Running   0          2m
pod/petclinic-xxx            1/1     Running   0          1m
pod/petclinic-yyy            1/1     Running   0          1m

NAME                TYPE       CLUSTER-IP     PORT(S)        AGE
service/mysql       ClusterIP  10.96.x.x      3306/TCP       2m
service/petclinic   NodePort   10.96.y.y      80:30080/TCP   1m
```

### Étape 6 : Accéder à l'Application

L'application est exposée sur le port **30080** de chaque worker node.

**Méthode 1 : Via IP du Worker**
```bash
# Obtenir l'IP d'un worker
kubectl get nodes -o wide

# Ouvrir dans le navigateur
# http://192.168.56.11:30080
# ou
# http://192.168.56.12:30080
```

**Méthode 2 : Via Port-Forward (pour test)**
```bash
kubectl port-forward -n petclinic-dev svc/petclinic 8080:80

# Ouvrir: http://localhost:8080
```

---

## 📁 Structure du Projet

```
petclinic-k8s/
├── README.md                    # Ce fichier
├── Vagrantfile                  # Configuration des VMs
├── docker/
│   └── dockerfile               # Build multi-stage de l'app
├── kubernetes/
│   ├── namespace.yaml           # Namespace petclinic-dev
│   ├── mysql/
│   │   ├── mysql-secret.yaml    # Credentials MySQL
│   │   ├── mysql-pvc.yaml       # Volume persistant (10Gi)
│   │   ├── mysql-deployment.yaml # Déploiement MySQL
│   │   └── mysql-service.yaml   # Service ClusterIP
│   ├── petclinic/
│   │   ├── petclinic-configmap.yaml    # Configuration app
│   │   ├── petclinic-deployment.yaml   # Déploiement app (2 réplicas)
│   │   └── petclinic-service.yaml      # Service NodePort
│   └── ingress/
│       └── petclinic-ingress.yaml      # (Optionnel) Ingress
├── scripts/
│   ├── build.sh                 # Build de l'image Docker
│   ├── deploy.sh                # Déploiement complet
│   ├── cleanup.sh               # Nettoyage des ressources
│   └── monitor.sh               # Monitoring en temps réel
└── docs/
    ├── architecture.md          # Documentation architecture
    ├── deployment-guide.md      # Guide détaillé
    └── screenshots/             # Captures d'écran
```

---

## ✅ Tests et Validation

### Test 1 : Vérifier les Pods

```bash
kubectl get pods -n petclinic-dev
```

**Statut attendu :**
- Tous les pods en `Running`
- Colonne `READY` : `1/1`
- Aucun `RESTART`

### Test 2 : Fonctionnalité de l'Application

1. Accéder à l'application : `http://192.168.56.11:30080`
2. Cliquer sur **"Find Owners"** → **"Add Owner"**
3. Remplir le formulaire :
   - First Name : `John`
   - Last Name : `Doe`
   - Address : `123 Main St`
   - City : `Springfield`
   - Telephone : `1234567890`
4. Cliquer sur **"Add Owner"**
5. Vérifier que le propriétaire apparaît dans la liste

### Test 3 : Persistance des Données

Ce test vérifie que les données MySQL survivent au redémarrage du pod.

```bash
# Supprimer le pod MySQL
kubectl delete pod -n petclinic-dev -l app=mysql

# Attendre la recréation (30 secondes)
kubectl get pods -n petclinic-dev -w

# Rafraîchir l'application dans le navigateur
# Les données (propriétaires) doivent toujours être présentes ✅
```

**Explication :** Le PersistentVolume conserve les données même quand le pod est supprimé.

### Test 4 : Haute Disponibilité

Ce test vérifie que l'application reste accessible même si un pod tombe.

```bash
# Supprimer un pod PetClinic
kubectl delete pod -n petclinic-dev -l app=petclinic --field-selector status.phase=Running | head -1

# L'application reste accessible pendant la recréation
# Rafraîchir plusieurs fois le navigateur → Aucune erreur

# Vérifier la recréation automatique
kubectl get pods -n petclinic-dev
```

**Explication :** Le Deployment maintient toujours 2 réplicas. Kubernetes recrée automatiquement les pods supprimés.

---

## 📊 Monitoring

### Métriques des Ressources

**Installer Metrics Server :**
```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

**Consulter les métriques :**
```bash
# Utilisation CPU/RAM des nœuds
kubectl top nodes

# Utilisation CPU/RAM des pods
kubectl top pods -n petclinic-dev
```

**Exemple de sortie :**
```
NAME                         CPU(cores)   MEMORY(bytes)
mysql-xxx                    50m          250Mi
petclinic-xxx                100m         450Mi
petclinic-yyy                95m          440Mi
```

### Logs

```bash
# Logs en temps réel de tous les pods PetClinic
kubectl logs -n petclinic-dev -l app=petclinic -f

# Logs d'un pod spécifique
kubectl logs -n petclinic-dev <pod-name>

# Logs MySQL
kubectl logs -n petclinic-dev -l app=mysql
```

### Monitoring Automatisé

Utiliser le script de monitoring :

```bash
./scripts/monitor.sh
```

Ce script affiche en temps réel :
- Utilisation CPU/RAM des nœuds
- Utilisation des pods
- Statut des pods
- Services actifs

---

## 🔧 Commandes Utiles

### Gestion des Pods

```bash
# Lister tous les pods du namespace
kubectl get pods -n petclinic-dev

# Détails d'un pod
kubectl describe pod <pod-name> -n petclinic-dev

# Se connecter à un pod
kubectl exec -it <pod-name> -n petclinic-dev -- /bin/bash

# Redémarrer un déploiement
kubectl rollout restart deployment petclinic -n petclinic-dev
```

### Gestion des Services

```bash
# Lister les services
kubectl get svc -n petclinic-dev

# Détails d'un service
kubectl describe svc petclinic -n petclinic-dev

# Tester la connectivité MySQL depuis un pod
kubectl run -it --rm debug --image=mysql:8.0 --restart=Never -n petclinic-dev -- \
  mysql -h mysql -u petclinic -p
```

### Debug

```bash
# Événements récents (utile pour diagnostiquer les erreurs)
kubectl get events -n petclinic-dev --sort-by='.lastTimestamp'

# Vérifier les ConfigMaps et Secrets
kubectl get configmap -n petclinic-dev
kubectl get secret -n petclinic-dev

# Vérifier le stockage
kubectl get pvc -n petclinic-dev
kubectl describe pvc mysql-pvc -n petclinic-dev
```

---

## 🔒 Sécurité

### Bonnes Pratiques Implémentées

✅ **Secrets Kubernetes**
- Credentials MySQL stockés dans un Secret
- Pas de mots de passe en clair dans les manifests
- Variables d'environnement injectées de manière sécurisée

✅ **Resource Limits**
- Limites CPU/RAM définies pour chaque pod
- Prévention de l'épuisement des ressources du cluster

✅ **Health Checks**
- Liveness Probes : Redémarrage automatique des pods défaillants
- Readiness Probes : Retrait des pods non prêts du load balancing

✅ **Namespace Isolation**
- Ressources isolées dans un namespace dédié
- Facilite la gestion des politiques de sécurité

### Améliorations Recommandées pour Production

- Network Policies (isolation réseau)
- RBAC (contrôle d'accès granulaire)
- Pod Security Standards
- TLS/SSL pour les communications
- Scan des images Docker (Trivy, Clair)

---

## 🧹 Nettoyage

### Supprimer le Déploiement Kubernetes

```bash
# Option 1 : Script automatisé
./scripts/cleanup.sh

# Option 2 : Suppression manuelle du namespace
kubectl delete namespace petclinic-dev
```

**Note :** Cela supprime tous les composants (pods, services, PVC, secrets, etc.)

### Arrêter/Supprimer les VMs

```bash
# Sortir de la VM (si connecté)
exit

# Arrêter les VMs (conserve les données)
vagrant halt

# Redémarrer les VMs
vagrant up

# Supprimer complètement les VMs
vagrant destroy -f
```

---

## 🐛 Dépannage

### Problème : Pods en "Pending"

**Cause possible :** Ressources insuffisantes sur les workers

**Solution :**
```bash
# Vérifier les ressources disponibles
kubectl top nodes
kubectl describe nodes

# Vérifier les événements
kubectl get events -n petclinic-dev | grep <pod-name>
```

### Problème : Pods en "CrashLoopBackOff"

**Cause possible :** Erreur de connexion à MySQL ou mauvaise configuration

**Solution :**
```bash
# Consulter les logs du pod
kubectl logs -n petclinic-dev <pod-name> --previous

# Vérifier que MySQL est prêt
kubectl get pods -n petclinic-dev -l app=mysql

# Attendre que MySQL soit complètement démarré
kubectl wait --for=condition=ready pod -l app=mysql -n petclinic-dev --timeout=180s

# Redémarrer PetClinic
kubectl rollout restart deployment petclinic -n petclinic-dev
```

### Problème : Image Docker non trouvée

**Cause possible :** Image non buildée ou non poussée vers un registry

**Solution :**
```bash
# Vérifier les images locales
docker images | grep petclinic

# Rebuilder si nécessaire
./scripts/build.sh

# Si vous utilisez Docker Hub, pousser l'image
docker tag petclinic:1.0 <votre-username>/petclinic:1.0
docker push <votre-username>/petclinic:1.0

# Mettre à jour le manifest petclinic-deployment.yaml avec la nouvelle image
```

### Problème : PVC non "Bound"

**Cause possible :** Pas de StorageClass disponible

**Solution :**
```bash
# Vérifier les StorageClass
kubectl get storageclass

# Vérifier le PVC
kubectl describe pvc mysql-pvc -n petclinic-dev

# Si nécessaire, créer un PV manuellement ou utiliser un storage provisioner
```

### Problème : Cannot connect to MySQL

**Cause possible :** Service DNS non résolu ou MySQL pas prêt

**Solution :**
```bash
# Vérifier le service MySQL
kubectl get svc -n petclinic-dev mysql

# Tester la résolution DNS depuis un pod PetClinic
kubectl exec -it -n petclinic-dev <petclinic-pod> -- nslookup mysql

# Vérifier les logs MySQL
kubectl logs -n petclinic-dev -l app=mysql
```

---

## 📚 Ressources et Documentation

**Documentation du projet :**
- [Guide de Déploiement Détaillé](./docs/deployment-guide.md)
- [Documentation Architecture](./docs/architecture.md)
- [Screenshots](./docs/screenshots/)

**Ressources externes :**
- [Documentation Kubernetes](https://kubernetes.io/docs/)
- [Spring PetClinic GitHub](https://github.com/spring-projects/spring-petclinic)
- [Docker Best Practices](https://docs.docker.com/develop/dev-best-practices/)
- [Kubernetes Patterns](https://kubernetes.io/docs/concepts/cluster-administration/manage-deployment/)

---

## 🎓 Compétences Développées

Ce projet m'a permis de maîtriser :

**Conteneurisation :**
- Création de Dockerfiles multi-stage optimisés
- Build d'images légères et sécurisées
- Gestion des registries Docker

**Kubernetes :**
- Déploiement d'applications stateful et stateless
- Configuration avec ConfigMaps et Secrets
- Gestion de la persistance avec PersistentVolumes
- Mise en place de la haute disponibilité
- Implémentation de health checks
- Exposition de services (ClusterIP, NodePort)

**DevOps :**
- Provisionnement d'infrastructure avec Vagrant
- Automatisation avec scripts Bash
- Monitoring et observabilité
- Debugging d'applications conteneurisées
- Documentation technique

---

## 📝 Licence

Ce projet est sous licence MIT. Voir [LICENSE](./LICENSE) pour plus d'informations.

---

## 👤 Auteur

**Votre Nom**
- GitHub : [@votre-username](https://github.com/votre-username)
- LinkedIn : [Votre Profil](https://linkedin.com/in/votre-profil)

---

## 🤝 Contribution

Les contributions sont les bienvenues ! N'hésitez pas à :
1. Forker le projet
2. Créer une branche pour votre fonctionnalité (`git checkout -b feature/AmazingFeature`)
3. Commiter vos changements (`git commit -m 'Add some AmazingFeature'`)
4. Pusher vers la branche (`git push origin feature/AmazingFeature`)
5. Ouvrir une Pull Request

---

**Projet réalisé dans le cadre du module DevOps - Kubernetes**
