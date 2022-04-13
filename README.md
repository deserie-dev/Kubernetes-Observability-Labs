# Kubernetes Observability

These are some of the projects I've done relating to the three pillars of observability - logs, metrics, and traces.

---

<details>
<summary><b>K8s Logging Using a Logging Agent and the Sidecar Pattern</b></summary><p>

### Use the sidecar multi-container Pod pattern to stream Pod logs to S3 using Fluentd.

---

1. Create a Namespace for the resources you'll create in this lab step and change your default kubectl context to use the Namespace:

```
# Create namespace
kubectl create namespace logs
# Set namespace as the default for the current context
kubectl config set-context $(kubectl config current-context) --namespace=logs
```

![](/images/a.png)

---

2. Create a multi-container Pod that runs a server and a client that sends requests to the server:

```
cat << 'EOF' > pod-logs.yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    test: logs
  name: pod-logs
spec:
  containers:
  - name: server
    image: busybox:1.30.1
    ports:
    - containerPort: 8888
    # Listen on port 8888
    command: ["/bin/sh", "-c"]
    # -v for verbose mode
    args: ["nc -p 8888 -v -lke echo Received request"]
    readinessProbe:
      tcpSocket:
        port: 8888
  - name: client
    image: busybox:1.30.1
    # Send requests to server every 5 seconds
    command: ["/bin/sh", "-c"]
    args: ["while true; do sleep 5; nc localhost 8888; done"]
EOF
kubectl create -f pod-logs.yaml
```

---

3. Retrieve the logs (standard output messages) from the server container:

```
kubectl logs pod-logs server
```

![](/images/b.png)

---

4. Display the most recent log (--tail=1) including the timestamp and stream (-f for follow) the logs from the client container:

```
kubectl logs -f --tail=1 --timestamps pod-logs client
```

![](/images/c.png)

---

5. Create an Apache web server and allow access to it via a load balancer:

```
cat << 'EOF' > pod-webserver.yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    test: logs
  name: webserver-logs
spec:
  containers:
  - name: server
    image: httpd:2.4.38-alpine
    ports:
    - containerPort: 80
    readinessProbe:
      httpGet:
        path: /
        port: 80
EOF
kubectl create -f pod-webserver.yaml
kubectl expose pod webserver-logs --type=LoadBalancer
```

![](/images/d.png)

---

![](/images/e.png)

---

6. Navigate to the DNS address in a new browser tab to confirm the Service has exposed the Pod over the Internet:

![](/images/f.png)

7. Refresh the page a few times and then append /oops to the address to cause a Not Found error:

![](/images/g.png)

8. Display the logs for the webserver Pod:

```
kubectl logs webserver-logs
```

![](/images/h.png)

9. Retrieve the last 10 lines from the conf/httpd.conf file:

```
kubectl exec webserver-logs -- tail -10 conf/httpd.conf
```

![](/images/i.png)

10. Copy the conf/httpd.conf from the container to the bastion host:

```
kubectl cp webserver-logs:conf/httpd.conf local-copy-of-httpd.conf
```

### The sidecar multi-container pattern uses a "sidecar" container to extend the primary container in the Pod. In the context of logging, the sidecar is a logging agent. The logging agent streams logs from the primary container, such as a web server, to a central location that aggregates logs. To allow the sidecar access to the log files, both containers mount a volume at the path of the log files. Use an S3 bucket to collect logs. Use a sidecar that uses Fluentd, a popular data collector often used as a logging layer, with an S3 plugin installed to stream log files in the primary container to S3.

---

11. Create S3 bucket then create a ConfigMap that stores the fluentd configuration file:

```
cat << EOF > fluentd-sidecar-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluentd-config
data:
  fluent.conf: |
    # First log source (tailing a file at /var/log/1.log)
    <source>
      @type tail
      format none
      path /var/log/1.log
      pos_file /var/log/1.log.pos
      tag count.format1
    </source>

    # Second log source (tailing a file at /var/log/2.log)
    <source>
      @type tail
      format none
      path /var/log/2.log
      pos_file /var/log/2.log.pos
      tag count.format2
    </source>

    # S3 output configuration (Store files every minute in the bucket's logs/ folder)
    <match **>
      @type s3

      s3_bucket $s3_bucket
      s3_region us-west-2
      path logs/
      buffer_path /var/log/
      store_as text
      time_slice_format %Y%m%d%H%M
      time_slice_wait 1m

      <instance_profile_credentials>
      </instance_profile_credentials>
    </match>
EOF
kubectl create -f fluentd-sidecar-config.yaml
```

![](/images/j.png)

Two log sources are configured in the /var/log directory and their log messages will be tagged with count.format1 and count.format2. The primary container in the Pod will stream logs to those two files. The configuration also describes streaming all the logs to the S3 logs bucket in the match section.

---

12. Create a multi-container Pod using a fluentd logging agent sidecar (count-agent):

```
cat << 'EOF' > pod-counter.yaml
apiVersion: v1
kind: Pod
metadata:
  name: counter
spec:
  containers:
  - name: count
    image: busybox
    command: ["/bin/sh", "-c"]
    args:
    - >
      i=0;
      while true;
      do
        # Write two log files along with the date and a counter
        # every second
        echo "$i: $(date)" >> /var/log/1.log;
        echo "$(date) INFO $i" >> /var/log/2.log;
        i=$((i+1));
        sleep 1;
      done
    # Mount the log directory /var/log using a volume
    volumeMounts:
    - name: varlog
      mountPath: /var/log
  - name: count-agent
    image: lrakai/fluentd-s3:latest
    env:
    - name: FLUENTD_ARGS
      value: -c /fluentd/etc/fluent.conf
    # Mount the log directory /var/log using a volume
    # and the config file
    volumeMounts:
    - name: varlog
      mountPath: /var/log
    - name: config-volume
      mountPath: /fluentd/etc
  # Use host network to allow sidecar access to IAM instance profile credentials
  hostNetwork: true
  # Declare volumes for log directory and ConfigMap
  volumes:
  - name: varlog
    emptyDir: {}
  - name: config-volume
    configMap:
      name: fluentd-config
EOF
kubectl create -f pod-counter.yaml
```

![](/images/k.png)

The count container writes the date and a counter variable ($i) in two different log formats to two different log files in the /var/log directory every second. The /var/log directory is mounted as a Volume in both the primary count container and the count-agent sidecar so both containers can access the logs. The sidecar also mounts the ConfigMap to access the fluentd configuration file. By using a ConfigMap, the same sidecar container can be used for any configuration compared to storing the configuration in the image and having to manage separate container images for each configuration.

---

13. View the logs in S3

![](/images/l.png)

---

![](/images/m.png)

---

</p></details>

---

<details>
<summary><b>Monitoring K8s with the Kubernetes Dashboard, Prometheus, and Grafana</b></summary><p>

# Deploy a Simple API Application

Introduction
In this Lab Step, you'll deploy a simple Python Flask web based API into the lab provided Kubernetes cluster. The API has been instrumented to provide various metrics which will be collected by Prometheus for observability purposes.

Instructions

1. Expand the Files tree view by clicking on the Files tab on the left handside menu, and then open the project/code/api directory:

2. The api directory contains the following 3 files which have then been used to build the cloudacademydevops/api-metrics container image. Open each of the following files within the editor view and review their contents.

api.py
Dockerfile
requirements.txt
The api.py file contains the Python source code which implements the example API. In particular take note of the following:

Line 5 - imports a PromethusMetrics module to automatically generate Flask based metrics and provide them for collection at the default endpoint /metrics
Line 10-32 - implements 5 x API endpoints:
/one
/two
/three
/four
/error
All example endpoints, except for the error endpoint, introduce a small amount of latency which will be measured and observed within both Prometheus and Grafana.
The error endpoint returns an HTTP 500 server error response code, which again will be measured and observed within both Prometheus and Grafana.
The Docker container image containing this source code has already been built using the tag cloudacademydevops/api-metrics 3. Within the Files tab on the left handside menu, open the project/code/k8s directory and click on the api.yml file:

4. The api.yml file contains the Kubernetes resources that will be created for the API when deployed into the cluster. In particular note the following:

Lines 1-25: API Deployment containing 2 pods
Line 22: Pods are based off the container image cloudacademydevops/api-metrics
Lines 27-46: API Service - loadbalances traffic across the 2 API Deployment pods
Lines 34-37: API Service is annotated to ensure that the Prometheus scraper will automatically discover the API pods behind it. Prometheus will then collect their metrics from the discovered targets

7. Deploy the API application. In the terminal execute the following command:

```
kubectl apply -f ./code/k8s
```

8. Confirm that the API pods are in a running status. In the terminal execute the following command:

```
kubectl get pods
```

9. Confirm that the service has been created. In the terminal execute the following command:

```
kubectl get svc
```

10. In order to generate traffic against the deployed API - spin up a single generator pod. In the terminal execute the following command:

```
kubectl run generator --env="API_URL=http://api-service:5000" --image=cloudacademydevops/api-generator --image-pull-policy IfNotPresent
```

11. Confirm that the generator pod is in a running status. In the terminal execute the following command:

```
kubectl get pods
```

Summary

In this Lab Step, you deployed a simple Python Flask web based API, which has been instrumented to automatically collect and provide metrics that will be collected by Prometheus. Additionally, you also deployed a single generator pod which will continually make HTTP requests against the API. In the next Lab Step you will install and configure Prometheus.

### Install and Configure K8s Dashboard

Introduction
In this Lab Step, you'll install and configure the Kubernetes Dashboard and expose it over the Internet on port 30990, allowing you to then access it from your own workstation. The Kubernetes Dashboard is a web-based Kubernetes user interface. You can use Kubernetes Dashboard to deploy containerized applications to a Kubernetes cluster, troubleshoot your containerized application, and/or manage other cluster resources.

Instructions

1. Create a new monitoring namespace within the cluster. In the terminal execute the following command:

```
kubectl create ns monitoring
```

2. Using Helm, install the Kubernetes Dashboard using the publicly available Kubernetes Dashboard Helm Chart. Deploy the dashboard into the monitoring namespace within the lab provided cluster. In the terminal execute the following commands:

```
{
helm repo add k8s-dashboard https://kubernetes.github.io/dashboard
helm repo update
helm install k8s-dashboard --namespace monitoring k8s-dashboard/kubernetes-dashboard --set=protocolHttp=true --set=serviceAccount.create=true --set=serviceAccount.name=k8sdash-serviceaccount --version 3.0.2
}
```

3. Establish permissions within the cluster to allow the Kubernetes Dashboard to read and write all cluster resources. In the terminal execute the following command:

```
kubectl create clusterrolebinding kubernetes-dashboard --clusterrole=cluster-admin --serviceaccount=monitoring:k8sdash-serviceaccount
```

4. The Kubernetes Dashboard web interface now needs to be exposed to the Internet so that you can browse to it. To do so, create a new NodePort based Service, and expose the web admin interface on port 30990. In the terminal execute the following command:

```
{
kubectl expose deployment k8s-dashboard-kubernetes-dashboard --type=NodePort --name=k8s-dashboard --port=30990 --target-port=9090 -n monitoring
kubectl patch service k8s-dashboard -n monitoring -p '{"spec":{"ports":[{"nodePort": 30990, "port": 30990, "protocol": "TCP", "targetPort": 9090}]}}'
}
```

5. Get the public IP address of the Kubernetes cluster that Prometheus has been deployed into . In the terminal execute the following command:

```
export | grep K8S_CLUSTER_PUBLICIP
```

6. Copy the Public IP address from the previous command and then using your local browser, browse to the URL: http://PUBLIC_IP:30990.

Summary

In this Lab Step, you installed the Kubernetes Dashboard into the monitoring namespace within the Kubernetes cluster. You then setup and exposed the dashboard using a NodePort based Service. You then logged into the dashboard and confirmed that it was functional. In the next Lab Step you will install and configure Prometheus to start collecting metrics.

### Install and Configure Prometheus

Introduction
In this Lab Step, you'll install and configure Prometheus into the lab provided Kubernetes cluster. Prometheus is an open-source systems monitoring and alerting service. You'll configure Prometheus to perform automatic service discovery of both the API pods launched in the previous Lab Step, and the cluster's nodes. Prometheus will then automatically begin to collect metrics for both the API pods and the cluster's nodes. HTTP request based metrics will be collected from the API pods, and Memory and CPU utilisation metrics will be collected from the cluster's nodes.

You will configure the Prometheus web admin interface to be exposed over the Internet on port 30900, allowing you to then access it from your own workstation.

Instructions

1. Using Helm, install Prometheus using the publicly available Prometheus Helm Chart. Deploy Prometheus into the monitoring namespace within the lab provided cluster. In the terminal execute the following commands:

```
{
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add stable https://charts.helm.sh/stable
helm repo update
helm install prometheus --namespace monitoring --values ./code/prometheus/values.yml prometheus-community/prometheus --version 13.0.0
}
```

2. Confirm that Prometheus has been successfully rolled out within the cluster. In the terminal execute the following command:

```
kubectl get deployment -n monitoring -w
```

Note: The previous command puts a watch on all deployments happening in the monitoring namespace. Exit the watch when all deployments have a READY status of 1/1 (CTRL+C to exit)

3. Confirm that the Prometheus Node Exporter DaemonSet resource has been created successfully. The Prometheus Node Exporter is used to collect Memory and CPU metrics off each node within the Kubernetes cluster. In the terminal execute the following command:

```
kubectl get daemonset -n monitoring
```

4. Patch the Prometheus Node Exporter DaemonSet to ensure that Prometheus can collect Memory and CPU node metrics. In the terminal execute the following command:

```
kubectl patch daemonset prometheus-node-exporter -n monitoring -p '{"spec":{"template":{"metadata":{"annotations":{"prometheus.io/scrape": "true"}}}}}'
```

5. The Prometheus web admin interface now needs to be exposed to the Internet so that you can browse to it. To do so, create a new NodePort based Service, and expose the web admin interface on port 30900. In the terminal execute the following command:

```
{
kubectl expose deployment prometheus-server --type=NodePort --name=prometheus-main --port=30900 --target-port=9090 -n monitoring
kubectl patch service prometheus-main -n monitoring -p '{"spec":{"ports":[{"nodePort": 30900, "port": 30900, "protocol": "TCP", "targetPort": 9090}]}}'
}
```

6. Get the public IP address of the Kubernetes cluster that Prometheus has been deployed into. In the terminal execute the following command:

```
export | grep K8S_CLUSTER_PUBLICIP
```

7. Copy the Public IP address from the previous command and then using your local browser, browse to the URL: http://PUBLIC_IP:30900.

8. Within Prometheus, click the Status top menu item and then select Service Discovery:

9. Within Prometheus, click the Status top menu item and then select Targets:

Summary

In this lab step, you installed Prometheus into the monitoring namespace within the Kubernetes cluster. You then set up and exposed the Prometheus web admin interface using a NodePort based Service. You then logged into the Prometheus web admin interface and confirmed the service discovery was working correctly. In the next lab step you will install and configure Grafana and import a prebuilt dashboard that pulls real-time data from Prometheus.

</p></details>

---
