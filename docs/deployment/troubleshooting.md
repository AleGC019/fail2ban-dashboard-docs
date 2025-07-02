# 🔧 Troubleshooting y Resolución de Problemas

## Introducción

Esta guía cubre los problemas más comunes que pueden surgir en el sistema Fail2ban Dashboard y sus soluciones paso a paso.

## 🚨 Problemas Críticos del Sistema

### Sistema No Responde

!!! danger "Síntomas"
    - El sitio web no carga
    - Error 502/503 en el navegador
    - Timeout en conexiones

=== "Diagnóstico Rápido"
    ```bash
    # Verificar estado de servicios críticos
    sudo systemctl status caddy fail2ban docker
    
    # Verificar contenedores
    docker compose ps
    
    # Verificar conectividad básica
    curl -I http://localhost:8000
    curl -I http://localhost:3100/ready
    ```

=== "Solución"
    ```bash
    # Reinicio ordenado de servicios
    sudo systemctl restart caddy
    docker compose restart
    
    # Si el problema persiste, reinicio completo
    docker compose down
    sudo systemctl restart docker
    docker compose up -d
    ```

### Certificado SSL Expirado o Inválido

!!! warning "Síntomas"
    - Advertencia de certificado en navegador
    - Error "NET::ERR_CERT_DATE_INVALID"
    - Fallo en HTTPS

=== "Verificar Certificado"
    ```bash
    # Verificar estado del certificado
    echo | openssl s_client -servername alertasfail2ban.xmakuno.com -connect alertasfail2ban.xmakuno.com:443 2>/dev/null | openssl x509 -noout -dates
    
    # Verificar logs de Caddy
    sudo journalctl -u caddy -f --since "1 hour ago"
    
    # Verificar configuración de Caddy
    sudo caddy validate --config /etc/caddy/Caddyfile
    ```

=== "Solución"
    ```bash
    # Forzar renovación de certificado
    sudo systemctl stop caddy
    sudo rm -rf /var/lib/caddy/.local/share/caddy/certificates/acme-v02.api.letsencrypt.org-directory/
    sudo systemctl start caddy
    
    # Verificar logs durante renovación
    sudo journalctl -u caddy -f
    ```

## 🐳 Problemas de Docker y Contenedores

### Contenedores No Inician

!!! error "Síntomas"
    - `docker compose ps` muestra contenedores como "Exited"
    - Errores en `docker compose logs`

=== "Diagnóstico"
    ```bash
    # Verificar logs detallados
    docker compose logs --details --timestamps
    
    # Verificar recursos del sistema
    df -h
    free -h
    
    # Verificar permisos de archivos montados
    ls -la /var/log/fail2ban.log
    ls -la /var/run/fail2ban/fail2ban.sock
    ```

=== "Soluciones Comunes"
    
    **Problema de Permisos:**
    ```bash
    # Ajustar permisos de archivos críticos
    sudo chmod 644 /var/log/fail2ban.log
    sudo chmod 666 /var/run/fail2ban/fail2ban.sock
    
    # Verificar que fail2ban esté corriendo
    sudo systemctl restart fail2ban
    ```
    
    **Falta de Espacio en Disco:**
    ```bash
    # Limpiar espacio en Docker
    docker system prune -af
    
    # Limpiar logs antiguos
    sudo journalctl --vacuum-time=3d
    ```
    
    **Variables de Entorno Faltantes:**
    ```bash
    # Verificar archivo .env
    cat .env
    
    # Recrear desde ejemplo si es necesario
    cp .env.example .env
    nano .env
    ```

### Contenedor API No Conecta a Servicios

!!! warning "Síntomas"
    - Error 500 en endpoints de la API
    - "Connection refused" en logs
    - Timeout en consultas a Loki

=== "Verificar Conectividad de Red"
    ```bash
    # Verificar red Docker
    docker network ls
    docker network inspect aca-fail2ban-dashboard_fail2ban_network
    
    # Probar conectividad entre contenedores
    docker compose exec api ping loki
    docker compose exec api curl http://loki:3100/ready
    ```

=== "Solución"
    ```bash
    # Recrear red Docker
    docker compose down
    docker network prune -f
    docker compose up -d
    
    # Verificar que todos los servicios estén en la misma red
    docker inspect $(docker compose ps -q) | grep NetworkMode
    ```

## 🛡️ Problemas de Fail2ban

### Fail2ban No Banea IPs

!!! info "Síntomas"
    - IPs maliciosas no son baneadas
    - No hay logs de baneos en `/var/log/fail2ban.log`
    - `fail2ban-client status` muestra 0 baneos

=== "Verificar Configuración"
    ```bash
    # Verificar que fail2ban esté activo
    sudo systemctl status fail2ban
    
    # Verificar jails configurados
    sudo fail2ban-client status
    
    # Verificar configuración específica
    sudo fail2ban-client get sshd logpath
    sudo fail2ban-client get sshd maxretry
    sudo fail2ban-client get sshd findtime
    sudo fail2ban-client get sshd bantime
    ```

=== "Revisar Logs de Entrada"
    ```bash
    # Verificar que los logs que monitorea existan
    ls -la /var/log/auth.log
    
    # Verificar que hay actividad en los logs
    sudo tail -f /var/log/auth.log
    
    # Buscar intentos SSH fallidos
    sudo grep "Failed password" /var/log/auth.log | tail -10
    ```

=== "Solución"
    ```bash
    # Verificar filtros de fail2ban
    sudo fail2ban-client set sshd addlogpath /var/log/auth.log
    
    # Recargar configuración
    sudo fail2ban-client reload sshd
    
    # Probar filtro manualmente
    sudo fail2ban-regex /var/log/auth.log /etc/fail2ban/filter.d/sshd.conf
    
    # Verificar que no esté en ignoreip
    sudo fail2ban-client get sshd ignoreip
    ```

### Fail2ban Banea IPs Legítimas

!!! danger "Síntomas"
    - Admins no pueden acceder vía SSH
    - IPs legítimas aparecen en listas de baneos
    - Servicios críticos afectados

=== "Desbanear IP Inmediatamente"
    ```bash
    # Verificar IP baneada
    sudo fail2ban-client status sshd
    
    # Desbanear IP específica
    sudo fail2ban-client set sshd unbanip IP_A_DESBANEAR
    
    # Verificar que se desbaneó
    sudo fail2ban-client status sshd
    ```

=== "Prevenir Auto-baneos Futuros"
    ```bash
    # Editar configuración para agregar IP a ignoreip
    sudo nano /etc/fail2ban/jail.local
    
    # Agregar IPs de administradores
    [DEFAULT]
    ignoreip = 127.0.0.1/8 ::1 TU_IP_PUBLICA_AQUI TU_IP_OFICINA_AQUI
    
    # Recargar configuración
    sudo fail2ban-client reload
    ```

## 🗄️ Problemas de Loki y Logs

### Loki No Recibe Logs

!!! warning "Síntomas"
    - Dashboard muestra datos vacíos
    - Consultas a Loki retornan resultados vacíos
    - Promtail no envía datos

=== "Verificar Pipeline Promtail → Loki"
    ```bash
    # Verificar que Promtail esté leyendo logs
    docker compose logs promtail | grep -i "reading\|tailing"
    
    # Verificar conectividad Promtail → Loki
    docker compose exec promtail curl http://loki:3100/ready
    
    # Verificar logs de Promtail
    docker compose logs promtail --tail=50
    ```

=== "Verificar Configuración de Promtail"
    ```bash
    # Verificar archivo de configuración
    cat promtail/promtail.yaml
    
    # Verificar que el archivo de log existe y es legible
    ls -la /var/log/fail2ban.log
    
    # Verificar posiciones de lectura
    cat /tmp/positions.yaml
    ```

=== "Solución"
    ```bash
    # Reiniciar pipeline de logs
    docker compose restart promtail
    
    # Verificar que los logs estén siendo generados
    sudo tail -f /var/log/fail2ban.log
    
    # Forzar actividad en fail2ban para generar logs
    sudo fail2ban-client status
    
    # Verificar en Loki directamente
    curl "http://localhost:3100/loki/api/v1/query?query={job=\"failban\"}&limit=10"
    ```

### Consultas Loki Muy Lentas

!!! info "Síntomas"
    - Dashboard tarda mucho en cargar
    - Timeouts en consultas de la API
    - Loki consume mucha CPU/memoria

=== "Optimizar Consultas"
    ```bash
    # Verificar estado de Loki
    curl http://localhost:3100/metrics | grep loki_ingester
    
    # Verificar tamaño de datos
    docker exec aca-fail2ban-dashboard-loki-1 du -sh /loki
    
    # Verificar queries activas
    curl http://localhost:3100/loki/api/v1/query_stats
    ```

=== "Solución"
    ```yaml
    # Optimizar configuración de Loki
    # loki/config.yaml
    limits_config:
      max_query_time: "2m"
      max_query_parallelism: 16
      max_streams_per_user: 5000
    
    chunk_store_config:
      max_look_back_period: "168h"  # 7 días
    
    table_manager:
      retention_deletes_enabled: true
      retention_period: "168h"  # 7 días
    ```

## 🌐 Problemas de Red y Conectividad

### Caddy No Puede Obtener Certificado SSL

!!! error "Síntomas"
    - Error 502 "Bad Gateway"
    - Logs de Caddy muestran errores de ACME
    - Certificado SSL no se renueva

=== "Verificar Conectividad Externa"
    ```bash
    # Verificar que el dominio apunte al servidor
    dig alertasfail2ban.xmakuno.com
    nslookup alertasfail2ban.xmakuno.com
    
    # Verificar desde externa (usar sitio web)
    # https://dnschecker.org/
    
    # Verificar puertos abiertos
    sudo netstat -tlnp | grep -E ":(80|443)"
    ```

=== "Verificar Configuración de Firewall"
    ```bash
    # Verificar Cloud Firewall de DigitalOcean
    # (revisar en panel web)
    
    # Verificar ufw local
    sudo ufw status verbose
    
    # Verificar que no hay otros servicios en puerto 80
    sudo lsof -i :80
    ```

=== "Solución"
    ```bash
    # Detener Caddy temporalmente
    sudo systemctl stop caddy
    
    # Probar validación manual con certbot
    sudo apt install certbot
    sudo certbot certonly --standalone -d alertasfail2ban.xmakuno.com
    
    # Si certbot funciona, el problema está en Caddy
    sudo systemctl start caddy
    
    # Verificar logs específicos
    sudo journalctl -u caddy -f --since "10 minutes ago"
    ```

### Puerto 8000 Accesible Externamente

!!! danger "Problema de Seguridad Crítico"
    Si el puerto 8000 de la API es accesible desde internet, es un riesgo de seguridad.

=== "Verificar Exposición"
    ```bash
    # Verificar desde otra máquina
    nmap -p 8000 tu_ip_servidor
    
    # O usar herramienta online como:
    # https://www.yougetsignal.com/tools/open-ports/
    ```

=== "Solución Inmediata"
    ```bash
    # Verificar Cloud Firewall de DigitalOcean
    # Puerto 8000 NO debe estar en las reglas
    
    # Verificar configuración Docker
    docker compose ps
    # Solo debe mostrar 8000:8000 sin 0.0.0.0:8000:8000
    
    # Si está mal configurado, editar docker-compose.yaml
    nano docker-compose.yaml
    
    # Cambiar de:
    # ports:
    #   - "0.0.0.0:8000:8000"
    # A:
    # ports:
    #   - "127.0.0.1:8000:8000"
    
    # Recrear contenedores
    docker compose up -d --force-recreate
    ```

## ⚡ Problemas de Performance

### Alta Carga del Sistema

!!! warning "Síntomas"
    - Load average muy alto
    - Sistema lento en respuesta
    - Servicios que no responden

=== "Identificar Causa"
    ```bash
    # Verificar carga actual
    uptime
    
    # Top procesos por CPU
    top -c
    
    # Top procesos por memoria
    ps aux --sort=-%mem | head -10
    
    # Verificar I/O de disco
    iotop -ao
    
    # Verificar uso de red
    iftop
    ```

=== "Soluciones por Causa"
    
    **CPU Alto por Loki:**
    ```bash
    # Reducir retención de datos
    # Editar loki/config.yaml
    limits_config:
      retention_period: "72h"  # Reducir de 744h a 72h
    
    # Reiniciar Loki
    docker compose restart loki
    ```
    
    **Memoria Alta:**
    ```bash
    # Verificar uso de memoria por contenedor
    docker stats --no-stream
    
    # Limpiar cache del sistema
    sync && echo 3 | sudo tee /proc/sys/vm/drop_caches
    
    # Agregar límites de memoria en docker-compose.yaml
    services:
      loki:
        deploy:
          resources:
            limits:
              memory: 512M
    ```
    
    **Disco Lleno:**
    ```bash
    # Verificar uso de disco
    df -h
    
    # Limpiar logs antiguos
    sudo journalctl --vacuum-time=3d
    
    # Limpiar Docker
    docker system prune -af --volumes
    
    # Rotar logs de Caddy
    sudo logrotate -f /etc/logrotate.d/caddy
    ```

## 🔍 Herramientas de Diagnóstico

### Script de Diagnóstico Automático

```bash
#!/bin/bash
# diagnostic.sh - Script completo de diagnóstico

echo "=== DIAGNÓSTICO COMPLETO DEL SISTEMA ==="
echo "Fecha: $(date)"
echo "=========================================="

# 1. Estado general del sistema
echo "📊 ESTADO DEL SISTEMA:"
uptime
free -h
df -h /

# 2. Servicios críticos
echo -e "\n🔧 SERVICIOS CRÍTICOS:"
for service in fail2ban caddy docker; do
    if systemctl is-active --quiet $service; then
        echo "✅ $service: ACTIVO"
    else
        echo "❌ $service: INACTIVO"
    fi
done

# 3. Contenedores Docker
echo -e "\n🐳 CONTENEDORES DOCKER:"
docker compose ps 2>/dev/null || echo "❌ Docker Compose no disponible"

# 4. Conectividad de red
echo -e "\n🌐 CONECTIVIDAD DE RED:"
if curl -s http://localhost:8000/health >/dev/null; then
    echo "✅ API local: DISPONIBLE"
else
    echo "❌ API local: NO DISPONIBLE"
fi

if curl -s http://localhost:3100/ready >/dev/null; then
    echo "✅ Loki: DISPONIBLE"
else
    echo "❌ Loki: NO DISPONIBLE"
fi

# 5. SSL/TLS
echo -e "\n🔒 CERTIFICADO SSL:"
cert_info=$(echo | openssl s_client -servername alertasfail2ban.xmakuno.com -connect alertasfail2ban.xmakuno.com:443 2>/dev/null | openssl x509 -noout -dates 2>/dev/null)
if [ $? -eq 0 ]; then
    echo "✅ Certificado SSL válido"
    echo "$cert_info"
else
    echo "❌ Problema con certificado SSL"
fi

# 6. Fail2ban status
echo -e "\n🛡️ FAIL2BAN STATUS:"
if sudo fail2ban-client status >/dev/null 2>&1; then
    echo "✅ Fail2ban funcionando"
    sudo fail2ban-client status | grep "Jail list"
else
    echo "❌ Problema con Fail2ban"
fi

# 7. Logs recientes de errores
echo -e "\n📝 ERRORES RECIENTES (últimas 2 horas):"
sudo journalctl --since="2 hours ago" --priority=err --no-pager | tail -5

echo -e "\n=========================================="
echo "Diagnóstico completado: $(date)"
```

### Monitoreo en Tiempo Real

```bash
#!/bin/bash
# monitor.sh - Monitoreo en tiempo real

watch -n 5 '
echo "=== MONITOREO EN TIEMPO REAL ==="
echo "Fecha: $(date)"
echo "================================"
echo ""
echo "🔧 Servicios:"
systemctl is-active fail2ban caddy docker | paste <(echo -e "fail2ban\ncaddy\ndocker") - | column -t
echo ""
echo "🐳 Contenedores:"
docker compose ps --format "table {{.Name}}\t{{.Status}}\t{{.Ports}}" 2>/dev/null
echo ""
echo "📊 Recursos:"
echo "CPU: $(top -bn1 | grep "Cpu(s)" | awk "{print \$2}" | cut -d"%" -f1)%"
echo "RAM: $(free | grep Mem | awk "{printf \"%.1f%%\", \$3/\$2 * 100.0}")"
echo "Disk: $(df / | tail -1 | awk "{print \$5}")"
echo ""
echo "🛡️ Fail2ban (últimos 5 min):"
sudo grep "$(date "+%Y-%m-%d %H:%M" -d "5 minutes ago")" /var/log/fail2ban.log 2>/dev/null | tail -3 || echo "Sin actividad"
'
```

!!! tip "Mejores Prácticas de Troubleshooting"
    1. **Siempre verifica logs** antes de hacer cambios
    2. **Haz backups** antes de modificar configuraciones críticas
    3. **Documenta todos los cambios** para futura referencia
    4. **Prueba en ambiente controlado** cuando sea posible
    5. **Ten un plan de rollback** para cada cambio importante

!!! success "Recursos Adicionales"
    - [Logs del sistema](../servidor/software.md#logs-y-monitoreo): Ubicaciones y formatos de logs
    - [Configuración de seguridad](../servidor/security.md): Hardening adicional
    - [API Reference](../api/reference.md): Documentación completa de endpoints

!!! info "Soporte Adicional"
    Si no puedes resolver un problema con esta guía:
    
    1. Ejecuta el script de diagnóstico completo
    2. Recopila logs relevantes con `journalctl`
    3. Documenta los pasos exactos que llevaron al problema
    4. Revisa la documentación oficial de cada componente