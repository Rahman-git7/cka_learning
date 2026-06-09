# kubectl Cheatsheet CKA

## etcd
```bash
export ETCDCTL_API=3
etcdctl snapshot save /backup/etcd-backup.db \
  --cacert /etc/kubernetes/pki/etcd/ca.crt \
  --cert /etc/kubernetes/pki/etcd/server.crt \
  --key /etc/kubernetes/pki/etcd/server.key
```

## Pods
```bash
kubectl run nginx --image=nginx                           # créer un pod
kubectl run nginx --image=nginx --dry-run=client -o yaml  # générer yaml sans créer
kubectl run nginx --image=nginx --dry-run=client -o yaml > pod.yaml  # sauvegarder yaml
kubectl describe pod nginx                                # détails + image du pod
kubectl get pods -o wide                                  # détails + node du pod
kubectl delete pod nginx                                  # supprimer un pod
kubectl delete -f pod.yaml                               # supprimer via fichier
```

## ReplicaSet
```bash
kubectl edit replicaset new-replicaset          # éditer (sans .yaml)
kubectl apply -f new-replicaset.yaml            # appliquer après édition fichier
kubectl scale --replicas=3 rs new-replicaset    # scaler directement
```
⚠️ Après `kubectl edit` → supprimer les pods existants pour qu'ils prennent la nouvelle config

## Deployments
```bash
kubectl create deployment nginx --image=nginx                           # créer
kubectl create deployment nginx --image=nginx --replicas=4              # avec replicas
kubectl create deployment nginx --image=nginx --dry-run=client -o yaml > deploy.yaml
kubectl scale deployment nginx --replicas=5                             # scaler
kubectl set image deployment nginx nginx=nginx:1.18                     # changer image
kubectl rollout status deployment nginx                                 # état du rollout
kubectl rollout undo deployment nginx                                   # rollback
kubectl rollout pause deployment nginx                                  # grouper changements
kubectl rollout resume deployment nginx                                 # appliquer
```

## Apply / Create / Delete
```bash
kubectl apply -f fichier.yaml     # créer ou mettre à jour ✅ (toujours utiliser ça)
kubectl create -f fichier.yaml    # créer seulement (erreur si existe déjà)
kubectl replace -f fichier.yaml   # remplacer de façon impérative
kubectl delete -f fichier.yaml    # supprimer
```

## Services
```bash
kubectl expose deployment nginx --port=80 --type=NodePort   # exposer un deployment
kubectl expose pod nginx --port=80 --name=nginx-service     # exposer un pod
```

## Namespaces
```bash
kubectl get pods --namespace=kube-system        # pods d'un namespace spécifique
kubectl get pods --all-namespaces               # tous les namespaces
kubectl create namespace dev                    # créer un namespace
kubectl apply -f pod.yaml --namespace=dev       # déployer dans un namespace
kubectl get all -n marketing                    # tout voir dans un namespace
kubectl config set-context $(kubectl config current-context) --namespace=dev  # changer namespace par défaut
```

## Troubleshooting
```bash
cat /etc/kubernetes/manifests/kube-apiserver.yaml       # config API server (kubeadm)
cat /etc/systemd/system/kube-apiserver.service          # config API server (sans kubeadm)
ps -aux | grep kube-apiserver                           # vérifier process API server
ps -aux | grep kube-scheduler                           # vérifier process scheduler
ps -aux | grep kubelet                                  # vérifier process kubelet
```