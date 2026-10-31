### We will Create Secrets Automatically using ESO Which Retrieve The Secrets from AWS Secret Manager Using SecretStore and ExternnalSecret Objects
# check NameSpace
### Create secretstore.yaml

```yaml
apiVersion: external-secrets.io/v1
kind: SecretStore
metadata:
  name: aws-secret-store
  namespace: webapps

spec:
  provider:
    aws:
      service: SecretsManager
      region: eu-west-1
```

### Verify

```bash
kubectl get secretstore -n webapps
```

---

## Create ExternalSecret

It tells ESO exactly which credentials to retrieve and what Kubernetes Secret to create.

### externalsecret.yaml

```yaml
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: bankapp-external-secret
  namespace: webapps

spec:
  refreshInterval: 1h

  secretStoreRef:
    name: aws-secret-store
    kind: SecretStore

  target:
    name: bankapp-db-secret
    creationPolicy: Owner

  data:
    - secretKey: SPRING_DATASOURCE_USERNAME
      remoteRef:
        key: bankapp/mysql
        property: SPRING_DATASOURCE_USERNAME


    - secretKey: SPRING_DATASOURCE_PASSWORD
      remoteRef:
        key: bankapp/mysql
        property: SPRING_DATASOURCE_PASSWORD

    - secretKey: MYSQL_ROOT_PASSWORD
      remoteRef:
        key: bankapp/mysql
        property: MYSQL_ROOT_PASSWORD
```

## Verify the Complete Integration

```bash
kubectl get secretstore -n webapps

kubectl get externalsecret -n webapps

kubectl get secret bankapp-db-secret -n webapps
```


## Connect BankApp Deployment Using envFrom/env

```yaml

spec:
  containers:
    - name: bankapp
      image: adijaiswal/bankapp:v6

      envFrom:
        - secretRef:
            name: bankapp-db-secret
```






# Deploying App with RDS DB
--> Creating 2 user one Root-user ---> admin  & pass --> root12345.... These created while creating DB
--> another user is creating using init container with DB  
--> So provide both user creds in AWS SM

```	
---
# Headless Service for MySQL StatefulSet
apiVersion: v1
kind: Service
metadata:
  name: rds-mysql
  namespace: webapps
spec:
  type: ExternalName
  externalName: rds-mysql.c1qjv0xwz3zv.us-east-1.rds.amazonaws.com
  ports:
    - port: 3306

# BankApp Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: bankapp
  namespace: webapps
spec:
  replicas: 1
  selector:
    matchLabels:
      app: bankapp
  template:
    metadata:
      labels:
        app: bankapp
    spec:
    # Creating a user and initial Db
      initContainers:
        - name: create-db-and-user
          image: mysql:8
          command:
            - sh
            - -c
            - |
              mysql -h rds-mysql -u admin -p"$MYSQL_ROOT_PASSWORD" <<EOF
              CREATE DATABASE IF NOT EXISTS bankappdb;

              CREATE USER IF NOT EXISTS 'bankapp_user'@'%'
              IDENTIFIED BY '${SPRING_DATASOURCE_PASSWORD}';
           
              GRANT ALL PRIVILEGES ON bankappdb.*
              TO 'bankapp_user'@'%';

              EOF
          
      containers:
        - name: bankapp
          image: surajkhanderao/bankapp:v8
          ports:
            - containerPort: 8080
          env:
            - name: SPRING_DATASOURCE_URL
              value: jdbc:mysql://rds-mysql:3306/bankappdb?useSSL=false&serverTimezone=UTC&allowPublicKeyRetrieval=true
            - name: SPRING_DATASOURCE_USERNAME
              valueFrom:
                secretKeyRef:
                  name: bankapp-db-secret
                  key: SPRING_DATASOURCE_USERNAME
              value: root
            - name: SPRING_DATASOURCE_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: bankapp-db-secret
                  key: SPRING_DATASOURCE_PASSWORD
          livenessProbe:
            httpGet:
              path: /login
              port: 8080
            initialDelaySeconds: 60
            timeoutSeconds: 5
            periodSeconds: 10
            failureThreshold: 3
          readinessProbe:
            httpGet:
              path: /login
              port: 8080
            initialDelaySeconds: 60
            timeoutSeconds: 5
            periodSeconds: 10
            failureThreshold: 3
          resources:
            requests:
              memory: "256Mi"
              cpu: "250m"
            limits:
              memory: "512Mi"
              cpu: "500m"

---
# BankApp Service
apiVersion: v1
kind: Service
metadata:
  name: bankapp-service
  namespace: webapps
spec:
  type: LoadBalancer
  selector:
    app: bankapp
  ports:
    - port: 80
      targetPort: 8080


```


