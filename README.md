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
kubectl get nodes -o wide

Weryfikacja, czy wszystkie nody są w stanie Ready.
<img width="1483" height="118" alt="image" src="https://github.com/user-attachments/assets/367d94d9-db2a-4390-b1b3-7dc0bb3aa432" />

3. Nadanie etykiet nodom (A,B,C)
kubectl label node minikube-m02 node=frontend
kubectl label node minikube-m03 node=backend
kubectl label node minikube node=mysql


Opis:
Przypisanie ról nodom: frontend, backend, mysql.

4. Deployment frontend (3 repliki)
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


Opis:
Tworzy deployment frontend na node=frontend.

5. Deployment backend (1 replika)
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


Opis:
Backend uruchamiany na node=backend.

6. Pod MySQL
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


Opis:
Uruchamia MySQL na node=mysql.

7. Tworzenie Service dla frontend / backend / mysql
Frontend – NodePort
kubectl expose deployment frontend --type=NodePort \
  --port=80 --target-port=80 --name=svc-frontend

Backend – ClusterIP
kubectl expose deployment backend --type=ClusterIP \
  --port=80 --target-port=80 --name=svc-backend

MySQL – ClusterIP
kubectl expose pod my-sql --type=ClusterIP \
  --port=3306 --target-port=3306 --name=svc-mysql


Opis:
Tworzą usługi dla wszystkich komponentów.

8. NetworkPolicy (backend może → mysql, frontend nie)
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


Opis:
Zezwala tylko backendowi na połączenie z MySQL na porcie 3306.

9. Testy NetworkPolicy
9.1 Uruchom tester
kubectl run tester --image=busybox:latest -it --restart=Never -- sh

9.2 Test backend → mysql (ma działać)

Nadaj testerowi etykietę backend (w drugim terminalu):

kubectl label pod tester app=backend --overwrite


W testerze:

wget -qO- svc-mysql.default.svc.cluster.local:3306


Oczekiwane:
bad header line: → MySQL zwrócił odpowiedź → działa.

# 9.3 Test frontend → mysql (ma być blokada)

Usuń etykietę:

kubectl label pod tester app- --overwrite


W testerze:

wget -qO- svc-mysql.default.svc.cluster.local:3306


Oczekiwane:
connection timed out → ruch zablokowany.
