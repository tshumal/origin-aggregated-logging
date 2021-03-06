# About the Logging Components

The aggregated logging subsystem consists of multiple components commonly
abbreviated as the "ELK" stack (though modified here to be the "EFK"
stack).

## ElasticSearch

ElasticSearch is a Lucene-based indexing object store into which logs
are fed. Logs for node services and all containers in the cluster are
fed into one deployed cluster. The ElasticSearch cluster should be deployed
with redundancy and persistent storage for scale and high availability.

## Fluentd

Fluentd is responsible for gathering log entries from all nodes, enriching
them with metadata, and feeding them into the ElasticSearch cluster.

## Kibana

Kibana presents a web UI for browsing and visualizing logs in ElasticSearch.

## Logging auth proxy

In order to authenticate the Kibana user against OpenShift's Oauth2 for
single sign-on, a proxy is required that runs in front of Kibana.

## Deployer

The deployer enables the system administrator to generate all of the
necessary key/certs/secrets and deploy all of the logging components
in concert.

## Curator

Curator allows the admin to remove old data from Elasticsearch on a per-project
basis, and is configurable on a per-project basis.  See the parent README.md
for details.

# Using the Logging Deployer

The deployer pod can enable deploying the full stack of the aggregated
logging solution with just a few prerequisites:

1. Sufficient volumes defined for ElasticSearch cluster storage.
2. A router deployment for serving cluster-defined routes for Kibana.

The deployer generates all the necessary certs/keys/etc for cluster
communication and defines secrets and templates for all of the necessary
API objects to implement aggregated logging. There are a few
manual steps you must run with cluster-admin privileges.

## Choose a Project

You will likely want to put all logging-related entities in their own project.
For examples in this document we will assume the `logging` project.

    $ oadm new-project logging --node-selector=""
    $ oc project logging

You can use the `default` or another project if you want. This
implementation has no need to run in any specific project.

## Create missing templates

If your installation did not create templates in the `openshift`
namespace, the `logging-deployer-template` and `logging-deployer-account-template`
templates may not exist. In that case you can create them with the following:

    $ oc apply -n openshift -f https://raw.githubusercontent.com/openshift/origin-aggregated-logging/master/deployer/deployer.yaml

## Create Supporting Service Accounts and Permissions

The deployer must run under a service account defined as follows:
(Note: change `:logging:` below to match the project name.)

    $ oc new-app logging-deployer-account-template
    $ oadm policy add-cluster-role-to-user oauth-editor \
              system:serviceaccount:logging:logging-deployer

The policy manipulation is required in order for the deployer pod to
create an OAuthClient for Kibana to authenticate against the master,
normally a cluster-admin privilege.

The fluentd component also requires special privileges for its service
account. Run the following command to add the aggregated-logging-fluentd
service account to the privileged SCC (to allow it to mount system logs)
and give it the `cluster-reader` role to allow it to read labels from
all pods (note, change `:logging:` below to the project of your choice):

    $ oadm policy add-scc-to-user privileged \
           system:serviceaccount:logging:aggregated-logging-fluentd
    $ oadm policy add-cluster-role-to-user cluster-reader \
           system:serviceaccount:logging:aggregated-logging-fluentd

The remaining steps do not require cluster-admin privileges to run.

## Specify Deployer Parameters

Parameters for the EFK deployment may be specified in the form of a
`[ConfigMap](https://docs.openshift.org/latest/dev_guide/configmaps.html)`,
a `[Secret](https://docs.openshift.org/latest/dev_guide/secrets.html)`,
or template parameters (which are passed as environment variables). The
deployer looks for each value first in a `logging-deployer` ConfigMap,
then a `logging-deployer` Secret, then as an environment variable. Any
or all may be omitted if not needed.

### Create Deployer ConfigMap

You will need to specify the hostname at which Kibana should be
exposed to client browsers, and also the master URL where client
browsers will be directed for authenticating to OpenShift. You should
read the [ElasticSearch](#elasticsearch) section below before choosing
ElasticSearch parameters for the deployer. These and other parameters
are available:

* `kibana-hostname`: External hostname where web clients will reach Kibana
* `public-master-url`: External URL for the master, for OAuth purposes
* `es-cluster-size`: How many instances of ElasticSearch to deploy. At least 3 are needed for redundancy, and more can be used for scaling.
* `es-instance-ram`: Amount of RAM to reserve per ElasticSearch instance (e.g. 1024M, 2G). Defaults to 8GiB; must be at least 512M (Ref.: [ElasticSearch documentation](https://www.elastic.co/guide/en/elasticsearch/guide/current/hardware.html#_memory).
* `es-pvc-size`: Size of the PersistentVolumeClaim to create per ElasticSearch ops instance, e.g. 100G. If empty, no PVCs will be created and emptyDir volumes are used instead.
* `es-pvc-prefix`: Prefix for the names of PersistentVolumeClaims to be created; a number will be appended per instance. If they don't already exist, they will be created with size `es-pvc-size`.
* `es-pvc-dynamic`: Set to `true` to have created PersistentVolumeClaims annotated such that their backing storage can be dynamically provisioned (if that is available for your cluster).
* `storage-group`: Number of a supplemental group ID for access to Elasticsearch storage volumes; backing volumes should allow access by this group ID (defaults to 65534).
* `fluentd-nodeselector`: The nodeSelector to use for the Fluentd DaemonSet. Defaults to "logging-infra-fluentd=true".
* `es-nodeselector`: Specify the nodeSelector that Elasticsearch should be use (label=value)
* `kibana-nodeselector`: Specify the nodeSelector that Kibana should be use (label=value)
* `curator-nodeselector`: Specify the nodeSelector that Curator should be use (label=value)
* `enable-ops-cluster`: If "true", configure a second ES cluster and Kibana for ops logs. (See [below](#ops-cluster) for details.)
* `kibana-ops-hostname`, `es-ops-instance-ram`, `es-ops-pvc-size`, `es-ops-pvc-prefix`, `es-ops-cluster-size`, `es-ops-nodeselector`, `kibana-ops-nodeselector`, `curator-ops-nodeselector`: Parallel parameters for the ops log cluster.
* `image-pull-secret`: Specify the name of an existing pull secret to be used for pulling component images from an authenticated registry.

An invocation supplying the most important parameters might be:

    $ oc create configmap logging-deployer \
       --from-literal kibana-hostname=kibana.example.com \
       --from-literal public-master-url=https://localhost:8443 \
       --from-literal es-cluster-size=3

It is also relatively easy to edit `ConfigMap` YAML after creating it:

    $ oc edit configmap logging-deployer

### Create Deployer Secret

Security parameters for the logging infrastructure
deployment can be supplied to the deployer in the form of a
Most other parameters can be supplied in the form of a ConfigMap.
Actually, the following parameters can all be supplied either way; but
files used for security purposes are typically supplied in a secret.

All contents of the secret and configmap are optional (they will be
generated/defaulted if not supplied). The following files may be supplied
in the `logging-deployer` secret:

* `kibana.crt` - A browser-facing certificate for the Kibana route. If not supplied, the route is secured with the default router cert.
* `kibana.key` - A key to be used with the Kibana certificate.
* `kibana-ops.crt` - A browser-facing certificate for the Ops Kibana route. If not supplied, the route is secured with the default router cert.
* `kibana-ops.key` - A key to be used with the Ops Kibana certificate.
* `kibana-internal.crt` - An internal certificate for the Kibana server.
* `kibana-internal.key` - A key to be used with the internal Kibana certificate.
* `server-tls.json` - JSON TLS options to override the internal Kibana TLS defaults; refer to
  [NodeJS docs](https://nodejs.org/api/tls.html#tls_tls_connect_options_callback) for
  available options and the [default options](conf/server-tls.json) for an example.
* `ca.crt` - A certificate for a CA that will be used to sign and validate any
  certificates generated by the deployer.
* `ca.key` - A matching CA key.

An invocation supplying a properly signed Kibana cert might be:

    $ oc create secret generic logging-deployer \
       --from-file kibana.crt=/path/to/cert \
       --from-file kibana.key=/path/to/key

### Choose Template Parameters

When running the deployer in the next step, there are a few parameters
that are specified directly if needed:

* `IMAGE_PREFIX`: Specify the prefix for logging component images; e.g. for "docker.io/openshift/origin-logging-deployer:v1.2.0", set prefix "docker.io/openshift/origin-"
* `IMAGE_VERSION`: Specify version for logging component images; e.g. for "docker.io/openshift/origin-logging-deployer:v1.2.0", set version "v1.2.0"
* `MODE`: Mode to run the deployer in; one of `install`, `uninstall`, `reinstall`, `upgrade`, `migrate`, `start`, `stop`. `migrate` refers to the ES UUID data migration that is required for upgrading from version 1.1. `stop` and `start` can be used to safely pause the cluster for maintenance.

## Run the Deployer

You run the deployer by instantiating a template. Here is an example with
some parameters (just for demonstration purposes -- none are required):

    $ oc new-app logging-deployer-template \
               -p IMAGE_VERSION=v1.2.0 \
               -p MODE=install

This creates a deployer pod and prints its name. As this is running in
`install` mode (which the default), it will create a new deployment of
the EFK stack. Wait until the pod is running; this can take up to a few
minutes to retrieve the deployer image from its registry. You can watch
it with:

    $ oc get pod/<pod-name> -w

If it seems to be taking too long, you can retrieve more details about
the pod and any associated events with:

    $ oc describe pod/<pod-name>

When it runs, check the logs of the resulting pod (`oc logs -f <pod name>`)
for some instructions to follow after deployment. More details
are given below.

## Adjusting the Deployment

Read on to learn about Elasticsearch parameters, how to have Fluentd
deployed, what the Ops cluster is for and explain the contents of the secrets the
deployer creates and how to change them.

### ElasticSearch

The deployer creates the number of ElasticSearch instances specified by
`es-cluster-size`. The nature of ElasticSearch and current Kubernetes
limitations require that we use a different scaling mechanism than the
standard Kubernetes scaling.

Scaling a standard deployment (a Kubernetes ReplicationController)
to multiple pods currently mounts the same volumes on all pods in the
deployment. However, multiple ElasticSearch instances in a cluster
cannot share storage; each pod requires its own storage. Work is under
way to enable specifying multiple volumes to be allocated individually
to instances in a deployment, but for now the deployer creates multiple
deployments in order to scale ElasticSearch to multiple instances. You
can view the deployments with:

    $ oc get dc --selector logging-infra=elasticsearch

These deployments all have different names but will cluster with each other
via `service/logging-es-cluster`.

It is possible to scale your cluster up after creation by adding more
deployments from a template; however, scaling up (or down) requires
the correct procedure and an awareness of clustering parameters (to be
described in a separate section). It is best if you can indicate the
desired scale at first deployment.

Refer to [Elastic's
documentation](https://www.elastic.co/guide/en/elasticsearch/guide/current/hardware.html#_disks)
for considerations involved in choosing storage and network location
as directed below.

#### Storage

By default, the deployer creates an ephemeral deployment in which all
of a pod's data will be lost any time it is restarted. For production
use you should specify a persistent storage volume for each deployment
of ElasticSearch. The deployer parameters with `-pvc-` in the name should
be used for this. You can either use a pre-existing set of PVCs (specify
a common prefix for their names and append numbers starting at 1, for
example with default prefix `logging-es-` supply PVCs `logging-es-1`,
`logging-es-2`, etc.), or the deployer can create them with a request
for a specified size. This is the recommended method of supplying
persistent storage.

You may instead choose to add volumes manually to deployments with the
`oc volume` command. For example, to use a local directory on the host
(which is actually recommended by Elastic in order to take advantage of
local disk performance):

    $ oc volume dc/logging-es-rca2m9u8 \
              --add --overwrite --name=elasticsearch-storage \
              --type=hostPath --path=/path/to/storage

Note: In order to allow the pods to mount host volumes, you would usually
need to add the `aggregated-logging-elasticsearch` service account to
the `hostmount-anyuid` SCC similar to Fluentd as shown above. Use node
selectors and node labels carefully to ensure that pods land on nodes
with the storage you intend.

See `oc volume -h` for further options. E.g. if you have a specific NFS volume
you would like to use, you can set it with:

    $ oc volume dc/logging-es-rca2m9u8 \
              --add --overwrite --name=elasticsearch-storage \
              --source='{"nfs": {"server": "nfs.server.example.com", "path": "/exported/path"}}'

#### Node selector

ElasticSearch can be very resource-heavy, particularly in RAM, depending
on the volume of logs your cluster generates. Per Elastic's guidance,
all members of the cluster should have low latency network connections
to each other.  You will likely want to direct the instances to dedicated
nodes, or a dedicated region in your cluster. You can do this by supplying
a node selector in each deployment.

The deployer has options to specify a nodeSelector label for Elasticsearch, Kibana
and Curator. If you have already deployed the EFK stack or would like to customize
your nodeSelector labels per deployment, see below.

There is no helpful command for adding a node selector (yet). You will
need to `oc edit` each DeploymentConfig and add or modify the `nodeSelector`
element to specify the label corresponding to your desired nodes, e.g.:

    apiVersion: v1
    kind: DeploymentConfig
    spec:
      template:
        spec:
          nodeSelector:
            nodelabel: logging-es-node-1

Alternatively, you can use `oc patch` to do this as well:
```
oc patch dc/logging-es-{unique name} -p '{"spec":{"template":{"spec":{"nodeSelector":{"nodelabel":"logging-es-node-1"}}}}}'
```

Recall that the default scheduler algorithm will spread pods to different
nodes (in the same region, if regions are defined). However this can
have unexpected consequences in several scenarios and you will most
likely want to label and specify designated nodes for ElasticSearch.

#### Cluster parameters

There are some administrative settings that can be supplied (ref. [Elastic documentation](https://www.elastic.co/guide/en/elasticsearch/guide/current/_important_configuration_changes.html)).

* `minimum_master_nodes` - the quorum required to elect a new master. Should be more than half the intended cluster size.
* `recover_after_nodes` - when restarting the cluster, require this many nodes to be present before starting recovery.
* `expected_nodes` and `recover_after_time` - when restarting the cluster, wait for number of nodes to be present or time to expire before starting recovery.

These are, respectively, the `NODE_QUORUM`, `RECOVER_AFTER_NODES`,
`RECOVER_EXPECTED_NODES`, and `RECOVER_AFTER_TIME` parameters in the
ES deployments and the ES template. The deployer also enables specifying
these parameters. However, usually the defaults
should be sufficient, unless you need to scale ES after deployment.

### Fluentd

Fluentd is deployed as a DaemonSet that deploys replicas according
to a node label selector (which you can choose; the default is
`logging-infra-fluentd`). Once you have ElasticSearch running as
desired, label the nodes to deploy Fluentd to in order to feed
logs into ES. The example below would label a node named
'ip-172-18-2-170.ec2.internal' using the default Fluentd node selector.

    $ oc label node/ip-172-18-2-170.ec2.internal logging-infra-fluentd=true

Alternatively, you can label all nodes with the following:

    $ oc label node --all logging-infra-fluentd=true

Note: Labeling nodes requires cluster-admin capability.

### Kibana

You may scale the Kibana deployment normally for redundancy:

    $ oc scale dc/logging-kibana --replicas=2
    $ oc scale rc/logging-kibana-1 --replicas=2

You should be able to visit the `KIBANA_HOSTNAME` specified in the
initial deployment to visit the UI (assuming DNS points correctly for
this domain). You will get a certificate warning if you did not provide
a properly signed certificate in the deployer secret. You should be able
to login with the same users that can login to the web console and have
index patterns defined for projects the user has access to.

### Ops cluster

If you set `ENABLE_OPS_CLUSTER` to `true` for the deployer, fluentd
expects to split logs between the main ElasticSearch cluster and another
cluster reserved for operations logs (node logs and `default` project).
Thus a separate ElasticSearch cluster and a separate Kibana are deployed
to index and access operations logs. These deployments are set apart with
the `-ops` included in their names. The same considerations apply as
for the main cluster.

### Curator

It is recommended to have one curator for each Elasticsearch cluster.

### About the Deployer generated secrets

Part of the installation process that is done by the logging deployer is to generate
certificates and keys and make them available to the logging components by means
of secrets.

The secrets that the components make use of are:

    logging-curator
    logging-curator-ops
    logging-elasticsearch
    logging-fluentd
    logging-kibana
    logging-kibana-proxy

#### logging-curator, logging-kibana, logging-fluentd

These three components all have a `ca`, `key` and `cert` entries.  These are what
are used for mutual TLS communication with Elasticsearch.
  1. `ca` contains the certificate to validate the Elasticsearch server certificate.
  2. `key` is the generated key specific to that component.
  3. `cert` is the client certificate created using `key`.

#### logging-kibana-proxy

The Kibana proxy container is used to serve requests from clients outside of the
cluster.  It contains `oauth-secret`, `server-cert`, `server-key`, `server-tls.json`
and `session-secret`.
  1. `oauth-secret` is used to communicate with the oauthclient created as part of
the aggregated logging installation to redirect requests for authentication with
the OpenShift console.
  2. `server-cert` is the browser-facing certificate served up by the auth proxy
  3. `server-key` is the browser-facing key served up by the auth proxy
  4. `server-tls.json` is the proxy TLS configuration file
  5. `session-secret` contains the generated proxy session that is used to secure
the user's cookie containing their auth token once obtained.

#### logging-elasticsearch

Elasticsearch is the central piece for aggregated logging.  It ensures that communication
between itself and the other components is secure as well as securing communication
between its other Elasticsearch cluster members.  It contains `admin-ca`, `admin-cert`,
`admin-key`, `key`, `searchguard.key` and `truststore`.
  1. `admin-ca` contains the ca to be used when doing ES operations as the admin user.
  2. `admin-cert` is the client certificate for the admin user corresponding to `admin-key`
  3. `admin-key` contains the generated key for the ES admin user.
  4. `key` contains the Elasticsearch server key and certificate used by Searchguard for mutual TLS
  5. `searchguard.key` contains the generated key used for communication with other
Elasticsearch cluster members.
  6. `truststore` contains the CA that validates client certificates

#### Changing secret contents

Disclaimer: Changing the contents of secrets may result in a non-working aggregated
logging installation if not done correctly. As with any other changes to your
aggregated logging cluster, you should stop your cluster prior to making any
changes to secrets to minimize the loss of log records.

The contents of secrets are base64 encoded, so when patching we need to ensure that
the value we are replacing with is encoded. If we wanted to change the `key` value
in `secret/logging-curator` and replace it with the contents of the file `new_key.key`
we would use (assuming the bash shell):

    $ oc patch secret/logging-curator -p='{"data":{"key": "'$(base64 -w 0 < new_key.key)'"}}'

## Using Kibana

The subject of using Kibana in general is covered in that [project's
documentation](https://www.elastic.co/guide/en/kibana/4.1/discover.html).
Here is some information specific to the aggregated logging deployment.

1. Login is performed via OAuth2, as with the web console. The default certificate
authentication used for the admin user isn't available, but you can create
other users and make them cluster admins.
2. Kibana and ElasticSearch have been customized to display logs only
to users that have access to the projects the logs came from. So if you login
and have no access to anything, be sure your user has access to at least one
project. Cluster admin users should have access to all project logs as
well as host logs.
3. To do anything with ElasticSearch and Kibana, Kibana has to have
defined some index patterns that match indices being recorded in
ElasticSearch. This should already be done for you, but you should be
aware how these work in case you want to customize anything. When logs
from applications in a project are recorded, they are indexed by project
name and date in the format `name.YYYY-MM-DD`. For matching a project's
logs for all dates, an index pattern will be defined in Kibana for each
project which looks like `name.*`.
4. When first visiting Kibana, the first page directs you to create
an index pattern.  In general this should not be necessary and you can
just click the "Discover" tab and choose a project index pattern to see
logs. If there are no logs yet for a project, you won't get any results;
keep in mind also that the default time interval for retrieving logs is
15 minutes and you will need to adjust it to find logs older than that.
5. Unfortunately there is no way to stream logs as they are created at
this time.

## Cleanup and removal

If you wish to remove everything generated or instantiated without having
to destroy the project:

    $ oc delete all --selector logging-infra=kibana
    $ oc delete all,daemonset --selector logging-infra=fluentd
    $ oc delete all --selector logging-infra=elasticsearch
    $ oc delete all,sa,oauthclient --selector logging-infra=support
    $ oc delete secret logging-fluentd logging-elasticsearch logging-es-proxy logging-kibana logging-kibana-proxy logging-kibana-ops-proxy

#### Adjusting ElasticSearch After Deployment

If you need to change the ElasticSearch cluster size after deployment,
DO NOT just scale existing deployments up or down. ElasticSearch cannot
scale by ordinary Kubernetes mechanisms, as explained above. Each instance
requires its own storage, and thus under current capabilities, its own
deployment. The deployer defined a template `logging-es-template` which
can be used to create new ElasticSearch deployments.

Adjusting the scale of the ElasticSearch cluster
typically requires adjusting cluster parameters that
vary by cluster size. [Elastic documentation discusses these
issues](https://www.elastic.co/guide/en/elasticsearch/guide/current/_important_configuration_changes.html)
and the corresponding parameters are coded as environment variables
in the existing deployments and parameters in the deployment template
(mentioned in the [Settings](#settings) section).  The deployer chooses sensible
defaults based on cluster size. These should be adjusted for both new
and existing deployments when changing the cluster size.

Changing cluster parameters (or any parameters/secrets, really) requires
re-deploying the instances. In order to minimize resynchronization
between the instances as they are restarted, we advise halting traffic to
ElasticSearch and then taking down the entire cluster for maintenance. No
logs will be lost; Fluentd simply blocks until the cluster returns.

Halting traffic to ElasticSearch requires scaling down Kibana and removing node labels for Fluentd:

    $ oc label node --all logging-infra-
    $ oc scale rc/logging-kibana-1 --replicas=0

Next scale all of the ElasticSearch deployments to 0 similarly.

    $ oc get rc --selector logging-infra=elasticsearch
    $ oc scale rc/logging-es-... --replicas=0

Now edit the existing DeploymentConfigs and modify the variables as needed:

    $ oc get dc --selector logging-infra=elasticsearch
    $ oc edit dc logging-es-...

You can adjust parameters in the template (`oc edit template logging-es-template`) and reuse it to add more instances:

    $ oc process logging-es-template | oc create -f -

Keep in mind that these are deployed immediately and will likely need
storage and node selectors defined. You may want to scale them down to
0 while operating on them.

Once all the deployments are properly configured, deploy them all at
about the same time.

    $ oc get dc --selector logging-infra=elasticsearch
    $ oc deploy --latest logging-es-...

The cluster parameters determine how cluster formation and recovery
proceeds, but the default is that the cluster will wait up to five minutes
for all instances to start up and join the cluster. After the cluster
is formed, new instances will begin replicating data from the existing
instances, which can take a long time and generate a lot of network
traffic and disk activity, but the cluster is operational immediately and
Kibana can be scaled back to its normal operating levels and nodes can be re-labeled
for Fluentd.

    $ oc label node --all logging-infra-fluentd=true
    $ oc scale rc/logging-kibana-1 --replicas=2


## Troubleshooting

There are a number of common problems with logging deployment that have simple
explanations but do not present useful errors for troubleshooting.

### Looping login on Kibana

The experience here is that when you visit Kibana, it redirects you to
login. Then when you login successfully, you are redirected back to Kibana,
which immediately redirects back to login again.

The typical reason for this is that the OAuth2 proxy in front of Kibana
is supposed to share a secret with the master's OAuth2 server, in order
to identify it as a valid client. This problem likely indicates that
the secrets do not match (unfortunately nothing reports this problem
in a way that can be exposed). This can happen when you deploy logging
more than once (perhaps to fix the initial deployment) and the `secret`
used by Kibana is replaced while the master `oauthclient` entry to match
it is not.

In this case, you should be able to do the following:

    $ oc delete oauthclient/kibana-proxy
    $ oc process logging-support-template | oc create -f -

This will replace the oauthclient (and then complain about the other
things it tries to create that are already there - this is normal). Then
your next successful login should not loop.

### "error":"invalid\_request" on login

When you visit Kibana directly and it redirects you to login, you instead
receive an error in the browser like the following:

     {"error":"invalid_request","error_description":"The request is missing a required parameter,
      includes an invalid parameter value, includes a parameter more than once, or is otherwise malformed."}

The reason for this is again a mismatch between the OAuth2 client and server.
The return address for the client has to be in a whitelist for the server to
securely redirect back after logging in; if there is a mismatch, then this
cryptic error message is shown.

As above, this may be caused by an `oauthclient` entry lingering from a
previous deployment, in which case you can replace it:

    $ oc delete oauthclient/kibana-proxy
    $ oc process logging-support-template | oc create -f -

This will replace the `oauthclient` (and then complain about the other
things it tries to create that are already there - this is normal).
Return to the Kibana URL and try again.

If the problem persists, then you may be accessing Kibana at
a URL that the `oauthclient` does not list. This can happen when, for
example, you are trying out logging on a vagrant-driven VirtualBox
deployment of OpenShift and accessing the URL at forwarded port 1443
instead of the standard 443 HTTPS port. Whatever the reason, you can
adjust the server whitelist by editing its `oauthclient`:

    $ oc edit oauthclient/kibana-proxy

This brings up a YAML representation in your editor, and you can edit
the redirect URIs accepted to include the address you are actually using.
After you save and exit, this should resolve the error.

### Deployment fails, RCs scaled to 0

When a deployment is performed, if it does not successfully bring up an
instance before a ten-minute timeout, it will be considered failed and
scaled down to zero instances. `oc get pods` will show a deployer pod
with a non-zero exit code, and no deployed pods, e.g.:

    NAME                           READY     STATUS             RESTARTS   AGE
    logging-es-2e7ut0iq-1-deploy   1/1       ExitCode:255       0          1m

(In this example, the deployer pod name for an ElasticSearch deployment is shown;
this is from ReplicationController `logging-es-2e7ut0iq-1` which is a deployment
of DeploymentConfig `logging-es-2e7ut0iq`.)

Deployment failure can happen for a number of transitory reasons, such as
the image pull taking too long, or nodes being unresponsive. Examine the
deployer pod logs for possible reasons; but often you can simply redeploy:

    $ oc deploy --latest logging-es-2e7ut0iq

Or you may be able to scale up the existing deployment:

    $ oc scale --replicas=1 logging-es-2e7ut0iq-1

If the problem persists, you can examine pods, events, and systemd unit
logs to determine the source of the problem.

### Image pull fails

If you specify an IMAGE\_PREFIX that results in images being defined that don't exist,
you will receive a corresponding error message, typically after creating the deployer.

    NAME                     READY     STATUS                                                                                       RESTARTS   AGE
    logging-deployer-1ub9k   0/1       Error: image registry.access.redhat.com:5000/openshift3logging-deployment:latest not found   0          1m

In this example, for the intended image name
`registry.access.redhat.com:5000/openshift3/logging-deployment:latest`
the `IMAGE\_PREFIX` needed a trailing `/`:

    $ oc process logging-deployer-template \
               -v IMAGE_PREFIX=registry.access.redhat.com:5000/openshift3/,...

You can just re-create the deployer with the proper parameters to proceed.

### Can't resolve kubernetes.default.svc.cluster.local

This internal alias for the master should be resolvable by the included
DNS server on the master. Depending on your platform, you should be able
to run the `dig` command (perhaps in a container) against the master to
check whether this is the case:

    master$ dig kubernetes.default.svc.cluster.local @localhost
    [...]
    ;; QUESTION SECTION:
    ;kubernetes.default.svc.cluster.local. IN A

    ;; ANSWER SECTION:
    kubernetes.default.svc.cluster.local. 30 IN A   172.30.0.1

Older versions of OpenShift did not automatically define this internal
alias for the master. You may need to upgrade your cluster in order to
use aggregated logging. If your cluster is up to date, there may be
a problem with your pods reaching the SkyDNS resolver at the master,
or it could have been blocked from running. You should resolve this
problem before deploying again.

### Can't connect to the master or services

If DNS resolution does not return at all or the address cannot be
connected to from within a pod (e.g. the deployer pod), this generally
indicates a system firewall/network problem and should be debugged
as such.

### Kibana access shows 503 error

If everything is deployed but visiting Kibana results in a proxy
error, then one of the following things is likely to be the issue.

First, Kibana might not actually have any pods that are recognized
as running. If ElasticSearch is slow in starting up, Kibana may
error out trying to reach it, and won't be considered alive. You can
check whether the relevant service has any endpoints:

    $ oc describe service logging-kibana
    Name:                   logging-kibana
    [...]
    Endpoints:              <none>

If any Kibana pods are live, endpoints should be listed. If they are
not, check the state of the Kibana pod(s) and deployment.

Second, the named route for accessing the Kibana service may be masked.
This tends to happen if you do a trial deployment in one project and
then try to deploy in a different project without completely removing the first one.
When multiple routes are declared for the same destination, the default router will route to
the first created. You can check if the route in question is defined in multiple places with:

    $ oc get route  --all-namespaces --selector logging-infra=support
    NAMESPACE   NAME         HOST/PORT                 PATH      SERVICE
    logging     kibana       kibana.example.com                  logging-kibana
    logging     kibana-ops   kibana-ops.example.com              logging-kibana-ops

(In this example there are no overlapping routes.)
