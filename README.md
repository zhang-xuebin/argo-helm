# Use ArgoCD to Manage (Private/Public) GKE clusters

[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)
[![Chart Publish](https://github.com/argoproj/argo-helm/actions/workflows/publish.yml/badge.svg?branch=main)](https://github.com/argoproj/argo-helm/actions/workflows/publish.yml)

If you land on this repo, probably you are looking for a solution to let ArgoCD manage your **(private)** GKE clusters, Anthos(VMWare, BareMetal) Clusters, etc. In this solution, we introduce how to use Connect Gateway and ArgoCD to achive that purpose.

This repo has twon special branches: 
 - `host-on-gke` this branch contains the configurations on how to set up when the central ArgoCD instance is running on a GKE cluster.
 - `host-on-anthos` this branch contains the configurations on how to set up when the central ArgoCD instance is running on a Anthos cluster.

## Host on GKE

Here we distinguish two projects, `HOST_PROJECT` is the GCP project which runs the central ArgoCD cluster, `FLEET_PROJECT` which is the GCP project which runs the managed external cluster. Note that these two cluster can be *different*. 

### Set up Central Host Cluster

Note the central host GKE cluster should have Workload Identity enabled.

```
gcloud iam service-accounts create argocd-fleet-admin --project $HOST_PROJECT

gcloud iam service-accounts add-iam-policy-binding --role roles/iam.workloadIdentityUser --member "serviceAccount:${HOST_PROJECT}.svc.id.goog[argocd/argocd-server]" argocd-fleet-admin@$HOST_PROJECT.iam.gserviceaccount.com
gcloud iam service-accounts add-iam-policy-binding --role roles/iam.workloadIdentityUser --member "serviceAccount:${HOST_PROJECT}.svc.id.goog[argocd/argocd-application-controller]" argocd-fleet-admin@$HOST_PROJECT.iam.gserviceaccount.com

gcloud projects add-iam-policy-binding $FLEET_PROJECT --member "serviceAccount:argocd-fleet-admin@${HOST_PROJECT}.iam.gserviceaccount.com" --role roles/gkehub.gatewayEditor
```

Install the ArgoCD using this branch.

```
# make sure current kubeconfig points to the central host cluster.

kubectl create ns argocd

git clone https://github.com/zhang-xuebin/argo-helm.git
helm dep update charts/argo-cd/
helm install argocd charts/argo-cd --namespace argocd
```


### Set up Managed External Cluster

The managed External cluster needs to be registered to a GCP Fleet. See this [document](https://cloud.google.com/anthos/fleet-management/docs/register/gke) for more in-depth explainations, as an example:

```
gcloud container clusters list --uri --project $FLEET_PROJECT
gcloud container fleet memberships register $MEMBERSHIP_NAME \
 --gke-uri=$GKE_URI \
 --enable-workload-identity --project $FLEET_PROJECT
```

After registration, you should be able to see it by running:
```
gcloud container fleet memberships list --project $FLEET_PROJECT
```

Enable Connect Gateway API for future use
```
gcloud services enable --project=FLEET_PROJECT connectgateway.googleapis.com 
```

Prepare a yaml file like the example in `cluster-enroll/target-gke.yaml`, apply the yaml file to ArgoCD cluster.

```
apiVersion: v1
kind: Secret
metadata:
  name: private-gke-cluster
  labels:
    argocd.argoproj.io/secret-type: cluster
type: Opaque
stringData:
  name: private-gke-cluster
  server: https://connectgateway.googleapis.com/v1beta1/projects/$FLEET_PROJECT_NUMBER/locations/global/gkeMemberships/private-gke-prod
  config: |
    {
      "execProviderConfig": {
        "command": "argocd-k8s-auth",
        "args": ["gcp"],
        "apiVersion": "client.authentication.k8s.io/v1beta1"
      },
      "tlsClientConfig": {
        "insecure": false,
        "caData": ""
      }
    }
```

Now, should be able to select the private GKE cluster in ArgoCD's UI.

### Read more:
- [Connect Gateway and ArgoCD: Deploy to Distributed Kubernetes](https://cloud.google.com/blog/products/containers-kubernetes/connect-gateway-with-argocd)





