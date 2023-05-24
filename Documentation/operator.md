# Gerrit Operator

1. [Gerrit Operator](#gerrit-operator)
   1. [Build](#build)
   2. [Versioning](#versioning)
   3. [Publish](#publish)
   4. [Tests](#tests)
   5. [Deploy](#deploy)
   6. [CustomResources](#customresources)
      1. [GerritCluster](#gerritcluster)
      2. [Gerrit](#gerrit)
      3. [GitGarbageCollection](#gitgarbagecollection)
      4. [Receiver](#receiver)
   7. [Configuration of Gerrit](#configuration-of-gerrit)

## Build

To build all components of the operator run:

```sh
cd operator
mvn clean install
```

## Versioning

The Gerrit Operator is still in an early state of development. The operator is
thus at the moment not semantically versioned. The CustomResources are as of now
independently versioned, i.e. the `GerritCluster` resource can have a different
version than the `GitGarbageCollection` resource, although they are in the same
group. At the moment, only the current version will be supported by the operator,
i.e. there won't be a migration path. As soon as the API reaches some stability,
this will change.

## Publish

To publish the container image of the Gerrit Operator run:

```sh
cd operator
mvn clean install -P publish
```

## Tests

Executing the E2E tests has a few infrastructure requirements that have to be
provided:

- An (unused) Kubernetes cluster
- The 'default' StorageClass that supports ReadWriteOnce access. It has to be
  possible to provision volumes using this StorageClass.
- A StorageClass that supports ReadWriteMany access. It has to be possible to
  provision volumes using this StorageClass. Such a StorageClass could be provided
  by the [NFS-subdir-provisioner chart](https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner).
- An [Nginx Ingress Controller](https://github.com/kubernetes/ingress-nginx)
- An installation of [OpenLDAP](../supplements/test-cluster/ldap/openldap.yaml)
  with at least one user.
- Istio installed with the [profile](../istio/gerrit.profile.yaml) provided by
  this project
- A secret containing valid certificates for the given hostnames. For istio this
  secret has to be named `tls-secret` and be present in the `istio-system` namespace.
  For the Ingress controller, the secret has to be either set as the default
  secret to be used or somehow automatically be provided in the namespaces created
  by the tests and named `tls-secret`, e.g. by using Gardener to manage DNS and
  certificates.

A sample setup for components required in the cluster is provided under
`$REPO_ROOT/supplements/test-cluster`. Some configuration has to be done manually
(marked by `#TODO`) and the `deploy.sh`-script can be used to install/update all
components.

In addition, some properties have to be set to configure the tests:

- `rwmStorageClass`: Name of the StorageClass providing RWM-access (default:nfs-client)
- `registry`: Registry to pull container images from
- `RegistryOrg`: Organization of the container images
- `tag`: Container tag
- `registryUser`: User for the container registry
- `registryPwd`: Password for the container registry
- `ingressDomain`: Domain to be used for the ingress
- `istioDomain`: Domain to be used for istio
- `ldapAdminPwd`: Admin password for LDAP server
- `gerritUser`: Username of a user in LDAP
- `gerritPwd`: The password of `gerritUser`

The properties should be set in the `test.properties` file. Alternatively, a
path of a properties file can be configured by using the
`-Dproperties=<path to properties file>`-option.

To run all E2E tests, use:

```sh
cd operator
mvn clean install -P integration-test -Dproperties=<path to properties file>
```

Note, that running the E2E tests will also involve pushing the container image
to the repository configured in the properties file.

## Deploy

First all CustomResourceDefinitions have to be deployed:

```sh
kubectl apply -f operator/target/classes/META-INF/fabric8/*-v1.yml
```

Note that these do not include the -v1beta1.yaml files, as those are for old
Kubernetes versions.

The operator requires a Java Keystore with a keypair inside to allow TLS
verification for Kubernetes Admission Webhooks. To create a keystore and
encode it with base64, run:

```sh
keytool \
  -genkeypair \
  -alias operator \
  -keystore keystore \
  -keyalg RSA \
  -keysize 2048 \
  -validity 3650
cat keystore | base64 -b 0
```

Add the result to the Secret in `k8s/operator.yaml` (see comments in the file)
and also add the base64-encoded password for the keystore to the secret.

Then the operator and associated RBAC rules can be deployed:

```sh
kubectl apply -f operator/k8s/operator.yaml
```

`k8s/operator.yaml` contains a basic deployment of the operator. Resources,
docker image name etc. might have to be adapted.

## CustomResources

The operator manages several CustomResources that are described in more detail
below.

The API reference for all CustomResources can be found [here](operator-api-reference.md).

### GerritCluster

The GerritCluster custom resource installs one or multiple Gerrit instances. The
operator takes over managing the state of all Gerrit instances within the cluster
and ensures that the state stays in sync. To this end it manages additional
resources that are shared between Gerrit instances or are required to synchronize
the state between Gerrit instances. These additional resources include:

- storage
- network / service mesh

Installing Gerrit with the GerritCluster resource is highly recommended over using
the [Gerrit](#gerrit) CustomResource directly, even if only a single deployment is
installed, since this reduces the requirements that have to be managed manually.
The same holds true for the [Receiver](#receiver) CustomResource, which without
a Gerrit instance using the same site provides little value.

### Gerrit

The Gerrit CustomResource deploys a Gerrit, which can run in multiple modes.

The Gerrit-CustomResource is mainly meant to be used by the GerritCluster-reconciler
to install Gerrit-instances managed by a GerritCluster. Gerrit-CustomResources
can however also be applied separately. Note, that the Gerrit operator will then
not create any storage resources or setup any network resources in addition to
the service.

### GitGarbageCollection

The GitGarbageCollection-CustomResource is used by the operator to set up CronJobs
that regularly run Git garbage collection on the git repositories that are served
by a GerritCluster.

A GitGarbageCollection can either handle all repositories, if no specific repository
is configured or a selected set of repositories. Multiple GitGarbageCollections
can exist as part of the same GerritCluster, but no two GitGarbageCollections
can work on the same project. This is prevented in three ways:

- ValidationWebhooks will prohibit the creation of a second GitGarbageCollection
  that does not specify projects, i.e. that would work on all projects.
- Projects for which a GitGarbageCollections that specifically selects it exists
  will be excluded from the GitGarbageCollection that works on all projects, if
  it exists.
- ValidationWebhooks will prohibit the creation of a GitGarbageCollection that
  specifies a project that was already specified by another GitGarbageCollection.

### Receiver

**NOTE:** A Receiver should never be installed for a GerritCluster that is already
managing a primary Gerrit to avoid conflicts when writing into repositories.

The Receiver-CustomResource installs a Deployment running Apache with a git-http-
backend that is meant to receive pushes performed by Gerrit's replication plugin.
It is meant to be installed into a GerritCluster that does not include a primary
Gerrit, but only Gerrit Replicas.

The Receiver-CustomResource is mainly meant to be used by the GerritCluster-reconciler
to install a Receiver-instance managed by a GerritCluster. Receiver-CustomResources
can however also be applied separately. Note, that the Gerrit operator will then
not create any storage resources or setup any network resources in addition to
the service.

## Configuration of Gerrit

The operator takes care of all configuration in Gerrit that depends on the
infrastructure, i.e. Kubernetes and the GerritCluster. This avoids duplicated
configuration and misconfiguration.

This means that some options in the gerrit.config are not allowed to be changed.
If these values are set and are not matching the expected value, a ValidationWebhook
will reject the resource creation/update. Thus, it is best to not set these values
at all. To see which values the operator assigned check the ConfigMap created by
the operator for the respective Gerrit.

These options are:

- `cache.directory`

    This should stay in the volume mounted to contain the Gerrit site and will
    thus be set to `cache`.

- `container.javaHome`

    This has to be set to `/usr/lib/jvm/java-11-openjdk-amd64`, since this is
    the path of the Java installation in the container.

- `container.javaOptions = -Djavax.net.ssl.trustStore`

    The keystore will be mounted to `/var/gerrit/etc/keystore`.

- `container.replica`

    This has to be set in the Gerrit-CustomResource under `spec.isReplica`.

- `container.user`

    The technical user in the Gerrit container is called `gerrit`.

- `gerrit.basePath`

    The git repositories are mounted to `/var/gerrit/git` in the container.

- `gerrit.canonicalWebUrl`

    The canonical web URL has to be set to the hostname used by the Ingress/Istio.

- `httpd.listenURL`

    This has to be set to `proxy-http://*:8080/` or `proxy-https://*:8080`,
    depending of TLS is enabled in the Ingress or not, otherwise the Jetty
    servlet will run into an endless redirect loop.

- `sshd.advertisedAddress`

    This is only enforced, if Istio is enabled. It can be configured otherwise.

- `sshd.listenAddress`

    Since the container port for SSH is fixed, this will be set automatically.
    If no SSH port is configured in the service, the SSHD is disabled.