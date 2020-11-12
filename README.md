# kubernetes-examples
Works in kubernetes v1.18.9
Ubuntu 18.04
Deployed from Kubespray


1 How to clear environment from previous Kubernetes installation
```

docker rm -f $(docker ps -a -q); docker rmi $(docker images -q)
docker rm -vf $(docker ps -a -q)
docker rmi -f $(docker images -a -q)

systemctl stop kubelet
docker volume rm $(docker volume ls -q)
cleanupdirs="/var/lib/etcd /etc/kubernetes /etc/cni /opt/cni /var/lib/cni /var/run/calico"
for dir in $cleanupdirs; do
  echo "Removing $dir"
  rm -rf $dir
done

# full remove docker
apt-get purge -y docker-engine docker docker.io docker-ce
apt-get autoremove -y --purge docker-engine docker docker.io docker-ce --allow-change-held-packages
umount /var/lib/docker/
rm -rf /var/lib/docker /etc/docker
rm /etc/apparmor.d/docker
groupdel docker
rm -rf /var/run/docker.sock
rm -rf /usr/bin/docker-compose

#detailed docker logs
docker logs $(docker ps -lq)

#remove kubernetes
apt-get purge kubeadm kubectl kubelet kubernetes-cni -y
apt-get autoremove
rm -rf ~/.kube
rm -rf /etc/cni /etc/kubernetes /var/lib/dockershim /var/lib/etcd /var/lib/kubelet /var/run/kubernetes ~/.kube/*
apt install kubelet kubeadm kubectl kubernetes-cni

#remove calico
rm -rf /etc/ceph \
       /etc/cni \
       /etc/kubernetes \
       /opt/cni \
       /opt/rke \
       /run/secrets/kubernetes.io \
       /run/calico \
       /run/flannel \
       /var/lib/calico \
       /var/lib/etcd \
       /var/lib/cni \
       /var/lib/kubelet \
       /var/lib/rancher/rke/log \
       /var/log/containers \
       /var/log/kube-audit \
       /var/log/pods \
       /var/run/calico \
       /etc/ssl/etcd

```

2 Typical daemonset with nginx

```
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: nginx-daemon
  namespace: default
  labels:
    k8s-app: nginx-daemon
spec:
  selector:
    matchLabels:
      name: nginx-daemon
  template:
    metadata:
      labels:
        name: nginx-daemon
    spec:
      hostNetwork: true
      dnsPolicy: ClusterFirstWithHostNet
      tolerations:
        # this toleration is to have the daemonset runnable on master nodes
        # remove it if your masters can't run pods
        - key: node-role.kubernetes.io/master
          effect: NoSchedule
      containers:
        - image: nginx
          name: nginx
          ports:
            - name: http
              containerPort: 80
            - name: https
              containerPort: 443
          volumeMounts:
            - mountPath: /etc/nginx/conf.d/
              name: nginx-confd
            - mountPath: /var/www/
              name: nginx-web-content
      volumes:
        - name: nginx-confd
          hostPath:
            path: /etc/nginx/conf.d/
        - name: nginx-web-content
          hostPath:
            path: /var/www/

```
 
3 Start rabbitmq in Kubernetes and create user for management plugin

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: rmq
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      app: rmq
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: rmq
    spec:
      imagePullSecrets:
        - name: gitlabregistrykey
      containers:
        - name: rmq
          image: rabbitmq:3.8.9-management-alpine # with management
#          env:
#            - name: RABBITMQ_DEFAULT_USER
#              value: "guest"
#            - name: RABBITMQ_DEFAULT_PASSWORD
#              value: "somepassword"
          ports:
            - name: http
              protocol: TCP
              containerPort: 15672
            - name: amqp
              protocol: TCP
              containerPort: 5672
          lifecycle:
            postStart:
              exec:
                command: ["/bin/sh", "-c", "sleep 15 && rabbitmqctl add_user admin MegaPassword && rabbitmqctl set_user_tags admin administrator && rabbitmqctl set_permissions admin '.*' '.*' '.*'"]




          livenessProbe:
            exec:
              # This is just an example. There is no "one true health check" but rather
              # several rabbitmq-diagnostics commands that can be combined to form increasingly comprehensive
              # and intrusive health checks.
              # Learn more at https://www.rabbitmq.com/monitoring.html#health-checks.
              #
              # Stage 2 check:
              command: ["rabbitmq-diagnostics", "status"]
            initialDelaySeconds: 60
            # See https://www.rabbitmq.com/monitoring.html for monitoring frequency recommendations.
            periodSeconds: 60
            timeoutSeconds: 15
          imagePullPolicy: Always
      restartPolicy: Always

---
apiVersion: v1
kind: Service
metadata:
  name: rmq
spec:
  type: NodePort
  ports:
    - name: http
      protocol: TCP
      port: 15672
      targetPort: 15672
      nodePort: 31672
    - name: amqp
      protocol: TCP
      port: 5672
      targetPort: 5672
      nodePort: 30672
  selector:
    app: rmq

```
4 Start typical NodeJS application in Kubernetes and expose ports

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mega-frontend
  namespace: default
spec:
  replicas: 3
  selector:
    matchLabels:
      app: mega-frontend
  strategy:
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: mega-frontend
    spec:
      imagePullSecrets:
        - name: gitlabregistrykey # You must create your own secret for access to Docker Repository
      containers:
        - name: mega-frontend
          image: gitlab.mega-frontend.com:5005/mega-frontend/mega-frontend:prod # Path to your Docker image
          ports:
            - containerPort: 4200
          imagePullPolicy: Always
      restartPolicy: Always

---
apiVersion: v1
kind: Service
metadata:
  name: mega-frontend
spec:
  type: NodePort
  ports:
    - port: 4200 # Expose the service on the specified port internally within the cluster. 
      nodePort: 31726 # Exposed outside Kubernetes on this port
      targetPort: 4200 # This is the port on the pod that the request gets sent to. 
  selector:
    app: mega-frontend

```

5 This is the secret for Docker Registry.


*Documentation - https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/

```
apiVersion: v1
kind: Secret
metadata:
  name: gitlabregistrykey
stringData:
  .dockerconfigjson: |
    {"auths":{"registry.gitlab.com":{"auth":"gitlabregistrykey"}}}
type: kubernetes.io/dockerconfigjson
```

