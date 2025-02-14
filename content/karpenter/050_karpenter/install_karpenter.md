---
title: "Install Karpenter"
date: 2021-11-07T11:05:19-07:00
weight: 20
draft: false
---

In this section we will install Karpenter and learn how to configure a default [Provisioner CRD](https://karpenter.sh/docs/provisioner-crd/) to set the configuration. Karpenter is installed in clusters with a [helm](https://helm.sh/) chart. Karpenter follows best practices for kubernetes controllers for its configuration. Karpenter uses [Custom Resource Definition(CRD)](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/) to declare its configuration. Custom Resources are extensions of the Kubernetes API. One of the premises of Kubernetes is the [declarative aspect of its APIs](https://kubernetes.io/docs/concepts/overview/kubernetes-api/). Karpenter simplifies its configuration by adhering to that principle.

## Install Karpenter Helm Chart

We will use helm to deploy Karpenter to the cluster. 

```
export KARPENTER_IAM_ROLE_ARN="arn:aws:iam::${AWS_ACCOUNT_ID}:role/${CLUSTER_NAME}-karpenter"
export CLUSTER_ENDPOINT="$(aws eks describe-cluster --name ${CLUSTER_NAME} --query "cluster.endpoint" --output text)"
echo "export KARPENTER_IAM_ROLE_ARN=${KARPENTER_IAM_ROLE_ARN}" >> ~/.bash_profile
echo "export CLUSTER_ENDPOINT=${CLUSTER_ENDPOINT}" >> ~/.bash_profile
helm repo add karpenter https://charts.karpenter.sh
helm repo update
helm upgrade --install --namespace karpenter --create-namespace \
  karpenter karpenter/karpenter \
  --version ${KARPENTER_VERSION}\
  --set serviceAccount.annotations."eks\.amazonaws\.com/role-arn"=${KARPENTER_IAM_ROLE_ARN} \
  --set clusterName=${CLUSTER_NAME} \
  --set clusterEndpoint=${CLUSTER_ENDPOINT} \
  --set nodeSelector.intent=control-apps \
  --set defaultProvisioner.create=false \
  --set aws.defaultInstanceProfile=KarpenterNodeInstanceProfile-${CLUSTER_NAME} \
  --wait
```

The command above:

* uses the both the **CLUSTER_NAME** and the **CLUSTER_ENDPOINT** so that Karpenter controller can contact the Cluster API Server.

* uses the `--set defaultProvisioner.create=false`. We will set a default Provisioner configuration in the next section. This will help us understand Karpenter Provisioners.

* uses the argument `--set nodeSelector.intent=control-apps` to deploy the controllers in the On-Demand managed node group that was created with the cluster.

* uses the argument `--set aws.defaultInstanceProfile=KarpenterNodeInstanceProfile-${CLUSTER_NAME}` to use the instance profile we created before to grant permissions necessary to instances to run containers and configure networking.

* Karpenter configuration is provided through a Custom Resource Definition. We will be learning about providers in the next section, the `--wait` notifies the webhook controller to wait until the Provisioner CRD has been deployed.

To check Karpenter is running you can check the Pods, Deployment and Service are Running.

To check running pods run the command below. There should be at least one pod `karpenter`
```
kubectl get pods --namespace karpenter
```

You should see an output similar to the one below. 
```
NAME                         READY   STATUS    RESTARTS   AGE
karpenter-6c59fc5c96-l8fwv   2/2     Running   0          3m51s
karpenter-6c59fc5c96-zw5jf   2/2     Running   0          3m51s
```


To check the deployment. Like with the pods, there should be one deployment  `karpenter`
```
kubectl get deployment -n karpenter
```

{{% notice info %}}
Since **v0.16.0** Karpenter deploys 2 replicas. One of the replicas is elected as as a Leader while the other stays in standby mode. Karpenter deployment also uses `topologySpreadConstraints` to spread each replica in a different AZ.
{{% /notice %}}
