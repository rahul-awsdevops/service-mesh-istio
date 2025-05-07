# Istio Canary Deployment on AWS EKS: Car Rental Application

This document provides a comprehensive guide to setting up an AWS EKS cluster, installing the AWS Load Balancer Controller, deploying Istio with a `NodePort` Ingress Gateway, and implementing a canary deployment for a car rental application with 90/10 traffic splitting (`v1`/`v2` frontend). It includes screenshots of the `v1` and `v2` application versions, detailed steps, YAML configurations, verification outputs, troubleshooting, and an explanation of traffic flow from the ALB to the application pods with an ASCII diagram.

## Prerequisites
- **Tools**:
  - `aws` CLI (configured with credentials)
  - `eksctl`
  - `kubectl`
  - `helm`
  - `istioctl` (Istio 1.25.2)
- **AWS Resources**:
  - VPC (`vpc-020555c635c09a5c4`)
  - IAM permissions for EKS, ALB, and ECR
  - RDS PostgreSQL instance (`car-rental-db.c9k8z3x4y5.us-east-1.rds.amazonaws.com`)
  - ECR repository (`676206914267.dkr.ecr.us-east-1.amazonaws.com/techcloud-academy-frontend-apps`)

## Step 1: Set Up EKS Cluster
Create an EKS cluster named `helm-eks-cluster` in `us-east-1`.

```bash
eksctl create cluster \
  --name helm-eks-cluster \
  --region us-east-1 \
  --version 1.29 \
  --nodegroup-name standard-workers \
  --node-type t3.medium \
  --nodes 2 \
  --nodes-min 1 \
  --nodes-max 3 \
  --managed
```

- **Output**: Cluster creation takes ~15-20 minutes. Verify:
  ```bash
  kubectl get nodes
  ```
  Example:
  ```
  NAME                          STATUS   ROLES    AGE   VERSION
  ip-192-168-49-247.ec2.internal   Ready    <none>   20m   v1.29.0
  ip-192-168-62-123.ec2.internal   Ready    <none>   20m   v1.29.0
  ```

## Step 2: Install AWS Load Balancer Controller
The AWS Load Balancer Controller manages ALBs for Kubernetes Ingress resources.

### Create IAM Service Account
Attach the `AWSLoadBalancerControllerIAMPolicy`.

```bash
eksctl create iamserviceaccount \
  --cluster helm-eks-cluster \
  --namespace kube-system \
  --name aws-load-balancer-controller \
  --attach-policy-arn arn:aws:iam::676206914267:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve
```

- **Output**:
  ```
  2025-05-07 16:14:29 [ℹ]  1 iamserviceaccount (kube-system/aws-load-balancer-controller) was included
  ...
  2025-05-07 16:15:02 [ℹ]  created serviceaccount "kube-system/aws-load-balancer-controller"
  ```

### Install Controller via Helm
```bash
helm repo add eks https://aws.github.io/eks-charts
helm repo update
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=helm-eks-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=us-east-1 \
  --set vpcId=vpc-020555c635c09a5c4 \
  --set image.repository=602401143452.dkr.ecr.us-east-1.amazonaws.com/amazon/aws-load-balancer-controller
```

- **Output**:
  ```
  NAME: aws-load-balancer-controller
  LAST DEPLOYED: Wed May  7 16:16:24 2025
  NAMESPACE: kube-system
  STATUS: deployed
  REVISION: 1
  NOTES:
  AWS Load Balancer controller installed!
  ```

- **Verify**:
  ```bash
  kubectl get pods -n kube-system
  ```
  Example:
  ```
  NAME                                            READY   STATUS    RESTARTS   AGE
  aws-load-balancer-controller-68d47d7ffb-5w7gt   1/1     Running   0          2m2s
  aws-load-balancer-controller-68d47d7ffb-c5j6k   1/1     Running   0          2m2s
  ...
  ```

## Step 3: Install Istio
Install Istio with the `demo` profile and `NodePort` Ingress Gateway.

### Download Istio
```bash
curl -L https://istio.io/downloadIstio | ISTIO_VERSION=1.25.2 sh -
cd istio-1.25.2
export PATH=$PATH:/Users/rahulranjan/istio-installation/istio-1.25.2/bin
```

### Install Istio
```bash
istioctl install \
  --set profile=demo \
  --set values.gateways.istio-ingressgateway.type=NodePort \
  -y
```

- **Output**:
  ```
  ✔ Istio core installed ⛵️
  ✔ Istiod installed 🧠
  ✔ Ingress gateways installed 🛬
  ✔ Egress gateways installed 🛫
  ✔ Installation complete
  ```

- **Verify**:
  ```bash
  kubectl get pods -n istio-system
  ```
  Example:
  ```
  NAME                                    READY   STATUS    RESTARTS   AGE
  istio-egressgateway-6659d5f648-bq75q    1/1     Running   0          37s
  istio-ingressgateway-54b88fc78b-smm4s   1/1     Running   0          37s
  istiod-c8d9c7897-z8777                  1/1     Running   0          46s
  ```

  ```bash
  kubectl get svc -n istio-system istio-ingressgateway
  ```
  Example:
  ```
  NAME                   TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)                                                                      AGE
  istio-ingressgateway   NodePort   10.100.187.233   <none>        15021:30714/TCP,80:31138/TCP,443:31710/TCP,31400:32567/TCP,15443:30977/TCP   2m33s
  ```

### Enable Istio Injection
```bash
kubectl label namespace default istio-injection=enabled
```

- **Verify**:
  ```bash
  kubectl get namespace -L istio-injection
  ```
  Example:
  ```
  NAME              STATUS   AGE   ISTIO-INJECTION
  default           Active   20m   enabled
  ...
  ```

## Step 4: Deploy Car Rental Application
Deploy the backend, frontend (`v1`), and frontend (`v2`) components.

### RDS PostgreSQL External Service
```yaml
apiVersion: v1
kind: Service
metadata:
  name: rds-postgres-external-service
  namespace: default
spec:
  type: ExternalName
  externalName: car-rental-db.c9k8z3x4y5.us-east-1.rds.amazonaws.com
  ports:
  - port: 5432
    protocol: TCP
    targetPort: 5432
```

Apply:
```bash
kubectl apply -f rds-postgres-external-nameservice.yml
```

### Backend Deployment
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: car-rental-backend #nameofdeployment
spec:
  replicas: 1
  selector: 
    matchLabels:
      app: car-rental-backend #matchlabelforpod
  template: 
    metadata:
      labels: 
        app: car-rental-backend #labelforpod
    spec: 
      containers:
        - name: car-rental-backend-container
          image: 676206914267.dkr.ecr.us-east-1.amazonaws.com/techcloud-academy-backend-apps:adec382
          ports:
            - containerPort: 4000 #default-port-pgadmin

          env:
            - name: DB_USER
              value: "dbadmin"
              
            - name: DB_PASSWORD
              value: "dbadmin123"

            - name: DB_NAME
              value: "car_rental_database"  # This is the name of the database inside PostgreSQL

            - name: DB_HOST
              value: "rds-postgres-external-service"  # RDS instance endpoint will also be listed in rds-postgres-external-nameservice

            - name: DB_PORT
              value: "5432"  # Default PostgreSQL port
---
apiVersion: v1
kind: Service
metadata:
  name: car-rental-backend-service
spec:
  type: ClusterIP
  selector:
    app: car-rental-backend #labelofpodsshouldmatch
  ports:
    - protocol: TCP
      port: 4000
      targetPort: 4000
```

Apply:
```bash
kubectl apply -f car-rental-backend.yml
```

### Frontend `v1` Deployment
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: car-rental-frontend #nameofdeployment
spec:
  replicas: 1
  selector: 
    matchLabels:
      app: car-rental-frontend #matchlabelforpod
      version: v1
  template: 
    metadata:
      labels: 
        app: car-rental-frontend #labelforpod
        version: v1
    spec: 
      containers:
        - name: nginx-reverse-proxy-container
          image: 676206914267.dkr.ecr.us-east-1.amazonaws.com/techcloud-academy-frontend-apps-nginx-proxy:adec382  # Docker image with Nginx reverse proxy
          ports:
            - containerPort: 80  # Exposes port 80 for reverse proxy
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
  type: ClusterIP #if you dont mention this type then default will be main ClusterIP
  selector:
    app: car-rental-frontend #labelofpodsshouldmatch
  ports:
    - protocol: TCP
      port: 80 #serviceport
      targetPort: 80 #containerPort from pod yml  
```

Apply:
```bash
kubectl apply -f car-rental-frontend.yml
```

### Frontend `v2` Deployment (Canary)
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
```

Apply:
```bash
kubectl apply -f car-rental-frontend-canary.yml
```

- **Verify Pods**:
  ```bash
  kubectl get pods
  ```
  Example:
  ```
  NAME                                      READY   STATUS    RESTARTS   AGE
  car-rental-backend-84c485d5cd-xzwf7       2/2     Running   0          29m
  car-rental-frontend-6c8b4c989b-46z55      3/3     Running   0          29m
  car-rental-frontend-v2-56c648b7d5-65z2s   3/3     Running   0          29m
  ```

## Step 5: Configure Istio Traffic Splitting
Set up Istio Gateway, VirtualService, and DestinationRule for traffic splitting.

### Destination Rule
```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: car-rental-frontend
spec:
  host: car-rental-frontend-service.default.svc.cluster.local
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
```

Apply:
```bash
kubectl apply -f car-rental-destination-rules.yml
```

### Gateway and VirtualService (90/10 Split)
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
  - match:
    - uri:
        prefix: /
    route:
    - destination:
        host: car-rental-frontend-service.default.svc.cluster.local
        subset: v1
      weight: 90
    - destination:
        host: car-rental-frontend-service.default.svc.cluster.local
        subset: v2
      weight: 10
```

Apply:
```bash
kubectl apply -f car-rental-istio-virtualservice-gateway.yml
```

## Step 6: Configure Ingress
### Ingress Class
```yaml
#this is for exposing load balancer

apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  name: my-aws-ingress-class
  annotations:
    ingressclass.kubernetes.io/is-default-class: "true"
spec:
  controller: ingress.k8s.aws/alb
```

Apply:
```bash
kubectl apply -f ingress-class.yml
```

### Ingress Resource
```yaml
# Ingress for ALB
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: car-rental-app
  namespace: istio-system
  annotations:
    alb.ingress.kubernetes.io/load-balancer-name: car-rental-app
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}]'
    alb.ingress.kubernetes.io/healthcheck-protocol: HTTP
    alb.ingress.kubernetes.io/healthcheck-port: "31138"
    alb.ingress.kubernetes.io/healthcheck-path: /
    alb.ingress.kubernetes.io/healthcheck-interval-seconds: "30"
    alb.ingress.kubernetes.io/healthcheck-timeout-seconds: "10"
    alb.ingress.kubernetes.io/success-codes: "200-399"
    alb.ingress.kubernetes.io/healthy-threshold-count: "2"
    alb.ingress.kubernetes.io/unhealthy-threshold-count: "2"
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

Apply:
```bash
kubectl apply -f ingress-resource.yml
```

- **Troubleshooting**: If error `json: cannot unmarshal number into Go struct field ObjectMeta.metadata.annotations` occurs, ensure annotations are quoted. Fix:
  ```bash
  kubectl delete -f ingress-resource.yml
  kubectl apply -f ingress-resource.yml
  ```

- **Verify**:
  ```bash
  kubectl get ingress -n istio-system
  ```
  Example:
  ```
  NAME             CLASS                  HOSTS   ADDRESS                                                PORTS   AGE
  car-rental-app   my-aws-ingress-class   *       car-rental-app-173958972.us-east-1.elb.amazonaws.com   80      24s
  ```

## Step 7: Visual Differences Between `v1` and `v2`
The `v1` and `v2` frontends differ primarily in the presence of the `/about` page.

### `v1` Frontend
- **Image**: `676206914267.dkr.ecr.us-east-1.amazonaws.com/techcloud-academy-frontend-apps:adec382`
- **Description**: The `v1` frontend provides the core car rental application (e.g., homepage, booking functionality) but lacks an `/about` page. Accessing `http://car-rental-app-173958972.us-east-1.elb.amazonaws.com/about` returns a 404 error.
- **Expected Screenshot**:
  - Likely shows the homepage (`/`) with a title like "Ally's Auto Rentals," booking options, and a clean, modern UI with a blue and yellow gradient.
  - Path: `[Insert path to v1 screenshot, e.g., ~/Desktop/v1-homepage.png]`
  - Placeholder: `[Insert v1 Screenshot here]` (e.g., upload to GitHub/Imgur and link: `![v1 Homepage](<URL>)`).

### `v2` Frontend
- **Image**: `676206914267.dkr.ecr.us-east-1.amazonaws.com/techcloud-academy-frontend-apps:new-features-about-page-d831d54`
- **Description**: The `v2` frontend includes all `v1` features plus a new `/about` page. The `/about` page contains:
  - Header: "About Ally's Auto Rentals" in bold, with "Ally's" in yellow.
  - Section: "Our Story" with text about revolutionizing car rentals, starting in 2020, and a diverse fleet.
  - Features: Cards for "Diverse Fleet," "Easy Booking," and "24/7 Support" with icons and colored backgrounds (blue, orange, yellow).
  - Call to Action: "Book Your Car Now" button.
- **Expected Screenshot**:
  - Shows the `/about` page with the above elements, accessed via `http://car-rental-app-173958972.us-east-1.elb.amazonaws.com/about`.
  - Path: `[Insert path to v2 screenshot, e.g., ~/Desktop/v2-about.png]`
  - Placeholder: `[Insert v2 screenshot here]` (e.g., `![v2 About Page](<URL>)`).

### Including Screenshots
To include screenshots in the final document:
1. **Markdown Viewers (e.g., GitHub, VS Code)**:
   - Upload images to a public host (e.g., GitHub repository, Imgur).
   - Add Markdown links:
     ```markdown
     ![v1 Homepage](https://example.com/v1-homepage.png)
     ![v2 About Page](https://example.com/v2-about.png)
     ```
2. **Word/PDF**:
   - Insert images directly at the placeholders `[Insert v1 screenshot here]` and `[Insert v2 screenshot here]`.
   - Use file paths (e.g., `~/Desktop/v1-homepage.png`) in your document editor.
3. **Local Markdown**:
   - Reference local files (limited compatibility):
     ```markdown
     ![v1 Homepage](file:///Users/rahulranjan/Desktop/v1-homepage.png)
     ```

## Step 8: Traffic Flow Explanation
Traffic flows from the client to the application pods as follows:

1. **Client Request**: Users access `http://car-rental-app-173958972.us-east-1.elb.amazonaws.com`.
2. **AWS ALB**: Receives HTTP on port 80, forwards to EKS nodes on `NodePort` 31138.
3. **EKS Worker Nodes**: Route traffic to `istio-ingressgateway` Service (port 80).
4. **Istio Ingress Gateway**: Matches HTTP requests via Gateway and forwards to VirtualService.
5. **Istio VirtualService**: Splits traffic (90% `v1`, 10% `v2` or 100% `v2`) to `car-rental-frontend-service`.
6. **Frontend Service**: Load-balances to `v1` or `v2` pods based on DestinationRule.
7. **Backend/RDS**: Frontend pods communicate with backend, which connects to RDS.

### Traffic Flow Diagram (ASCII)
```
[Client]
    |
    | HTTP (car-rental-app-173958972.us-east-1.elb.amazonaws.com:80)
    v
[AWS ALB]
    | Forward to NodePort 31138
    | Health check: / (port 31138, 200-399)
    v
[EKS Worker Nodes] (e.g., 3.87.34.177:31138)
    | NodePort Service (istio-ingressgateway:80)
    v
[Istio Ingress Gateway Pod] (e.g., 192.168.62.61:80)
    | Istio Gateway (car-rental-gateway)
    | VirtualService (car-rental-frontend)
    |   - 90% -> v1 subset
    |   - 10% -> v2 subset
    v
[car-rental-frontend-service] (ClusterIP)
    | DestinationRule (subsets: v1, v2)
    |-----------------|-----------------|
    v                 v
[v1 Pod]          [v2 Pod]
(car-rental-frontend) (car-rental-frontend-v2)
(image: adec382)  (image: new-features-about-page-d831d54)
(no /about, 404)  (has /about, 200)
    |                 |
    | HTTP (NEXT_PUBLIC_API_URL)
    v                 v
[car-rental-backend-service]
    |
    v
[Backend Pod]
(image: cd4c2b6)
    |
    v
[RDS PostgreSQL]
(car-rental-db.c9k8z3x4y5.us-east-1.rds.amazonaws.com:5432)
```

## Step 9: Verify Application Access
- **Health Check**:
  ```bash
  curl -v http://3.87.34.177:31138/
  ```
  Example:
  ```
  < HTTP/1.1 200 OK
  <html><title>Ally's Auto Rentals</title>...
  ```

- **Ingress Gateway Health**:
  ```bash
  kubectl exec -it car-rental-frontend-6c8b4c989b-46z55 -n default -- curl -v http://192.168.62.61:15021/healthz/ready
  ```
  Example:
  ```
  < HTTP/1.1 200 OK
  content-length: 0
  ```

- **ALB DNS**:
  ```bash
  curl -v http://car-rental-app-173958972.us-east-1.elb.amazonaws.com
  ```
  Expect homepage HTML.

## Step 10: Test Canary Deployment
The `v2` frontend includes an `/about` page, while `v1` returns 404 for `/about`.

### Test 100% `v2` Traffic
```bash
kubectl apply -f v2-virtualservice.yml
for i in {1..100}; do curl -s http://car-rental-app-173958972.us-east-1.elb.amazonaws.com/about | grep -q "Our Story" && echo -e "\033[32m[v2]\033[0m"; done | wc -l
```

- **Output**:
  ```
      100
  ```
  - All 100 requests hit `v2`, returning 200 with `/about` page.

### Test 90/10 Split
```bash
kubectl apply -f car-rental-istio-virtualservice-gateway.yml
for i in {1..100}; do curl -s http://car-rental-app-173958972.us-east-1.elb.amazonaws.com/about | grep -q "Our Story" && echo -e "\033[32m[v2]\033[0m"; done | wc -l
```

- **Output**:
  ```
        9
  ```
  - ~9/100 requests hit `v2`, confirming 10% traffic.

- **Colored Test**:
  ```bash
  for i in {1..20}; do response=$(curl -s http://car-rental-app-173958972.us-east-1.elb.amazonaws.com/about | grep "Our Story"); if [ -n "$response" ]; then echo -e "\033[32m[v2]\033[0m Our Story found"; fi; done
  ```
  Example:
  ```
  [v2] Our Story found
  [v2] Our Story found
  ```

- **Status Codes**:
  ```bash
  for i in {1..20}; do status=$(curl -s -o /dev/null -w "%{http_code}" http://car-rental-app-173958972.us-east-1.elb.amazonaws.com/about); if [ "$status" = "200" ]; then echo -e "\033[32m[v2] $status\033[0m"; else echo -e "\033[31m[v1] $status\033[0m"; fi; done
  ```
  Example:
  ```
  [v1] 404
  [v2] 200
  [v1] 404
  ...
  ```

## Step 11: Cleanup
```bash
kubectl delete -f .
```

- **Output**:
  ```
  deployment.apps "car-rental-backend" deleted
  service "car-rental-backend-service" deleted
  destinationrule.networking.istio.io "car-rental-frontend" deleted
  deployment.apps "car-rental-frontend-v2" deleted
  deployment.apps "car-rental-frontend" deleted
  service "car-rental-frontend-service" deleted
  gateway.networking.istio.io "car-rental-gateway" deleted
  virtualservice.networking.istio.io "car-rental-frontend" deleted
  ingressclass.networking.k8s.io "my-aws-ingress-class" deleted
  ingress.networking.k8s.io "car-rental-app" deleted
  service "rds-postgres-external-service" deleted
  ```

Uninstall Istio:
```bash
istioctl uninstall --purge -y
```

Delete EKS cluster:
```bash
eksctl delete cluster --name helm-eks-cluster
```

## Troubleshooting
### Ingress Resource Error
- **Issue**: `json: cannot unmarshal number into Go struct field ObjectMeta.metadata.annotations`
- **Fix**: Quote annotation values. Reapply:
  ```bash
  kubectl delete -f ingress-resource.yml
  kubectl apply -f ingress-resource.yml
  ```

### Unhealthy Target Group
- **Fix**:
  - Update security groups:
    - ALB SG: Allow outbound TCP to node SG on `31138`.
    - Node SG: Allow inbound TCP from ALB SG on `31138`.
  - Verify health check:
    ```yaml
    alb.ingress.kubernetes.io/healthcheck-port: "31138"
    alb.ingress.kubernetes.io/healthcheck-path: /
    ```

### Filtering Issues
- **Issue**: `grep "Our Story"` returned full HTML or no output.
- **Fix**:
  - Use `"revolutionize car rentals"`:
    ```bash
    for i in {1..20}; do response=$(curl -s http://car-rental-app-173958972.us-east-1.elb.amazonaws.com/about | grep "revolutionize car rentals"); if [ -n "$response" ]; then echo -e "\033[32m[v2]\033[0m revolutionize car rentals found"; fi; done
    ```

## Conclusion
This setup demonstrated EKS cluster creation, Istio deployment, and a canary deployment with 90/10 traffic splitting. Screenshots of `v1` and `v2` provide visual confirmation of the differences, validated by traffic split tests (100% `v2`: 100/100 hits; 90/10: 9/100 hits).


---

### Guidance on Screenshots
Since you have the screenshot paths:
1. **Share Paths**: Provide the file paths (e.g., `~/Desktop/v1-homepage.png`, `~/Desktop/v2-about.png`) and describe what each shows (e.g., UI elements, colors, text).
2. **Upload Images**: Upload to a public host (e.g., GitHub repository, Imgur) and share URLs. I can update the Markdown with:
   ```markdown
   ![v1 Homepage](https://example.com/v1-homepage.png)
   ![v2 About Page](https://example.com/v2-about.png)
   ```
3. **Local Document**: If compiling locally (e.g., Word, PDF), insert images at the placeholders using the file paths.

### Assumptions
- `v1` Screenshot: Likely the homepage (`/`) with booking features, no `/about` page.
- `v2` Screenshot: The `/about` page with "Our Story," feature cards, and a booking button, as described in the HTML output.

Please provide the screenshot paths and descriptions, or confirm if the placeholders are sufficient. I can further refine the notes if needed!