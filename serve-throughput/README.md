## 🔁 OSS vs Turbo: Ray Serve Throughput Benchmark

### 📦 Benchmark detail

This benchmark runs a recommendation model with 2 Ray Serve deployments - the Ingress Deployment that receives the request and generates candidates, the Ranker Deployment that runs a Deep Learning Recommendation Model on a GPU. The throughput measurement is run using locust - locust file is provided.

### Cluster Specs

- **1 head node** — 8 CPUs, 32 GB RAM  
- **Worker Group 1: 1 g6e.12xlarge nodes (g2-standard-48 in GCP)**

### 📊 Benchmark Results: Turbo vs OSS 

To reproduce the numbers you need to:
1. Build an image with the python requirements and environment variables - The Dockerfile is provided that works with both RayTurbo and OSS Ray.
2. Deploy a service through KubeRay or Anyscale (eg. `anyscale service deploy app:recsys_app --name my-test-service`). 
2. Run locust on your client side. Make sure that your client node has sufficient CPUs and you run enough locust users to saturate the throughput. Example locust command - `locust --headless --host <HOST_URL> -r 800 -u 800 --proc 30 -t 2m`

Keep a 1:2 ratio between replicas of the Ranker and Ingress Deployment

| Number of GPU replicas | Ray Turbo (req/s) | Ray OSS (req/s) | Latency (p50) Ray Turbo | Latency (p50) OSS | Latency (p90) Ray Turbo | Latency (p90) OSS | Locust users RT | Locust users OSS | 
|--------|--------------------|--------------------|--------------------|--------------------|--------------------|--------------------|--------------------|--------------------|
| 1  | 1920             | 570              | 100              | 130              | 120              | 190              | 200              | 75              | 
| 2  | 3540              | 850              | 110              | 110              | 150              | 180              | 400              | 100              | 
| 4  | 6300              | 900              | 110              |  120             |  180             | 190              | 750             | 100              | 

---

### Reproduce with RayService on KubeRay

1. Build an image as above.
2. Deploy a Kubernetes cluster with GPUs available. The example rayservice.yaml
   is configured for GKE machine types. If you are using GKE, you can create
   your cluster as follows:
   ```
   gcloud container clusters create my-benchmark-cluster \
     --location us-central2-b \  # Important: Choose a location with L4 available: https://docs.cloud.google.com/compute/docs/regions-zones/gpu-regions-zones
     --enable-ray-operator  # Optional, you may install kuberay with helm as well.

   gcloud container node-pools create gpu-pool \
     --cluster my-benchmark-cluster \
     --location us-central2-b \
     --machine-type g2-standard-48 \
     --accelerator type=nvidia-l4,count=4,gpu-driver-version=latest \
     --num-nodes 1
   ```
3. Deploy a configmap with the code. Rename the imports and create the
   configmap:
   ```
   sed -i 's/from model/from throughput.model/' app.py
   sed -i 's/from config/from throughput.config/' app.py
   kubectl create configmap serve-throughput-src \
       --from-file=app.py \
       --from-file=model.py \
       --from-file=config.py
   ```
4. Deploy the RayService. Remember to change the image or use envsubst:
   ```
   envsubst < rayservice.yaml | kubectl apply -f -
   ```

Fetch the status with `kubectl get rayservice`. When it is ready, route
throughput testing traffic through the service created (`kubectl get services`).
