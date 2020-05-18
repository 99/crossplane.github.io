---
title: Publish Infrastructure
toc: true
weight: 4
indent: true
---

# Publish Infrastructure

Provisioning infrastructure using the Kubernetes API is a powerful capability,
but combining primitive infrastructure resources into a single unit and
publishing them to be provisioned by developers and consumed by applications
greatly enhances this functionality.

As mentioned in the [last section], CRDs that represent infrastructure resources
on a provider are installed at the *cluster scope*. However, applications are
typically provisioned at the *namespace scope* using Kubernetes primitives such
as `Pod` or `Deployment`. To make infrastructure resources available to be
provisioned at the namespace scope, they can be *published*. This consists of
creating the following resources:

- `InfrastructureDefinition`: defines a new kind of composite resource
- `InfrastructurePublication`: makes an `InfrastructureDefinition` available at
  the namespace scope
- `Composition`: defines one or more resources and their configuration

In addition to making provisioning available at the namespace scope,
[composition] also allows for multiple types of managed resources to satisfy the
same namespace-scoped resource. In the examples below, we will define and
publish a new `MySQLInstance` resource that only takes a single `storageGB`
parameter, and specifies that it will create a connection `Secret` with keys for
`username`, `password`, and `endpoint`. We will then create a `Composition` for
each provider that can satisfy and be parameterized by a `MySQLInstance`. Let's
get started!

> Note: Crossplane must be granted RBAC permissions to managed new
> infrastructure types that we define. This is covered in greater detail in the
> [composition] section, but you can run the following command to grant all
> necessary RBAC permissions for this quick start guide: `kubectl apply -f
> https://raw.githubusercontent.com/crossplane/crossplane/master/docs/snippets/quick-start-clusterrole.yaml`

## Create InfrastructureDefinition

The next step is defining an `InfrastructureDefinition` that declares the schema
for a `MySQLInstance`. You will notice that this looks very similar to a CRD,
and after the `InfrastructureDefinition` is created, we will in fact have a
`MySQLInstance` CRD present in our cluster.

```yaml
apiVersion: apiextensions.crossplane.io/v1alpha1
kind: InfrastructureDefinition
metadata:
  name: mysqlinstances.database.example.org
spec:
  connectionSecretKeys:
    - username
    - password
    - endpoint
  crdSpecTemplate:
    group: database.example.org
    version: v1alpha1
    names:
      kind: MySQLInstance
      listKind: MySQLInstanceList
      plural: mysqlinstances
      singular: mysqlinstance
    validation:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            properties:
              parameters:
                type: object
                properties:
                  storageGB:
                    type: integer
                required:
                  - storageGB
            required:
              - parameters
```

You are now able to create instances of kind `MySQLInstance` at the cluster
scope.

## Create InfrastructurePublication

The `InfrastructureDefinition` will make it possible to create `MySQLInstance`
objects in our Kubernetes cluster at the cluster scope, but we want to make them
available at the namespace scope as well. This is done by defining an
`InfrastructurePublication` that references the new `MySQLInstance` type.

```yaml
apiVersion: apiextensions.crossplane.io/v1alpha1
kind: InfrastructurePublication
metadata:
  name: mysqlinstances.database.example.org
spec:
  infrastructureDefinitionRef:
    name: mysqlinstances.database.example.org
```

This will create a new CRD named `MySQLInstanceRequirement`, which is the
namespace-scoped variant of the `MySQLInstance` CRD. You are now able to create
instances of kind `MySQLInstanceRequirement` at the namespace scope.

## Create Compositions

Now it is time to define the resources that represent the primitive
infrastructure units that actually get provisioned. For each provider we will
define a `Composition` that satisfies the requirements of the `MySQLInstance`
`InfrastructureDefinition`. In this case, each will result in the provisioning
of a public MySQL instance on the provider.


<ul class="nav nav-tabs">
<li class="active"><a href="#aws-tab-1" data-toggle="tab">AWS</a></li>
<li><a href="#gcp-tab-1" data-toggle="tab">GCP</a></li>
<li><a href="#azure-tab-1" data-toggle="tab">Azure</a></li>
<li><a href="#alibaba-tab-1" data-toggle="tab">Alibaba</a></li>
</ul>
<br>
<div class="tab-content">
<div class="tab-pane fade in active" id="aws-tab-1" markdown="1">

```yaml
apiVersion: apiextensions.crossplane.io/v1alpha1
kind: Composition
metadata:
  name: mysqlinstances.aws.database.example.org
  labels:
    provider: aws
    guide: quickstart
spec:
  writeConnectionSecretsToNamespace: crossplane-system
  reclaimPolicy: Delete
  from:
    apiVersion: database.example.org/v1alpha1
    kind: MySQLInstance
  to:
    - base:
        apiVersion: database.aws.crossplane.io/v1beta1
        kind: RDSInstance
        spec:
          forProvider:
            dbInstanceClass: db.t2.small
            masterUsername: masteruser
            engine: mysql
            skipFinalSnapshotBeforeDeletion: true
            publiclyAccessible: true
          writeConnectionSecretToRef:
            namespace: crossplane-system
          providerRef:
            name: aws-provider
          reclaimPolicy: Delete
      patches:
        - fromFieldPath: "metadata.uid"
          toFieldPath: "spec.writeConnectionSecretToRef.name"
          transforms:
            - type: string
              string:
                fmt: "%s-mysql"
        - fromFieldPath: "spec.parameters.storageGB"
          toFieldPath: "spec.forProvider.allocatedStorage"
      connectionDetails:
        - fromConnectionSecretKey: username
        - fromConnectionSecretKey: password
        - fromConnectionSecretKey: endpoint
```

</div>
<div class="tab-pane fade" id="gcp-tab-1" markdown="1">

```yaml
apiVersion: apiextensions.crossplane.io/v1alpha1
kind: Composition
metadata:
  name: mysqlinstances.gcp.database.example.org
  labels:
    provider: gcp
    guide: quickstart
spec:
  writeConnectionSecretsToNamespace: crossplane-system
  reclaimPolicy: Delete
  from:
    apiVersion: database.example.org/v1alpha1
    kind: MySQLInstance
  to:
    - base:
        apiVersion: database.gcp.crossplane.io/v1beta1
        kind: CloudSQLInstance
        spec:
          forProvider:
            databaseVersion: MYSQL_5_7
            region: us-central1
            ipConfiguration:
              ipv4Enabled: true
              authorizedNetworks:
                - value: "0.0.0.0/0"
            settings:
              tier: db-n1-standard-1
              dataDiskType: PD_SSD
          writeConnectionSecretToRef:
            namespace: crossplane-system
          providerRef:
            name: gcp-provider
          reclaimPolicy: Delete
      patches:
        - fromFieldPath: "metadata.uid"
          toFieldPath: "spec.writeConnectionSecretToRef.name"
          transforms:
            - type: string
              string:
                fmt: "%s-mysql"
        - fromFieldPath: "spec.parameters.storageGB"
          toFieldPath: "spec.forProvider.settings.dataDiskSizeGb"
      connectionDetails:
        - fromConnectionSecretKey: username
        - fromConnectionSecretKey: password
        - fromConnectionSecretKey: endpoint
```

</div>
<div class="tab-pane fade" id="azure-tab-1" markdown="1">

> Note: the `Composition` for Azure also includes a `ResourceGroup` and
> `MySQLServerFirewallRule` that are required to provision a publicly available
> MySQL instance on Azure. Composition enables scenarios such as this, as well
> as far more complex ones. See the [composition] documentation for more
> information.

```yaml
apiVersion: apiextensions.crossplane.io/v1alpha1
kind: Composition
metadata:
  name: mysqlinstances.azure.database.example.org
  labels:
    provider: azure
    guide: quickstart
spec:
  writeConnectionSecretsToNamespace: crossplane-system
  reclaimPolicy: Delete
  from:
    apiVersion: database.example.org/v1alpha1
    kind: MySQLInstance
  to:
    - base:
        apiVersion: azure.crossplane.io/v1alpha3
        kind: ResourceGroup
        spec:
          location: West US 2
          reclaimPolicy: Delete
          providerRef:
            name: azure-provider
    - base:
        apiVersion: database.azure.crossplane.io/v1beta1
        kind: MySQLServer
        spec:
          forProvider:
            administratorLogin: myadmin
            resourceGroupNameSelector:
              matchControllerRef: true
            location: West US 2
            sslEnforcement: Disabled
            version: "5.7"
            sku:
              tier: GeneralPurpose
              capacity: 2
              family: Gen5
            storageProfile:
          writeConnectionSecretToRef:
            namespace: crossplane-system
          providerRef:
            name: azure-provider
          reclaimPolicy: Delete
      patches:
        - fromFieldPath: "metadata.uid"
          toFieldPath: "spec.writeConnectionSecretToRef.name"
          transforms:
            - type: string
              string:
                fmt: "%s-mysql"
        - fromFieldPath: "spec.parameters.storageGB"
          toFieldPath: "spec.forProvider.storageProfile.storageMB"
          transforms:
            - type: math
              math:
                multiply: 1024
      connectionDetails:
        - fromConnectionSecretKey: username
        - fromConnectionSecretKey: password
        - fromConnectionSecretKey: endpoint
    - base:
        apiVersion: database.azure.crossplane.io/v1alpha3
        kind: MySQLServerFirewallRule
        spec:
          forProvider:
            serverNameSelector:
              matchControllerRef: true
            resourceGroupNameSelector:
              matchControllerRef: true
            properties:
              startIpAddress: 0.0.0.0
              endIpAddress: 255.255.255.254
          reclaimPolicy: Delete
          providerRef:
            name: azure-provider
```

</div>
<div class="tab-pane fade" id="alibaba-tab-1" markdown="1">

```yaml
apiVersion: apiextensions.crossplane.io/v1alpha1
kind: Composition
metadata:
  name: mysqlinstances.alibaba.database.example.org
  labels:
    provider: alibaba
    guide: quickstart
spec:
  writeConnectionSecretsToNamespace: crossplane-system
  reclaimPolicy: Delete
  from:
    apiVersion: database.example.org/v1alpha1
    kind: MySQLInstance
  to:
    - base:
        apiVersion: database.alibaba.crossplane.io/v1alpha1
        kind: RDSInstance
        spec:
          forProvider:
            engine: mysql
            engineVersion: "5.7"
            dbInstanceClass: rds.mysql.s1.small
            dbInstanceStorageInGB: 20
            securityIPList: "0.0.0.0/0"
            masterUsername: "myuser"
          writeConnectionSecretToRef:
            namespace: crossplane-system
          providerRef:
            name: alibaba-provider
          reclaimPolicy: Delete
      patches:
        - fromFieldPath: "metadata.uid"
          toFieldPath: "spec.writeConnectionSecretToRef.name"
          transforms:
            - type: string
              string:
                fmt: "%s-mysql"
        - fromFieldPath: "spec.parameters.storageGB"
          toFieldPath: "spec.forProvider.dbInstanceStorageInGB"
      connectionDetails:
        - fromConnectionSecretKey: username
        - fromConnectionSecretKey: password
        - fromConnectionSecretKey: endpoint
```

</div>
</div>

## Create Requirement

Now that we have defined our new type of infrastructure (`MySQLInstance`) and
created at least one composition that can satisfy it, we can create a
`MySQLInstanceRequirement` in the namespace of our choosing. In each
`Composition` we defined we added a `provider: <name-of-provider>` label. In the
`MySQLInstanceRequirement` below we can use the `compositionSelector` to match
our `Composition` of choice.

<ul class="nav nav-tabs">
<li class="active"><a href="#aws-tab-2" data-toggle="tab">AWS</a></li>
<li><a href="#gcp-tab-2" data-toggle="tab">GCP</a></li>
<li><a href="#azure-tab-2" data-toggle="tab">Azure</a></li>
<li><a href="#alibaba-tab-2" data-toggle="tab">Alibaba</a></li>
</ul>
<br>
<div class="tab-content">
<div class="tab-pane fade in active" id="aws-tab-2" markdown="1">

```yaml
apiVersion: database.example.org/v1alpha1
kind: MySQLInstanceRequirement
metadata:
  name: my-db
  namespace: default
spec:
  parameters:
    storageGB: 20
  compositionSelector:
    matchLabels:
      provider: aws
  writeConnectionSecretToRef:
    name: db-conn
```

</div>
<div class="tab-pane fade" id="gcp-tab-2" markdown="1">

```yaml
apiVersion: database.example.org/v1alpha1
kind: MySQLInstanceRequirement
metadata:
  name: my-db
  namespace: default
spec:
  parameters:
    storageGB: 20
  compositionSelector:
    matchLabels:
      provider: gcp
  writeConnectionSecretToRef:
    name: db-conn
```

</div>
<div class="tab-pane fade" id="azure-tab-2" markdown="1">

```yaml
apiVersion: database.example.org/v1alpha1
kind: MySQLInstanceRequirement
metadata:
  name: my-db
  namespace: default
spec:
  parameters:
    storageGB: 20
  compositionSelector:
    matchLabels:
      provider: azure
  writeConnectionSecretToRef:
    name: db-conn
```

</div>
<div class="tab-pane fade" id="alibaba-tab-2" markdown="1">

```yaml
apiVersion: database.example.org/v1alpha1
kind: MySQLInstanceRequirement
metadata:
  name: my-db
  namespace: default
spec:
  parameters:
    storageGB: 20
  compositionSelector:
    matchLabels:
      provider: alibaba
  writeConnectionSecretToRef:
    name: db-conn
```

</div>
</div>

After creating the `MySQLInstanceRequirement` Crossplane will provision a
database instance on your provider of choice. Once provisioning is complete, you
should see `READY: True` in the output when you run:
```
kubectl get mysqlinstancerequirement.database.example.org my-db
```

> Note: while waiting for the `MySQLInstanceRequirement` to become ready, you
> may want to look at other resources in your cluser. The following commands
> will allow you to view groups of Crossplane resources:
> - `kubectl get managed`: get all resources that represent a unit of external
>   infrastructure
> - `kubectl get <name-of-provider>`: get all resources related to `<provider>`
> - `kubectl get crossplane`: get all resources related to Crossplane

You should also see a `Secret` in the `default` namespace named `db-conn` that
contains fields for `username`, `password`, and `endpoint`:
```
kubectl get secrets db-conn
```

## Consume Infrastructure

Because connection secrets are written as a Kubernetes `Secret` they can easily
be consumed by Kubernetes primitives. The most basic building block in
Kubernetes is the `Pod`. Let's define a `Pod` that will show that we are able to
connect to our newly provisioned database.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: see-db
  namespace: default
spec:
  containers:
  - name: see-db
    image: mysql:5.7
    command: ['mysql']
    args: ['-u', '$(MYSQL_USER)', '-e', 'SELECT CURRENT_USER();']
    env:
    - name: MYSQL_HOST
      valueFrom:
        secretKeyRef:
          name: db-conn
          key: endpoint
    - name: MYSQL_USER
      valueFrom:
        secretKeyRef:
          name: db-conn
          key: username
    - name: MYSQL_PWD
      valueFrom:
        secretKeyRef:
          name: db-conn
          key: password
```

This `Pod` simply connects to a PostgreSQL database and prints its name, so you
should see the following output (or similar) after creating it if you run
`kubectl logs see-db`:
```
CURRENT_USER()
myadmin@%
```

## Clean Up

To clean up the infrastructure that was provisioned, you can delete the
`MySQLInstanceRequirement`:
```
kubectl delete mysqlinstancerequirement.database.example.org my-db
```

To clean up the `Pod`, run:
```
kubectl delete pod see-db
```

## Next Steps

Now you have seen how to provision and publish more complex infrastructure
setups. In the [next section] you will learn how to consume infrastructure
alongside your [OAM] application manifests.


<!-- Named Links -->

[last section]: provision-infrastructure.yaml
[composition]: ../composition.md
[next section]: consume-infrastructure.md
[OAM]: https://oam.dev/
