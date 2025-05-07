
# Notes: Installing Istio on AWS EKS Cluster (v1.30) via Mac Terminal

## Prerequisites
- EKS cluster v1.30 running
- `kubectl` configured to access the EKS cluster
- `helm` installed (optional, for add-ons)
- `istioctl` installed (steps below)

## Step 1: Install `istioctl`
1. Download Istio 1.25.2 for macOS: [istio-1.25.2-osx.tar.gz](https://istio.io/latest/docs/setup/getting-started/#download)
2. Create directory and move file:
   ```
   mkdir istio-installation
   mv istio-1.25.2-osx.tar.gz /Users/rahulranjan/istio-installation
   cd istio-installation
   ```
3. Extract tarball:
   ```
   tar -xvzf istio-1.25.2-osx.tar.gz
   ```
4. Remove tarball:
   ```
   rm -rf istio-1.25.2-osx.tar.gz
   ```
5. Navigate to extracted directory:
   ```
   cd istio-1.25.2
   ls
   # Output: LICENSE  README.md  bin  manifest.yaml  manifests  samples  tools
   ```
6. Add `istioctl` to PATH:
   ```
   export PATH="$PWD/bin:$PATH"
   # OR
   export PATH=$PATH:/Users/rahulranjan/istio-installation/istio-1.25.2/bin
   ```
7. Verify installation:
   ```
   istioctl version
   ```

## Step 2: Verify EKS Cluster Connection
- Check cluster nodes:
  ```
  kubectl get nodes
  # Output:
  # NAME                             STATUS   ROLES    AGE     VERSION
  # ip-192-168-13-207.ec2.internal   Ready    <none>   3m21s   v1.30.9-eks-5d632ec
  # ip-192-168-53-199.ec2.internal   Ready    <none>   3m32s   v1.30.9-eks-5d632ec
  ```

## Step 3: Install Istio
1. Check namespaces:
   ```
   kubectl get ns
   # Output:
   # NAME              STATUS   AGE
   # default           Active   10m
   # kube-node-lease   Active   10m
   # kube-public       Active   10m
   # kube-system       Active   10m
   ```
2. Install Istio with default profile:
   ```
   istioctl install
   # Prompt: Proceed? (y/N) y
   # Output:
   # ✔ Istio core installed
   # ✔ Istiod installed
   # ✔ Ingress gateways installed
   # ✔ Installation complete
   ```
3. Verify installation:
   ```
   kubectl get ns
   # Output includes: istio-system      Active   66s
   kubectl get pods -n istio-system
   # Output:
   # NAME                                    READY   STATUS    RESTARTS   AGE
   # istio-ingressgateway-6d8d69dd75-hw5qv   1/1     Running   0          67s
   # istiod-647f57cfc6-6mhv2                 1/1     Running   0          80s
   ```

## Step 4: Enable Automatic Envoy Sidecar Injection
1. Check current pods and services:
   ```
   kubectl get pods
   # Output:
   # NAME                                   READY   STATUS    RESTARTS   AGE
   # car-rental-backend-84c485d5cd-cjb4b    1/1     Running   0          19s
   # car-rental-frontend-54ff75945c-4ckkf   2/2     Running   0          18s
   kubectl get svc
   # Output:
   # NAME                            TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
   # car-rental-backend-service      ClusterIP      10.100.118.151   <none>        4000/TCP       30s
   # car-rental-frontend-service     NodePort       10.100.114.170   <none>        80:32400/TCP   29s
   # kubernetes                      ClusterIP      10.100.0.1       <none>        443/TCP        17m
   # rds-postgres-external-service   ExternalName   <none>           ...           <none>         28s
   ```
2. Check namespace labels:
   ```
   kubectl get ns default --show-labels
   # Output: default   Active   33m   kubernetes.io/metadata.name=default
   ```
3. Enable Istio injection:
   ```
   kubectl label namespace default istio-injection=enabled
   kubectl get ns default --show-labels
   # Output: default   Active   34m   istio-injection=enabled,kubernetes.io/metadata.name=default
   ```
4. Redeploy applications:
   ```
   kubectl delete -f .
   kubectl apply -f .
   ```
5. Verify sidecar injection:
   ```
   kubectl get pods
   # Output:
   # NAME                                   READY   STATUS    RESTARTS   AGE
   # car-rental-backend-84c485d5cd-dv9nt    2/2     Running   0          5s
   # car-rental-frontend-54ff75945c-6d98b   3/3     Running   0          5s
   kubectl describe pod car-rental-frontend-54ff75945c-6d98b
   # Output includes: istio-proxy container and annotations
   ```

## Step 5: Install Add-ons (Optional)
1. Apply add-ons (Kiali, Grafana, Jaeger, Prometheus):
   ```
   kubectl apply -f istio-1.25.2/samples/addons
   ```
2. Verify pods:
   ```
   kubectl get pods -n istio-system
   # Output includes:
   # grafana-75f6f7cb78-l4km4                1/1     Running   0          76s
   # jaeger-7b4845685b-k8c5h                 1/1     Running   0          73s
   # kiali-58c55848b8-pbbnm                  1/1     Running   0          67s
   # prometheus-bbc7d764f-889mf              2/2     Running   0          58s
   # Note: loki-0 may be Pending due to PVC issues
   ```
3. Remove problematic Loki add-on:
   ```
   kubectl delete -f istio-1.25.2/samples/addons/loki.yaml
   ```
4. Verify services:
   ```
   kubectl get svc -n istio-system
   # Output includes:
   # grafana                ClusterIP      10.100.49.175    <none>        3000/TCP
   # istio-ingressgateway   LoadBalancer   10.100.109.131   ...           15021:32160/TCP,80:31964/TCP,443:32638/TCP
   # istiod                 ClusterIP      10.100.98.105    <none>        15010/TCP,15012/TCP,443/TCP,15014/TCP
   # jaeger-collector       ClusterIP      10.100.28.40     <none>        14268/TCP,14250/TCP,9411/TCP,4317/TCP,4318/TCP
   # kiali                  ClusterIP      10.100.124.149   <none>        20001/TCP,9090/TCP
   # prometheus             ClusterIP      10.100.78.63     <none>        9090/TCP
   # tracing                ClusterIP      10.100.59.139    <none>        80/TCP,16685/TCP
   # zipkin                 ClusterIP      10.100.107.17    <none>        9411/TCP
   ```

## Step 6: Access Kiali Dashboard
1. Port-forward to access locally:
   ```
   kubectl port-forward svc/kiali -n istio-system 20001:20001
   open http://localhost:20001
   ```
2. Patch Kiali service to NodePort:
   ```
   kubectl patch svc kiali -n istio-system -p '{"spec": {"type": "NodePort"}}'
   ```
3. Verify NodePort:
   ```
   kubectl get svc kiali -n istio-system
   # Output:
   # NAME    TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)                          AGE
   # kiali   NodePort   10.100.124.149   <none>        20001:30465/TCP,9090:31575/TCP   13m
   ```

## Step 7: Uninstall Istio (if needed)
```
istioctl uninstall --purge
kubectl delete namespace istio-system
kubectl label namespace default istio-injection-
```
