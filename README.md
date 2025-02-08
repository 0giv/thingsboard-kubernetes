# Installation Guide for ThingsBoard


## Installation Steps


### Step 0: Prerequisites

You need a certificate for mqtt:


- In mqtt-envars.yml you need to replace the MQTT_SSL_PEM_CERT variable with your domain crt file.

- In mqtt-envars.yml you need to replace the MQTT_SSL_PEM_KEY variable with your domain key file.


- In mqtt-envars.yml you need to replace the MQTT_SSL_PEM_KEY_PASSWORD variable with your domain key password if you have, if you dont just leave it "".


### Step 1: Create Namespace and Set default to thingsboard
```bash
kubectl create namespace thingsboard

kubectl config set-context $(kubectl config current-context) --namespace=thingsboard
```
### Step 2: Deploy Third-party Dependencies

```bash
kubectl apply -f storage.yml

kubectl apply -f ./configmap/kafka-config.yml && helm upgrade --install tb-kafka oci://registry-1.docker.io/bitnamicharts/kafka -f ./values/kafka-values.yaml --namespace thingsboard --create-namespace

helm upgrade --install tb-redis oci://registry-1.docker.io/bitnamicharts/redis -f ./values/redis-values.yaml --namespace tb-redis --create-namespace
```

- Wait until pods are running successfully.

### Step 3: Install PostgreSQL
```bash
helm upgrade --install tb-postgres oci://registry-1.docker.io/bitnamicharts/postgresql -f ./values/postgres-values.yaml --namespace tb-postgres --create-namespace

kubectl config set-context $(kubectl config current-context) --namespace=thingsboard
```

- Wait until the PostgreSQL pod is running successfully.

### Step 4: Configure ThingsBoard Components

#### 1 Database Configuration
```bash
kubectl apply -f ./configmap/tb-node-db-configmap.yml
```
#### 2 Cache Configuration
```bash
kubectl apply -f ./configmap/tb-cache-configmap.yml
```
#### 3 Node Configuration
```bash
kubectl apply -f ./configmap/tb-node-configmap.yml
```

### Step 5: Set Up the Database

To set up the database, execute the following commands:
- Without demo data: 
```bash
kubectl apply -f database-setup.yml && kubectl wait --for=condition=Ready pod/tb-db-setup --timeout=120s && kubectl exec tb-db-setup -- sh -c "export INSTALL_TB=true; export LOAD_DEMO='false'; start-tb-node.sh; touch /tmp/install-finished;"
```
- With demo data:

```bash
kubectl apply -f database-setup.yml && kubectl wait --for=condition=Ready pod/tb-db-setup --timeout=120s && kubectl exec tb-db-setup -- sh -c "export INSTALL_TB=true; export LOAD_DEMO='true'; start-tb-node.sh; touch /tmp/install-finished;"
```
- If you get any timeout error ust execute the same command when the pod is running

After the database setup is complete, delete the tb-db-setup pod:

```bash
kubectl delete pod tb-db-setup
```
### Step 6: Deploy Ingress If Needed

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

helm upgrade --install ingress-nginx ingress-nginx/ingress-nginx -f ./values/ingress.yaml --namespace thingsboard
```

### Step 7: Deploy ThingsBoard Resources

Finally, deploy ThingsBoard components by applying the following files in sequence:


#### 0 MQTT Secret:
```bash
kubectl apply -f mqtt-envars.yml
```
#### 1 Transport ConfigMap:
```bash
kubectl apply -f ./configmap/tb-transport-configmap.yml
```
#### 2 Main ThingsBoard Deployment:
```bash
kubectl apply -f thingsboard.yml
```
#### 3 Node Deployment:
```bash
kubectl apply -f tb-node.yml
```
#### 4 Routes:
```bash
kubectl apply -f routes.yml
```

Wait for 15 minutes to ensure resources are initialized and you're ready to go