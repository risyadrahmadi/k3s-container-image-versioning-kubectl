# K3s Container Image Versioning with kubectl

Deploying versioned Docker images to a Kubernetes K3s cluster using GitLab CI/CD, GitLab Container Registry, and `kubectl`.

## Overview

This project implements application deployment using **Kubernetes K3s** as the container orchestration platform.

Unlike the previous Docker Compose approach, where containers are managed through a Compose file, this implementation uses Kubernetes resources such as:

* Deployment
* Pod
* Service
* Secret

The build and deployment processes are separated into different CI/CD responsibilities.

The Docker image is built using a GitLab Runner with a Docker executor. The deployment process uses a separate Kubernetes runner with access to the K3s cluster and executes Kubernetes commands through `kubectl`.

The deployment process starts when source code changes are committed to GitLab and a Git Tag is created. GitLab CI/CD uses the value of `CI_COMMIT_TAG` as the Docker image version.

The image is then pushed to the GitLab Container Registry. After the image is available, the Kubernetes deployment is applied to the K3s cluster using the corresponding image version.

---

## Objectives

The main objectives of this implementation are:

* Build Docker images automatically using GitLab CI/CD.
* Use Git Tags for Docker image versioning.
* Store images in GitLab Container Registry.
* Deploy applications to Kubernetes K3s.
* Separate image building from Kubernetes deployment.
* Use `kubectl` to manage Kubernetes resources.
* Authenticate K3s against a private GitLab Container Registry.
* Verify Deployment, Pod, Service, and image versions after deployment.
* Perform controlled application updates using Kubernetes rollout.

---

## K3s Environment

The Kubernetes cluster used in this implementation runs on a server with the following example configuration:

| Component        | Example             |
| ---------------- | ------------------- |
| Hostname         | `k3s-vm`            |
| Operating System | Debian GNU/Linux 13 |
| IP Address       | `192.168.1.100`     |
| K3s Version      | `v1.36.4+k3s1`      |
| Kubernetes CLI   | `kubectl`           |

> The IP address above is an example and should be replaced with the address of the target K3s server.

Basic environment information can be checked with:

```bash
hostname
hostname -I
kubectl version
```

The K3s node can be checked using:

```bash
sudo kubectl get nodes
```

Example:

```text
NAME      STATUS   ROLES
k3s-vm    Ready    control-plane
```

The node should be in the `Ready` state before deploying workloads.

---

## Inspect Kubernetes Pods

All Pods across namespaces can be inspected using:

```bash
sudo kubectl get pods -A
```

Pods in the default namespace can be checked with:

```bash
sudo kubectl get pods -n default
```

These commands help verify that Kubernetes resources are being created and managed correctly.

---

## GitLab Runner Architecture

Two different GitLab Runners are used for the CI/CD process.

### Docker Runner

The Docker runner is responsible for building and pushing Docker images.

Example runner:

```text
runner-docker
```

Example tag:

```text
docker
```

The build process uses the Dockerfile from the repository:

```bash
docker build \
  -t "${CI_REGISTRY_IMAGE}:${CI_COMMIT_TAG}" .
```

After the image is successfully built, it is pushed to GitLab Container Registry:

```bash
docker push "${CI_REGISTRY_IMAGE}:${CI_COMMIT_TAG}"
```

Using `CI_COMMIT_TAG` ensures that the Docker image version follows the Git Tag.

For example:

```text
Git Tag:
v0.1.6

Docker Image:
registry.gitlab.com/k3s/franken-php:v0.1.6
```

### K3s Runner

The Kubernetes deployment is handled by a separate runner:

```text
runner-k3s
```

This runner is responsible for executing Kubernetes commands such as:

```bash
kubectl apply
kubectl rollout status
kubectl get deployments
kubectl get pods
kubectl get services
```

Separating the runners provides a clearer CI/CD responsibility:

* `runner-docker` → build and push Docker images
* `runner-k3s` → deploy and verify Kubernetes resources

---

## GitLab Container Registry Authentication

The Docker image is stored in the GitLab Container Registry.

If the registry is private, Kubernetes requires credentials to pull the image.

The registry credentials are stored as a Kubernetes Secret:

```text
gitlab-registry
```

The Secret can be checked using:

```bash
sudo kubectl -n default get secret gitlab-registry
```

To inspect the Secret type:

```bash
sudo kubectl -n default get secret gitlab-registry \
  -o jsonpath="{.type}"
```

Expected result:

```text
kubernetes.io/dockerconfigjson
```

The Secret is referenced by the Pod through `imagePullSecrets`:

```yaml
spec:
  imagePullSecrets:
    - name: gitlab-registry
```

This allows Kubernetes to authenticate against the GitLab Container Registry when pulling a private Docker image.

---

## Kubernetes Deployment

The application is deployed using a Kubernetes YAML manifest.

Example file:

```text
yaml/deployment.yaml
```

A simplified Deployment configuration for the FrankenPHP application is:

```yaml
apiVersion: apps/v1
kind: Deployment

metadata:
  name: franken-php

spec:
  replicas: 1

  selector:
    matchLabels:
      app: franken-php

  template:
    metadata:
      labels:
        app: franken-php

    spec:
      imagePullSecrets:
        - name: gitlab-registry

      containers:
        - name: franken-php
          image: registry.gitlab.com/k3s/franken-php:{{VERSION}}
```

The `{{VERSION}}` placeholder is used to dynamically define the Docker image version during the CI/CD process.

This allows the same Kubernetes manifest to be reused for multiple image versions.

---

## Replacing the Image Version

During deployment, the `{{VERSION}}` placeholder can be replaced using the GitLab `CI_COMMIT_TAG` variable.

Example:

```bash
sed -e "s|{{VERSION}}|${CI_COMMIT_TAG}|g" \
  yaml/deployment.yaml | \
  kubectl apply -n default -f -
```

If the pipeline is triggered by:

```text
v0.1.6
```

the placeholder:

```text
{{VERSION}}
```

becomes:

```text
v0.1.6
```

The resulting image becomes:

```text
registry.gitlab.com/k3s/franken-php:v0.1.6
```

This approach avoids manually modifying the Deployment manifest whenever a new image version is released.

---

## Apply Kubernetes Configuration

A Deployment manifest containing the correct image version can be applied using:

```bash
sudo kubectl apply -n default -f yaml/deployment.yaml
```

`kubectl apply` creates the resource if it does not exist or updates the existing resource when its configuration changes.

When the image version changes, Kubernetes creates a new ReplicaSet and performs a rollout to replace the previous Pod.

---

## Monitor Deployment Rollout

After applying the Deployment, its rollout status should be checked:

```bash
sudo kubectl rollout status deployment/franken-php \
  -n default \
  --timeout=120s
```

A successful rollout indicates that the new ReplicaSet has become available and the application has been updated successfully.

---

## Verify Deployment

The Deployment status can be checked with:

```bash
sudo kubectl get deployments -n default
```

Example:

```text
NAME          READY   UP-TO-DATE   AVAILABLE
franken-php   1/1     1            1
```

The following fields are useful for verification:

* `READY` — number of ready Pods.
* `UP-TO-DATE` — Pods using the latest Deployment configuration.
* `AVAILABLE` — Pods available to serve the application.

---

## Verify Pods

The application Pods can be inspected using:

```bash
sudo kubectl get pods -n default -o wide
```

The Pod should normally show:

```text
Running
```

The `-o wide` option also displays additional information such as the Pod IP and the node where the Pod is running.

Checking the Pod is important because a successful Deployment resource does not necessarily mean that the application container started successfully.

---

## Verify Service

Kubernetes Services can be checked with:

```bash
sudo kubectl get services -n default
```

A Service provides a stable network endpoint for communicating with application Pods.

Instead of connecting directly to a Pod IP, other applications can communicate through the Service. This is important because Pod IP addresses can change when Pods are recreated.

---

## Verify the Docker Image

The image currently used by the Pod can be inspected using:

```bash
sudo kubectl get pods \
  -n default \
  -o jsonpath="{.items[*].spec.containers[*].image}"
```

This command displays the image configured for the containers running in the Pods.

For example:

```text
registry.gitlab.com/k3s/franken-php:v0.1.6
```

This verification confirms that the Kubernetes Deployment is using the expected image version.

---

## Image Version Update

Image versioning becomes particularly useful when a new application version is released.

For example, the previous application version is:

```text
v0.1.5
```

A new source code release is then created using:

```text
v0.1.6
```

The CI/CD process builds the new image:

```text
registry.gitlab.com/k3s/franken-php:v0.1.6
```

and pushes it to the GitLab Container Registry.

The deployment pipeline then replaces the image version in the Kubernetes manifest and applies the updated configuration.

Kubernetes performs a rollout to replace the Pod using the previous image with a Pod using the new image.

The rollout can be monitored using:

```bash
sudo kubectl rollout status deployment/franken-php \
  -n default \
  --timeout=120s
```

This makes application updates traceable through Git Tags and Docker image versions.

---

## Example Versioning Workflow

A typical release process is:

### 1. Create a Git Tag

```bash
git tag v0.1.6
```

### 2. Push the Tag

```bash
git push origin v0.1.6
```

### 3. Build the Image

GitLab CI/CD uses:

```bash
docker build \
  -t "${CI_REGISTRY_IMAGE}:${CI_COMMIT_TAG}" .
```

### 4. Push the Image

```bash
docker push "${CI_REGISTRY_IMAGE}:${CI_COMMIT_TAG}"
```

### 5. Deploy to K3s

The Kubernetes runner replaces the image version and applies the manifest:

```bash
sed -e "s|{{VERSION}}|${CI_COMMIT_TAG}|g" \
  yaml/deployment.yaml | \
  kubectl apply -n default -f -
```

### 6. Verify the Rollout

```bash
kubectl rollout status deployment/franken-php \
  -n default \
  --timeout=120s
```

### 7. Verify the Image

```bash
kubectl get pods \
  -n default \
  -o jsonpath="{.items[*].spec.containers[*].image}"
```

The final image should contain the same version as the Git Tag:

```text
registry.gitlab.com/k3s/franken-php:v0.1.6
```

---

## CI/CD Responsibility

The implementation separates the CI/CD workflow into two main responsibilities.

The **Docker runner** is responsible for building the application image and pushing it to the GitLab Container Registry.

The **K3s runner** is responsible for deploying the selected image version to Kubernetes and checking the resulting resources.

This separation prevents the image-building process from being tightly coupled with Kubernetes deployment and makes the pipeline easier to maintain.

---

## Security Considerations

This repository is intended to be public. Therefore, sensitive information must not be committed to Git.

Do not include:

* Registry passwords
* Access tokens
* Kubernetes credentials
* SSH private keys
* Real private infrastructure credentials
* Production `.env` files
* Kubernetes Secrets containing real credentials

The `gitlab-registry` Secret should be created directly in the target Kubernetes cluster or through a secure CI/CD mechanism.

Example manifests in this repository use placeholder values and example IP addresses.

---

## Result

The K3s `kubectl` deployment method successfully connects GitLab CI/CD image versioning with Kubernetes deployment.

The process starts from a Git Tag and continues through the following stages:

1. Git Tag is created.
2. GitLab CI/CD starts the pipeline.
3. Docker runner builds the image.
4. Image is tagged using `CI_COMMIT_TAG`.
5. Image is pushed to GitLab Container Registry.
6. K3s runner updates the Kubernetes image version.
7. `kubectl apply` applies the Deployment.
8. Kubernetes performs the rollout.
9. Deployment and Pod status are verified.
10. The running image version is checked.

This workflow allows application versions to be tracked consistently from source code to the running Kubernetes workload.

---

## Conclusion

This implementation demonstrates **container image versioning with Kubernetes K3s and `kubectl`** using GitLab CI/CD.

The Docker image build process is handled by `runner-docker`, while Kubernetes deployment is handled by `runner-k3s`. The Git Tag is used as the image version through the `CI_COMMIT_TAG` variable.

GitLab Container Registry provides centralized image storage, while the Kubernetes Secret `gitlab-registry` allows K3s to authenticate when pulling private images.

Compared with Docker Compose, this approach introduces Kubernetes resources such as **Deployment, Pod, Service, and Secret** for application management. Kubernetes rollout mechanisms also provide a controlled way to update application versions and verify the result of each deployment.

The next stage of this project is to move from direct `kubectl` deployment toward **ArgoCD and GitOps**, where application deployment can be managed declaratively through a Git repository.
