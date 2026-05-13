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