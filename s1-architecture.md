## Composant de Kubernetes

Deux grands groupes : 

**Master node :** 

* **etcd** → la base de données du cluster, stocke tout l'état
* **kube-scheduler** → décide sur quel node un pod va tourner
* **controller-manager** → surveille et maintient l'état désiré
* **kube-apiserver** → permet la communication entre les composants


**Worker node :** Host Application as container

* **kubelet** → l'agent sur chaque worker, fait tourner les pods
* **kube-proxy** → gère le réseau entre les pods

## ETCD bases

C'est quoi ***etcd*** ? 

Une base de données clé-valeur. Comme un dictionnaire — tu cherches une clé, tu obtiens une valeur.

```bash
clé : "node1"  →  valeur : "ready"
clé : "pod/nginx"  →  valeur : "running"
```

**2. Son rôle dans Kubernetes** : 

Il stocke tout l'état du cluster. Quand tu fais `kubectl get pods`, Kubernetes va lire dans etcd. Quand tu déploies un pod, Kubernetes écrit dans etcd.

**3. Pourquoi c'est critique** : 

Si etcd meurt, le cluster perd toute sa mémoire. C'est pour ça que le backup etcd est une question quasi-certaine à l'exam CKA.


### `etcd` dans Kubernetes
- Stocke tout l'état du cluster (pods, nodes, configs...)
- Tourne dans un pod dans le namespace kube-system
- La seule commande importante : etcdctl snapshot save → backup
- Le reste est géré automatiquement par K8s


* Cluster installé avec kubeadm → etcd tourne comme un pod dans kube-system
* Cluster installé manuellement → etcd tourne comme un service directement sur le node


## Kube Api Server 

**Exemple de flow pour la création d'un pod**

```
kubectl run nginx --image=nginx
        ↓
1. API server → authentifie et valide ta requête
        ↓
2. API server → crée l'objet pod dans etcd (sans node assigné)
        ↓
3. Scheduler → détecte un pod sans node, choisit le meilleur node
        ↓
4. API server → met à jour etcd avec le node choisi
        ↓
5. Kubelet → détecte que le pod lui est assigné, crée le container
        ↓
6. Kubelet → remonte le statut à l'API server → etcd mis à jour
        ↓
pod qui tourne
```

### Cas de troubleshooting 

Savoir la chaine complète du kube api server sert notamment a résoudre les soucis relatifs a Kube 

```bash
# Voir la config de l'API server (cluster monté avec kubeadm)
cat /etc/kubernetes/manifests/kube-apiserver.yaml

# Voir la config de l'API server (cluster monté manuellement)
cat /etc/systemd/system/kube-apiserver.service

# Voir si l'API server tourne bien
ps -aux | grep kube-apiserver
````

À quoi ça sert :

* Le fichier `.yaml` → voir les options de démarrage (certs, ports, flags activés)
* Le fichier `service` → pareil mais pour les clusters sans kubeadm
* `Le ps -aux` → vérifier que le processus tourne bien sur le node

*Quand tu t'en sers :*

Uniquement en troubleshooting quand quelque chose cloche au niveau du cluster lui-même. En exam ça peut tomber dans une question où un composant du control plane est cassé.


## Kube Controller Manager
- Surveille en boucle que l'état réel = état désiré
- Contient plein de controllers : node, replication, deployment...
- Node controller : détecte les nodes morts et replanifie les pods
```bash
# Emplacement config (kubeadm)
cat /etc/kubernetes/manifests/kube-controller-manager.yaml

# Sans kubeadm
cat /etc/systemd/system/kube-controller-manager.service
```

## Kube Scheduler

**Rôle** : décide sur quel node placer un pod, mais ne le crée pas (c'est le kubelet qui crée)

**Comment il choisit :**
1. Filtre — élimine les nodes incompatibles (pas assez CPU/RAM, taints, node selector...)
2. Score — choisit le node qui aura le plus de ressources libres après placement

**Si aucun node dispo** → pod reste en état `Pending`

**Config :**
```bash
cat /etc/kubernetes/manifests/kube-scheduler.yaml  # kubeadm
ps -aux | grep kube-scheduler                       # vérifier le process
```

**Personnalisable via** : resource limits, taints/tolerations, node selector (détails section scheduling)

## Kubelet

Le **kubelet** c'est l'agent sur chaque worker node. C'est lui qui fait le vrai travail sur la machine.
Il fait 3 choses :

* Reçoit les instructions du scheduler via l'API server — "crée ce pod sur ce node"
* Demande au container runtime (containerd) de lancer le container
* Remonte le statut en permanence à l'API server — "mon pod tourne bien / il est mort"

Analogie : si le scheduler est le chef de chantier qui décide où construire, le kubelet c'est l'ouvrier sur le terrain qui construit vraiment.

**Point important** : le kubelet est le seul composant qui est jamais déployé automatiquement par **kubeadm**.

 Tu dois l'installer manuellement sur chaque node. Ça tombe parfois en exam.

 ## Kube Proxy

**Rôle** : tourne sur chaque node, gère les règles réseau pour que les pods communiquent entre eux

**Problème qu'il résout :**
Un pod DB peut mourir et se relancer avec une nouvelle IP
→ solution : créer un Service devant la DB

**Le Service :**
- Pas un pod, c'est un objet virtuel K8s gardé en mémoire
- IP fixe qui change jamais
- Sert d'intermédiaire stable devant des pods

pod app → Service DB (IP fixe) → pod DB (IP variable)


**kube-proxy** maintient ces règles de redirection sur chaque node

## Pods

**C'est quoi** : la plus petite unité déployable dans K8s, tourne à l'intérieur d'un node
Un pod = une instance de ton application

**Scaling :**
- Plus d'users → on ajoute des pods, pas des containers dans le même pod
- Node saturé → on ajoute un nouveau node avec ses pods

```bash
pod1 → container app
pod2 → container app   ✅ bonne façon de scaler
pod3 → container app
````

**Multi-container pods :**
- Un pod peut avoir plusieurs containers mais c'est rare
- Cas d'usage : un container principal + un helper (ex: collecteur de logs)
- Ils partagent le même réseau et stockage
- Le helper existe uniquement pour assister l'app principale


**Avantages :**
- Partagent automatiquement le même réseau et stockage
- Pas de config manuelle contrairement à Docker --link
- Vivent et meurent ensemble

pod → container app web + container log collector

**Commande de base :**
```bash
kubectl run nginx --image=nginx    # créer un pod
kubectl get pods                   # lister les pods
```

## Pods avec YAML

**Les 4 champs obligatoires dans tout fichier K8s :**
```yaml
apiVersion: v1        # version de l'API K8s
kind: Pod             # type d'objet
metadata:             # infos du pod
spec:                 # ce qu'on veut faire tourner
```

**Structure complète :**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod        # identifiant unique du pod
  labels:
    app: myapp           # étiquette pour grouper/filtrer
    type: frontend       # autre étiquette optionnelle
spec:
  containers:
    - name: nginx        # nom du container
      image: nginx       # image à pull
```

**name vs labels :**
- `name` → identifiant unique, deux pods peuvent pas avoir le même
- `labels` → étiquettes pour grouper et filtrer plusieurs pods

**Indentation :**
- Toujours 2 espaces, jamais de tabulations
- Si quelque chose appartient à quelque chose → indent de 2 espaces
- Le tiret `-` = élément d'une liste

**Commandes :**
```bash
kubectl create -f pod.yaml    # créer depuis un fichier yaml
kubectl get pods              # lister les pods
kubectl describe pod myapp-pod # détails d'un pod
```

## ReplicaSet

**Rôle** : garantit qu'un nombre défini de pods tourne en permanence
- Si un pod meurt → le ReplicaSet en recrée un automatiquement
- Utile même avec 1 seul pod (relance si crash)
- Permet le load balancing entre plusieurs pods

**ReplicationController vs ReplicaSet :**
- ReplicationController → ancienne version, oublie-la
- ReplicaSet → nouvelle version, toujours utiliser celle-ci ✅

**Structure yaml :**
```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: myapp-replicaset
spec:
  template:             
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: nginx
          image: nginx
  replicas: 3
  selector:
    matchLabels:
      app: myapp
```

**selector vs template :**
- `selector` → identifie les pods existants à gérer (peut adopter des pods déjà existants)
- `template` → utilisé pour créer de nouveaux pods si un pod meurt

**Commandes :**
```bash
kubectl create -f replicaset.yaml
kubectl get replicaset
kubectl get pods
kubectl delete replicaset myapp-replicaset
```

## Deployments

**Rôle** : gère les ReplicaSets et permet les mises à jour sans coupure

**Avantages vs ReplicaSet seul :**
- Rolling updates → pods mis à jour un par un, aucune coupure pour les users
- Rollback → revenir à la version précédente si problème
- Pause/resume → grouper plusieurs changements avant de les appliquer

**Rolling update :**
```
v1 v1 v1 v1   # état initial
v2 v1 v1 v1   # pod 1 mis à jour
v2 v2 v2 v2   # terminé, aucune coupure
```

**Structure yaml :**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      containers:
        - name: nginx
          image: nginx
```

**Commandes :**
```bash
kubectl create -f deployment.yaml
kubectl get deployments
kubectl get all                          # voir tout : deployment, replicaset, pods
kubectl rollout pause deployment myapp   # grouper des changements
kubectl rollout resume deployment myapp  # appliquer tous les changements
kubectl rollout status deployment myapp  # voir l'état du déploiement
kubectl rollout undo deployment myapp    # rollback version précédente
```

**En prod** : c'est le pipeline CI/CD (GitLab CI + ArgoCD) qui gère tout ça automatiquement