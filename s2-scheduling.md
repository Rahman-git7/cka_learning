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


## Node Selector

**Rôle** : forcer un pod à aller sur un node avec un label spécifique

**Différence avec taints/tolerations :**
- `taint/toleration` → empêche un pod d'aller quelque part (restriction)
- `nodeSelector` → force un pod à aller quelque part (ciblage)

**Pourquoi l'utiliser :**
Le scheduler optimise selon CPU/RAM dispo, mais connaît pas les contraintes métier (besoin SSD, GPU, région spécifique...)

**Cas d'usage réels :**
- Pod avec besoin de RAM élevé → node taillé pour ça
- Pod qui a besoin d'un SSD → `nodeSelector: disk=ssd`
- Pod GPU → `nodeSelector: gpu=true`
- Contrainte de région → `nodeSelector: region=eu-west`

**Utilisation en 2 étapes :**

**1. Labelliser le node**
```bash
kubectl label nodes node1 size=Large
```

**2. Cibler le label dans le pod**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
spec:
  nodeSelector:
    size: Large
  containers:
    - name: nginx
      image: nginx
```

**Modifier le placement d'un pod existant :**
- Pod seul → immuable, faut delete + recreate
- Deployment → modifier `nodeSelector` dans le yaml + `kubectl apply` → rolling update automatique ✅

**Commandes utiles :**
```bash
kubectl label nodes node1 size=Large       # ajouter un label à un node
kubectl get nodes --show-labels            # voir les labels des nodes
kubectl label nodes node1 size-            # supprimer un label (- à la fin)
```

## Node Affinity

**Rôle** : version avancée de nodeSelector — permet des règles plus flexibles (plusieurs valeurs, préférences)

**nodeSelector vs nodeAffinity :**
| | nodeSelector | nodeAffinity |
|---|---|---|
| Critères | 1 seule valeur exacte | plusieurs valeurs, opérateurs (In, NotIn...) |
| Flexibilité | obligatoire uniquement | obligatoire OU préférence |

**Les 2 combos qui existent :**

**1. requiredDuringScheduling + IgnoredDuringExecution**
```
Au placement → node DOIT avoir le label, sinon pod reste Pending
Pendant l'exécution → si le label du node change → rien ne se passe, pod reste où il est
```

**2. preferredDuringScheduling + IgnoredDuringExecution**
```
Au placement → on essaie de matcher, sinon pod placé ailleurs quand même
Pendant l'exécution → pareil, rien ne se passe si le label change
```

**Résumé simple :**
```
preferred → pas de match → pod placé quand même ✅
required  → pas de match → pod reste Pending ❌
```

**yaml — required :**
```yaml
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: size
                operator: In
                values:
                  - Large
                  - Medium
```

**Pourquoi "IgnoredDuringExecution" :**
Si le label du node change après que le pod soit placé → K8s n'expulse pas le pod, par souci de stabilité

**Cas d'usage en prod :**
- Clusters avec types d'instances variés (EKS node groups)
- Multi-AZ → forcer un pod à rester dans une AZ
- Spot vs On-Demand
- Isolation dev/staging/prod

**Bonne pratique** : utiliser `preferred` plus souvent que `required` pour rester flexible