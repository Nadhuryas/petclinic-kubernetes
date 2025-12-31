# petclinic-kubernetes
# Spring PetClinic sur Kubernetes

Projet de déploiement de l'application Spring PetClinic sur un cluster Kubernetes local.

## 📝 À propos

Ce projet consiste à déployer une application Spring Boot (PetClinic) sur Kubernetes avec une base de données MySQL persistante. L'objectif est de mettre en pratique les concepts d'orchestration de conteneurs, de gestion de configuration et de monitoring.

Le déploiement se fait en **local sur une VM provisionnée via Vagrant**, ce qui me permet de simuler un environnement de production sans dépendre du cloud.

## 🎯 Objectifs du projet

- Conteneuriser une application Spring Boot
- Déployer une stack applicative complète sur Kubernetes
- Configurer la persistance des données
- Mettre en place la haute disponibilité
- Sécuriser les credentials avec Secrets
- Implémenter du monitoring basique

## 🛠️ Stack technique

- **Application**: Spring PetClinic (Java/Spring Boot)
- **Base de données**: MySQL 8.0
- **Conteneurisation**: Docker
- **Orchestration**: Kubernetes (Minikube)
- **Provisionnement**: Vagrant
- **OS**: Ubuntu 22.04

## 📋 Prérequis

Avant de commencer, assurez-vous d'avoir installé :

- Vagrant (2.3+)
- VirtualBox (ou un autre provider compatible)
- Git

## 🚀 Installation et déploiement

### Étape 1 : Cloner le repository

```bash
git clone https://github.com/votre-username/spring-petclinic-k8s.git
cd spring-petclinic-k8s
```

### Étape 2 : Démarrer la VM avec Vagrant

```bash
# Lancer la VM (provisionnée avec Minikube, Docker, kubectl)
vagrant up

# Se connecter à la VM
vagrant ssh
```

### Étape 3 : Builder l'image Docker

Une fois connecté à la VM :

```bash
# Cloner le code source de Spring PetClinic
git clone https://github.com/spring-projects/spring-petclinic.git
cd spring-petclinic

# Copier le Dockerfile depuis le projet
cp /vagrant/Dockerfile .

# Builder l'image (prend environ 5-10 minutes)
docker build -t petclinic:v1.0 .

# Vérifier que l'image est créée
docker images | grep petclinic
```

### Étape 4 : Déployer sur Kubernetes

```bash
# Retour au dossier du projet
cd /vagrant

# Déployer tous les composants
./scripts/deploy.sh

# Suivre le déploiement
kubectl get pods -n petclinic -w
```

Le script va :
1. Créer le namespace `petclinic`
2. Déployer MySQL avec son volume persistant
3. Déployer l'application PetClinic (2 réplicas)
4. Exposer l'application via un Service

### Étape 5 : Accéder à l'application

```bash
# Méthode 1 : Via Minikube service (ouvre automatiquement le navigateur)
minikube service petclinic -n petclinic

# Méthode 2 : Via port-forward
kubectl port-forward svc/petclinic 8080:80 -n petclinic
# Puis ouvrir http://localhost:8080 dans votre navigateur
```

## 🏗️ Architecture

L'application est déployée selon l'architecture suivante :

```
┌─────────────────────────────────────────┐
│         Namespace: petclinic            │
│                                         │
│  ┌──────────────┐    ┌──────────────┐  │
│  │  PetClinic   │◄───┤  ConfigMap   │  │
│  │  (2 replicas)│    └──────────────┘  │
│  │              │                       │
│  │  Port: 8080  │◄───┤   Secret     │  │
│  └──────┬───────┘    └──────────────┘  │
│         │                               │
│         ▼                               │
│  ┌──────────────┐    ┌──────────────┐  │
│  │    MySQL     │◄───┤     PVC      │  │
│  │  Port: 3306  │    │    (5Gi)     │  │
│  └──────────────┘    └──────────────┘  │
└─────────────────────────────────────────┘
         │
         ▼
    LoadBalancer
    (Port 80)
```

### Composants déployés

| Composant | Type | Replicas | Stockage |
|-----------|------|----------|----------|
| PetClinic | Deployment | 2 | - |
| MySQL | Deployment | 1 | 5Gi (PVC) |
| ConfigMap | - | - | - |
| Secret | - | - | - |

Pour plus de détails, consultez la [documentation architecture](docs/architecture.md).

## ✅ Tests et validation

### Test 1 : Vérifier que tous les pods tournent

```bash
kubectl get pods -n petclinic

# Résultat attendu :
# NAME                         READY   STATUS    RESTARTS   AGE
# mysql-xxxx                   1/1     Running   0          2m
# petclinic-xxxx               1/1     Running   0          1m
# petclinic-yyyy               1/1     Running   0          1m
```

### Test 2 : Tester l'application

1. Ouvrir l'application dans le navigateur
2. Naviguer vers "Find Owners" → "Add Owner"
3. Ajouter un propriétaire de test :
   - First Name: John
   - Last Name: Doe
   - Address: 123 Main Street
   - City: Paris
   - Telephone: 0123456789
4. Vérifier que le propriétaire apparaît dans la liste

### Test 3 : Tester la persistance des données

```bash
# Exécuter le script de test automatisé
./scripts/test-persistence.sh

# Le script va :
# 1. Compter le nombre d'entrées dans la base
# 2. Supprimer le pod MySQL
# 3. Attendre la recréation
# 4. Vérifier que les données sont toujours présentes
```

### Test 4 : Tester la haute disponibilité

```bash
# Supprimer un pod PetClinic
POD=$(kubectl get pod -n petclinic -l app=petclinic -o jsonpath='{.items[0].metadata.name}')
kubectl delete pod $POD -n petclinic

# Observer la recréation automatique
kubectl get pods -n petclinic -w

# L'application reste accessible pendant la recréation
```

## 📊 Monitoring

### Métriques des ressources

```bash
# Voir l'utilisation CPU/Mémoire des pods
kubectl top pods -n petclinic

# Voir l'utilisation des nodes
kubectl top nodes
```

### Logs

```bash
# Voir les logs de tous les pods PetClinic
kubectl logs -f deployment/petclinic -n petclinic

# Voir les logs d'un pod spécifique
kubectl logs -f <pod-name> -n petclinic

# Voir les logs MySQL
kubectl logs -f deployment/mysql -n petclinic
```

### Dashboard Kubernetes

```bash
# Ouvrir le dashboard (depuis la VM)
minikube dashboard
```

## 🔍 Commandes utiles

### Gestion des pods

```bash
# Lister tous les pods du namespace
kubectl get pods -n petclinic

# Voir les détails d'un pod
kubectl describe pod <pod-name> -n petclinic

# Se connecter à un pod
kubectl exec -it <pod-name> -n petclinic -- /bin/sh

# Redémarrer un déploiement
kubectl rollout restart deployment petclinic -n petclinic
```

### Gestion des services

```bash
# Lister les services
kubectl get svc -n petclinic

# Voir les détails d'un service
kubectl describe svc petclinic -n petclinic
```

### Debug

```bash
# Voir les événements récents
kubectl get events -n petclinic --sort-by='.lastTimestamp'

# Vérifier les ConfigMaps et Secrets
kubectl get configmap -n petclinic
kubectl get secret -n petclinic

# Vérifier le stockage
kubectl get pvc -n petclinic
```

## 🔐 Sécurité

Les bonnes pratiques de sécurité mises en œuvre :

- ✅ Aucun mot de passe en clair dans les manifests
- ✅ Utilisation de Secrets Kubernetes pour les credentials
- ✅ Utilisateur non-root dans le conteneur PetClinic
- ✅ Resources limits pour éviter l'épuisement des ressources
- ✅ Health checks pour détecter les pods défaillants

## 🗂️ Structure du projet

```
.
├── README.md
├── Vagrantfile
├── Dockerfile
├── .dockerignore
├── .gitignore
├── k8s/
│   ├── namespace.yaml
│   ├── mysql/
│   │   ├── mysql-secret.yaml
│   │   ├── mysql-pvc.yaml
│   │   ├── mysql-deployment.yaml
│   │   └── mysql-service.yaml
│   └── petclinic/
│       ├── petclinic-configmap.yaml
│       ├── petclinic-deployment.yaml
│       └── petclinic-service.yaml
├── scripts/
│   ├── deploy.sh
│   ├── cleanup.sh
│   └── test-persistence.sh
└── docs/
    ├── architecture.md
    └── screenshots/
        ├── app-running.png
        ├── pods-list.png
        ├── persistence-proof.png
        └── metrics.png
```

## 🧹 Nettoyage

### Supprimer le déploiement

```bash
# Supprimer tous les composants
./scripts/cleanup.sh

# Ou manuellement
kubectl delete namespace petclinic
```

### Arrêter la VM

```bash
# Sortir de la VM
exit

# Arrêter la VM
vagrant halt

# Supprimer complètement la VM
vagrant destroy
```

## 🐛 Troubleshooting

### Problème : Les pods ne démarrent pas

**Solution** :
```bash
# Vérifier les événements
kubectl get events -n petclinic --sort-by='.lastTimestamp'

# Voir les logs
kubectl logs <pod-name> -n petclinic

# Décrire le pod pour plus d'infos
kubectl describe pod <pod-name> -n petclinic
```

### Problème : L'image PetClinic n'est pas trouvée

**Solution** :
```bash
# Vérifier que l'image existe
docker images | grep petclinic

# Si besoin, rebuild l'image
cd spring-petclinic
docker build -t petclinic:v1.0 .
```

### Problème : PetClinic ne peut pas se connecter à MySQL

**Solution** :
```bash
# Vérifier que MySQL est bien démarré
kubectl get pods -n petclinic -l app=mysql

# Attendre que MySQL soit prêt
kubectl wait --for=condition=ready pod -l app=mysql -n petclinic --timeout=180s

# Redémarrer PetClinic
kubectl rollout restart deployment petclinic -n petclinic
```

### Problème : Le stockage persistant ne fonctionne pas

**Solution** :
```bash
# Vérifier le PVC
kubectl get pvc -n petclinic
kubectl describe pvc mysql-pvc -n petclinic

# Sur Minikube, le provisioner par défaut devrait fonctionner
# Si problème, vérifier les StorageClass
kubectl get storageclass
```

## 📚 Ce que j'ai appris

Au cours de ce projet, j'ai acquis les compétences suivantes :

- Création de Dockerfiles multi-stage optimisés
- Déploiement d'applications stateful sur Kubernetes
- Gestion de la persistance avec PersistentVolumes
- Configuration d'applications avec ConfigMaps et Secrets
- Mise en place de health checks et probes
- Debugging d'applications conteneurisées
- Documentation technique et architecture

## 🔗 Ressources

- [Documentation Kubernetes](https://kubernetes.io/docs/)
- [Spring PetClinic GitHub](https://github.com/spring-projects/spring-petclinic)
- [Docker Best Practices](https://docs.docker.com/develop/dev-best-practices/)
- [Minikube Documentation](https://minikube.sigs.k8s.io/docs/)

