# Prometheus Monitoring with Dynatrace
In this exercise you will learn how to deploy a NGINX server, expose Metrics via a Prometheus exporter and ingest the metrics into your Dynatrace Environment.

## Prerequisites
- A Dynatrace environment / tenant with "Manage monitoring settings" permissions. [Free Trial](https://www.dynatrace.com/signup/)
- A Kubernetes Cluster to deploy the resources

## Tasks
- Create a namespace called prometheus-demo
- Create a configMap:
    - Namespace: prometheus-demo
    - Name: nginx-conf
    - Data:
        - key: nginx.conf
        - value: 
        ```yml
        server {
        listen       80;
        server_name  localhost;
        location / {
          root   /usr/share/nginx/html;
          index  index.html index.htm;
        }
        location /metrics {
            stub_status;
            }
        }
        ```
- Create NGINX deployment
    - Namespace: Prometheus-demo
    - image: nginx
        - name: nginx
        - Port: 80
        - Mount configMap "nginx-conf" at `/etc/nginx/conf.d/`
    - image
        - name: nginx-

    ame: nginx-exporter
          image: 'nginx/nginx-prometheus-exporter'
          args:
            - '-nginx.scrape-uri=http://localhost:80/metrics'
          resources:
            limits:
              memory: 128Mi
              cpu: 500m
          ports:
            - containerPort: 9113