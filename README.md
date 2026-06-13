Enterprise Infrastructure Documentation: High-Availability Load Balancing & Observability Stack
This documentation outlines the architecture, deployment strategy, and troubleshooting methodology implemented for the MaxSMS infrastructure task. The solution leverages Traefik v3 as a high-availability reverse proxy and dynamic load balancer, an automated WhoAmI application cluster running multiple replicas, and an infrastructure observability layer via Google cAdvisor.

1. System Architecture Overview
The infrastructure is designed using a decoupled, modular architecture where services are isolated into logical directories but communicate seamlessly via a unified external Docker overlay network layer.

Plaintext
                                 [ Internet Traffic ]
                                          │
                                   Ports 80 / 443
                                          ▼
                             ┌─────────────────────────┐
                             │  Traefik Core Gateway   │ (Dynamic Routing & SSL)
                             └────────────┬────────────┘
                                          │
                           Shared Network: traefik-public
                                          │
                  ┌───────────────────────┼───────────────────────┐
                  ▼                       ▼                       ▼
      ┌───────────────────────┐ ┌───────────────────────┐ ┌───────────────────────┐
      │  maxsms_whoami (R1)   │ │  maxsms_whoami (R2)   │ │  maxsms_whoami (R3)   │
      │    Port 80 (Internal) │ │    Port 80 (Internal) │ │    Port 80 (Internal) │
      └───────────────────────┘ └───────────────────────┘ └───────────────────────┘
                                          │
                                          ▼
                              ┌───────────────────────┐
                              │  monitoring_cadvisor  │ (Host Metric Export)
                              │  Port 8080 (Internal) │
                              └───────────────────────┘
Infrastructure Components
Traefik Gateway: Acts as the Single Point of Entry, managing SSL/TLS termination automatically via Let's Encrypt and performing Round-Robin load balancing.

Application Replicas: Three stateless, identical instances of the application container to guarantee horizontal scaling and high availability.

Observability Node: A containerized cAdvisor instance with native daemon-level access to the host subsystems to gather real-time execution metrics.

2. Directory Structure
The absolute directory hierarchy inside the project repository is organized as follows:

Plaintext
/root/maxsms/
├── docker-compose.yml       # Application deployment spec & replica definitions
├── traefik/
│   ├── docker-compose.yml   # Core reverse proxy engine configuration
│   └── acme.json            # Encrypted keystore for Let's Encrypt certificates
└── monitoring/
    └── docker-compose.yml   # cAdvisor container spec for system metrics
3. Step-by-Step Deployment Runbook
Follow this execution sequence strictly to provision the environment on a clean virtual private server.

Step 3.1: Global Network Initialization
Before bringing up any containers, initialize the isolated bridging network:

Bash
docker network create traefik-public
Step 3.2: Gateway Provisioning (Traefik)
Navigate to the proxy directory, enforce secure file permissions for the SSL storage configuration, and run the service orchestration:

Bash
cd /root/maxsms/traefik
touch acme.json
chmod 600 acme.json
docker compose up -d --force-recreate
Step 3.3: Cluster Scaling (MaxSMS Application)
Deploy the horizontal core instances. The engine dynamically populates the upstream pools via Docker API sockets:

Bash
cd /root/maxsms
docker compose up -d --remove-orphans
Step 3.4: Metric Agent Deployment (cAdvisor)
Deploy the monitoring container linked to the dynamic backend routers:

Bash
cd /root/maxsms/monitoring
docker compose up -d --force-recreate
4. Production Configuration Manifests
4.1: Traefik Engine (traefik/docker-compose.yml)
YAML
services:
  traefik:
    image: traefik:v3.6.6
    container_name: traefik
    restart: always
    security_opt:
      - no-new-privileges:true
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - ./acme.json:/acme.json
    networks:
      - traefik-public
    command:
      - "--providers.docker=true"
      - "--providers.docker.exposedbydefault=false"
      - "--providers.docker.network=traefik-public"
      - "--entrypoints.web.address=:80"
      - "--entrypoints.web.http.redirections.entrypoint.to=websecure"
      - "--entrypoints.web.http.redirections.entrypoint.scheme=https"
      - "--entrypoints.websecure.address=:443"
      - "--certificatesresolvers.myresolver.acme.httpchallenge=true"
      - "--certificatesresolvers.myresolver.acme.httpchallenge.entrypoint=web"
      - "--certificatesresolvers.myresolver.acme.email=yasharrezvaniofficial@gmail.com"
      - "--certificatesresolvers.myresolver.acme.storage=/acme.json"

networks:
  traefik-public:
    external: true
4.2: Application Stack (./docker-compose.yml)
YAML
services:
  whoami:
    image: traefik/whoami:latest
    restart: always
    deploy:
      replicas: 3
    networks:
      - traefik-public
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.maxsms.rule=Host(`devopspractice.ir`)"
      - "traefik.http.routers.maxsms.entrypoints=websecure"
      - "traefik.http.routers.maxsms.tls.certresolver=myresolver"
      - "traefik.http.services.maxsms.loadbalancer.server.port=80"

networks:
  traefik-public:
    external: true
4.3: Observability Stack (monitoring/docker-compose.yml)
YAML
services:
  cadvisor:
    image: gcr.io/cadvisor/cadvisor:v0.49.1
    container_name: monitoring_cadvisor
    restart: always
    privileged: true
    devices:
      - /dev/kmsg
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
      - /dev/disk/:/dev/disk:ro
    networks:
      - traefik-public
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.cadvisor.rule=Host(`cadvisor.devopspractice.ir`)"
      - "traefik.http.routers.cadvisor.entrypoints=websecure"
      - "traefik.http.routers.cadvisor.tls.certresolver=myresolver"
      - "traefik.http.services.cadvisor.loadbalancer.server.port=8080"

networks:
  traefik-public:
    external: true
5. Engineering Challenges & Core Resolutions
5.1: Multi-Compose Service Discovery Isolation (HTTP 404 Resolution)
Problem: When separating stack definitions across individual directories, Traefik initially dropped incoming requests intended for the monitoring cluster, emitting an HTTP 404 Page Not Found error. This occurred because Traefik's internal Docker provider could not determine which network card to use when communicating across decoupled compose structures.

Resolution: The core gateway command abstraction was appended with an explicit network scoping declaration: --providers.docker.network=traefik-public. This optimization instructed Traefik to uniquely evaluate cross-stack labels through the shared external ingress scope, solving the discovery isolation bug.

5.2: Traefik v3 Dynamic Routing & Versioning Overhaul
Problem: During the configuration of dynamic middleware rules, compatibility errors arose within Traefik's internal schema parser due to subtle design shifts between legacy architectures (v2) and modern environments (v3). Routing engines failed to properly register subdirectories and ports because of deprecated parsing patterns within the label configurations.

Resolution: This specific regression was resolved by cross-referencing upstream validation cases found in Traefik Issue #12253 (https://github.com/traefik/traefik/issues/12253). By adapting the labels to the strict v3 semantic requirements—specifically separating clear network constraints from dynamic routing protocols—we successfully bypassed configuration lockups and stabilized the container backend discovery lifecycle.

5.3: Let's Encrypt Ingress Rate Limiting
Problem: Frequent infrastructure teardowns and continuous validation requests targeting the domain led to temporary ACME API throttling (Rate Limits) from Let's Encrypt authority endpoints. This caused SSL negotiations to temporarily fail, displaying untrusted fallback certificates on valid domain targets.

Resolution: This was mitigated by purging the transactional metadata via a specialized clean execution sequence: wiping the acme.json local state and executing a forced orchestration recreation via the --force-recreate operational flag. To maintain development velocity under strict limit states, external cloud edge proxies (Cloudflare DNS proxy layers) can alternatively be toggled to handle instant downstream TLS offloading without exhausting localized origin ACME quotas.

6. Infrastructure Verification & Live Browser Testing
The infrastructure deployment can be fully validated live using any web browser.

6.1: Application & Load Balancer Verification (WhoAmI Stack)
To test the dynamic Round-Robin load balancing mechanism across the horizontally scaled container replicas, open your browser and access the root endpoint:

Target URL: https://devopspractice.ir

Browser Testing Procedure:
Load the URL in a browser tab. The page will render standard plain text container metadata.

Observe the Hostname value (e.g., Hostname: df63af6b46fc).

Force-refresh the browser (Ctrl + F5 or Cmd + Shift + R).

You will observe that the Hostname dynamically shifts between three distinct, unique container alphanumeric hashes on subsequent requests. This behavior confirms that Traefik is actively distributing ingress traffic sequentially across the 3 independent stateless application nodes.

6.2: Infrastructure Observability Verification (cAdvisor Stack)
To monitor daemon-level container analytics, resource usage histories, and memory profiles via the web control panel, navigate to the dedicated metrics endpoint:

Target URL: https://cadvisor.devopspractice.ir

Browser Testing Procedure:
Open the subdomain URL in your web browser.

The page will immediately load the official native graphical dashboard interface provided by Google cAdvisor.

Users can navigate into specific Docker subnet pathways to inspect real-time CPU consumption, active sub-process isolation layers, network interface memory pipelines, and overall cluster stability metrics.
