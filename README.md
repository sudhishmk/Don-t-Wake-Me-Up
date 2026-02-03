# Don't Wake Me Up: Automated Incident Response with Argo Events & Tekton

This repository contains the Proof of Concept (PoC) for the talk **"Don't Wake Me Up: Automating Incident Response with Argo Events and Tekton"**.

It demonstrates an **"Ops-as-Code"** architecture where a production alert (simulated via Webhook) automatically triggers a remediation workflow (Tekton Pipeline) to fix a broken application without human intervention.

## 🏗 Architecture

`Alert (Prometheus/Curl)` → `Argo Events (Sensor)` → `Tekton (Task)` → `Kubernetes (Restart Pod)`

## 🚀 Prerequisites

* **Kubernetes Cluster** (Minikube, Kind, or OpenShift)
* **kubectl** CLI installed
* **tkn** CLI installed (optional, for debugging)

## 🛠️ Installation & Setup

### 1. Install Controllers
First, install the required operators for Tekton and Argo Events.

```bash
# Install Tekton Pipelines
kubectl apply -f [https://storage.googleapis.com/tekton-releases/pipeline/latest/release.yaml](https://storage.googleapis.com/tekton-releases/pipeline/latest/release.yaml)

# Install Argo Events (Namespace first to avoid race conditions)
kubectl create namespace argo-events
kubectl apply -f [https://raw.githubusercontent.com/argoproj/argo-events/stable/manifests/install.yaml](https://raw.githubusercontent.com/argoproj/argo-events/stable/manifests/install.yaml)
