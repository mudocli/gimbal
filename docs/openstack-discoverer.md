# Openstack Discoverer

## Overview

The Openstack discoverer provides service discovery for an Openstack cluster. It does this by monitoring all Load Balancer as a Service (LBaaS) configured as well as the corresponding Members. They are synchronized to the Team namespace as Services and Endpoints, with the Namespace being configured as the TenantName in Openstack.

The Discoverer will poll the Openstack API on a customizable interval and update the Gimbal cluster accordingly.

The discoverer will only be responsible for monitoring a single cluster at a time. If multiple clusters are required to be watched, then multiple discoverers will need to be deployed.

## Technical Details

The following sections outline the technical implementations of the discoverer.

### Arguments

Arguments are available to customize the discoverer, most have defaults but others are required to be configured by the cluster administrators:

| flag  | default  | description  |
|---|---|---|
| version  |  false | Show version, build information and quit  
| num-threads  | 2  |  Specify number of threads to use when processing queue items
| gimbal-kubecfg-file  | ""  | Location of kubecfg file for access to Kubernetes cluster hosting Gimbal
| cluster-name  | ""  |   Name of cluster scraping for services & endpoints (Cannot start or end with a hyphen and must be lowercase alpha-numeric)
| debug | false | Enable debug logging 
| reconciliation-period | 30s | The interval of time between reconciliation loop runs 
| http-client-timeout | 5s | The HTTP client request timeout
| openstack-certificate-authority | "" | Path to cert file of the OpenStack API certificate authority

### Credentials

The discoverer requires a username/password, auth URL, as well as the TenantName of the Openstack cluster to be discovered:

Credentials required:

- Username: User with access to the API
- Password: Password for User
- AuthURL: Openstack API Url
- TenantName: Tenant used to discover services
- Cluster Name: Unique name of cluster to identify
- Certificate Authority Data: CA Certificate (if required)

_NOTE: These are exposed to the deployment via environment variables._

#### Example

Following example creates a Kubernetes secret which the Openstack discoverer will consume and get credentials & other information to be able to discover services & endpoints: 

```sh
$ kubectl create secret generic remote-discover-openstack --from-file=certificate-authority-data=./ca.pem --from-literal=cluster-name=openstack --from-literal=username=admin --from-literal=password=abc123 --from-literal=auth-url=https://api.openstack:5000/ --from-literal=tenant-name=heptio
```

### Updating Credentials

Credentials to the backend OpenStack cluster can be updated at any time if necessary. To do so, we recommend taking advantage of the Kubernetes deployment's update features:

1. Create a new secret with the new credentials.
2. Update the deployment to reference the new secret.
3. Wait until the discoverer pod is rolled over.
4. Verify the discoverer is up and running.
5. Delete the old secret, or rollback the deployment if the discoverer failed to start.

### Data flow

Data flows from the remote cluster into the Gimbal cluster. The steps on how they replicate are as follows:

1. Connection is made to remote cluster and all LBaaS's and corresponding Members are retrieved from the cluster
2. Those objects are then translated into Kubernetes Services and Endpoints, then synchronized to the Gimbal cluster in the same namespace as the remote cluster. Labels will also be added during the synchronization (See the [labels](#labels) section for more details).
3. Once the initial list of objects is synchronized, any further updates will happen based upon the configured `reconciliation-period` which will start a new reconciliation loop.

### Labels

All synchronized services & endpoints will have additional labels added to assist in understanding where the object were sourced from. 

Labels added to service and endpoints:
```
gimbal.heptio.com/service=<serviceName>
gimbal.heptio.com/cluster=<nodeName>
gimbal.heptio.com/load-balancer-id=<LoadBalancer.ID>
gimbal.heptio.com/load-balancer-name=<LoadBalancer..Name>
```