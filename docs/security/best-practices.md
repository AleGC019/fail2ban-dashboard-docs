# 🔐 Mejores Prácticas de Seguridad

## Configuración del Servidor

### SSH Security

```bash
# Configuración SSH segura
# /etc/ssh/sshd_config
Port 22
Protocol 2
PasswordAuthentication no
PubkeyAuthentication yes
PermitRootLogin no
MaxAuthTries 3
ClientAliveInterval 300
AllowUsers makuno
```

### Firewall Configuration

```bash
# Configuración UFW restrictiva
sudo ufw --force reset
sudo ufw default deny incoming
sudo ufw default allow outgoing

# Solo puertos necesarios
sudo ufw allow from TU_IP_ADMIN to any port 22
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw enable
```

## Seguridad de Contenedores

### Docker Security

```yaml
# docker-compose.yaml - Configuración segura
services:
  api:
    # No privilegios de root
    user: "1000:1000"
    
    # Read-only filesystem
    read_only: true
    
    # Límites de recursos
    deploy:
      resources:
        limits:
          memory: 512M
          cpus: '0.5'
    
    # Security options
    security_opt:
      - no-new-privileges:true
    
    # Capabilities mínimas
    cap_drop:
      - ALL
```

### Volúmenes Seguros

```yaml
volumes:
  # Read-only mounts cuando sea posible
  - /var/log/fail2ban.log:/var/log/fail2ban.log:ro
  - /var/run/fail2ban/fail2ban.sock:/var/run/fail2ban/fail2ban.sock:ro
```

## Monitoreo y Auditoría

### Script de Auditoría Diaria

```bash
#!/bin/bash
# daily-security-audit.sh

echo "=== AUDITORÍA DE SEGURIDAD DIARIA ==="

# 1. Verificar servicios críticos
for service in ssh fail2ban caddy docker; do
    if systemctl is-active --quiet $service; then
        echo "✅ $service: ACTIVO"
    else
        echo "❌ $service: INACTIVO"
    fi
done

# 2. Verificar intentos SSH fallidos
echo "🔍 Intentos SSH fallidos (últimas 24h):"
grep "Failed password" /var/log/auth.log | grep "$(date '+%b %d')" | wc -l

# 3. Verificar IPs baneadas
echo "🛡️ IPs baneadas actualmente:"
sudo fail2ban-client status sshd | grep "Currently banned"

# 4. Verificar certificado SSL
echo "🔒 Estado certificado SSL:"
days_until_expiry=$(openssl s_client -servername alertasfail2ban.xmakuno.com -connect alertasfail2ban.xmakuno.com:443 2>/dev/null | openssl x509 -noout -checkend 604800)
if [ $? -eq 0 ]; then
    echo "✅ Certificado válido (>7 días)"
else
    echo "⚠️ Certificado expira pronto"
fi

# 5. Verificar actualizaciones de seguridad
echo "📦 Actualizaciones de seguridad disponibles:"
apt list --upgradable 2>/dev/null | grep -i security | wc -l
```

### Alertas Automáticas

```bash
# Configurar en crontab: 0 6 * * * /path/to/security-alerts.sh
#!/bin/bash
# security-alerts.sh

# Verificar eventos críticos
CRITICAL_EVENTS=$(grep -E "(Ban|CRITICAL|ERROR)" /var/log/fail2ban.log | grep "$(date '+%Y-%m-%d')" | wc -l)

if [ $CRITICAL_EVENTS -gt 5 ]; then
    echo "ALERTA: $CRITICAL_EVENTS eventos críticos detectados" | mail -s "Alerta de Seguridad" admin@tudominio.com
fi
```

## Checklist de Seguridad

### Configuración Inicial
- [ ] SSH configurado solo con claves
- [ ] Firewall configurado (ufw + Cloud Firewall)
- [ ] Fail2ban instalado y configurado
- [ ] Usuario no-root creado
- [ ] Actualizaciones del sistema aplicadas

### Configuración Web
- [ ] Certificado SSL válido
- [ ] Security headers configurados
- [ ] Rate limiting habilitado
- [ ] Puerto 8000 no expuesto públicamente

### Monitoreo
- [ ] Logs centralizados en Loki
- [ ] Alertas configuradas
- [ ] Backup de configuraciones
- [ ] Scripts de auditoría programados

!!! danger "Nunca Hacer"
    - Exponer puerto 8000 directamente
    - Usar contraseñas para SSH
    - Ejecutar contenedores como root
    - Ignorar alertas de seguridad

!!! tip "Mantenimiento Regular"
    - Revisar logs semanalmente
    - Actualizar sistema mensualmente
    - Auditar configuración trimestralmente
    - Backup de configuraciones antes de cambios