# La seule commande etcdctl qui compte pour le CKA
export ETCDCTL_API=3
etcdctl snapshot save /backup/etcd-backup.db \
  --cacert /etc/kubernetes/pki/etcd/ca.crt \
  --cert /etc/kubernetes/pki/etcd/server.crt \
  --key /etc/kubernetes/pki/etcd/server.key