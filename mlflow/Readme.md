


1. Instalação do MLFlow

Para instalarmos os componentes do MLFlow no kubernetes precisamos executar os seguintes comandos usando o kubectl.

1.1 Criando namespace para o mlflow:

```
> kubectl create namespace mlflow

```
1.2 Criando pod para MLflow Postgres:

```
> cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
 name: mlflow-postgres-config
 namespace: mlflow
 labels:
   app: mlflow-postgres
data:
 POSTGRES_DB: mlflow_db
 POSTGRES_USER: mlflow_user
 POSTGRES_PASSWORD: mlflow_pwd
 PGDATA: /var/lib/postgresql/mlflow/data
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
 name: mlflow-postgres
 namespace: mlflow
 labels:
   app: mlflow-postgres
spec:
 selector:
   matchLabels:
     app: mlflow-postgres
 serviceName: "mlflow-postgres-service"
 replicas: 1
 template:
   metadata:
     labels:
       app: mlflow-postgres
   spec:
     containers:
     - name: mlflow-postgres
       image: postgres:11
       ports:
       - containerPort: 5432
         protocol: TCP
       envFrom:
       - configMapRef:
           name: mlflow-postgres-config
       resources:
         requests:
           memory: "1Gi"
           cpu: "500m"
       volumeMounts:
       - name: mlflow-pvc
         mountPath: /var/lib/postgresql/mlflow
 volumeClaimTemplates:
 - metadata:
     name: mlflow-pvc
   spec:
     accessModes: [ "ReadWriteOnce" ]
     resources:
       requests:
         storage: 100Mi
---
apiVersion: v1
kind: Service
metadata:
 name: mlflow-postgres-service
 namespace: mlflow
 labels:
   svc: mlflow-postgres-service
spec:
 type: NodePort
 ports:
 - port: 5432
   targetPort: 5432
   protocol: TCP
 selector:
   app: mlflow-postgres
EOF
```

1.3 Criando pod para o Minio (Artifact Storage) do MLFlow:

```
> cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
 name: mlflow-minio
 namespace: mlflow
spec:
 selector:
   matchLabels:
     app: mlflow-minio
 template:
   metadata:
     labels:
       app: mlflow-minio
   spec:
     volumes:
     - name: mlflow-pvc
       persistentVolumeClaim:
         claimName: mlflow-pvc
     containers:
     - name: mlflow-minio
       image: minio/minio:latest
       args:
       - server
       - /data
       volumeMounts:
       - name: mlflow-pvc
         mountPath: '/data'
       env:
       - name: MINIO_ACCESS_KEY
         value: "minio"
       - name: MINIO_SECRET_KEY
         value: "minio123"
       ports:
       - containerPort: 9000
---
apiVersion: v1
kind: Service
metadata:
 name: mlflow-minio-service
 namespace: mlflow
spec:
 type: NodePort
 ports:
 - port: 9000
   targetPort: 9000
   protocol: TCP
 selector:
   app: mlflow-minio
---
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
 name: mlflow-minio-ingress
 namespace: mlflow
 annotations:
   kubernetes.io/ingress.class: nginx
   nginx.ingress.kubernetes.il/add-base-url: "true"
   nginx.ingress.kubernetes.io/ssl-redirect: "false"
spec:
 rules:
 - host: mlflow-minio.local
   http:
     paths:
       - backend:
           serviceName: mlflow-minio-service
           servicePort: 9000
         path: /
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
 name: mlflow-pvc
 namespace: mlflow
spec:
 accessModes:
   - ReadWriteMany
 resources:
   requests:
     storage: 100Mi
EOF

```

1.4 Criando pod para o MLFLow Server:

```
> cat <<EOF | kubectl apply -f -
# Creating MLflow deployment
apiVersion: apps/v1
kind: Deployment
metadata:
 name: mlflow-deployment
 namespace: mlflow
spec:
 replicas: 1
 selector:
   matchLabels:
     app: mlflow-deployment
 template:
   metadata:
     labels:
       app: mlflow-deployment
   spec:
     containers:
     - name: mlflow-deployment
       image: dkhundley/mlflow-server:1.0.3
       imagePullPolicy: Always
       args:
       - --host=0.0.0.0
       - --port=5000
       - --backend-store-uri=postgresql://mlflow_user:mlflow_pwd@mlflow-postgres-service.mlflow:5432/mlflow_db
       - --default-artifact-root=s3://mlflow/
       - --workers=2
       env:
       - name: MLFLOW_S3_ENDPOINT_URL
         value: http://mlflow-minio-service.mlflow:9000
       - name: AWS_ACCESS_KEY_ID
         value: "minio"
       - name: AWS_SECRET_ACCESS_KEY
         value: "minio123"
       ports:
       - name: http
         containerPort: 5000
         protocol: TCP
---
apiVersion: v1
kind: Service
metadata:
 name: mlflow-service
 namespace: mlflow
spec:
 type: NodePort
 ports:
   - port: 5000
     targetPort: 5000
     protocol: TCP
     name: http
 selector:
   app: mlflow-deployment
---
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
 name: mlflow-ingress
 namespace: mlflow
 annotations:
   kubernetes.io/ingress.class: nginx
   nginx.ingress.kubernetes.il/add-base-url: "true"
spec:
 rules:
 - host: mlflow-server.local
   http:
     paths:
       - backend:
           serviceName: mlflow-service
           servicePort: 5000
         path: /
EOF

```

1.5 Acessando as UI do MLFlow e Minio

Para acessar as respectivas UI, é necessário criar uma entrada em seu /etc/hosts conforme exemplo abaixo, substituindo o IP pelo IP de seu ambiente:

```
10.190.24.157 mlflow-server.local
10.190.24.157 mlflow-minio.local
```

Para acessa-la, abra em seu browser as urls http://mlflow-server.local  e http://mlflow-minio.local

Outra forma de expor é executando o comando kubectl port-forward, conforme exemplo abaixo:

```
> kubectl port-forward -n mlflow  service/mlflow-minio-service 8080:9000 --address 0.0.0.0 &!

> kubectl port-forward -n mlflow  service/mlflow-service 8443:5000 --address 0.0.0.0 &!

```

1.5 Criando um bucket para ser root do Artifact Storage 

Acesse http://mlflow-minio.local , se autentique e crie um bucket com o nome *mlflow* .


1.6 Criando uma secret no kubernetes para não expor credenciais

```
> kubectl create secret generic aws-secret --from-literal=AWS_ACCESS_KEY_ID=minio --from-literal=AWS_SECRET_ACCESS_KEY=minio123 -n <user_namespace>
```