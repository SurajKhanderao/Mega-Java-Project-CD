
   ### We will Create Secrets Automatically using ESO(External Secret Oprator) 
   ### ESO Retrieve The Secrets from AWS Secret Manager Using SecretStore and ExternnalSecret Objects



### Create secretstore.yaml ,(Note: Check NameSpace before creating)
```
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



