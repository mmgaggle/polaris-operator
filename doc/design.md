
Build packages
Build container images
Build operator

# Introduction

The Polaris operator should be created using the Operator SDK. 

# Custom resource definitions

The purpose of the Polaris operator custom resource definitions is to provide end users with a declarative way of constructing a set of Polaris resources to provide comprehensive access control scheme for a Data Lakehouse. An example construction is illustrated below:
 
 
Initially we will limit the custom resource to the set that are managed through the Polaris management API, which notably excludes namespaces and tables. This should be acceptable because these can be created through SQL commands using a query engine that is interacting with the catalog.
The prefix for the management API is /api/management/v1, and by default Polaris will listen on port 8181.

## Polaris

The Polaris Catalog CRD will be used to create a deployment and services and will be a dependency of all other resources because those resources require calls be made against the management service endpoint.

### Deployments

Polaris API deployment

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: polaris-api-server
  labels:
    app: polaris
spec:
  replicas: 2
  selector:
    matchLabels:
      app: polaris
  template:
    metadata:
      labels:
        app: polaris
    spec:
      containers:
      - name: polaris
        image: polaris:tag
        ports:
        - containerPort: 8181
       livenessProbe:
         httpGet:
           path: /q/health
           port: 8182
         initialDelaySeconds: 3
         periodSeconds: 5
       readinessProbe:
         httpGet:
           path: /q/health
           port: 8182
         initialDelaySeconds: 3
         periodSeconds: 5
         
```

Polaris API service

```
# ClusterIP Service (for internal access)
apiVersion: v1
kind: Service
metadata:
  name: polaris-api-server
spec:
  type: ClusterIP
  selector:
    app: polaris-api-server
  ports:
    - port: 8181
      targetPort: 8181
```

Polaris DB secret

```
apiVersion: v1
kind: Secret
metadata:
  name: postgres-credentials
type: Opaque
data:
  password: <insert base64 encoded secret>
```

Polaris DB deployment

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: polaris-db
spec:
  replicas: 1
  selector:
    matchLabels:
      app: poalris-db
  template:
    metadata:
      labels:
        app: polaris-db
    spec:
      containers:
        - name: postgres
          image: postgres:14
          env:
            - name: POSTGRES_PASSWORD
              valueFrom:
                name: polaris-db-credentials
                key: password
            - name: POSTGRES_DB
              value: polaris
            - name: PGDATA
              value: /var/lib/postgresql/data/pgdata
          ports:
            - containerPort: 5432
          volumeMounts:
            - name: polaris-db-storage
              mountPath: /var/lib/postgresql/data
      volumes:
        - name: polaris-db-storage
          persistentVolumeClaim:
            claimName: polaris-db-pvc
```

Polaris DB service

```
apiVersion: v1
kind: Service
metadata:
  name: polaris-db
spec:
  type: ClusterIP
  selector:
    app: polaris-db
  ports:
    - port: 5432
      targetPort: 5432
```

Additional Polaris configuration

Should include provisions for federating with a OIDC provider to supply principals. This may be through a properties file that configures the Spring OAuth2ResrouceServer (application.properties/yaml). This might be projected into the Polaris container with information sourced from a config map.

```
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: https://your-oidc-provider/auth/realms/your-realm
          jwk-set-uri: https://your-oidc-provider/auth/realms/your-realm/protocol/openid-connect/certs
```

Should configure EclipseLink for metastore persistence with the PostgreSQL container

https://polaris.io/metastores.md

## Principal

The operator can create a principal by sending an OpenAPI PUT request to

`/principals`

The operator can read, update, or delete a principal by sending a OpenAPI PUT request to

`/principals/{principalName}`

## Principal Role

The operator can create a principal role by sending an OpenAPI PUT request to

`/principal-roles`

The operator can read, update, or delete a principal role by sending a OpenAPI PUT request to

`/principal-roles/{principalRoleName}`

## PrincipalRoleGrant

Principal role grants establish a relationship between a principal role and a catalog role.

`/principal-roles/{principalRoleName}/catalog-roles/{catalogName}`
`/principal-roles/{principalRoleName}/catalog-roles/{catalogName}/{catalogRoleName}`

## Catalog

The operator can create a catalog by sending an OpenAPI PUT request to

`/catalogs`

The operator can read, update, or delete a catalog by sending a OpenAPI PUT request to

`/catalogs/{catalogName}`

An example cURL request structure is provided below

```
curl -s -i -X POST \
           -H "Authorization: Bearer ${POLARIS_BEARER_TOKEN:-principal:root;realm:default-realm}" \
           -H 'Accept: application/json' \
           -H 'Content-Type: application/json' \
           http://${POLARIS_HOST:-localhost}:8181/api/management/v1/catalogs \
           -d "{
                \"name\": \"warehouse\",
                \"id\": 100,
                \"type\": \"INTERNAL\",
                \"readOnly\": false,
                \"properties\": {
                  \"default-base-location\": \"${S3_LOCATION}\"
                 },
                \"storageConfigInfo\": {
                  \"storageType\": \"S3_COMPATIBLE\",
                  \"allowedLocations\": [\"${S3_LOCATION}/\"],
                  \"s3.region\": \"default\",
                  \"s3.endpoint\": \"https://s3.example.com\",
                  \"s3.credentials.catalog.accessKeyId\": \"S3_ACCESS_KEY\",
                  \"s3.credentials.catalog.secretAccessKey\": \"S3_SECRET_KEY\",
                  \"s3.roleArn\": \"arn:aws:iam::RGW25531238860968914:role/polaris/catalog/client\"
                }
              }"
````

## Catalog Role

The operator can create a catalog role by sending an OpenAPI PUT request to

`/catalogs/{catalogName}/catalog-roles`

The operator can read, update, or delete a catalog role by sending a OpenAPI PUT request to

`/catalogs/{catalogName}/catalog-roles/{catalogRoleName}`

## CatalogRoleGrant

Catalog role grants establish a relationship between a catalog role and a catalog and are organized as an array.

