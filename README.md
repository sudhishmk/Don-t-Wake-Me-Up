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

Markdown
# Verify Installation:
Ensure all pods in argo-events and tekton-pipelines are Running.

**Bash**
```bash
kubectl get pods -n argo-events
kubectl get pods -n tekton-pipelines


2. Deploy the "Broken" App
We will deploy a simple Nginx app that we pretend has a memory leak.

Bash

Bash
kubectl create deployment leaky-app --image=nginx --replicas=1
📄 Manifests
Apply the following manifests to configure the remediation loop.

1. RBAC Permissions (rbac.yaml)
Allows the remediation ServiceAccount to manage pods and allows the Sensor to trigger tasks.

YAML

YAML
apiVersion: v1
kind: ServiceAccount
metadata:
  name: remediation-sa
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-restarter
rules:
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch", "delete"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: remediation-binding
subjects:
  - kind: ServiceAccount
    name: remediation-sa
roleRef:
  kind: Role
  name: pod-restarter
  apiGroup: rbac.authorization.k8s.io
---
# Grant admin rights to the SA so the Sensor can create Tekton TaskRuns
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: sensor-admin-binding
subjects:
  - kind: ServiceAccount
    name: remediation-sa
    namespace: default
roleRef:
  kind: ClusterRole
  name: admin
  apiGroup: rbac.authorization.k8s.io
2. The Remediation Runbook (remediation-task.yaml)
The Tekton Task that performs the actual fix (deleting the pod).

YAML

YAML
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: restart-deployment
spec:
  params:
    - name: app-label
      description: "The label value of the app to restart"
  steps:
    - name: restart
      image: bitnami/kubectl:latest
      command: ["/bin/sh", "-c"]
      args:
        - |
          echo "🚨 Alert Received! Restarting application: $(params.app-label)..."
          kubectl delete pods -l app=$(params.app-label)
          echo "✅ Remediation Complete."
3. The Event Bus (eventbus.yaml)
Required for Argo Events internal communication.

YAML

YAML
apiVersion: argoproj.io/v1alpha1
kind: EventBus
metadata:
  name: default
spec:
  nats:
    native:
      auth: token
4. The Event Source (webhook-source.yaml)
Listens for HTTP POST requests (simulating an alert).

YAML

YAML
apiVersion: argoproj.io/v1alpha1
kind: EventSource
metadata:
  name: webhook
spec:
  service:
    ports:
      - port: 12000
        targetPort: 12000
  webhook:
    remediate:
      port: "12000"
      method: "POST"
      endpoint: "/remediate"
5. The Sensor (sensor.yaml)
Connects the EventSource to the Tekton Task.

YAML

YAML
apiVersion: argoproj.io/v1alpha1
kind: Sensor
metadata:
  name: remediation-sensor
spec:
  template:
    serviceAccountName: remediation-sa
  dependencies:
    - name: test-dep
      eventSourceName: webhook
      eventName: remediate
  triggers:
    - template:
        name: trigger-tekton
        k8s:
          operation: create
          source:
            resource:
              apiVersion: tekton.dev/v1beta1
              kind: TaskRun
              metadata:
                generateName: auto-remediate-
              spec:
                serviceAccountName: remediation-sa
                taskRef:
                  name: restart-deployment
                params:
                  - name: app-label
                    value: "leaky-app"
📥 Apply All Manifests
Bash

Bash
kubectl apply -f rbac.yaml
kubectl apply -f remediation-task.yaml
kubectl apply -f eventbus.yaml
kubectl apply -f webhook-source.yaml
kubectl apply -f sensor.yaml
🎬 Running the Demo
1. Wait for Components
Ensure the EventBus, EventSource, and Sensor pods are running.

Bash

Bash
kubectl get pods
Note: If the Sensor or EventSource pods are in CrashLoopBackOff, delete them once the EventBus is Running to restart them.

2. Port Forward the Listener
Open a tunnel to the EventSource so you can trigger it from your laptop.

Bash

Bash
kubectl port-forward $(kubectl get pod -l eventsource-name=webhook -o name) 12000:12000
3. Watch the Magic
Open a new terminal to watch the pods react.

Bash

Bash
watch kubectl get pods
4. Trigger the Incident
In a third terminal, send the mock alert signal.

Bash

Bash
curl -d '{"message":"Memory High"}' -H "Content-Type: application/json" -X POST http://localhost:12000/remediate
Expected Result:

A new auto-remediate-xxxxx TaskRun pod appears.

The leaky-app pod is terminated and restarted.

The output of the TaskRun logs will show: ✅ Remediation Complete.

🐛 Troubleshooting
"Error: namespaces argo-events not found"

Run kubectl create ns argo-events manually before installing the manifest.

"Sensor / EventSource in CrashLoopBackOff"

This is a race condition where they tried to start before the NATS EventBus was ready.

Fix: kubectl delete pod -l sensor-name=remediation-sensor and kubectl delete pod -l eventsource-name=webhook. They will recreate instantly and connect.


Would you like me to generate a diagram of how these components (EventSource -> Sensor -> Tekton
