# observability-project-ec2-k8s-grafana-prom-loki

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

