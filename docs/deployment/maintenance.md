# 🔧 Mantenimiento del Sistema

## Introducción

Esta guía cubre todas las tareas de mantenimiento necesarias para mantener el sistema Fail2ban Dashboard funcionando de manera óptima y segura.

## 📅 Tareas de Mantenimiento Rutinario

### Diarias

!!! tip "Tareas Automáticas Diarias"
    - ✅ **Backup de datos** (configurado via cron)
    - ✅ **Rotación de logs** (configurado via logrotate)
    - ✅ **Renovación de certificados SSL** (automático con Caddy)

### Semanales

```bash
# Verificar estado general del sistema
~/scripts/weekly-health-check.sh
```

Crea el script de verificación semanal:

```bash
#!/bin/bash
# weekly-health-check.sh

echo "=== HEALTH CHECK SEMANAL - $(date) ==="

# 1. Estado de servicios críticos
echo "📊 Estado de servicios:"
sudo systemctl status fail2ban --no-pager -l
sudo systemctl status caddy --no-pager -l
sudo systemctl status docker --no-pager -l

# 2. Estado de contenedores
echo "🐳 Estado de contenedores:"
docker compose ps

# 3. Uso de disco
echo "💾 Uso de disco:"
df -h /

# 4. Uso de memoria
echo "🧠 Uso de memoria:"
free -h

# 5. IPs baneadas actualmente
echo "🛡️ IPs baneadas:"
sudo fail2ban-client status sshd

# 6. Logs recientes de errores
echo "📝 Errores recientes:"
sudo journalctl --since="24 hours ago" --priority=err --no-pager

# 7. Tamaño de volúmenes Docker
echo "📦 Tamaño de volúmenes:"
docker system df

echo "=== FIN HEALTH CHECK ==="
```

### Mensuales

=== "Actualizaciones de Sistema"
    ```bash
    # Actualizar sistema operativo
    sudo apt update && sudo apt list --upgradable
    sudo apt upgrade -y
    
    # Reiniciar si es necesario
    if [ -f /var/run/reboot-required ]; then
        echo "Reinicio requerido"
        sudo reboot
    fi
    ```

=== "Limpieza de Docker"
    ```bash
    # Limpiar imágenes no utilizadas
    docker image prune -f
    
    # Limpiar contenedores detenidos
    docker container prune -f
    
    # Limpiar volúmenes no utilizados
    docker volume prune -f
    
    # Limpiar redes no utilizadas
    docker network prune -f
    
    # Limpieza completa (cuidado!)
    docker system prune -af --volumes
    ```

=== "Rotación de Backups"
    ```bash
    # Limpiar backups antiguos (mantener 30 días)
    find ~/backups -name "*.tar.gz" -mtime +30 -delete
    
    # Verificar integridad de backups recientes
    for backup in $(find ~/backups -name "*.tar.gz" -mtime -7); do
        if tar -tzf "$backup" >/dev/null 2>&1; then
            echo "✅ $backup - OK"
        else
            echo "❌ $backup - CORRUPTO"
        fi
    done
    ```

## 📊 Monitoreo y Métricas

### Dashboard de Monitoreo

Crea un script para generar métricas:

```bash
#!/bin/bash
# metrics-report.sh

echo "=== REPORTE DE MÉTRICAS - $(date) ==="

# Métricas de Fail2ban
echo "🛡️ Estadísticas de Fail2ban:"
for jail in $(sudo fail2ban-client status | grep "Jail list:" | cut -d: -f2 | tr ',' '\n' | xargs); do
    echo "  Jail: $jail"
    sudo fail2ban-client status $jail | grep -E "(Currently banned|Total banned|Currently failed|Total failed)"
done

# Métricas de la API
echo "⚙️ Estadísticas de la API:"
curl -s http://localhost:8000/metrics 2>/dev/null || echo "API no disponible"

# Métricas de Loki
echo "🗄️ Estadísticas de Loki:"
curl -s http://localhost:3100/metrics | grep -E "loki_ingester_streams|loki_distributor_lines_received_total" | head -5

# Top IPs baneadas (últimos 7 días)
echo "🔴 Top IPs baneadas (últimos 7 días):"
sudo grep "Ban " /var/log/fail2ban.log | grep "$(date -d '7 days ago' '+%Y-%m-%d')" | awk '{print $NF}' | sort | uniq -c | sort -nr | head -10

echo "=== FIN REPORTE ==="
```

### Alertas Automáticas

Configura alertas por email para eventos críticos:

```bash
# /etc/fail2ban/action.d/sendmail-alert.conf
[Definition]
actionstart = echo "Fail2ban iniciado en $(hostname)" | mail -s "Fail2ban: Servicio iniciado" admin@tudominio.com
actionstop = echo "Fail2ban detenido en $(hostname)" | mail -s "Fail2ban: Servicio detenido" admin@tudominio.com
actioncheck = 
actionban = echo "IP <ip> baneada en jail <name> en $(hostname) por <failures> intentos fallidos" | mail -s "Fail2ban: Nueva IP baneada" admin@tudominio.com
actionunban = echo "IP <ip> desbaneada en jail <name> en $(hostname)" | mail -s "Fail2ban: IP desbaneada" admin@tudominio.com

[Init]
name = default
```

## 🔄 Actualizaciones del Sistema

### Actualización de Contenedores

=== "Actualización Planificada"
    ```bash
    # 1. Crear backup antes de actualizar
    ~/backup-loki.sh
    
    # 2. Descargar nuevas imágenes
    docker compose pull
    
    # 3. Recrear contenedores con nuevas imágenes
    docker compose up -d --force-recreate
    
    # 4. Verificar que todo funciona
    docker compose ps
    curl -I https://alertasfail2ban.xmakuno.com/health
    ```

=== "Rollback en caso de problemas"
    ```bash
    # Volver a imágenes anteriores
    docker compose down
    
    # Restaurar backup si es necesario
    cd ~/backups
    latest_backup=$(ls -t loki_data_*.tar.gz | head -1)
    docker run --rm -v aca-fail2ban-dashboard_loki_data:/data -v $(pwd):/backup ubuntu tar xzf /backup/$latest_backup -C /data
    
    # Iniciar servicios
    docker compose up -d
    ```

### Actualización de la API

```bash
# Actualizar código desde Git
cd ~/aca-fail2ban-dashboard
git fetch origin
git checkout main
git pull origin main

# Rebuild y restart de la API
docker compose build api
docker compose up -d api

# Verificar logs
docker compose logs api -f
```

### Actualización de Caddy

```bash
# Actualizar Caddy
sudo apt update
sudo apt upgrade caddy

# Verificar nueva versión
caddy version

# Restart del servicio
sudo systemctl restart caddy
sudo systemctl status caddy
```

## 🗄️ Gestión de Datos y Backups

### Estrategia de Backup

!!! info "Tipos de Backup"
    === "Backup Completo"
        - **Frecuencia**: Semanal
        - **Contenido**: Todos los datos de Loki + configuraciones
        - **Retención**: 4 backups (1 mes)
    
    === "Backup Incremental"
        - **Frecuencia**: Diario
        - **Contenido**: Solo cambios recientes
        - **Retención**: 7 backups (1 semana)
    
    === "Backup de Configuración"
        - **Frecuencia**: Antes de cada cambio
        - **Contenido**: Archivos de configuración críticos
        - **Retención**: 30 backups

### Scripts de Backup Avanzado

```bash
#!/bin/bash
# advanced-backup.sh

BACKUP_DIR="/home/makuno/backups"
DATE=$(date +%Y%m%d_%H%M%S)
RETENTION_DAYS=7

# Función para logging
log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" | tee -a /var/log/backup.log
}

# Crear directorio de backup
mkdir -p $BACKUP_DIR

log "Iniciando backup completo..."

# 1. Backup de volumen Loki
log "Backing up Loki data..."
docker run --rm \
    -v aca-fail2ban-dashboard_loki_data:/data \
    -v $BACKUP_DIR:/backup \
    ubuntu tar czf /backup/loki_data_$DATE.tar.gz -C /data .

if [ $? -eq 0 ]; then
    log "✅ Loki data backup completado"
else
    log "❌ Error en Loki data backup"
    exit 1
fi

# 2. Backup de configuraciones
log "Backing up configurations..."
tar czf $BACKUP_DIR/configs_$DATE.tar.gz \
    /etc/caddy/Caddyfile \
    /etc/fail2ban/jail.local \
    ~/aca-fail2ban-dashboard/.env \
    ~/aca-fail2ban-dashboard/docker-compose.yaml

# 3. Backup de base de datos de fail2ban
log "Backing up fail2ban database..."
if [ -f /var/lib/fail2ban/fail2ban.sqlite3 ]; then
    cp /var/lib/fail2ban/fail2ban.sqlite3 $BACKUP_DIR/fail2ban_db_$DATE.sqlite3
fi

# 4. Verificar integridad de backups
log "Verificando integridad de backups..."
for backup in $BACKUP_DIR/*_$DATE.tar.gz; do
    if tar -tzf "$backup" >/dev/null 2>&1; then
        log "✅ $backup - Integridad OK"
    else
        log "❌ $backup - Integridad FALLIDA"
    fi
done

# 5. Limpiar backups antiguos
log "Limpiando backups antiguos..."
find $BACKUP_DIR -name "*.tar.gz" -mtime +$RETENTION_DAYS -delete
find $BACKUP_DIR -name "*.sqlite3" -mtime +$RETENTION_DAYS -delete

# 6. Enviar resumen por email (opcional)
if command -v mail >/dev/null 2>&1; then
    backup_size=$(du -sh $BACKUP_DIR | cut -f1)
    echo "Backup completado: $DATE
    Tamaño total: $backup_size
    Ubicación: $BACKUP_DIR" | mail -s "Backup Report - $DATE" admin@tudominio.com
fi

log "Backup completado exitosamente"
```

### Restauración desde Backup

```bash
#!/bin/bash
# restore-backup.sh

BACKUP_DIR="/home/makuno/backups"
RESTORE_DATE=$1

if [ -z "$RESTORE_DATE" ]; then
    echo "Uso: $0 YYYYMMDD_HHMMSS"
    echo "Backups disponibles:"
    ls -la $BACKUP_DIR/*_*.tar.gz | awk '{print $9}' | sort
    exit 1
fi

echo "⚠️  ADVERTENCIA: Esto restaurará datos desde el backup $RESTORE_DATE"
echo "Todos los datos actuales se perderán."
read -p "¿Continuar? (yes/no): " confirm

if [ "$confirm" != "yes" ]; then
    echo "Operación cancelada"
    exit 1
fi

# 1. Detener servicios
echo "Deteniendo servicios..."
docker compose down
sudo systemctl stop caddy

# 2. Restaurar datos de Loki
echo "Restaurando datos de Loki..."
docker run --rm \
    -v aca-fail2ban-dashboard_loki_data:/data \
    -v $BACKUP_DIR:/backup \
    ubuntu sh -c "rm -rf /data/* && tar xzf /backup/loki_data_$RESTORE_DATE.tar.gz -C /data"

# 3. Restaurar configuraciones
echo "Restaurando configuraciones..."
tar xzf $BACKUP_DIR/configs_$RESTORE_DATE.tar.gz -C /

# 4. Restaurar base de datos de fail2ban
if [ -f $BACKUP_DIR/fail2ban_db_$RESTORE_DATE.sqlite3 ]; then
    echo "Restaurando base de datos de fail2ban..."
    sudo cp $BACKUP_DIR/fail2ban_db_$RESTORE_DATE.sqlite3 /var/lib/fail2ban/fail2ban.sqlite3
    sudo chown fail2ban:fail2ban /var/lib/fail2ban/fail2ban.sqlite3
fi

# 5. Reiniciar servicios
echo "Iniciando servicios..."
sudo systemctl start caddy
docker compose up -d

# 6. Verificar que todo funciona
sleep 10
echo "Verificando servicios..."
docker compose ps
curl -I https://alertasfail2ban.xmakuno.com/health

echo "✅ Restauración completada"
```

## 🔐 Mantenimiento de Seguridad

### Auditoría de Seguridad Mensual

```bash
#!/bin/bash
# security-audit.sh

echo "=== AUDITORÍA DE SEGURIDAD - $(date) ==="

# 1. Verificar actualizaciones de seguridad
echo "🔍 Verificando actualizaciones de seguridad..."
sudo apt list --upgradable | grep -i security

# 2. Verificar configuración SSH
echo "🔐 Configuración SSH:"
sudo sshd -T | grep -E "(PasswordAuthentication|PubkeyAuthentication|PermitRootLogin)"

# 3. Verificar firewall
echo "🔥 Estado del firewall:"
sudo ufw status verbose

# 4. Verificar certificados SSL
echo "🔒 Certificados SSL:"
echo | openssl s_client -servername alertasfail2ban.xmakuno.com -connect alertasfail2ban.xmakuno.com:443 2>/dev/null | openssl x509 -noout -dates

# 5. Verificar intentos de acceso fallidos
echo "🚨 Intentos de acceso fallidos (últimas 24h):"
sudo grep "Failed password" /var/log/auth.log | tail -10

# 6. Verificar jails activos
echo "🛡️ Jails de Fail2ban:"
sudo fail2ban-client status

# 7. Top IPs atacantes
echo "🔴 Top IPs atacantes (última semana):"
sudo grep "Ban " /var/log/fail2ban.log | awk '{print $NF}' | sort | uniq -c | sort -nr | head -10

echo "=== FIN AUDITORÍA ==="
```

### Rotación de Claves SSH

```bash
# Generar nuevas claves SSH
ssh-keygen -t ed25519 -C "nueva-clave-$(date +%Y%m%d)" -f ~/.ssh/nueva_clave

# Agregar nueva clave a authorized_keys
cat ~/.ssh/nueva_clave.pub >> ~/.ssh/authorized_keys

# Probar nueva clave desde otra sesión antes de eliminar la antigua
# ssh -i ~/.ssh/nueva_clave makuno@tu_servidor

# Eliminar clave antigua después de confirmar que la nueva funciona
# sed -i '/clave_antigua/d' ~/.ssh/authorized_keys
```

## 📈 Optimización de Performance

### Monitoreo de Recursos

```bash
#!/bin/bash
# performance-monitor.sh

echo "=== MONITOREO DE PERFORMANCE - $(date) ==="

# CPU
echo "💻 Uso de CPU:"
top -bn1 | grep "Cpu(s)" | awk '{print $2}' | cut -d'%' -f1

# Memoria
echo "🧠 Uso de memoria:"
free -h

# Disco
echo "💾 Uso de disco:"
df -h / | tail -1

# Docker stats
echo "🐳 Recursos de contenedores:"
docker stats --no-stream --format "table {{.Container}}\t{{.CPUPerc}}\t{{.MemUsage}}\t{{.MemPerc}}"

# Conexiones de red
echo "🌐 Conexiones activas:"
ss -tuln | grep -E ":(80|443|8000|3100|22)"

# Procesos top
echo "🔝 Top procesos por CPU:"
ps aux --sort=-%cpu | head -6

echo "=== FIN MONITOREO ==="
```

### Optimización de Loki

```yaml
# Configuración optimizada para Loki
# loki/config.yaml
limits_config:
  # Optimización de ingesta
  ingestion_rate_mb: 4
  ingestion_burst_size_mb: 6
  max_streams_per_user: 10000
  max_line_size: 256000
  
  # Optimización de queries
  max_query_parallelism: 32
  max_query_time: "5m"
  max_query_length: "12000h"
  
  # Límites de retención
  retention_period: "744h"  # 31 días
```

!!! tip "Mejores Prácticas de Mantenimiento"
    1. **Automatiza todo lo posible** con scripts y cron jobs
    2. **Monitorea proactivamente** en lugar de reactivamente
    3. **Mantén backups regulares** y prueba la restauración
    4. **Documenta todos los cambios** y procedimientos
    5. **Mantén el sistema actualizado** pero prueba en staging primero

!!! success "Próximo Paso"
    Revisa la guía de [troubleshooting](troubleshooting.md) para resolver problemas comunes que pueden surgir durante el mantenimiento.