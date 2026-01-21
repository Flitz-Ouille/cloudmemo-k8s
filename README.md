# CloudMemo - Déploiement Multi-Locataire Kubernetes

## Aperçu

Ce projet démontre un **déploiement multi-locataire Kubernetes** de l'application CloudMemo utilisant :

- **Docker** pour la conteneurisation et la gestion des images
- **Kubernetes** pour l'orchestration avec isolation par namespace
- **Network Policies** pour la sécurité et le contrôle de la communication inter-namespace
- **Redis** comme service avec état par namespace
- **Ansible** pour l'automation de l'infrastructure (optionnel)

Le déploiement comprend deux namespaces séparés (`team-blue` et `team-green`), chacun avec des instances isolées de CloudMemo et Redis, démontrant les bonnes pratiques de multi-location et les politiques de sécurité réseau.

---

## Architecture

### Structure du Projet

```
cloudmemo-k8s/
├── Dockerfile              # Build de l'image Docker pour CloudMemo
├── docker-compose.yaml     # Tests locaux avec Docker Compose
├── README.md               # Ce fichier
├── kubernetes/
│  ├── namespaces.yaml      # Définitions des namespaces (team-blue, team-green)
│  ├── cloudmemo.yaml       # Deployment & Service CloudMemo
│  ├── redis.yaml           # StatefulSet & Service Redis
│  ├── ingress.yaml         # Routes Ingress (blue/green)
│  ├── network-policies/
│  │  ├── default-deny.yaml
│  │  └── allow-app-redis.yaml
│  └── ansible/             # Playbooks d'automation (optionnel)
│     └── deploy.yaml       # Automation de déploiement Kubernetes
```

---

## Prérequis

- **Docker** installé (pour les tests locaux)
- Cluster **Kubernetes** (v1.20+)
- **kubectl** configuré pour accéder à votre cluster
- Plugin CNI **Calico** installé (pour les NetworkPolicies)
- Compte **Docker Hub** (pour pousser les images)

---

## Étape 1 : Build et Test Local avec Docker

### Construire l'image Docker CloudMemo

```bash
docker build -t flitzouille/cloudmemo:1.0 .
```

### Tester avec Docker Compose

```bash
docker-compose up -d
```

Cela démarre :
- **CloudMemo** sur `http://localhost:5000`
- **Redis** sur `localhost:6379`

Vérifier que tout fonctionne :

```bash
curl http://localhost:5000/
```

### Pousser l'image vers Docker Hub

```bash
docker login
docker push flitzouille/cloudmemo:1.0
```

---

## Étape 2 : Déployer sur Kubernetes

### 1. Créer les namespaces

```bash
kubectl apply -f kubernetes/namespaces.yaml
```

### 2. Déployer Redis dans les deux namespaces

```bash
kubectl apply -n team-blue  -f kubernetes/redis.yaml
kubectl apply -n team-green -f kubernetes/redis.yaml
```

Vérifier que Redis fonctionne :

```bash
kubectl get pods -n team-blue
kubectl get pods -n team-green
```

### 3. Déployer CloudMemo dans les deux namespaces

```bash
kubectl apply -n team-blue  -f kubernetes/cloudmemo.yaml
kubectl apply -n team-green -f kubernetes/cloudmemo.yaml
```

Vérifier le déploiement :

```bash
kubectl get all -n team-blue
kubectl get all -n team-green
```

### 4. Déployer l'Ingress (optionnel)

```bash
kubectl apply -f kubernetes/ingress.yaml
```

Cela crée :
- `blue.lucas.tp.kaas-mvp-ts.fr` → CloudMemo team-blue
- `green.lucas.tp.kaas-mvp-ts.fr` → CloudMemo team-green

---

## Étape 3 : Appliquer les Politiques Réseau

Les Network Policies appliquent l'isolation multi-locataire :

### Deny par Défaut (bloque tout le trafic entrant)

```bash
kubectl apply -n team-blue  -f kubernetes/network-policies/default-deny.yaml
kubectl apply -n team-green -f kubernetes/network-policies/default-deny.yaml
```

### Autoriser App → Redis (port 6379, même namespace)

```bash
kubectl apply -n team-blue  -f kubernetes/network-policies/allow-app-redis.yaml
kubectl apply -n team-green -f kubernetes/network-policies/allow-app-redis.yaml
```

**Résultat** : CloudMemo ne peut accéder à Redis que dans son propre namespace. L'accès inter-namespace est bloqué.

---

## Tests

### Accéder aux applications localement

Utiliser `kubectl port-forward` pour un accès local :

```bash
# Terminal 1 : team-blue
kubectl port-forward -n team-blue deploy/cloudmemo 5001:5000

# Terminal 2 : team-green
kubectl port-forward -n team-green deploy/cloudmemo 5002:5000
```

Accès depuis votre PC :
- **team-blue** : `http://localhost:5001`
- **team-green** : `http://localhost:5002`

### Tester l'isolation des données

1. Créer un mémo dans `http://localhost:5001` (team-blue)
2. Créer un mémo différent dans `http://localhost:5002` (team-green)
3. Vérifier que chaque namespace ne voit que ses propres données

### Tester le scénario de "piratage"

Prouver que les Network Policies empêchent l'accès inter-locataires :

```bash
# Modifier CloudMemo team-blue pour pointer vers Redis team-green
kubectl edit deploy cloudmemo -n team-blue
```

Changer `REDIS_HOST` de `redis` à `redis.team-green.svc.cluster.local`

Résultat attendu :
- CloudMemo blue échouera à se connecter (timeout de connexion / erreur DNS)
- La Network Policy bloque le trafic inter-namespace
- Restaurer `REDIS_HOST` à `redis` pour corriger

---

## Variables d'Environnement

### CloudMemo

| Variable | Défaut | Description |
|----------|--------|-------------|
| `REDIS_HOST` | `redis` | Nom d'hôte Redis (même namespace) |
| `REDIS_PORT` | `6379` | Port Redis |
| `FLASK_ENV` | `production` | Environnement Flask |

### Surcharger Docker Compose

Éditer `docker-compose.yaml` pour personnaliser les ports, les paramètres Redis, etc.

---

## Dépannage

### Les pods restent bloqués en `ContainerCreating`

**Cause** : Plugins CNI non trouvés  
**Solution** : S'assurer que `/usr/lib/cni` contient les binaires Calico :

```bash
ln -s /opt/cni/bin/* /usr/lib/cni/
systemctl restart kubelet
```

### PVC Redis en `Pending`

**Cause** : Pas de StorageClass / PV disponible  
**Solution** : Utiliser un volume `emptyDir` au lieu de `volumeClaimTemplates` (données perdues au redémarrage)

### CloudMemo ne peut pas atteindre Redis

**Cause 1** : Network Policy bloque le trafic  
**Solution** : Vérifier que `allow-app-redis.yaml` est appliquée

**Cause 2** : Mauvais nom DNS  
**Solution** : Vérifier que le pod utilise `REDIS_HOST=redis` (intra-namespace) ou `redis.team-green.svc.cluster.local` (inter-namespace)

### Port-forward non accessible depuis le PC Windows

**Cause** : Port-forward écoute seulement sur 127.0.0.1  
**Solution** : Utiliser un tunnel SSH depuis Windows :

```powershell
ssh -L 5001:127.0.0.1:5001 -L 5002:127.0.0.1:5002 lucas@192.168.20.150
```

Ensuite accéder à `http://localhost:5001` et `http://localhost:5002` sur votre PC.

---

## Checklist de Validation

- [ ] L'image Docker se construit et fonctionne localement
- [ ] Les deux namespaces sont créés dans Kubernetes
- [ ] Les pods CloudMemo s'exécutent dans les deux namespaces
- [ ] Les pods Redis s'exécutent dans les deux namespaces
- [ ] CloudMemo blue est accessible et fonctionnel
- [ ] CloudMemo green est accessible et fonctionnel
- [ ] Isolation des données confirmée (mémos différents dans chaque namespace)
- [ ] Les Network Policies sont appliquées avec succès
- [ ] La politique default-deny bloque le trafic inconnu
- [ ] La politique app-to-redis autorise l'accès intra-namespace
- [ ] La tentative d'accès inter-namespace échoue comme prévu
- [ ] La restauration du bon `REDIS_HOST` corrige l'application

---

## Concepts Clés

### Multi-Location

Chaque namespace (`team-blue`, `team-green`) est un locataire séparé avec :
- Des pods isolés
- Des services isolés
- Du stockage isolé (Redis)
- Aucune fuite de données entre locataires

### Network Policies

**Deny par Défaut** : Tout le trafic entrant vers tous les pods est bloqué par défaut.  
**Allow Explicite** : Seul le trafic spécifié (app → Redis sur port 6379) est autorisé.  
**Même Namespace Uniquement** : La communication inter-namespace est bloquée.

### Objets Kubernetes Utilisés

- **Namespace** : Partitionnement logique du cluster
- **Deployment** : Gère les réplicas CloudMemo
- **StatefulSet** : Gère Redis avec identité stable
- **Service** : ClusterIP pour la communication inter-pods
- **NetworkPolicy** : Contrôle du trafic entrant

---

## Auteurs

**Lucas LESENS**, Vincent ARSON, Théophane DE SAN NICOLAS

Projet : Kubernetes - Déploiement Multi-Tenant Infrastructure

---

## Licence

Non spécifiée (à des fins éducatives).
