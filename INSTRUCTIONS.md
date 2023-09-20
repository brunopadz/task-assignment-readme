# Instructions

## Prerequisites

- Kubernetes 1.22-1.26
- Helm 3.2.0+
- PV provisioner support in the underlying infrastructure, such as [aws-ebs-csi](https://github.com/kubernetes-sigs/aws-ebs-csi-driver).

## Helm values

### Global parameters

The following parameters are used to configure global settings for the Elasticsearch cluster.

| Parameter                | Description                                                                                                                                                                                                                                                                                                                                  | Default                                 |
|--------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------------------------------------|
| name                     | Name of the Elasticsearch cluster                                                                                                                                                                                                                                                                                                            | `""`                                    |
| version                  | Elasticsearch version                                                                                                                                                                                                                                                                                                                        | `"8.9.2"`                               |
| volumeClaimDeletePolicy  | Controls what to do with PersistentVolumeClaims when the cluster or a node is deleted. Possible values are `DeleteOnScaledownAndClusterDeletion` and `DeleteOnScaledownOnly`. See the [official docs](https://www.elastic.co/guide/en/cloud-on-k8s/2.7/k8s-volume-claim-templates.html#k8s_controlling_volume_claim_deletion) for more info. | `"DeleteOnScaledownAndClusterDeletion"` |
| pdb.enabled              | Enable Pod Disruption Budget                                                                                                                                                                                                                                                                                                                 | `true`                                  |
| pdb.minAvailable         | Minimum number of Pods that must still be available after the eviction, even in the absence of the evicted Pod.                                                                                                                                                                                                                              | `2`                                     |
| injectSecrets.enabled    | Enable injection of secrets                                                                                                                                                                                                                                                                                                                  | `false`                                 |
| injectSecrets.secretName | Name of the secret to inject                                                                                                                                                                                                                                                                                                                 | `[]`                                    |
| service.type             | Type of the service to expose the cluster API. Possible values are `ClusterIP`, `LoadBalancer` and `NodePort`.                                                                                                                                                                                                                               | `ClusterIP`                             |

### Elasticsearch / nodeSets parameters

The following parameters are used to configure the nodeSets for the Elasticsearch cluster.

| Parameter                                         | Description                                                                                                                                                                           | Default           |
|---------------------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-------------------|
| nodeSets.nodes.name                               | Name of the nodeSet.                                                                                                                                                                  | `""`              |
| nodeSets.nodes.replicas                           | Number of nodes in the nodeSet.                                                                                                                                                       | `3`               |
| nodeSets.nodes.config.roles                       | Elasticsearch roles. For supported values see the Configuring Elasticsearch [node section](https://www.elastic.co/guide/en/elasticsearch/reference/8.9/modules-node.html#node-roles). | `[]`              |
| nodeSets.nodes.config.storeAllowMmap              | Enables a initContainer to configure the `vm.max_map_count` setting.                                                                                                                  | `true`            |
| nodeSets.nodes.labels.team                        | The name of the team responsible for running the Elasticsearch cluster.                                                                                                               | `"undefined"`     |
| nodeSets.nodes.updateStrategy.maxSurge            | Maximum number of Pods that can be scheduled above the desired number of Pods.                                                                                                        | `3`               |
| nodeSets.nodes.updateStrategy.maxUnavailable      | Maximum number of Pods that can be unavailable during the update process.                                                                                                             | `1`               |
| nodeSets.nodes.securityContext.id                 | Specifies a non-root user and group ID to run Elasticsearch.                                                                                                                          | `2000`            |
| nodeSets.nodes.resources.limits.cpu               | Specifies the CPU limits for the Elasticsearch container.                                                                                                                             | `"1000m"`         |
| nodeSets.nodes.resources.limits.memory            | Specifies the memory limits for the Elasticsearch container.                                                                                                                          | `"4Gi"`           |
| nodeSets.nodes.resources.requests.cpu             | Specifies the requested CPU for the Elasticsearch container.                                                                                                                          | `"100m"`          |
| nodeSets.nodes.resources.requests.memory          | Specifies the requested memory limits for the Elasticsearch container.                                                                                                                | `"4Gi"`           |
| nodeSets.nodes.readinessProbe.failureThreshold    | Failure threshold for readinessProbe.                                                                                                                                                 | `3`               |
| nodeSets.nodes.readinessProbe.initialDelaySeconds | Initial delay in seconds for readinessProbe.                                                                                                                                          | `60`              |
| nodeSets.nodes.readinessProbe.periodSeconds       | Period in seconds for readinessProbe.                                                                                                                                                 | `10`              |
| nodeSets.nodes.readinessProbe.successThreshold    | Success threshold for readinessProbe.                                                                                                                                                 | `5`               |
| nodeSets.nodes.readinessProbe.timeoutSeconds      | Timeout in seconds for readinessProbe.                                                                                                                                                | `12`              |
| nodeSets.nodes.externalVolume.size                | Specifies the PVC storage request for data volume.                                                                                                                                    | `"2Gi"`           |
| nodeSets.nodes.externalVolume.accessMode          | Specifies the PVC access mode for data volume.                                                                                                                                        | `"ReadWriteOnce"` |
| nodeSets.nodes.externalVolume.storageClassName    | Specifies the `storageClass` to proivison the data volume.                                                                                                                            | `"gp2"`           |
| nodeSets.nodes.env                                | Specifies ENV VARs for additional Elasticsearch configuration.                                                                                                                        | `[]`              |

## Deployment process

Follow the steps below to deploy the solution.

1. Install ECK CRDs.

```bash
kubectl create -f https://download.elastic.co/downloads/eck/2.7.0/crds.yaml
```

2. Install the operator.

```bash
kubectl apply -f https://download.elastic.co/downloads/eck/2.7.0/operator.yaml
```

3. Make sure everything is installed and running.

```bash 
# Check for CRDs
kubectl get crds |grep elastic

# Check for operator
kubectl get pods -n elastic-system

# Monitor the operator logs
kubectl logs -f -n elastic-system statefulset.apps/elastic-operator
```

4. Install the Elasticsearch Helm Chart

```bash
cd elasticsearch/
helm install es-cluster . --create-namespace -n elasticsearch -f values.yaml
```

The release name `es-cluster` and the namespace `elasticsearch` can be changed to whatever you want.

5. After the deployment process, the Chart provides instructions on how to check the status of the cluster and how to make requests to the Elasticsearch API.

```bash
NAME: es-cluster
LAST DEPLOYED: Wed Sep 20 13:36:13 2023
NAMESPACE: elasticsearch
STATUS: deployed
REVISION: 1
NOTES:
=======================================================================
The Elasticsearch cluster is now up and running!
=======================================================================

To check the status of the release, run:

  $ helm status es-cluster -n elasticsearch

To get an overview of the cluster, run the following command:

  $ kubectl get elasticsearch -n elasticsearch

To check the status of the cluster, run:

  export PASSWORD=$(kubectl get secret elasticsearch-es-elastic-user -n elasticsearch -o jsonpath="{.data.elastic}" | base64 -d)
  kubectl port-forward -n elasticsearch service/elasticsearch-es-http 9200
  curl -u "elastic:$PASSWORD" -k "https://localhost:9200/_cluster/health?pretty"
```
