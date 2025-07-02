# 🔐 Configuración de Seguridad

## Hardening del Sistema

### Configuración SSH Avanzada

```bash
# Editar configuración SSH
sudo nano /etc/ssh/sshd_config
```

```ini
# /etc/ssh/sshd_config - Configuración segura
Port 22
Protocol 2
HostKey /etc/ssh/ssh_host_rsa_key
HostKey /etc/ssh/ssh_host_ecdsa_key
HostKey /etc/ssh/ssh_host_ed25519_key

# Autenticación
PubkeyAuthentication yes
PasswordAuthentication no
ChallengeResponseAuthentication no
UsePAM yes

# Restricciones
PermitRootLogin no
MaxAuthTries 3
MaxSessions 2
LoginGraceTime 30

# Timeout
ClientAliveInterval 300
ClientAliveCountMax 2

# Restricciones de usuarios
AllowUsers makuno
DenyUsers root
```

### Fail2ban Configuración

```bash
# Configuración personalizada
sudo nano /etc/fail2ban/jail.local
```

```ini
# /etc/fail2ban/jail.local
[DEFAULT]
ignoreip = 127.0.0.1/8 ::1 TU_IP_PUBLICA
bantime = 3600
findtime = 600
maxretry = 5

[sshd]
enabled = true
port = ssh
filter = sshd
logpath = /var/log/auth.log
maxretry = 3
bantime = 86400

[caddy]
enabled = true
port = http,https
filter = caddy
logpath = /var/log/caddy/access.log
maxretry = 10
```

### Configuración de Firewall

```bash
# ufw configuración avanzada
sudo ufw --force reset
sudo ufw default deny incoming
sudo ufw default allow outgoing

# Permitir servicios esenciales
sudo ufw allow from TU_IP_PUBLICA to any port 22
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp

# Habilitar firewall
sudo ufw enable
sudo ufw status verbose
```

## 🔒 Certificados SSL

### Configuración Caddy SSL

```caddyfile
# /etc/caddy/Caddyfile
{
    email admin@tudominio.com
}

alertasfail2ban.xmakuno.com {
    reverse_proxy localhost:8000
    
    header {
        Strict-Transport-Security max-age=31536000
        X-Frame-Options DENY
        X-Content-Type-Options nosniff
    }
}
```

### Verificar SSL

```bash
# Verificar certificado
echo | openssl s_client -servername alertasfail2ban.xmakuno.com -connect alertasfail2ban.xmakuno.com:443 2>/dev/null | openssl x509 -noout -dates

# Logs de Caddy
sudo journalctl -u caddy -f
```

## 📊 Auditoría de Seguridad

### Scripts de Monitoreo

```bash
#!/bin/bash
# security-check.sh

echo "=== AUDITORÍA DE SEGURIDAD ==="

# Verificar intentos SSH fallidos
echo "Intentos SSH fallidos (últimas 24h):"
sudo grep "Failed password" /var/log/auth.log | tail -5

# Verificar baneos de Fail2ban
echo "IPs baneadas actualmente:"
sudo fail2ban-client status sshd

# Verificar firewall
echo "Estado del firewall:"
sudo ufw status

# Verificar certificados SSL
echo "Certificado SSL:"
curl -I https://alertasfail2ban.xmakuno.com 2>/dev/null | grep -i "strict-transport"
```

!!! warning "Recordatorios de Seguridad"
    - Cambiar `TU_IP_PUBLICA` por tu IP real
    - Revisar logs regularmente
    - Mantener sistema actualizado
    - Backup de configuraciones críticas

!!! tip "Próximo Paso"
    Continúa con el [despliegue del proyecto](../deployment/installation.md).