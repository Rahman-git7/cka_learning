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

Il stocke tout l'état du cluster. Quand tu fais kubectl get pods, Kubernetes va lire dans etcd. Quand tu déploies un pod, Kubernetes écrit dans etcd.

**3. Pourquoi c'est critique** : 

Si etcd meurt, le cluster perd toute sa mémoire. C'est pour ça que le backup etcd est une question quasi-certaine à l'exam CKA.