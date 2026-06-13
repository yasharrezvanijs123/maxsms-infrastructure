<h1 align="center">🏢 Enterprise Infrastructure Documentation</h1>
<h3 align="center">High-Availability Load Balancing & Observability Stack</h3>

<p align="center">
  <img src="https://img.shields.io/badge/Version-v3.6.6-blue?style=for-the-badge&logo=traefik" alt="Traefik Version" />
  <img src="https://img.shields.io/badge/Status-Production--Ready-green?style=for-the-badge&logo=docker" alt="Deployment Status" />
  <img src="https://img.shields.io/badge/Architecture-ARM64%20%2F%20AMD64-orange?style=for-the-badge" alt="Architecture" />
</p>

<hr />

<h2 align="left">🗺️ 1. System Architecture Overview</h2>

<p align="left">The infrastructure leverages a decoupled, modular architecture where services are isolated into logical directories but communicate seamlessly via a unified external Docker overlay network layer.</p>

<pre>
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
</pre>

<h3 align="left">⚙️ Core Components</h3>
<ul>
  <li><strong>Traefik Gateway:</strong> Single Point of Entry, managing SSL/TLS termination automatically via Let's Encrypt and performing Round-Robin load balancing.</li>
  <li><strong>Application Replicas:</strong> Three stateless, identical instances of the application container to guarantee horizontal scaling.</li>
  <li><strong>Observability Node:</strong> A containerized cAdvisor instance with native daemon-level access to the host subsystems.</li>
</ul>

<hr />

<h2 align="left">📂 2. Directory Structure</h2>

<pre>
/root/maxsms/
├── docker-compose.yml       # Application deployment spec & replica definitions
├── traefik/
│   ├── docker-compose.yml   # Core reverse proxy engine configuration
│   └── acme.json            # Encrypted keystore for Let's Encrypt certificates
└── monitoring/
    └── docker-compose.yml   # cAdvisor container spec for system metrics
</pre>

<hr />

<h2 align="left">🚀 3. Step-by-Step Deployment Runbook</h2>

<h4 align="left">Step 3.1: Global Network Initialization</h4>
<p align="left">Before bringing up any containers, initialize the isolated bridging network:</p>
<pre><code>docker network create traefik-public</code></pre>

<h4 align="left">Step 3.2: Gateway Provisioning (Traefik)</h4>
<p align="left">Navigate to the proxy directory, enforce secure file permissions for the SSL storage configuration, and run the service orchestration:</p>
<pre><code>cd /root/maxsms/traefik
touch acme.json && chmod 600 acme.json
docker compose up -d --force-recreate</code></pre>

<h4 align="left">Step 3.3: Cluster Scaling (MaxSMS Application)</h4>
<p align="left">Deploy the horizontal core instances. The engine dynamically populates the upstream pools via Docker API sockets:</p>
<pre><code>cd /root/maxsms
docker compose up -d --remove-orphans</code></pre>

<h4 align="left">Step 3.4: Metric Agent Deployment (cAdvisor)</h4>
<p align="left">Deploy the monitoring container linked to the dynamic backend routers:</p>
<pre><code>cd /root/maxsms/monitoring
docker compose up -d --force-recreate</code></pre>

<hr />

<h2 align="left">📝 4. Production Configuration Manifests</h2>

<h3 align="left">🛠️ 4.1: Traefik Engine (traefik/docker-compose.yml)</h3>
<pre><code>services:
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
    external: true</code></pre>

<h3 align="left">📦 4.2: Application Stack (./docker-compose.yml)</h3>
<pre><code>services:
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
    external: true</code></pre>

<h3 align="left">📊 4.3: Observability Stack (monitoring/docker-compose.yml)</h3>
<pre><code>services:
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
    external: true</code></pre>

<hr />

<h2 align="left">🧠 5. Engineering Challenges & Core Resolutions</h2>
<ul>
  <li>
    <strong>5.1: Multi-Compose Service Discovery Isolation (HTTP 404 Resolution)</strong>
    <br /><em>Problem:</em> Traefik dropped incoming requests intended for the decoupled monitoring cluster, emitting an HTTP 404.
    <br /><em>Resolution:</em> Appended explicit network scoping declaration: <code>--providers.docker.network=traefik-public</code>.
  </li>
  <li>
    <strong>5.2: Traefik v3 Dynamic Routing & Versioning Overhaul</strong>
    <br /><em>Problem:</em> Dynamic middleware schema errors due to syntactic design shifts between v2 and v3.
    <br /><em>Resolution:</em> Overhauled labels based on strict v3 semantic requirements cross-referenced in <a href="https://github.com/traefik/traefik/issues/12253" target="_blank">Traefik Issue #12253</a>.
  </li>
  <li>
    <strong>5.3: Let's Encrypt Ingress Rate Limiting</strong>
    <br /><em>Problem:</em> Frequent teardowns caused temporary ACME API throttling.
    <br /><em>Resolution:</em> Purged transactional metadata via wiping <code>acme.json</code> state and executing forced container recreation via <code>--force-recreate</code>.
  </li>
</ul>

<hr />

<h2 align="left">🌐 6. Infrastructure Verification & Live Browser Testing</h2>
<p align="left">The live deployment endpoints are active and can be verified directly via any web browser interface:</p>

<h3 align="left">🔗 Production Links:</h3>
<h4 align="left">🔗 Use VPN</h4>
<p align="left">
  <a href="https://devopspractice.ir" target="_blank">
    <img src="https://img.shields.io/badge/Application%20Endpoint-https%3A%2F%2Fdevopspractice.ir-blue?style=for-the-badge&logo=nginx&logoColor=white" alt="App Endpoint" />
  </a>
  <a href="https://cadvisor.devopspractice.ir" target="_blank">
    <img src="https://img.shields.io/badge/Metrics%20Endpoint-https%3A%2F%2Fcadvisor.devopspractice.ir-orange?style=for-the-badge&logo=prometheus&logoColor=white" alt="Metrics Endpoint" />
  </a>
</p>

<h3 align="left">🔍 6.1: How to Test Load Balancing (WhoAmI Cluster in Browser)</h3>
<ol>
  <li>Click the <strong>Application Endpoint</strong> badge above to open the target domain in a new browser tab.</li>
  <li>Look for the explicit metadata row stating <strong><code>Hostname: [alphanumeric_hash]</code></strong>.</li>
  <li>Force-refresh the browser tab (<code>Ctrl + F5</code> on Windows/Linux or <code>Cmd + Shift + R</code> on macOS).</li>
  <li><strong>Expected Live Output:</strong> The <code>Hostname</code> hash value will dynamically switch and rotate between three unique container hashes on subsequent refreshes. This explicitly validates that Traefik's internal Round-Robin dynamic balancing engine is distributing ingress workflows across the 3 backend replicas.</li>
</ol>

<h3 align="left">📈 6.2: How to Test Container Metrics (cAdvisor Panel in Browser)</h3>
<ol>
  <li>Click the <strong>Metrics Endpoint</strong> badge above to open the monitoring subdomain.</li>
  <li>The active browser tab will instantly render the native web control interface provided by <strong>Google cAdvisor</strong>.</li>
  <li>Users can navigate into the localized Docker runtime sub-links to monitor live CPU graphs, core daemon system states, memory cache bounds, and active host subprocesses isolation logs without terminal dependencies.</li>
</ol>
