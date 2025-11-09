# Lab5_Wojciech_Wojciechewicz

## 1. Utworzenie przestrzeni nazw

Utworzyłem dwie przestrzenie nazw:
- ns-dev – środowisko deweloperskie,  
- ns-prod – środowisko produkcyjne.

Polecenia:
```
kubectl create namespace ns-dev
kubectl create namespace ns-prod
kubectl get namespaces
```
## 2. Konfiguracja ograniczeń zasobów dla ns-dev
Plik quota-dev.yaml
```
apiVersion: v1
kind: ResourceQuota
metadata:
  name: quota-dev
  namespace: ns-dev
spec:
  hard:
    pods: "10"
    requests.cpu: "2"
    requests.memory: "2Gi"
    limits.cpu: "2"
    limits.memory: "2Gi"
```
Zastosowanie:
```
kubectl apply -f quota-dev.yaml
kubectl describe resourcequota quota-dev -n ns-dev
```
Plik limitrange-dev.yaml
```
apiVersion: v1
kind: LimitRange
metadata:
  name: limits-dev
  namespace: ns-dev
spec:
  limits:
  - type: Container
    default:
      cpu: 200m
      memory: 256Mi
    defaultRequest:
      cpu: 100m
      memory: 128Mi
    max:
      cpu: 200m
      memory: 256Mi
```
Zastosowanie:
```
kubectl apply -f limitrange-dev.yaml
kubectl describe limitrange limits-dev -n ns-dev
```
## 3. Konfiguracja ograniczeń zasobów dla ns-prod

Namespace produkcyjny ma dwukrotnie większe limity.
Plik quota-prod.yaml
```
apiVersion: v1
kind: ResourceQuota
metadata:
  name: quota-prod
  namespace: ns-prod
spec:
  hard:
    pods: "20"
    requests.cpu: "4"
    requests.memory: "4Gi"
    limits.cpu: "4"
    limits.memory: "4Gi"
```
Plik limitrange-prod.yaml
```
apiVersion: v1
kind: LimitRange
metadata:
  name: limits-prod
  namespace: ns-prod
spec:
  limits:
  - type: Container
    default:
      cpu: 400m
      memory: 512Mi
    defaultRequest:
      cpu: 200m
      memory: 256Mi
    max:
      cpu: 400m
      memory: 512Mi
```
Zastosowanie:
```
kubectl apply -f quota-prod.yaml
kubectl apply -f limitrange-prod.yaml
kubectl describe resourcequota quota-prod -n ns-prod
kubectl describe limitrange limits-prod -n ns-prod
```
## 4. Testowanie Deploymentów
4.1 Deployment no-test (niepoprawny – przekracza limity)
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: no-test
  namespace: ns-dev
spec:
  replicas: 1
  selector:
    matchLabels:
      app: no-test
  template:
    metadata:
      labels:
        app: no-test
    spec:
      containers:
      - name: nginx
        image: nginx
        resources:
          requests:
            cpu: "500m"
            memory: "512Mi"
          limits:
            cpu: "1"
            memory: "1Gi"
```
Polecenia:
```
kubectl apply -f no-test.yaml
kubectl get pods -n ns-dev
kubectl describe deployment no-test -n ns-dev
```
Wynik:
Pod nie został uruchomiony z powodu przekroczenia limitów (exceeded quota).
## 4.2 Deployment yes-test (poprawny – mieści się w limitach)
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: yes-test
  namespace: ns-dev
spec:
  replicas: 1
  selector:
    matchLabels:
      app: yes-test
  template:
    metadata:
      labels:
        app: yes-test
    spec:
      containers:
      - name: nginx
        image: nginx
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "200m"
            memory: "256Mi"
```
Polecenia:
```
kubectl apply -f yes-test.yaml
kubectl get pods -n ns-dev
kubectl describe deployment yes-test -n ns-dev
```
Wynik:
Pod uruchomił się poprawnie (STATUS: Running).
## 4.3 Deployment zero-test (bez deklaracji – używa LimitRange)
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: zero-test
  namespace: ns-dev
spec:
  replicas: 1
  selector:
    matchLabels:
      app: zero-test
  template:
    metadata:
      labels:
        app: zero-test
    spec:
      containers:
      - name: nginx
        image: nginx
```
Polecenia:
```
kubectl apply -f zero-test.yaml
kubectl get pods -n ns-dev
kubectl describe pod -n ns-dev $(kubectl get pod -n ns-dev -l app=zero-test -o jsonpath='{.items[0].metadata.name}') | grep -A5 "Limits"
```
Wynik:
Pod otrzymał automatycznie limity z LimitRange (requests: 100m/128Mi, limits: 200m/256Mi).
## 5. Sprawdzenie wykorzystania Quota

Polecenia:
```
kubectl describe resourcequota quota-dev -n ns-dev
kubectl describe resourcequota quota-prod -n ns-prod
```
Cel:
Zweryfikowanie, że utworzone Pody wykorzystują przypisane limity i zasoby.
