# GitOps Repository for Kubernetes Applications

## Goal of This Repository

This repository implements the Argo CD **"App of Apps"** pattern to manage the deployment of Kubernetes applications. It contains both the parent "App of Apps" Helm chart (`apps`) and the deployable application charts themselves (starting with `workprofile`).

The primary goal is to provide a single, version-controlled source of truth that defines:

1.  **What applications should exist** on the cluster (managed by the `apps` chart).
2.  **How each application is configured** and deployed (managed by the `workprofile` chart and its values).

## The Deployment Pattern and Workflow

The entire system is bootstrapped via Infrastructure as Code and managed through GitOps.

1.  **Bootstrapping with Terraform:** A single "root" Argo CD `Application` resource is initially created using the **Terraform ArgoCD Provider**. This root application is configured to point to the `apps` chart within this repository.
2.  **Parent Chart Sync:** Argo CD syncs the `apps` chart. This does not create any application resources like `Deployments` or `Services`. Instead, its sole purpose is to render and apply the templates inside `apps/templates`, which are Argo CD `Application` manifests.
3.  **Child Application Creation:** This sync creates the child `Application` resources in the cluster, such as `workprofile-staging` and `workprofile-prod`.
4.  **Child Chart Sync:** These newly created `Application` resources, in turn, point to the `workprofile` chart *within this same repository*. Each specifies a different values file (`values-prod.yaml` or `values-staging.yaml`) to ensure the correct configuration for its environment.
5.  **Application Deployment:** Finally, Argo CD syncs these child applications, which renders the `workprofile` Helm chart with the correct environment values and deploys the 3-tier application (Nginx, Python, MySQL) into the target namespace.

## Repository Structure

The repository is organized into two main parts: the parent `apps` chart and the child `workprofile` chart.

```
.
├── apps
│   ├── Chart.yaml                  # Helm metadata for the parent 'apps' chart.
│   ├── templates
│   │   ├── namespaces.yaml         # Template to create namespaces like 'staging', 'production'.
│   │   ├── workprofile-prod.yaml   # Argo CD Application manifest for the production environment.
│   │   └── workprofile-staging.yaml# Argo CD Application manifest for the staging environment.
│   └── values.yaml                 # Defines configuration for the child applications (repo URL, path, etc.).
└── workprofile
    ├── .helmignore                 # Specifies files to exclude when packaging the workprofile chart.
    ├── Chart.yaml                  # Helm metadata for the child 'workprofile' application chart.
    ├── README.md                   # Documentation specific to the workprofile application.
    ├── charts/                     # Directory for sub-charts (currently empty).
    ├── templates
    │   ├── NOTES.txt               # Post-deployment notes shown after a Helm install.
    │   ├── _helpers.tpl            # Helm helper templates and functions.
    │   ├── app-config.yaml         # ConfigMap for the Python application tier.
    │   ├── app-deployment.yaml     # Kubernetes Deployment for the Python application tier.
    │   ├── app-service.yaml        # Kubernetes Service for the Python application tier.
    │   ├── db-service.yaml         # Kubernetes Service for the MySQL database tier.
    │   ├── db-sts.yaml             # Kubernetes StatefulSet for the MySQL database tier.
    │   ├── hpa.yaml                # HorizontalPodAutoscaler for the application deployments.
    │   ├── ingress.yaml            # Kubernetes Ingress to expose the web tier to external traffic.
    │   ├── initdb-config.yaml      # ConfigMap for initial MySQL database setup scripts.
    │   ├── sealed-secret.yaml      # Encrypted secrets for the application.
    │   ├── serviceaccount.yaml     # Kubernetes ServiceAccount for the application pods.
    │   ├── tests
    │   │   └── test-connection.yaml# Helm test to verify connectivity after deployment.
    │   ├── web-config.yaml         # ConfigMap for the Nginx web tier.
    │   ├── web-deployment.yaml     # Kubernetes Deployment for the Nginx web tier.
    │   └── web-service.yaml        # Kubernetes Service for the Nginx web tier.
    ├── values-prod.yaml            # Environment-specific overrides for Production.
    ├── values-staging.yaml         # Environment-specific overrides for Staging.
    └── values.yaml                 # Default shared configuration for the workprofile application.
```


## The Cluster Architecture
![Kubernetes cluster architecture](./docs/images/Develeap%20-%20Portfolio%20-%20Architecture-Kubernetes%20Deployment.jpg)

## The GitOps Update Cycle

Ongoing deployments are fully automated and triggered by commits to this repository.

1.  A CI pipeline for the `workprofile` application source code builds a new Docker image.
2.  Upon success, the CI pipeline automatically commits a change to this repository, updating the `image.tag` value within `workprofile/values-staging.yaml` or `workprofile/values-prod.yaml`.
3.  The pipeline commits and pushes this change.
4.  Argo CD, which is constantly monitoring this repository, detects the commit. It recognizes that the desired state in Git no longer matches the live state for that specific application (`workprofile-staging` or `workprofile-prod`).
5.  Argo CD automatically re-syncs the relevant child application, which triggers a rolling update of the pods to the new image version.

## Cluster Prerequisites

For this system to function, the target Kubernetes cluster must have the following controllers installed and configured:

1.  **Argo CD:** Manages the GitOps workflow and application synchronization.
2.  **Sealed Secrets Controller:** Must be running with the **pre-existing private key** that was used to encrypt the `sealed-secret.yaml` file. The system will fail if the cluster cannot decrypt the secrets required by the `workprofile` application.
