# 🛠️ Instalación de Software

## Software Base

### Paquetes Esenciales

```bash
# Actualizar repositorios
sudo apt update

# Instalar herramientas básicas
sudo apt install -y curl wget git htop neofetch unzip
```

### Docker Engine

```bash
# Desinstalar versiones anteriores
sudo apt remove -y docker docker-engine docker.io containerd runc

# Instalar dependencias
sudo apt install -y apt-transport-https ca-certificates curl gnupg lsb-release

# Agregar clave GPG de Docker
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

# Agregar repositorio
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list

# Instalar Docker
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin

# Configurar usuario
sudo usermod -aG docker $USER
sudo systemctl enable docker
```

### Caddy Web Server

```bash
# Instalar dependencias
sudo apt install -y debian-keyring debian-archive-keyring apt-transport-https

# Agregar clave GPG
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | sudo gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg

# Agregar repositorio
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' | sudo tee /etc/apt/sources.list.d/caddy-stable.list

# Instalar Caddy
sudo apt update
sudo apt install -y caddy
sudo systemctl enable caddy
```

### Fail2ban

```bash
# Instalar Fail2ban
sudo apt install -y fail2ban

# Habilitar servicio
sudo systemctl enable fail2ban
sudo systemctl start fail2ban

# Verificar instalación
sudo fail2ban-client status
```

## ⚙️ Configuración Inicial

### Verificar Servicios

```bash
# Estado de servicios críticos
sudo systemctl status docker caddy fail2ban

# Verificar que Docker funciona
docker --version
docker ps
```

### Logs y Monitoreo

```bash
# Ubicaciones importantes de logs
ls -la /var/log/fail2ban.log
ls -la /var/log/auth.log
ls -la /var/log/syslog

# Verificar espacio en disco
df -h /var/log
```

!!! tip "Próximo Paso"
    Continúa con la [configuración de seguridad](security.md) avanzada del servidor.