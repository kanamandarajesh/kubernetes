### Horizontal Pod Autoscaler (HPA) Setup in Kubernetes

The **Horizontal Pod Autoscaler (HPA)** automatically scales the number of pods in a deployment, replica set, or stateful set based on observed CPU utilization or custom metrics. To use HPA, you need to:

1. Ensure your cluster is running a **metrics server** (for resource-based autoscaling like CPU and memory).
2. Create an HPA resource that specifies how to scale your deployment based on CPU, memory, or custom metrics.

### Prerequisites

1. **Metrics Server**: The HPA relies on the metrics server to gather resource usage data for scaling decisions (like CPU and memory usage).

   If the **metrics-server** is not installed, you can install it using:

   ```bash
   kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
   ```

   After installing, you can check if the metrics server is working:

   ```bash
   kubectl get deployment metrics-server -n kube-system
   ```

   You should see the `metrics-server` pod running.

### Step 1: Create a Deployment

HPA typically works with a **Deployment**. Below is an example deployment YAML (`deployment.yml`).

#### Example: `deployment.yml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  labels:
    app: my-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: my-app-container
        image: nginx:latest
        resources:
          requests:
            cpu: "250m"
            memory: "256Mi"
          limits:
            cpu: "500m"
            memory: "512Mi"
```

- **`replicas: 1`**: The initial number of replicas is set to 1.
- **`resources`**: `requests` and `limits` for CPU and memory are defined, which is important for HPA to function based on resource usage.

### Step 2: Create the Horizontal Pod Autoscaler (HPA)

You can create an HPA based on **CPU** utilization (or other metrics). Below is an example of an HPA YAML that scales the `my-app` deployment based on CPU usage.

#### Example: `hpa.yml`

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-app  # This should match the deployment name
  minReplicas: 1
  maxReplicas: 5
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 50  # Target CPU utilization (50%)
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 60  # Target memory utilization (60%)
```

### Key Fields:
- **`scaleTargetRef`**: Points to the deployment that HPA will scale.
  - `apiVersion`: The Kubernetes API version for the target.
  - `kind`: The type of resource to scale (in this case, `Deployment`).
  - `name`: The name of the deployment to scale.
  
- **`minReplicas` and `maxReplicas`**: Defines the minimum and maximum number of replicas for the deployment. The HPA will scale between these values based on resource usage.

- **`metrics`**: Defines the metrics on which the scaling decision will be made.
  - **`cpu`**: This example uses CPU utilization to scale. You can also use memory or custom metrics.
  - **`averageUtilization`**: The target percentage of CPU or memory utilization.

### Step 3: Apply the Deployment and HPA

1. Apply the deployment YAML to create the `my-app` deployment:

   ```bash
   kubectl apply -f deployment.yml
   ```

2. Apply the HPA YAML to create the Horizontal Pod Autoscaler:

   ```bash
   kubectl apply -f hpa.yml
   ```

### Step 4: Verify HPA

You can check the status of the HPA using:

```bash
kubectl get hpa my-hpa
```

This will show the current number of replicas and the observed CPU/memory utilization:

```
NAME      REFERENCE                    TARGETS         MINPODS   MAXPODS   REPLICAS   AGE
my-hpa    Deployment/my-app   50%/50% (cpu), 60%/60% (memory)    1         5         1         1m
```

### Step 5: Testing the Autoscaler

You can test the scaling behavior by simulating increased CPU usage. For example, you could use a tool like `stress` or `kubectl run` to simulate load on your deployment.

#### Example: Simulate CPU Load

Run a `stress` container to generate CPU load:

```bash
kubectl run -i --tty --rm stress --image=progrium/stress --restart=Never -- cpu 4
```

This will create a pod that will stress the CPU of your cluster. The HPA should trigger a scale-up based on the metrics you defined in the HPA configuration.

### Key Points:
- **Metrics Server**: The HPA relies on the metrics server for resource metrics like CPU and memory.
- **Resource Requests & Limits**: Ensure your deployment has CPU and memory `requests` and `limits` set, as HPA uses these to make scaling decisions.
- **API Version**: Use `autoscaling/v2` for more advanced metrics scaling (including multiple metrics, like CPU and memory).

### Advanced HPA (Using Custom Metrics)

If you want to scale based on **custom metrics** (e.g., queue length, response time), you need to install a custom metrics adapter (like Prometheus Adapter) and then reference these metrics in your HPA.

---
