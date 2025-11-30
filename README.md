# Lab8_Wojciech_Wojciechewicz

# 1. Uruchomienie klastra Minikube z Calico (3 nody)
minikube start \
  --nodes=3 \
  --cni=calico \
  --network-plugin=cni \
  --container-runtime=docker \
  --memory=4500 \
  --cpus=2

Start klastra z Calico i 3 nodami: master + 2 workery.

# 2. Sprawdzenie nodów
```
kubectl get nodes -o wide
```
Weryfikacja, czy wszystkie nody są w stanie Ready.
<img width="1483" height="118" alt="image" src="https://github.com/user-attachments/assets/367d94d9-db2a-4390-b1b3-7dc0bb3aa432" />

# 3. Nadanie etykiet nodom (A,B,C)
```
kubectl label node minikube-m02 node=frontend
kubectl label node minikube-m03 node=backend
kubectl label node minikube node=mysql
```
Przypisanie ról: frontend, backend, mysql.

# 4. Deployment frontend (3 repliki)
```
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      nodeSelector:
        node: frontend
      containers:
      - name: nginx
        image: nginx
EOF

```
Tworzy deployment frontend na node=frontend.

# 5. Deployment backend (1 replika)
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
spec:
  replicas: 1
  selector:
    matchLabels:
      app: backend
  template:
    metadata:
      labels:
        app: backend
    spec:
      nodeSelector:
        node: backend
      containers:
      - name: nginx
        image: nginx
EOF

Backend uruchamiany na node=backend.

# 6. Pod MySQL
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: my-sql
  labels:
    app: my-sql
spec:
  nodeSelector:
    node: mysql
  containers:
  - name: mysql
    image: mysql
    env:
    - name: MYSQL_ROOT_PASSWORD
      value: root
EOF

Uruchamia MySQL na node=mysql.

# 7. Tworzenie Service dla frontend / backend / mysql
# Frontend – NodePort
kubectl expose deployment frontend --type=NodePort \
  --port=80 --target-port=80 --name=svc-frontend

# Backend – ClusterIP
kubectl expose deployment backend --type=ClusterIP \
  --port=80 --target-port=80 --name=svc-backend

# MySQL – ClusterIP
kubectl expose pod my-sql --type=ClusterIP \
  --port=3306 --target-port=3306 --name=svc-mysql


Tworzą usługi dla wszystkich komponentów.

# 8. NetworkPolicy (backend może → mysql, frontend nie)
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: mysql-access
spec:
  podSelector:
    matchLabels:
      app: my-sql
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: backend
    ports:
    - protocol: TCP
      port: 3306
EOF

Zezwala tylko backendowi na połączenie z MySQL na porcie 3306.

# 9. Testy NetworkPolicy
# 9.1 Uruchomienie tester
kubectl run tester --image=busybox:latest -it --restart=Never -- sh

# 9.2 Test backend → mysql

kubectl label pod tester app=backend --overwrite

wget -qO- svc-mysql.default.svc.cluster.local:3306

bad header line: → MySQL zwrócił odpowiedź → działa.

# 9.3 Test frontend → mysql

kubectl label pod tester app- --overwrite

wget -qO- svc-mysql.default.svc.cluster.local:3306

<img width="833" height="157" alt="image" src="https://github.com/user-attachments/assets/ac63557d-9e21-43df-96ba-feee2f61ebd4" />
<img width="803" height="173" alt="image" src="https://github.com/user-attachments/assets/70c285d1-d8e0-4138-95c8-694b5347ed7f" />

