# 🗺️ Diagramas de Arquitectura

## Introducción

Esta sección presenta diagramas técnicos detallados que muestran la arquitectura del sistema desde diferentes perspectivas: infraestructura, flujo de datos, redes y despliegue.

## 🏗️ Arquitectura de Infraestructura

### Vista General del Sistema

```mermaid
graph TB
    subgraph "Internet"
        USER[👤 Usuarios]
        DNS[🌐 Namecheap DNS]
        LE[🔒 Let's Encrypt]
    end
    
    subgraph "DigitalOcean Cloud"
        subgraph "Cloud Firewall"
            FW_SSH[SSH:22]
            FW_HTTP[HTTP:80]
            FW_HTTPS[HTTPS:443]
        end
        
        subgraph "Ubuntu 22.04 Droplet"
            subgraph "Host Services"
                CADDY[🔒 Caddy v2]
                FAIL2BAN[🛡️ Fail2ban]
                DOCKER[🐳 Docker Engine]
                UFW[🔥 ufw Firewall]
            end
            
            subgraph "Docker Network (fail2ban_network)"
                API[⚙️ FastAPI Container]
                LOKI[🗄️ Loki Container]
                PROMTAIL[📜 Promtail Container]
            end
            
            subgraph "File System"
                LOGS[📁 /var/log/fail2ban.log]
                SOCKET[🔌 /var/run/fail2ban/fail2ban.sock]
                LOKI_DATA[💾 loki_data volume]
            end
        end
    end
    
    USER -->|HTTPS Request| DNS
    DNS -->|Resolves to| FW_HTTPS
    FW_HTTPS --> CADDY
    
    CADDY -->|Reverse Proxy| API
    CADDY <-->|Certificate Validation| LE
    
    API -->|Unix Socket| SOCKET
    API -->|HTTP Queries| LOKI
    
    FAIL2BAN --> LOGS
    FAIL2BAN --> SOCKET
    
    PROMTAIL -->|Reads| LOGS
    PROMTAIL -->|Ships Logs| LOKI
    
    LOKI --> LOKI_DATA
    
    DOCKER -->|Manages| API
    DOCKER -->|Manages| LOKI
    DOCKER -->|Manages| PROMTAIL
```

## 🌐 Arquitectura de Red

### Configuración de Puertos y Firewall

```mermaid
graph LR
    subgraph "External Network"
        INTERNET[🌍 Internet]
    end
    
    subgraph "DigitalOcean Cloud Firewall"
        CF_22[Port 22/TCP<br/>SSH Access]
        CF_80[Port 80/TCP<br/>HTTP<br/>Let's Encrypt]
        CF_443[Port 443/TCP<br/>HTTPS<br/>Production Traffic]
        CF_BLOCK[❌ Port 8000/TCP<br/>BLOCKED<br/>Direct API Access]
    end
    
    subgraph "Droplet Internal Network"
        subgraph "Host Ports"
            H_22[22 → SSH]
            H_80[80 → Caddy]
            H_443[443 → Caddy]
            H_8000[8000 → API Container]
        end
        
        subgraph "Docker Bridge Network"
            D_API[API:8000]
            D_LOKI[Loki:3100]
            D_PROMTAIL[Promtail:9080]
        end
    end
    
    INTERNET --> CF_22
    INTERNET --> CF_80
    INTERNET --> CF_443
    INTERNET -.->|BLOCKED| CF_BLOCK
    
    CF_22 --> H_22
    CF_80 --> H_80
    CF_443 --> H_443
    
    H_80 --> D_API
    H_443 --> D_API
    H_8000 --> D_API
    
    D_API <--> D_LOKI
    D_PROMTAIL --> D_LOKI
```

### Flujo de Red Detallado

!!! info "Configuración de Seguridad de Red"
    === "Puertos Expuestos"
        - **22/TCP**: SSH para administración (restringir por IP)
        - **80/TCP**: HTTP para validación Let's Encrypt
        - **443/TCP**: HTTPS para tráfico de producción
    
    === "Puertos Internos"
        - **8000/TCP**: API FastAPI (solo localhost)
        - **3100/TCP**: Loki (solo red Docker)
        - **9080/TCP**: Promtail (solo red Docker)
    
    === "Bloqueos Críticos"
        - ❌ Puerto 8000 **NO** debe estar abierto en Cloud Firewall
        - ❌ Puertos 3100, 9080 **NO** deben ser accesibles externamente

## 📊 Flujo de Datos y Procesamiento

### Pipeline Completo de Logs

```mermaid
sequenceDiagram
    participant SS as Suspicious Source
    participant SRV as Server Service
    participant FB as Fail2ban
    participant FS as File System
    participant PT as Promtail
    participant LK as Loki
    participant API as FastAPI
    participant CD as Caddy
    participant USR as User Dashboard
    
    Note over SS,SRV: 1. Intento de Intrusión
    SS->>+SRV: Múltiples intentos SSH fallidos
    SRV->>FS: Escribe en /var/log/auth.log
    
    Note over FB: 2. Detección y Baneo
    FB->>FS: Monitorea auth.log
    FB->>FB: Detecta patrón malicioso
    FB->>SRV: Aplica regla iptables
    FB->>FS: Escribe en fail2ban.log
    
    Note over PT,LK: 3. Recolección de Logs
    PT->>FS: Lee fail2ban.log
    PT->>PT: Procesa con regex
    PT->>+LK: Envía logs estructurados
    LK->>LK: Indexa por etiquetas
    
    Note over API,USR: 4. Visualización
    USR->>CD: Accede al dashboard
    CD->>API: Proxy HTTPS request
    API->>LK: Query logs recientes
    LK->>API: Retorna datos JSON
    API->>CD: Response con datos
    CD->>USR: Dashboard actualizado
    
    Note over USR: 5. Acción del Usuario
    USR->>CD: Solicita desbanear IP
    CD->>API: POST /jails/sshd/unban
    API->>FB: fail2ban-client unban
    FB->>SRV: Remueve regla iptables
    FB->>FS: Log de desbaneo
```

### Estados del Sistema

```mermaid
stateDiagram-v2
    [*] --> Normal
    
    Normal --> Suspicious: Detecta actividad maliciosa
    Suspicious --> Analyzing: Fail2ban evalúa patrones
    
    Analyzing --> Banned: Excede threshold
    Analyzing --> Normal: Dentro de límites
    
    Banned --> Monitoring: IP baneada
    Monitoring --> Banned: Mantiene baneo
    Monitoring --> Normal: Tiempo de baneo expirado
    
    Banned --> Normal: Admin desbanea manualmente
    
    state Monitoring {
        [*] --> Active
        Active --> Logging: Registra actividad
        Logging --> Active
    }
```

## 🔄 Arquitectura de Despliegue

### Flujo de CI/CD y Despliegue

```mermaid
graph TB
    subgraph "Development"
        DEV[👨‍💻 Developer]
        GIT[📚 Git Repository]
    end
    
    subgraph "Build & Deploy"
        GHA[🔄 GitHub Actions]
        DOCKER_BUILD[🐳 Docker Build]
        DOCKER_PUSH[📤 Docker Push]
    end
    
    subgraph "Production Server"
        PULL[📥 Docker Pull]
        COMPOSE[🎼 Docker Compose]
        HEALTH[🏥 Health Checks]
    end
    
    subgraph "Monitoring"
        LOGS_MON[📊 Log Monitoring]
        ALERTS[🚨 Alerts]
        METRICS[📈 Metrics]
    end
    
    DEV --> GIT
    GIT --> GHA
    
    GHA --> DOCKER_BUILD
    DOCKER_BUILD --> DOCKER_PUSH
    
    DOCKER_PUSH --> PULL
    PULL --> COMPOSE
    COMPOSE --> HEALTH
    
    HEALTH --> LOGS_MON
    LOGS_MON --> ALERTS
    LOGS_MON --> METRICS
    
    ALERTS -->|Notifica| DEV
```

### Estrategia de Actualización

```mermaid
graph LR
    subgraph "Actualización Zero-Downtime"
        BACKUP[💾 Backup Data]
        PULL_NEW[📥 Pull New Images]
        STOP_OLD[⏹️ Stop Old Containers]
        START_NEW[▶️ Start New Containers]
        VERIFY[✅ Verify Health]
        ROLLBACK[🔄 Rollback if Error]
    end
    
    BACKUP --> PULL_NEW
    PULL_NEW --> STOP_OLD
    STOP_OLD --> START_NEW
    START_NEW --> VERIFY
    VERIFY -->|❌ Error| ROLLBACK
    VERIFY -->|✅ Success| END[🎉 Deploy Complete]
    ROLLBACK --> STOP_OLD
```

## 🔐 Arquitectura de Seguridad

### Capas de Seguridad

```mermaid
graph TB
    subgraph "Security Layers"
        subgraph "Layer 1: Cloud Infrastructure"
            CF[🛡️ DigitalOcean Cloud Firewall]
            DOS[🚫 DDoS Protection]
        end
        
        subgraph "Layer 2: Host Level"
            UFW[🔥 ufw Firewall]
            FAIL2BAN[🛡️ Fail2ban IPS]
            SSH_KEY[🔑 SSH Key Auth]
        end
        
        subgraph "Layer 3: Application Level"
            TLS[🔒 TLS/SSL Encryption]
            CADDY_SEC[🛡️ Caddy Security Headers]
            API_AUTH[🔐 API Authentication]
        end
        
        subgraph "Layer 4: Container Level"
            DOCKER_NET[🌐 Isolated Docker Networks]
            CONTAINER_USER[👤 Non-root Containers]
            VOLUME_PERM[📁 Volume Permissions]
        end
    end
    
    INTERNET[🌍 Internet] --> CF
    CF --> DOS
    DOS --> UFW
    UFW --> FAIL2BAN
    FAIL2BAN --> SSH_KEY
    SSH_KEY --> TLS
    TLS --> CADDY_SEC
    CADDY_SEC --> API_AUTH
    API_AUTH --> DOCKER_NET
    DOCKER_NET --> CONTAINER_USER
    CONTAINER_USER --> VOLUME_PERM
    VOLUME_PERM --> SECURE_APP[🎯 Secure Application]
```

### Flujo de Autenticación y Autorización

```mermaid
sequenceDiagram
    participant C as Client
    participant CF as Cloud Firewall
    participant CD as Caddy
    participant API as FastAPI
    participant FB as Fail2ban
    
    Note over C,FB: Security Check Flow
    
    C->>CF: Request to server
    CF->>CF: Check allowed ports
    alt Port not allowed
        CF->>C: ❌ Block request
    else Port allowed
        CF->>CD: Forward request
    end
    
    CD->>CD: Verify SSL certificate
    CD->>CD: Apply security headers
    CD->>API: Proxy to API
    
    API->>API: Validate request format
    API->>API: Check rate limits
    
    alt Rate limit exceeded
        API->>FB: Report suspicious activity
        FB->>FB: Evaluate for banning
        API->>CD: ❌ Rate limit error
    else Request valid
        API->>API: Process request
        API->>CD: ✅ Success response
    end
    
    CD->>C: Final response
```

## 📈 Arquitectura de Monitoreo

### Sistema de Observabilidad

```mermaid
graph TB
    subgraph "Data Sources"
        APP_LOGS[📝 Application Logs]
        SYS_LOGS[📋 System Logs]
        FAIL2BAN_LOGS[🛡️ Fail2ban Logs]
        CADDY_LOGS[🔒 Caddy Logs]
    end
    
    subgraph "Collection Layer"
        PROMTAIL[📜 Promtail Agent]
        FILEBEAT[📊 Log Shippers]
    end
    
    subgraph "Storage & Processing"
        LOKI[🗄️ Loki Log Store]
        ELASTIC[🔍 Search Engine]
    end
    
    subgraph "Visualization"
        API_ENDPOINT[⚙️ API Endpoints]
        DASHBOARD[📊 Web Dashboard]
        GRAFANA[📈 Grafana (Optional)]
    end
    
    subgraph "Alerting"
        WEBHOOKS[🔗 Webhooks]
        EMAIL[📧 Email Alerts]
        SLACK[💬 Slack Integration]
    end
    
    APP_LOGS --> PROMTAIL
    SYS_LOGS --> PROMTAIL
    FAIL2BAN_LOGS --> PROMTAIL
    CADDY_LOGS --> FILEBEAT
    
    PROMTAIL --> LOKI
    FILEBEAT --> ELASTIC
    
    LOKI --> API_ENDPOINT
    ELASTIC --> API_ENDPOINT
    
    API_ENDPOINT --> DASHBOARD
    API_ENDPOINT --> GRAFANA
    
    API_ENDPOINT --> WEBHOOKS
    WEBHOOKS --> EMAIL
    WEBHOOKS --> SLACK
```

## 🚀 Escalabilidad y Alta Disponibilidad

### Arquitectura Escalable

```mermaid
graph TB
    subgraph "Load Balancer Tier"
        LB[⚖️ Load Balancer]
        CDN[🌐 CDN (Optional)]
    end
    
    subgraph "Application Tier"
        API1[⚙️ API Instance 1]
        API2[⚙️ API Instance 2]
        API3[⚙️ API Instance N]
    end
    
    subgraph "Data Tier"
        LOKI_CLUSTER[🗄️ Loki Cluster]
        DB_BACKUP[💾 Database Backup]
    end
    
    subgraph "Monitoring Tier"
        PROMTAIL_MULTI[📜 Promtail Instances]
        METRICS[📊 Metrics Collection]
    end
    
    CDN --> LB
    LB --> API1
    LB --> API2
    LB --> API3
    
    API1 --> LOKI_CLUSTER
    API2 --> LOKI_CLUSTER
    API3 --> LOKI_CLUSTER
    
    LOKI_CLUSTER --> DB_BACKUP
    
    PROMTAIL_MULTI --> LOKI_CLUSTER
    METRICS --> LOKI_CLUSTER
```

!!! tip "Consideraciones de Escalabilidad"
    === "Escala Horizontal"
        - **Múltiples instancias API** detrás de load balancer
        - **Cluster de Loki** para mayor capacidad de almacenamiento
        - **Múltiples Promtail** para recolección distribuida
    
    === "Escala Vertical"
        - **Incrementar recursos** de los contenedores existentes
        - **Optimizar consultas** a Loki para mejor rendimiento
        - **Tuning de configuraciones** para mayor throughput
    
    === "Alta Disponibilidad"
        - **Health checks** automáticos en todos los servicios
        - **Failover automático** con restart policies
        - **Backups regulares** de datos críticos
        - **Monitoreo proactivo** con alertas

!!! success "Próximo Paso"
    Ahora que entiendes la arquitectura completa, continúa con la [configuración del servidor](../servidor/droplet-setup.md) para implementar este sistema paso a paso.