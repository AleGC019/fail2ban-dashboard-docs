# 🛡️ Configuración de Fail2ban

## Configuración Principal

### Archivo jail.local

```ini
# /etc/fail2ban/jail.local
[DEFAULT]
# IPs que nunca serán baneadas
ignoreip = 127.0.0.1/8 ::1 TU_IP_ADMIN

# Configuración de baneos escalonados
bantime = 3600          # 1 hora inicial
bantime.increment = true
bantime.factor = 2      # Duplicar tiempo cada reincidencia
bantime.multipliers = 2 4 8 16 32 64
bantime.maxtime = 86400 # Máximo 24 horas

# Ventana de tiempo para contar fallos
findtime = 600          # 10 minutos

# Número máximo de intentos
maxretry = 5

# Configuración de notificaciones
destemail = admin@tudominio.com
sendername = Fail2ban-Alert
mta = sendmail
action = %(action_mwl)s

[sshd]
enabled = true
port = ssh
filter = sshd
logpath = /var/log/auth.log
maxretry = 3
bantime = 86400
findtime = 600

[caddy]
enabled = true
port = http,https
filter = caddy
logpath = /var/log/caddy/access.log
maxretry = 10
bantime = 3600
findtime = 600
```

## Filtros Personalizados

### Filtro para Caddy

```ini
# /etc/fail2ban/filter.d/caddy.conf
[Definition]
failregex = ^.*"remote_ip":"<HOST>".*"status":(?:401|403|404|429).*$
            ^.*<HOST>.*"GET \/\.env.*$
            ^.*<HOST>.*"GET \/wp-admin.*$
            ^.*<HOST>.*"POST \/.*\.php.*$

ignoreregex =

[Init]
datepattern = ^"ts":"%%Y-%%m-%%dT%%H:%%M:%%S\.%%fZ"
```

## Comandos Útiles

### Gestión de Jails

```bash
# Ver estado general
sudo fail2ban-client status

# Ver estado específico de jail
sudo fail2ban-client status sshd

# Banear IP manualmente
sudo fail2ban-client set sshd banip 192.168.1.100

# Desbanear IP
sudo fail2ban-client set sshd unbanip 192.168.1.100

# Recargar configuración
sudo fail2ban-client reload
```

### Monitoreo

```bash
# Ver logs de fail2ban
sudo tail -f /var/log/fail2ban.log

# IPs baneadas actualmente
sudo fail2ban-client banned

# Estadísticas por jail
sudo fail2ban-client status sshd | grep "Currently banned"
```

!!! warning "Importante"
    - Cambiar `TU_IP_ADMIN` por tu IP real
    - Verificar filtros antes de activar
    - Mantener logs de respaldo