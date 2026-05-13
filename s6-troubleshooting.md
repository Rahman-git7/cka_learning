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