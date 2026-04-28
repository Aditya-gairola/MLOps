# MLOps Step 4: Deploying the Model Using KServe

## Setup Architecture
Our setup contains the following.

 1. The Flask Frontend App: Deployed as a standard Kubernetes Deployment

 2. The Inference Deployment: Deployed via KServe


<img width="1793" height="1647" alt="image" src="https://github.com/user-attachments/assets/ccb854c5-08ad-4665-8c6c-6f0f6b0a40ee" />


## Inference API (FastAPI + model.pkl)
model.pkl is just a file on disk. On its own, it can only be called from a Python script like the predict.py script . No HTTP, no JSON, no way for any other application to talk to it.

To make it callable over the network from a frontend, we need to wrap it with a service that exposes an HTTP endpoint.. So we use FastAPI for that.

The inference API has three key endpoints as shown below. /health and /ready are not optional, KServe uses them to manage pod lifecycle and traffic routing. /predict is the actual inference call.


<img width="2094" height="2207" alt="image" src="https://github.com/user-attachments/assets/b3706453-eeb4-40ae-80dc-c261d9bae81b" />


---

> 💡 key Insight: Model Loading
>
> The model loads at startup and stays in memory. Every request reuses the same in-memory object and it doesn't read model.pkl from disk on every call. 
>
> This is the same pattern used in production model serving systems. Models are loaded during service startup and reused for inference to ensure low latency and efficient           > performance.

## The Frontend App  (Flask)
The Flask app is the UI layer. It renders the HTML form, takes the HR person's input, POSTs it to the KServe inference endpoint, and displays the result. 

It has no model logic inside it,  just a form and an HTTP call call to the inference service.


> If you have ever deployed a Node.js or Go web app on Kubernetes, this is no different. Flask handles routing and template rendering. The model prediction is just an HTTP call to > another service. The same pattern as any microservice architecture.

## What Is KServe and Why Not Just Use a Deployment?
Here is the key question every DevOps engineer asks. 

You already know how to deploy a container on Kubernetes. Why not just write a Deployment + Service + Ingress and be done with it?

Well you could. But KServe gives you things that a basic Deployment doesn't. 

KServe is a Kubernetes operator purpose-built for ML model serving. It handles operational concerns like scaling, traffic management, and observability.


<img width="1567" height="2072" alt="image" src="https://github.com/user-attachments/assets/cf4d16da-08e1-4fba-995c-a8d5ae11ba95" />

---

Think of KServe as an Ingress Controller, but for ML models. Just like Nginx Ingress handles HTTP routing rules via Ingress objects, KServe handles model serving via InferenceService objects. 

The following image illustrates how Kserve fits in to our workflow.


<img width="1060" height="906" alt="image" src="https://github.com/user-attachments/assets/8f5833bc-afc9-437d-b1ff-16bf3f52f89b" />

---

> If you have worked with operators before like Prometheus Operator, Argo CD etc, KServe follows the same pattern. It installs a custom resource definition (CRD) into your cluster. > You create an InferenceService object. The KServe controller watches for it and provisions the underlying Pods, Services, and routing rules automatically.

Now lets implement everything.

## Dockerize The Model  
First, we need to containerize model.pkl with all the dependencies it requires for predictions like scikit-learn, numpy, and the predictor logic that loads and calls it

The Dockerfile is inside the phase-1-local-dev/inference folder. It bakes model.pkl directly into the image at build time. The image is the versioned model artifact (1.0.0).


<img width="2562" height="1895" alt="image" src="https://github.com/user-attachments/assets/90488054-6117-4939-b8ea-61a491cba033" />



For this local setup, i push the images to Docker Hub so my local Kubernetes cluster can pull it. 

My Dockerfile is inside phase-1-local-dev/inference folder. Now running the following commands from the phase-1-local-dev directory to build the image.

```
$ cd inference 

$ docker build -t <your-dockerhub-username>/attrition-inference:1.0.0 .

$ docker push <your-dockerhub-username>/attrition-inference:1.0.0
```

---
<img width="1047" height="563" alt="Screenshot from 2026-04-28 10-34-15" src="https://github.com/user-attachments/assets/45effcf0-3163-420f-93ea-195e53dfe530" />



<img width="935" height="221" alt="Screenshot from 2026-04-28 10-38-33" src="https://github.com/user-attachments/assets/1a42d7ab-0c52-43c7-8d09-1fe972d8a6d5" />

---

model.pkl is baked into the image at build time. The image is the versioned artifact. v1.0.0 always contains exactly the model trained in Step 3. So no surprises at runtime.

## Dockerize The Frontend 
Now, Dockerize the frontend.

My frontend codes is inside phase-1-local-dev/frontend folder. Running the following commands from the phase-1-local-dev folder to build the image

```
$ cd frontend

$ docker build -t <your-dockerhub-username>/attrition-frontend:1.0.0 .

$ docker push <your-dockerhub-username>/attrition-frontend:1.0.0
```

---

<img width="1008" height="557" alt="Screenshot from 2026-04-28 10-44-47" src="https://github.com/user-attachments/assets/34fdafd8-9b05-4ea1-a748-dd5258689bb2" />

---

## Deploy KServe
To quickly install Kserve on my cluster using Helm, running the following commands. Cert-manager is also mandatory for Kserve, so  will install that as well.

```
curl -sfL https://get.rke2.io | sudo sh -
sudo systemctl enable rke2-server
sudo systemctl start rke2-server
```
```
mkdir -p ~/.kube
sudo cp /etc/rancher/rke2/rke2.yaml ~/.kube/config
sudo chown $USER:$USER ~/.kube/config
export KUBECONFIG=~/.kube/config
echo 'export KUBECONFIG=~/.kube/config' >> ~/.bashrc
source ~/.bashrc
```
```
# Install cert-manager

$ kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.19.0/cert-manager.yaml

# Install kserve in standard mode

$ helm install kserve-crd oci://ghcr.io/kserve/charts/kserve-crd --version v0.16.0 -n kserve --create-namespace

# Install KServe (RKE2-safe method)
# First pass (creates webhook/controller)
helm install kserve oci://ghcr.io/kserve/charts/kserve \
--version v0.16.0 \
-n kserve \
--set kserve.controller.deploymentMode=Standard \
--set kserve.servingruntime.enabled=false \
--set kserve.clusterServingRuntime.enabled=false || true

# Wait until pod is healthy:

kubectl wait --for=condition=Ready pod -l control-plane=kserve-controller-manager -n kserve --timeout=300s
# Second pass (finishes install)
helm upgrade --install kserve oci://ghcr.io/kserve/charts/kserve \
--version v0.16.0 \
-n kserve \
--set kserve.controller.deploymentMode=Standard \
--set kserve.servingruntime.enabled=false \
--set kserve.clusterServingRuntime.enabled=false \
--wait \
--timeout 10m
```

---
<img width="999" height="452" alt="Screenshot from 2026-04-28 11-07-31" src="https://github.com/user-attachments/assets/38bd644f-149c-4e15-8f1c-7ff6364d94aa" />


<img width="900" height="341" alt="Screenshot from 2026-04-28 11-22-09" src="https://github.com/user-attachments/assets/dad605d8-c4ad-4acc-9505-bf62aa8cefc2" />

---


## Deploy The Kserve InferenceService
This is the only Kubernetes resource  need for the inference container. One YAML. The InferenceService CRD creates a Deployment, Service, HPA, and routing config. 

 manifest file to create an inference is inside phase-1-local-dev/k8s/inference.yaml.

 <img width="793" height="408" alt="Screenshot from 2026-04-28 11-53-22" src="https://github.com/user-attachments/assets/cd6019b7-b86a-449d-b3d3-b3a50d9b4501" />




 To deploy the inference API, running the following command inside phase-1-local-dev/k8s directory.

```
$ cd k8s

$ kubectl apply -f inference.yaml

$ kubectl get deploy,svc
```
<img width="611" height="98" alt="Screenshot from 2026-04-28 12-52-12" src="https://github.com/user-attachments/assets/1975edf0-fc85-4c0d-b349-8f1e05ad8327" />



And, i can see pod up and running in the default namespace. 

Also, it exposes the inference service at employee-attrition-predictor.default.svc.cluster. The frontend application uses this service endpoint internally using INFERENCE_URL  environment variable to call the inference service.

## Deploy the Frontend
Now deploying the frontend app.

The frontend is a standard Flask web app. It serves the HTML form, takes the HR person's input, calls the inference endpoint and shows the result. It has no knowledge of the model, just an HTTP URL it POSTs to.

It reads INFERENCE_URL from an environment variable at runtime. I set this in the Deployment manifest.

frontend deployment file is inside  phase-1-local-dev/k8s/deployment.yaml.

To deploy the inference, running the following command from inside phase-1-local-dev/k8s directory.
```
$ cd k8s

$ kubectl apply -f deployment.yaml
```

<img width="692" height="136" alt="Screenshot from 2026-04-28 13-14-23" src="https://github.com/user-attachments/assets/6f36e777-a896-45b2-9d3c-93162e386249" />



Now, if i list the pods,  can see an additional pod running in the default namespace. 

It also creates a NodePort service on port 31010. So that i can access the UI.

## Test the Deployments
To test the deployments, accessing the UI using the URL http://localhost:31010 as shown below.

<img width="1280" height="696" alt="Screenshot from 2026-04-28 13-16-57" src="https://github.com/user-attachments/assets/f95fa562-ccae-4ece-adb9-5ee43f5deec0" />

---

Entering the values and clicking the Predict Attrition button. I got the prediction values as follows.


<img width="793" height="310" alt="image" src="https://github.com/user-attachments/assets/0384a404-9da7-411f-86e2-83339b35ddb1" />



## Real World Context: What If the Model Is 15 GB?
Baking a 15 GB model into a Docker image and loading it entirely into memory is not realistic. A 15 GB image takes minutes to pull, burns memory on every pod replica, and makes scaling painful.

In production, large models are stored separately in a model registry or object storage (S3, GCS), and the container pulls just the model weights it needs at startup, not the entire file baked into the image. KServe supports this natively.

For very large models (LLMs, foundation models), you go further. Model sharding across multiple GPUs and dedicated serving runtimes like vLLM or Triton Inference Server instead of a plain FastAPI app.

For our employee attrition model, the file is a few KBs. Baking it into the image is perfectly fine and keeps things simple for Phase 1.

## Completely Remove the RKE2 Cluster
1. Stop RKE2 server
   
```sudo systemctl stop rke2-server```

3. Disable auto-start on boot

```sudo systemctl disable rke2-server```

4. Run official uninstall script

```sudo /usr/bin/rke2-uninstall.sh```


This usually removes:

RKE2 binaries
systemd services
kubelet/containerd managed by RKE2
cluster state
manifests
4. Optional Cleanup (Recommended)

Remove kubeconfig and leftovers:

```
rm -rf ~/.kube
sudo rm -rf /etc/rancher/rke2
sudo rm -rf /var/lib/rancher/rke2
sudo rm -rf /var/lib/kubelet
sudo rm -rf /run/k3s
sudo rm -rf /var/lib/cni
sudo rm -rf /etc/cni
```
5. Verify Removed
   
```kubectl get nodes```





