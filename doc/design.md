
Build packages
Build container images
Build operator

# Introduction

We can start with a prototype operator based on helm scripts that we use with [kubebuilder](https://github.com/kubernetes-sigs/kubebuilder) to convert into an operator.

# Building

We will need to create BuildConfig jobs that

* Build any packages that are not included in UBI.
* Build container images using RPMs

Instructions for building Polaris can be found here:

https://polaris.apache.org/in-dev/unreleased/quickstart/

# Architecture

The core design features deploying polaris-api and polaris-db pods, the former is stateless and the latter is the persistence layer for the Polaris metastore.

To get started, we can create a toolbox pod that can be used to run administrative Polaris CLI commands.

https://polaris.apache.org/in-dev/unreleased/command-line-interface/

Eventually, we'll want to have custom resources for each command to facilitate declarative configuration of an access control model. Ideally, the operator would monitor for the creation of Polaris CRs and in response make API calls against the Polaris management API.

The Polaris Management API follows an OpenAPI specification:

https://github.com/apache/polaris/blob/main/spec/polaris-management-service.yml

The Polaris Management API by default will be `:8181` and the prefix is `/api/management/v1`.

# Custom resource definitions

The purpose of the Polaris operator custom resource definitions is to provide end users with a declarative way of constructing a set of Polaris resources to provide comprehensive access control scheme for a Data Lakehouse. An example construction is illustrated below:
 
Initially we will limit the custom resources to the set that are managed through the Polaris management API, which notably excludes namespaces and tables. This should be acceptable because these can be created through SQL commands using a query engine that is interacting with the catalog. For example, tables can be created a la:

```
sql
CREATE NS foo;
```
```
sql
CREATE TABLE foo.bar;
```

## Polaris

The Polaris Catalog CRD will be used to create a deployment and services and will be a dependency of all other resources because those resources require calls be made against the management service endpoint.

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

## PrincipalRoleGrant

Principal role grants establish a relationship between a principal role and a catalog role.

`/principal-roles/{principalRoleName}/catalog-roles/{catalogName}`
`/principal-roles/{principalRoleName}/catalog-roles/{catalogName}/{catalogRoleName}`

## CatalogRoleGrant

Catalog role grants establish a relationship between a catalog role and a catalog and are organized as an array.

