## Summary of this Project:
- Observability and monitoring stack in a AWS EC2 instance running Ubuntu 26. A pipeline using Docker Compose, combining Prometheus for metrics collection, Node Exporter for system metrics, Grafana for visualization/alerting, and Loki with Grafana Alloy for log aggregation.
- Prometheus monitors /metrics endpoint of a python-fastapi blog CRUD webapp
- SMTP alert from Grafana set up using 'google app password' to detect and target downtime automatically

## Architectural Diagram
<img width="1073" height="578" alt="diagram" src="https://github.com/user-attachments/assets/d5ed03dd-3d4a-462b-98cf-5b15ab65e537" />

## Work Log:  

### Preparing Ubuntu, Docker and Kubernetes in EC2

- Instead of Amazon Linux I have selected Ubuntu 26. I still have credits left, so using the largest free EC2 I can find because I will be installing many things on this:  
  <img width="1173" height="495" alt="image" src="https://github.com/user-attachments/assets/0df80b68-7975-42e9-bfc8-dff1239c5c96" />   

-  Taking a 30 GB volume instead of the default 8GB.  
-  I could start the ubuntu machine and install everything 1 by 1. But I can also pass a script in the [user-data](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/user-data.html#userdata-linux) field. It can be shell script  or yaml also. [more docs](https://docs.cloud-init.io/en/24.1/reference/examples.html):  
  <img width="689" height="188" alt="image" src="https://github.com/user-attachments/assets/e055b3ff-2965-46d0-bb42-14df3ee75987" />  

- refer to the ec2_user_data_field.yaml file.  
- Verification of installations...  
<img width="761" height="322" alt="image" src="https://github.com/user-attachments/assets/19ace719-3d69-48a9-91a9-e4698fa1fd56" />  
<img width="678" height="258" alt="Screenshot 2026-11-12 081815" src="https://github.com/user-attachments/assets/95328e6d-2c1d-4fdd-9704-57efee82113e" />  
<img width="504" height="70" alt="Screenshot 2026-11-12 081822" src="https://github.com/user-attachments/assets/e1c474b7-1022-40fa-a62d-1821b6f4c789" />  

### metrics and logging backend: Prometheus and Loki

### The webapp to Observe: set up & instrumentation
I've previously built this blog webapp: [fullstack-blog-FastAPI](https://github.com/9mAhmad/fullstack-blog-FastAPI.git).
Following the Readme and the docker-compose file, a combined compose file looks like this:
```
services:
  prometheus:
    image: prom/prometheus:v2.45.0
    container_name: prometheus
    restart: unless-stopped
    ports:
      - "127.0.0.1:9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
    networks:
      - monitoring

  grafana:
    image: grafana/grafana:10.1.0
    container_name: grafana
    restart: unless-stopped
    ports:
      - "127.0.0.1:3000:3000"
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=admin
    volumes:
      - grafana_data:/var/lib/grafana
    networks:
      - monitoring
    depends_on:
      - prometheus
      - loki

  node-exporter:
    image: prom/node-exporter:v1.6.1
    container_name: node-exporter
    restart: unless-stopped
    ports:
      - "127.0.0.1:9100:9100"
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - '--path.procfs=/host/proc'
      - '--path.sysfs=/host/sys'
      - '--path.rootfs=/rootfs'
      - '--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)'
    networks:
      - monitoring

  loki:
    image: grafana/loki:2.8.4
    container_name: loki
    restart: unless-stopped
    ports:
      - "127.0.0.1:3100:3100"
    volumes:
      - loki_data:/tmp/loki
    command: -config.file=/etc/loki/local-config.yaml
    networks:
      - monitoring

  db:
    container_name: postgres-blog-composed
    image: postgres:14.22-trixie
    restart: always
    ports:
      - "127.0.0.1:5432:5432"
    environment:
      POSTGRES_DB: "blog"
      POSTGRES_USER: "bloguser"
      POSTGRES_PASSWORD: "bloguser"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U $$POSTGRES_USER -d $$POSTGRES_DB"]
      interval: 5s
      timeout: 5s
      retries: 5
    volumes:
      - blogdata-pg:/var/lib/postgresql/data
    networks:
      - monitoring

  webapp:
    build: ./
    container_name: blog-webapp
    restart: unless-stopped
    env_file: ".env"
    ports:
      - "127.0.0.1:8080:8080"
    depends_on:
      db:
        condition: service_healthy
    networks:
      - monitoring

volumes:
  prometheus_data:
  grafana_data:
  loki_data:
  blogdata-pg:

networks:
  monitoring:
    driver: bridge

```

And a `prometheus.yml` file, ( the `compose up` looks for "yml" not yaml):
```
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'node-exporter'
    static_configs:
      - targets: ['node-exporter:9100']

  - job_name: 'blog-webapp'
    static_configs:
      - targets: ['webapp:8080']
```

**Relevant screenshots**:    
<img width="1355" height="138" alt="image" src="https://github.com/user-attachments/assets/86384d01-be2e-4259-802a-ef645c310551" />   
<img width="1336" height="560" alt="image" src="https://github.com/user-attachments/assets/e651a21c-f906-4016-b2ff-00be440340a0" />  


Prometheus scrape config:
```
ubuntu@ip-172-31-3-125:~$ cat prometheus.yml 
global:
  scrape_interval: 15s

  - job_name: 'node-exporter'
    static_configs:
      - targets: ['node-exporter:9100']


  - job_name: 'blog-webapp'
    static_configs:
      - targets: ['webapp:8080']

```
In this endpoint, we can see prometheus can reach its target endpoint as per the yml file above:  
<img width="1235" height="317" alt="image" src="https://github.com/user-attachments/assets/e8f84502-b4e9-4439-a7ae-1586f1048279" />    
Note that /metics endpoint was not in my blog-webapp previously, hence needed to add this code to main.py  

```
from prometheus_client import generate_latest, CONTENT_TYPE_LATEST
from fastapi import Response
@app.get("/metrics", include_in_schema=False)
async def metrics():
    return Response(content=generate_latest(), media_type=CONTENT_TYPE_LATEST)

```

### SMTP Configuration:

Going to Google account settings (https://myaccount.google.com/apppasswords), generate an 'app password' then in grafana's compose yaml:
```
    environment:
      - GF_SECURITY_ADMIN_USER=admin
      - GF_SECURITY_ADMIN_PASSWORD=admin
      - GF_SMTP_ENABLED=true
      - GF_SMTP_HOST=smtp.gmail.com:587
      - GF_SMTP_USER=9mahin.ahmad@gmail.com
      - GF_SMTP_PASSWORD=16-char-app-password from https://myaccount.google.com/apppasswords
      - GF_SMTP_FROM_ADDRESS=your-email@gmail.com
      - GF_SMTP_FROM_NAME=Grafana Alerting
      - GF_SMTP_STARTTLS_POLICY=Mandatory

```

#### Alert Firing:

If I cause the webapp to malfunction: ``
Then prometheus target throws error > grafana alert is fired > email is sent to my inbox:  
<img width="1310" height="451" alt="image" src="https://github.com/user-attachments/assets/c45a45a4-2f56-4949-a235-081435e6b7e1" />  

<img width="1212" height="597" alt="image" src="https://github.com/user-attachments/assets/983a7d04-e892-4b1d-947d-6a7ad39e2752" />  
<img width="1281" height="208" alt="image" src="https://github.com/user-attachments/assets/8fb76367-6b58-4afc-ac7d-2d1de4867f5d" />  
<img width="916" height="371" alt="image" src="https://github.com/user-attachments/assets/de374175-e514-4aa2-8ccd-1f65ee414d51" />  

**After restarting the container**, the down status of prometheus and grafana is back to normal.  

### Troubleshooting:

**Grafana cannnot connect to loki or "Data source connected, but no labels were received. Verify that Loki and Promtail are correctly configured."**

Go to `alloy-values.yaml` and change this block:
```
loki.write "loki_endpoint" {
  endpoint {
    // internval
    url = "http://172.31.3.125:3100/loki/api/v1/push"
  }
}

```  
Refresh the grafana page and click 'test and save' again.  

More issues:  

<img width="772" height="86" alt="image" src="https://github.com/user-attachments/assets/cf64b842-c90e-43cb-8fc9-ab585c56c9fa" />  

correct alloy-values.yaml

```

cat alloy-values.yaml
controller:
  type: daemonset

alloy:
  configMap:
    content: |
      logging {
        level  = "info"
        format = "logfmt"
      }

      // 1. Discover Kubernetes Pods
      discovery.k8s "pods" {
        role = "pod"
      }

      discovery.relabel "pod_logs" {
        targets = discovery.k8s.pods.targets

        rule {
          source_labels = ["__meta_kubernetes_namespace"]
          action        = "replace"
          target_label  = "namespace"
        }
        rule {
          source_labels = ["__meta_kubernetes_pod_name"]
          action        = "replace"
          target_label  = "pod"
        }
        rule {
          source_labels = ["__meta_kubernetes_pod_container_name"]
          action        = "replace"
          target_label  = "container"
        }
      }

      // 2. Collect Kubernetes Pod logs natively
      loki.source.kubernetes "pods" {
        targets    = discovery.relabel.pod_logs.output
        forward_to = [loki.write.loki_endpoint.receiver]
      }

      // 3. Forward logs to your host-based Loki instance
      loki.write "loki_endpoint" {
        endpoint {
          url = "http://172.31.3.125:3100/loki/api/v1/push"
        }
      }
      
```
Apply using: `helm upgrade alloy grafana/alloy -n monitoring -f alloy-values.yaml`.  Make sure that `kubectl get configmap alloy -n monitoring -o yaml | grep -c loki.source.kubernetes` returns 1




