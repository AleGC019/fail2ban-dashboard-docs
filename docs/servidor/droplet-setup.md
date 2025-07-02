# 🚀 Configuración del Droplet

## Creación del Droplet

### Especificaciones Recomendadas

!!! info "Configuración DigitalOcean"
    - **SO**: Ubuntu 22.04 LTS
    - **Plan**: Basic ($24/mes)
    - **CPU**: 2 vCPUs
    - **RAM**: 4GB
    - **SSD**: 80GB
    - **Transferencia**: 4TB

### Configuración Inicial

```bash
# Conectar como root
ssh root@your_droplet_ip

# Actualizar sistema
apt update && apt upgrade -y

# Crear usuario admin
adduser makuno
usermod -aG sudo makuno

# Configurar SSH keys
mkdir -p /home/makuno/.ssh
cp ~/.ssh/authorized_keys /home/makuno/.ssh/
chown -R makuno:makuno /home/makuno/.ssh
chmod 700 /home/makuno/.ssh
chmod 600 /home/makuno/.ssh/authorized_keys
```

## 🔐 Configuración de Seguridad

### SSH Hardening

```bash
# Editar configuración SSH
sudo nano /etc/ssh/sshd_config
```

```ini
# /etc/ssh/sshd_config
Port 22
PasswordAuthentication no
PubkeyAuthentication yes
PermitRootLogin no
MaxAuthTries 3
ClientAliveInterval 300
```

```bash
# Reiniciar SSH
sudo systemctl restart sshd
```

### Firewall Básico

```bash
# Configurar ufw
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw enable
```

## 🌐 Configuración de Red

### Cloud Firewall (DigitalOcean)

| Puerto | Protocolo | Origen | Propósito |
|--------|-----------|---------|-----------|
| 22 | TCP | Tu IP | SSH Admin |
| 80 | TCP | Anywhere | HTTP/Let's Encrypt |
| 443 | TCP | Anywhere | HTTPS |

!!! danger "Importante"
    **NO** abrir puerto 8000 en Cloud Firewall. Solo debe ser accesible vía localhost.

### Configuración de Dominio

```bash
# Verificar DNS
dig alertasfail2ban.xmakuno.com
nslookup alertasfail2ban.xmakuno.com

# Debe retornar la IP del Droplet
```

## 📊 Monitoreo Básico

### Métricas del Sistema

```bash
# Verificar recursos
htop
df -h
free -h
iostat
```

### Logs del Sistema

```bash
# Logs importantes
sudo journalctl -f
sudo tail -f /var/log/auth.log
sudo tail -f /var/log/syslog
```

!!! tip "Próximo Paso"
    Continúa con la [instalación de software](software.md) necesario para el sistema.