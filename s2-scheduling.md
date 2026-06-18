## Manual Scheduling

**Le scheduler en résumé :**
```
pod créé → nodeName: (vide)
scheduler choisit un node
pod → nodeName: node-1 (rempli par le scheduler)
```

**Sans scheduler** → pod reste en état `Pending`

**Assigner manuellement un node (2 façons) :**

**1. Pod pas encore créé → nodeName dans le yaml**
```yaml
spec:
  nodeName: node-1
  containers:
    - name: nginx
      image: nginx
```

**2. Pod déjà créé → Binding object**
- On peut pas modifier `nodeName` sur un pod existant
- Il faut passer par un objet Binding envoyé à l'API server (anecdotique, jamais utilisé en vrai)

**En prod :**
- Le scheduler automatique gère tout, 99% du temps ✅
- Pour influencer le placement → `nodeSelector`, `nodeAffinity`, `taints/tolerations` (à venir)
- Binding manuel → quasi jamais utilisé



## Labels, Selectors & Annotations

**Labels** : étiquettes clé-valeur attachées à un objet (pod, service, deployment...)
**Selectors** : filtres pour sélectionner des objets selon leurs labels

**Analogie** : labels = post-its sur des boîtes, selector = "donne-moi les boîtes avec le post-it rouge"

**Où mettre les labels :**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: simple-webapp
  labels:
    app: App1
    function: Front-end
spec:
  containers:
    - name: simple-webapp
      image: simple-webapp
```

**Filtrer avec selector :**
```bash
kubectl get pods --selector app=App1
kubectl get pods --selector app=App1,function=Front-end   # plusieurs critères
```

**ReplicaSet — le piège classique :**
3 endroits où mettre des labels, attention à les faire matcher :
```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: simple-webapp
  labels:                  # 1) labels du ReplicaSet lui-même
    app: App1
spec:
  replicas: 3
  selector:
    matchLabels:
      app: App1             # 2) doit matcher le label des pods ↓
  template:
    metadata:
      labels:
        app: App1           # 3) label appliqué aux pods créés
    spec:
      containers:
        - name: simple-webapp
          image: simple-webapp
```
⚠️ Si `selector.matchLabels` ≠ `template.metadata.labels` → erreur de validation

**Annotations** : métadonnées informatives, pas utilisées pour filtrer
```yaml
metadata:
  labels:
    app: App1
  annotations:
    buildversion: "1.34"     # juste de l'info, pas de filtre possible
```

**Résumé :**
| Concept | Usage |
|---------|-------|
| labels | grouper et filtrer |
| selectors | sélectionner selon les labels |
| annotations | métadonnées informatives uniquement |


## Taints & Tolerations

**Concept :**
- `taint` → restriction posée sur un node ("je refuse les pods par défaut")
- `toleration` → permission sur un pod ("j'accepte ce taint, laisse-moi entrer")

**Analogie :**
- Taint = panneau "accès interdit" sur le node
- Toleration = badge qui permet de passer quand même

**⚠️ Point important :**
Une toleration ne force pas le pod sur ce node — elle lui donne juste le droit d'y aller.
Le scheduler peut encore le placer ailleurs. Pour forcer → utiliser `nodeAffinity`

**Les 3 effets de taint :**
| Effect | Comportement |
|--------|-------------|
| `NoSchedule` | nouveaux pods sans toleration refusés, pods existants restent |
| `PreferNoSchedule` | le scheduler évite ce node mais pas garanti |
| `NoExecute` | pods sans toleration refusés + pods existants expulsés immédiatement |

**Commandes :**
```bash
# Ajouter un taint
kubectl taint nodes node1 app=blue:NoSchedule

# Supprimer un taint (- à la fin)
kubectl taint nodes node1 app=blue:NoSchedule-

# Voir les taints d'un node
kubectl describe node node1 | grep Taint
```

**Toleration dans le yaml d'un pod :**
```yaml
spec:
  tolerations:
    - key: "app"
      operator: "Equal"
      value: "blue"
      effect: "NoSchedule"
  containers:
    - name: nginx
      image: nginx
```

**Cas d'usage en prod :**
- Node GPU réservé aux workloads ML
- Node DB réservé aux bases de données
- Maintenance → `kubectl drain` pose un taint `NoExecute` automatiquement

**Master node :**
- K8s pose un taint automatique sur le master
- Best practice : ne jamais y déployer des pods applicatifs