# La seule commande etcdctl qui compte pour le CKA
export ETCDCTL_API=3
etcdctl snapshot save /backup/etcd-backup.db \
  --cacert /etc/kubernetes/pki/etcd/ca.crt \
  --cert /etc/kubernetes/pki/etcd/server.crt \
  --key /etc/kubernetes/pki/etcd/server.key


  ## Lab pod 

Pour créer un pod avec l'image nginx: 

  `kubectl run nginx --image=nginx`

Pour describe un pod notamment pour voir son image : 

  `kubectl describe pod nginx`

Pour plus de details sur le pods (par exemple le node sur lequel il tourne) :

`kubectl get pods -o wide`

## Replicaset

Apès avoir edit un replicaset 

`kubectl edit replicaset new-replicaset.yaml`

il faut supp les pods pour qu'ils prennent la nouvelle config 

Pour scale un replicaset :

`kubectl scale --replicas=3 rs new-replicaset`

Si on passe par edit pour scale : 

`kubectl edit replicaset new-replicaset.yaml`

il faut faire un appy juste après pour appliquer les changements : 

`kubectl apply -f new-replicaset.yaml`