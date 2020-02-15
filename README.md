# How to deploy strapi in k8s

## Customize

1. Clone this repo.
2. Open the cloned repo with your editor, and make a 'search and replace' to change all occurrences of `mystrapiapp` to the name of your app.
3. Open the files in the `k8s` directory and search for the comments starting with the word `@custom` and modify the values below them according to your needs.

## Database

**Note:** postgres is used, if you are using a different database software modify as needed.

### Generate a password

```sh
tr -cd '[:alnum:]' < /dev/urandom | fold -w $((20 + RANDOM % 10)) | head -n1
```

### Create a k8s secret to store the password

(change `generated_pass` with the output of the command above)

```sh
kubectl create secret generic strapi-mystrapiapp --from-literal=db_password=generated_pass
```

### Start psql session

```sh
kubectl exec -it your-postgres-pod-name -- psql -U postgres
```

### Create postgres user and database

(change 'generated_pass')

```sql
CREATE USER strapi_mystrapiapp WITH PASSWORD 'generated_pass';

CREATE DATABASE strapi_mystrapiapp OWNER strapi_mystrapiapp;
```

### Strapi database settings

In `config/environments/production/database.json` set `client` to `postgres` and add `"ssl": true`

```json
{
  "defaultConnection": "default",
  "connections": {
    "default": {
      "connector": "bookshelf",
      "settings": {
        "client": "postgres",
        "host": "${process.env.DATABASE_HOST || '127.0.0.1'}",
        "port": "${process.env.DATABASE_PORT || 27017}",
        "database": "${process.env.DATABASE_NAME || 'strapi'}",
        "username": "${process.env.DATABASE_USERNAME || ''}",
        "password": "${process.env.DATABASE_PASSWORD || ''}",
        "ssl": true
      },
      "options": {}
    }
  }
}
```


## Persistent volume claim

```sh
kubectl apply -f k8s/pvc.yaml
```

### Copy/clone the app to the created persistent volume

One way of achieving this is to run a temporary pod with the volume mounted, and use this pod to copy or clone the app into the volume:

```yaml
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: temporary-strapi-pod
spec:
  containers:
  - name: strapi
    image: strapi/base
    volumeMounts:
    - name: strapi-mystrapiapp
      mountPath: /strapi
    command: ["sleep", "infinity"]
  volumes:
  - name: strapi-mystrapiapp
    persistentVolumeClaim:
      claimName: strapi-mystrapiapp
EOF
```

then

```sh
kubectl cp /local/path/to/mystrapiapp temporary-strapi-pod:/strapi
```

Or

```sh
kubectl exec temporary-strapi-pod -- \
git clone https://link/to/app-repo.git /strapi/mystrapiapp && \
yarn --cwd /strapi/mystrapiapp install && \
NODE_ENV=production yarn --cwd /strapi/mystrapiapp build
```

When finished you can delete the temporary pod

```sh
kubectl delete po temporary-strapi-pod
```

## App manifest

```sh
kubectl apply -f k8s/manifest.yaml
```

This will create an ingress, service and a deployment for the app.

## TLS certificate

If you have [cert manager](https://cert-manager.io/) installed, you can generate a tls certificate with:

```sh
kubectl apply -f k8s/tls_certificate.yaml
```
