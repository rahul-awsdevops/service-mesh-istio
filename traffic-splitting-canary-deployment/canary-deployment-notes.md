To implement a canary deployment for your car-rental frontend application using Istio, you'll deploy a new version of the frontend application alongside the existing one, route a small percentage of traffic to the new version via the Istio Gateway and VirtualService, and continue accessing the application through the existing Application Load Balancer (ALB) DNS. The new frontend image is `676206914267.dkr.ecr.us-east-1.amazonaws.com/techcloud-academy-frontend-apps:new-features-about-page-d831d54`. Below is a step-by-step guide with all necessary code and instructions.

### Prerequisites
- Your existing setup is running fine with the provided manifest files (`rds-postgres-external-service.yml`, `car-rental-backend.yml`, `car-rental-frontend.yml`, `ingress-class.yml`, `ingress-resource.yml`).
- Istio is installed on your AWS EKS cluster (v1.30) with automatic sidecar injection enabled in the `default` namespace (as per your previous setup).
- The ALB is configured and accessible via its DNS name.
- You have `kubectl` and `istioctl` configured to interact with your EKS cluster.

### Overview of Canary Deployment
- **Objective**: Deploy the new frontend image (`new-features-about-page-d831d54`) as a canary version, initially routing 10% of traffic to it and 90% to the stable version, using the existing ALB DNS.
- **Approach**:
  - Create a new Deployment for the canary version of the frontend with a distinct `version` label (e.g., `v2`).
  - Define an Istio DestinationRule to identify the stable (`v1`) and canary (`v2`) subsets based on pod labels.
  - Configure an Istio Gateway and VirtualService to route traffic through the `istio-ingressgateway`, splitting 90% to `v1` and 10% to `v2`.
  - Update the ALB Ingress to point to the `istio-ingressgateway` service instead of the `car-rental-frontend-service`.
- **Access**: Traffic will continue to reach the application via the ALB DNS, but Istio will handle the routing internally.

### Step-by-Step Guide

#### Step 1: Create a Canary Deployment for the Frontend
Create a new Deployment for the canary version of the frontend application. This Deployment will use the new image and include a `version: v2` label to distinguish it from the stable version (`v1`).

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: car-rental-frontend-v2
spec:
  replicas: 1
  selector:
    matchLabels:
      app: car-rental-frontend
      version: v2
  template:
    metadata:
      labels:
        app: car-rental-frontend
        version: v2
    spec:
      containers:
        - name: nginx-reverse-proxy-container
          image: 676206914267.dkr.ecr.us-east-1.amazonaws.com/techcloud-academy-frontend-apps-nginx-proxy:adec382
          ports:
            - containerPort: 80
          env:
            - name: NEXTJS_APP_URL
              value: "localhost"
            - name: NEXTJS_APP_PORT
              value: "3000"
            - name: API_BACKEND_URL
              value: "car-rental-backend-service"
            - name: API_BACKEND_PORT
              value: "4000"
        - name: car-rental-frontend-container
          image: 676206914267.dkr.ecr.us-east-1.amazonaws.com/techcloud-academy-frontend-apps:new-features-about-page-d831d54
          ports:
            - containerPort: 3000
---
apiVersion: v1
kind: Service
metadata:
  name: car-rental-frontend-v2-service
spec:
  type: ClusterIP
  selector:
    app: car-rental-frontend
    version: v2
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
```

**Notes**:
- The `app: car-rental-frontend` label ensures both versions can be targeted by a single Kubernetes Service.
- The `version: v2` label distinguishes the canary pods.
- A separate Service (`car-rental-frontend-v2-service`) is created for the canary version, though it’s primarily for internal routing clarity; the VirtualService will handle traffic splitting.
- The `nginx-reverse-proxy-container` image remains unchanged, assuming it doesn’t need updating for the canary.

Apply the canary Deployment and Service:
```bash
kubectl apply -f car-rental-frontend-canary.yml
```

#### Step 2: Update the Stable Deployment
Modify the existing `car-rental-frontend.yml` to add a `version: v1` label to the stable Deployment’s pods, ensuring Istio can distinguish between the stable and canary versions.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: car-rental-frontend
spec:
  replicas: 1
  selector:
    matchLabels:
      app: car-rental-frontend
      version: v1
  template:
    metadata:
      labels:
        app: car-rental-frontend
        version: v1
    spec:
      containers:
        - name: nginx-reverse-proxy-container
          image: 676206914267.dkr.ecr.us-east-1.amazonaws.com/techcloud-academy-frontend-apps-nginx-proxy:adec382
          ports:
            - containerPort: 80
          env:
            - name: NEXTJS_APP_URL
              value: "localhost"
            - name: NEXTJS_APP_PORT
              value: "3000"
            - name: API_BACKEND_URL
              value: "car-rental-backend-service"
            - name: API_BACKEND_PORT
              value: "4000"
        - name: car-rental-frontend-container
          image: 676206914267.dkr.ecr.us-east-1.amazonaws.com/techcloud-academy-frontend-apps:adec382
          ports:
            - containerPort: 3000
---
apiVersion: v1
kind: Service
metadata:
  name: car-rental-frontend-service
spec:
  type: ClusterIP
  selector:
    app: car-rental-frontend
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
```

**Changes**:
- Added `version: v1` to the `matchLabels` and pod `labels`.
- Changed the Service `selector` to only include `app: car-rental-frontend`, allowing it to target both `v1` and `v2` pods.
- Removed the `nodePort` field since the ALB will route traffic via the Istio Gateway, not directly to the Service.

Apply the updated stable Deployment and Service:
```bash
kubectl apply -f car-rental-frontend.yml
```

#### Step 3: Create Istio DestinationRule
Define a DestinationRule to create subsets for the stable (`v1`) and canary (`v2`) versions based on the `version` label.

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: car-rental-frontend
spec:
  host: car-rental-frontend-service
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
```

**Notes**:
- The `host` points to the `car-rental-frontend-service`, which selects both `v1` and `v2` pods.
- Subsets `v1` and `v2` are defined based on the `version` label.

Apply the DestinationRule:
```bash
kubectl apply -f car-rental-destination-rule.yml
```

#### Step 4: Create Istio Gateway and VirtualService
Configure an Istio Gateway to receive traffic from the ALB and a VirtualService to split traffic (90% to `v1`, 10% to `v2`).

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: car-rental-gateway
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "*"
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: car-rental-frontend
spec:
  hosts:
  - "*"
  gateways:
  - car-rental-gateway
  http:
  - route:
    - destination:
        host: car-rental-frontend-service
        subset: v1
      weight: 90
    - destination:
        host: car-rental-frontend-service
        subset: v2
      weight: 10
```

**Notes**:
- The Gateway uses the `istio: ingressgateway` selector to target the `istio-ingressgateway` pods.
- The `hosts: "*"` allows the Gateway to accept traffic for any host, as the ALB will forward requests to the `istio-ingressgateway`.
- The VirtualService splits traffic: 90% to the `v1` subset and 10% to the `v2` subset.

Apply the Gateway and VirtualService:
```bash
kubectl apply -f car-rental-istio-gateway.yml
```

#### Step 5: Update the Ingress to Route via Istio Gateway
Modify the existing `ingress-resource.yml` to route traffic to the `istio-ingressgateway` service instead of `car-rental-frontend-service`.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: car-rental-app
  annotations:
    alb.ingress.kubernetes.io/load-balancer-name: car-rental-app
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}]'
    alb.ingress.kubernetes.io/healthcheck-protocol: HTTP
    alb.ingress.kubernetes.io/healthcheck-port: traffic-port
    alb.ingress.kubernetes.io/healthcheck-path: /
    alb.ingress.kubernetes.io/healthcheck-interval-seconds: '15'
    alb.ingress.kubernetes.io/healthcheck-timeout-seconds: '5'
    alb.ingress.kubernetes.io/success-codes: '200'
    alb.ingress.kubernetes.io/healthy-threshold-count: '2'
    alb.ingress.kubernetes.io/unhealthy-threshold-count: '2'
spec:
  ingressClassName: my-aws-ingress-class
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: istio-ingressgateway
            port:
              number: 80
```

**Changes**:
- Changed the `service.name` to `istio-ingressgateway` and `service.port.number` to `80`, directing ALB traffic to the Istio Gateway.

Apply the updated Ingress:
```bash
kubectl apply -f ingress-resource.yml
```

#### Step 6: Verify the Canary Deployment
1. **Check Pods**:
   ```bash
   kubectl get pods -l app=car-rental-frontend
   ```
   Expected output:
   ```
   NAME                                    READY   STATUS    RESTARTS   AGE
   car-rental-frontend-...                 3/3     Running   0          5m
   car-rental-frontend-v2-...              3/3     Running   0          5m
   ```
   The `3/3` READY status indicates the application containers plus the Istio sidecar.

2. **Get ALB DNS**:
   ```bash
   kubectl get ingress car-rental-app -o jsonpath='{.status.loadBalancer.ingress[0].hostname}'
   ```
   Use this DNS to access the application (e.g., `http://<ALB-DNS>`).

3. **Test Traffic Splitting**:
   Make multiple requests to the ALB DNS using `curl` or a browser:
   ```bash
   for i in {1..20}; do curl -s http://<ALB-DNS> | grep -i "version"; done
   ```
   Since your new image has an "about page" feature, check for differences (e.g., a new `/about` endpoint or UI changes). Approximately 2 out of 20 requests should hit the `v2` version, identifiable by the new feature.

4. **Monitor with Kiali** (if installed):
   ```bash
   kubectl port-forward svc/kiali -n istio-system 20001:20001
   ```
   Open `http://localhost:20001` in a browser to visualize traffic. You should see ~90% of traffic to `v1` and ~10% to `v2`.

#### Step 7: Adjust Traffic Weights (Optional)
If the canary version performs well, gradually shift more traffic to `v2`. Edit the VirtualService:
```bash
kubectl edit virtualservice car-rental-frontend
```
Update the weights, e.g., to 50% `v1` and 50% `v2`:
```yaml
http:
- route:
  - destination:
      host: car-rental-frontend-service
      subset: v1
    weight: 50
  - destination:
      host: car-rental-frontend-service
      subset: v2
    weight: 50
```
To fully transition to `v2`, set `v2` weight to 100 and `v1` to 0, then delete the `v1` Deployment and update the stable Deployment’s image.

#### Step 8: Roll Back (If Needed)
If the canary fails:
1. Shift all traffic to `v1`:
   ```bash
   kubectl edit virtualservice car-rental-frontend
   ```
   Set `v1` weight to 100 and `v2` to 0.
2. Delete the canary Deployment:
   ```bash
   kubectl delete -f car-rental-frontend-canary.yml
   ```

#### Step 9: Clean Up (After Full Transition)
Once `v2` is stable and handling 100% traffic:
1. Update the stable Deployment’s image to `new-features-about-page-d831d54` in `car-rental-frontend.yml`.
2. Delete the canary Deployment and Service:
   ```bash
   kubectl delete -f car-rental-frontend-canary.yml
   ```
3. Update the VirtualService to route all traffic to `v1` or remove the `v2` subset.
4. Delete the DestinationRule if no longer needed:
   ```bash
   kubectl delete -f car-rental-destination-rule.yml
   ```

### Additional Notes
- **Autoscaling**: If you want the canary and stable Deployments to scale independently, configure HorizontalPodAutoscalers (HPAs):
  ```yaml
  apiVersion: autoscaling/v2
  kind: HorizontalPodAutoscaler
  metadata:
    name: car-rental-frontend-v1
  spec:
    scaleTargetRef:
      apiVersion: apps/v1
      kind: Deployment
      name: car-rental-frontend
    minReplicas: 1
    maxReplicas: 5
    metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 50
  ---
  apiVersion: autoscaling/v2
  kind: HorizontalPodAutoscaler
  metadata:
    name: car-rental-frontend-v2
  spec:
    scaleTargetRef:
      apiVersion: apps/v1
      kind: Deployment
      name: car-rental-frontend-v2
    minReplicas: 1
    maxReplicas: 5
    metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 50
  ```
  Apply with `kubectl apply -f hpa.yml`.

- **Monitoring**: Use Prometheus and Grafana (installed as Istio add-ons) to monitor metrics like request latency and error rates for `v1` vs. `v2`. Access Grafana via:
  ```bash
  kubectl port-forward svc/grafana -n istio-system 3000:3000
  ```
  Open `http://localhost:3000`.

- **ALB DNS Continuity**: The ALB DNS remains unchanged because the Ingress now points to `istio-ingressgateway`, which handles routing internally.

### Troubleshooting
- **Pods Not Ready**: Ensure Istio sidecar injection is enabled (`kubectl get ns default --show-labels` should show `istio-injection=enabled`).
- **Traffic Not Splitting**: Verify the VirtualService and DestinationRule with `kubectl describe virtualservice car-rental-frontend` and `kubectl describe destinationrule car-rental-frontend`.
- **ALB Not Routing**: Check the Ingress status (`kubectl get ingress car-rental-app`) and ensure the ALB DNS is resolving correctly.

This guide leverages Istio’s traffic management capabilities to implement a canary deployment while maintaining access via the ALB DNS. Let me know if you need further assistance![](https://kublr.com/blog/hands-on-canary-deployments-with-istio-and-kubernetes/)[](https://www.digitalocean.com/community/tutorials/how-to-do-canary-deployments-with-istio-and-kubernetes)[](https://istio.io/latest/blog/2017/0.1-canary/)