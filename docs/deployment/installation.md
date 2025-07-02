# 🚀 Instalación del Sistema

## Introducción

Esta guía te llevará paso a paso por la instalación completa del sistema Fail2ban Dashboard en un Droplet de DigitalOcean desde cero.

## 📋 Requisitos Previos

### Infraestructura Mínima

!!! info "Especificaciones del Servidor"
    - **Proveedor**: DigitalOcean Droplet
    - **SO**: Ubuntu 22.04 LTS
    - **RAM**: Mínimo 2GB (recomendado 4GB)
    - **Almacenamiento**: Mínimo 20GB SSD
    - **CPU**: 1 vCPU (recomendado 2 vCPUs)
    - **Dominio**: Configurado y apuntando al Droplet

### Servicios Externos

=== "Dominio y DNS"
    - Dominio registrado (ej. Namecheap)
    - Record A apuntando al IP del Droplet
    - Record AAAA para IPv6 (opcional)

=== "Acceso SSH"
    - Par de claves SSH generadas
    - Clave pública agregada al Droplet

## 🔧 Preparación del Servidor

### 1. Configuración Inicial del Usuario

Conéctate al Droplet y crea un usuario no-root:

```bash
# Conectar como root
ssh root@tu_droplet_ip

# Crear usuario no-root
adduser makuno

# Agregar a grupo sudo
usermod -aG sudo makuno

# Cambiar a nuevo usuario
su - makuno
```

### 2. Configuración SSH

```bash
# Crear directorio SSH para el nuevo usuario
mkdir -p ~/.ssh
chmod 700 ~/.ssh

# Copiar clave pública autorizada
sudo cp /root/.ssh/authorized_keys ~/.ssh/
sudo chown makuno:makuno ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys

# Configurar SSH para mayor seguridad
sudo nano /etc/ssh/sshd_config
```

Configuración SSH recomendada:

```ini
# /etc/ssh/sshd_config
PasswordAuthentication no
ChallengeResponseAuthentication no
PermitRootLogin prohibit-password
PubkeyAuthentication yes
```

```bash
# Reiniciar SSH
sudo systemctl restart sshd
```

### 3. Actualización del Sistema

```bash
# Actualizar repositorios y paquetes
sudo apt update && sudo apt upgrade -y

# Instalar paquetes esenciales
sudo apt install -y curl wget git ufw htop neofetch
```

## 🔐 Configuración de Firewall

### Cloud Firewall (DigitalOcean)

Configura las reglas en el panel de DigitalOcean:

| Tipo | Protocolo | Puerto | Origen |
|------|-----------|---------|---------|
| **SSH** | TCP | 22 | Tu IP o IPs específicas |
| **HTTP** | TCP | 80 | All IPv4, All IPv6 |
| **HTTPS** | TCP | 443 | All IPv4, All IPv6 |

!!! danger "Importante"
    **NO** abras el puerto 8000 en el Cloud Firewall. La API debe ser accesible solo a través de Caddy.

### ufw (Opcional)

Si quieres una capa adicional de firewall:

```bash
# Habilitar ufw
sudo ufw enable

# Permitir SSH
sudo ufw allow 22/tcp

# Permitir HTTP y HTTPS
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp

# Verificar estado
sudo ufw status verbose
```

## 🛠️ Instalación de Software Base

### 1. Docker Engine

```bash
# Desinstalar versiones anteriores
sudo apt remove -y docker docker-engine docker.io containerd runc

# Instalar dependencias
sudo apt install -y apt-transport-https ca-certificates curl gnupg lsb-release

# Agregar clave GPG oficial de Docker
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

# Agregar repositorio
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Instalar Docker
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin

# Agregar usuario al grupo docker
sudo usermod -aG docker $USER

# Habilitar Docker al arranque
sudo systemctl enable docker
sudo systemctl start docker
```

!!! tip "Reiniciar Sesión"
    Cierra y abre una nueva sesión SSH para que los cambios del grupo docker tomen efecto.

### 2. Fail2ban

```bash
# Instalar Fail2ban
sudo apt install -y fail2ban

# Habilitar al arranque
sudo systemctl enable fail2ban
sudo systemctl start fail2ban

# Verificar estado
sudo systemctl status fail2ban
```

### 3. Caddy

```bash
# Instalar dependencias
sudo apt install -y debian-keyring debian-archive-keyring apt-transport-https

# Agregar clave GPG de Caddy
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | sudo gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg

# Agregar repositorio
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' | sudo tee /etc/apt/sources.list.d/caddy-stable.list

# Instalar Caddy
sudo apt update
sudo apt install -y caddy

# Habilitar al arranque
sudo systemctl enable caddy
```

## 📥 Despliegue del Proyecto

### 1. Clonación del Repositorio

```bash
# Navegar al directorio home
cd ~

# Clonar el repositorio del proyecto
git clone https://github.com/tu-usuario/aca-fail2ban-dashboard.git
cd aca-fail2ban-dashboard

# Verificar estructura del proyecto
ls -la
```

### 2. Configuración de Variables de Entorno

```bash
# Copiar archivo de ejemplo
cp .env.example .env

# Editar variables de entorno
nano .env
```

Configuración `.env` requerida:

```bash
# .env
# =================================
# Configuración de la API
# =================================
API_HOST=0.0.0.0
API_PORT=8000
API_RELOAD=false

# =================================
# Configuración de Loki
# =================================
LOKI_URL=http://loki:3100
LOKI_QUERY_URL=http://loki:3100/loki/api/v1/query_range
LOKI_WS_URL=ws://loki:3100/loki/api/v1/tail

# =================================
# Configuración de Fail2ban
# =================================
FAIL2BAN_LOG_PATH=/var/log/fail2ban.log
FAIL2BAN_SOCKET_PATH=/var/run/fail2ban/fail2ban.sock

# =================================
# Configuración de Dominio
# =================================
DOMAIN_NAME=alertasfail2ban.xmakuno.com
API_BASE_URL=https://alertasfail2ban.xmakuno.com

# =================================
# Configuración de Logs
# =================================
LOG_LEVEL=INFO
LOG_FORMAT=json

# =================================
# Configuración de Promtail
# =================================
PROMTAIL_PORT=9080
PROMTAIL_LOG_LEVEL=info

# =================================
# Configuración de Loki Storage
# =================================
LOKI_RETENTION_PERIOD=744h  # 31 días
LOKI_MAX_CHUNK_AGE=1h
```

### 3. Configuración de Fail2ban

```bash
# Crear configuración local
sudo nano /etc/fail2ban/jail.local
```

Configuración recomendada:

```ini
# /etc/fail2ban/jail.local
[DEFAULT]
# IPs que nunca serán baneadas (CAMBIAR POR TU IP)
ignoreip = 127.0.0.1/8 ::1 TU_IP_PUBLICA_AQUI

# Tiempo de banneo por defecto (1 hora)
bantime = 3600

# Ventana de tiempo para contar intentos (10 minutos)
findtime = 600

# Número máximo de intentos fallidos
maxretry = 5

# Configuración de email (opcional)
destemail = tu_email@example.com
sendername = Fail2ban-Alert
mta = sendmail

[sshd]
enabled = true
port = ssh
filter = sshd
logpath = /var/log/auth.log
maxretry = 3
bantime = 86400  # 24 horas para SSH
findtime = 600   # 10 minutos

[caddy]
enabled = true
port = http,https
filter = caddy
logpath = /var/log/caddy/access.log
maxretry = 10
bantime = 3600
findtime = 600
```

```bash
# Reiniciar Fail2ban para aplicar configuración
sudo systemctl restart fail2ban

# Verificar jails activos
sudo fail2ban-client status
```

### 4. Construcción y Despliegue de Contenedores

```bash
# Verificar que Docker funciona sin sudo
docker --version
docker ps

# Construir e iniciar servicios
docker compose up -d --build

# Verificar que los contenedores estén corriendo
docker compose ps
```

Salida esperada:

```
NAME                IMAGE                      STATUS
fail2ban-api        aca-fail2ban-dashboard-api Up 2 minutes
fail2ban-loki       grafana/loki:latest        Up 2 minutes  
fail2ban-promtail   grafana/promtail:latest    Up 2 minutes
```

### 5. Configuración de Caddy

```bash
# Crear directorio de configuración
sudo mkdir -p /etc/caddy

# Crear Caddyfile
sudo nano /etc/caddy/Caddyfile
```

Configuración de Caddy:

```caddyfile
# /etc/caddy/Caddyfile
{
    # Configuración global
    email tu_email@example.com
}

alertasfail2ban.xmakuno.com {
    # Proxy inverso a la API FastAPI
    reverse_proxy localhost:8000

    # Headers de seguridad
    header {
        # HSTS - Forzar HTTPS
        Strict-Transport-Security max-age=31536000; includeSubDomains; preload
        
        # Prevenir clickjacking
        X-Frame-Options DENY
        
        # Prevenir MIME type sniffing
        X-Content-Type-Options nosniff
        
        # XSS Protection
        X-XSS-Protection "1; mode=block"
        
        # Referrer Policy
        Referrer-Policy strict-origin-when-cross-origin
        
        # Content Security Policy
        Content-Security-Policy "default-src 'self'; script-src 'self' 'unsafe-inline'; style-src 'self' 'unsafe-inline'"
    }

    # Logs de acceso
    log {
        output file /var/log/caddy/access.log {
            roll_size 100mb
            roll_keep 5
            roll_keep_for 720h
        }
        format json
    }

    # Manejo de errores personalizado
    handle_errors {
        @5xx expression {http.error.status_code} >= 500
        respond @5xx "Error interno del servidor - Por favor intenta más tarde" 500
        
        @4xx expression {http.error.status_code} >= 400
        respond @4xx "Recurso no encontrado" 404
    }

    # Rate limiting (opcional)
    rate_limit {
        zone dynamic {
            key {remote_host}
            events 30
            window 1m
        }
    }
}

# Redirección de www (opcional)
www.alertasfail2ban.xmakuno.com {
    redir https://alertasfail2ban.xmakuno.com{uri} permanent
}
```

```bash
# Crear directorio de logs
sudo mkdir -p /var/log/caddy
sudo chown caddy:caddy /var/log/caddy

# Verificar configuración
sudo caddy validate --config /etc/caddy/Caddyfile

# Reiniciar Caddy
sudo systemctl restart caddy

# Verificar estado
sudo systemctl status caddy
```

## ✅ Verificación de la Instalación

### 1. Verificar Servicios Base

```bash
# Estado de todos los servicios
sudo systemctl status fail2ban caddy docker

# Verificar logs
sudo journalctl -u fail2ban -f --lines=20
sudo journalctl -u caddy -f --lines=20
```

### 2. Verificar Contenedores Docker

```bash
# Estado de contenedores
docker compose ps

# Logs de contenedores
docker compose logs -f --tail=20

# Logs específicos por servicio
docker compose logs api -f
docker compose logs loki -f
docker compose logs promtail -f
```

### 3. Verificar Conectividad

```bash
# Verificar API local
curl http://localhost:8000/health

# Verificar Loki
curl http://localhost:3100/ready

# Verificar SSL y dominio
curl -I https://alertasfail2ban.xmakuno.com
```

### 4. Verificar Fail2ban

```bash
# Estado general
sudo fail2ban-client status

# Estado del jail SSH
sudo fail2ban-client status sshd

# Verificar logs
sudo tail -f /var/log/fail2ban.log
```

## 🧪 Pruebas Post-Instalación

### 1. Prueba de la API

=== "Endpoint de Health"
    ```bash
    curl https://alertasfail2ban.xmakuno.com/health
    ```
    
    Respuesta esperada:
    ```json
    {
        "status": "healthy",
        "timestamp": "2025-01-21T10:30:00Z",
        "services": {
            "loki": "connected",
            "fail2ban": "connected"
        }
    }
    ```

=== "Endpoint de Jails"
    ```bash
    curl https://alertasfail2ban.xmakuno.com/api/jails
    ```
    
    Respuesta esperada:
    ```json
    [
        {
            "name": "sshd",
            "status": "active",
            "banned_ips": [],
            "total_banned": 0,
            "total_failed": 0
        }
    ]
    ```

### 2. Prueba de Dashboard Web

Abre en tu navegador:
```
https://alertasfail2ban.xmakuno.com
```

Deberías ver:
- ✅ Dashboard cargando correctamente
- ✅ Certificado SSL válido
- ✅ Datos de jails mostrados
- ✅ Logs en tiempo real

### 3. Prueba de Logs en Tiempo Real

```bash
# Generar actividad en logs
sudo fail2ban-client status

# Verificar que Promtail esté enviando a Loki
curl "http://localhost:3100/loki/api/v1/query?query={component=\"fail2ban\"}&limit=10"
```

## 🔧 Configuración Opcional

### 1. Configurar Logrotate para Loki

```bash
sudo nano /etc/logrotate.d/loki
```

```bash
# /etc/logrotate.d/loki
/var/log/caddy/*.log {
    daily
    missingok
    rotate 14
    compress
    notifempty
    create 644 caddy caddy
    postrotate
        systemctl reload caddy
    endscript
}
```

### 2. Configurar Backup Automático

```bash
# Crear script de backup
nano ~/backup-loki.sh
```

```bash
#!/bin/bash
# backup-loki.sh

BACKUP_DIR="/home/makuno/backups"
DATE=$(date +%Y%m%d_%H%M%S)

mkdir -p $BACKUP_DIR

# Backup de volumen Loki
docker run --rm -v aca-fail2ban-dashboard_loki_data:/data -v $BACKUP_DIR:/backup ubuntu tar czf /backup/loki_data_$DATE.tar.gz -C /data .

# Backup de configuraciones
tar czf $BACKUP_DIR/configs_$DATE.tar.gz /etc/caddy /etc/fail2ban/jail.local .env

# Limpiar backups antiguos (mantener 7 días)
find $BACKUP_DIR -name "*.tar.gz" -mtime +7 -delete

echo "Backup completado: $DATE"
```

```bash
# Hacer ejecutable
chmod +x ~/backup-loki.sh

# Configurar cron para backup diario
crontab -e
```

```bash
# Backup diario a las 2 AM
0 2 * * * /home/makuno/backup-loki.sh >> /home/makuno/backup.log 2>&1
```

## 🚨 Solución de Problemas Comunes

### Problema: Contenedores no inician

```bash
# Verificar logs detallados
docker compose logs --details

# Verificar permisos de archivos
ls -la /var/log/fail2ban.log
ls -la /var/run/fail2ban/fail2ban.sock

# Reiniciar servicios
docker compose down
sudo systemctl restart fail2ban
docker compose up -d
```

### Problema: Caddy no obtiene certificado SSL

```bash
# Verificar logs de Caddy
sudo journalctl -u caddy -f

# Verificar que el dominio apunte al servidor
dig alertasfail2ban.xmakuno.com

# Verificar que el puerto 80 esté abierto
sudo netstat -tlnp | grep :80

# Forzar renovación de certificado
sudo caddy reload --config /etc/caddy/Caddyfile
```

### Problema: API no puede conectarse a Fail2ban

```bash
# Verificar socket de Fail2ban
ls -la /var/run/fail2ban/fail2ban.sock

# Verificar permisos
sudo chmod 666 /var/run/fail2ban/fail2ban.sock

# Reiniciar contenedor API
docker compose restart api
```

!!! success "¡Instalación Completa!"
    Si todos los pasos se completaron exitosamente, tu sistema Fail2ban Dashboard ya está funcionando y listo para usar.

!!! tip "Próximos Pasos"
    - Revisa la guía de [mantenimiento](maintenance.md) para tareas rutinarias
    - Consulta [troubleshooting](troubleshooting.md) para resolver problemas
    - Explora la documentación de la [API](../api/reference.md) para integraciones