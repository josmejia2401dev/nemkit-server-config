# Guía de Configuración y Optimización Máxima: Ubuntu Server 26.04 LTS

Configuración optimizada para **Lenovo ThinkCentre M73 Tiny** (Intel Core i5 4.ª Gen, 8 GB RAM, SSD 256 GB) ejecutando contenedores **Docker (Node.js + Angular)**.

---

## 1. Red Wi-Fi con IP Estática (Netplan)

Edita el archivo de configuración de Netplan:

```bash
sudo nano /etc/netplan/50-cloud-init.yaml

```

Asegúrate de respetar la indentación con **espacios** (sin tabuladores):

```yaml
network:
  version: 2
  renderer: networkd
  wifis:
    wlp2s0:
      dhcp4: false
      dhcp6: false
      addresses:
        - 192.168.1.55/24
      routes:
        - to: default
          via: 192.168.1.254
          metric: 100
      nameservers:
        search: [local]
        addresses: [8.8.8.8, 1.1.1.1, 192.168.1.254, 8.8.4.4]
      access-points:
        "FMC ":
          auth:
            key-management: "psk"
          password: "Familiamc1"
```

_(Guarda con `Ctrl + O`, `Enter` y sal con `Ctrl + X`)_.

> **Nota:** Mantén las comillas y el espacio final en `"FMC "`.

Aplica permisos seguros y activa la red:

```bash
sudo chmod 600 /etc/netplan/50-cloud-init.yaml
sudo netplan apply
ping -c 3 1.1.1.1

```

---

## 2. Seguridad Base: SSH y Firewall (UFW)

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install openssh-server ufw -y
sudo systemctl enable --now ssh

# Reglas de Firewall
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow ssh
# Descomenta según los puertos que exponga tu proxy inverso / Nginx:
# sudo ufw allow 80/tcp
# sudo ufw allow 443/tcp

sudo ufw enable
sudo ufw status verbose

```

---

## 3. Limpieza de Servicios Innecesarios (Ahorro de RAM y CPU)

Elimina demonios que consumen memoria y ciclos de disco en un servidor doméstico:

```bash
# Eliminar Snapd (ahorra ~150-250 MB de RAM y procesos en segundo plano)
sudo systemctl stop snapd.service snapd.socket
sudo apt purge snapd -y
sudo apt-mark hold snapd

# Desactivar servicios de hardware inexistente en el M73
sudo systemctl disable --now ModemManager.service multipathd.service 2>/dev/null || true
sudo apt autoremove --purge -y

```

---

## 4. Optimización de Energía, CPU (Haswell) y Kernel

### 4.1. Instalar herramientas de gestión térmica y energía

```bash
sudo apt install tuned tuned-utils thermald linux-tools-common linux-tools-$(uname -r) -y

# Activar gestión térmica de Intel y TuneD
sudo systemctl enable --now thermald
sudo systemctl enable --now tuned

# Perfil 'powersave' en TuneD: mantiene el escalado dinámico rápido de intel_pstate
# pero activa ahorro agresivo en enlaces SATA, PCIe y USB
sudo tuned-adm profile powersave

```

### 4.2. Parámetros de arranque (GRUB)

Edita `/etc/default/grub`:

```bash
sudo nano /etc/default/grub

```

Modifica `GRUB_CMDLINE_LINUX_DEFAULT` para apagar el video a los 60s, forzar ahorro PCIe ASPM, dormir el chip de audio integrado y recuperar rendimiento I/O en Haswell:

```text
GRUB_CMDLINE_LINUX_DEFAULT="consoleblank=60 pcie_aspm=force pcie_aspm.policy=powersave snd_hda_intel.power_save=1 mitigations=off"

```

_(Nota: Si prefieres mantener los parches de seguridad de CPU antiguos a costa de un ~20 % menos de rendimiento I/O, elimina `mitigations=off`)_.

Actualiza GRUB:

```bash
sudo update-grub

```

### 4.3. Evitar microcortes en Wi-Fi (Mantener baja latencia)

Crea una regla persistente para evitar que la tarjeta Wi-Fi se duerma y degrade tus sesiones SSH o peticiones HTTP:

```bash
echo 'ACTION=="add", SUBSYSTEM=="net", KERNEL=="wlp2s0", RUN+="/usr/sbin/iw dev $name set power_save off"' | sudo tee /etc/udev/rules.d/70-wifi-powersave.rules

```

---

## 5. Optimización de Memoria RAM (ZRAM) y Red (TCP BBR)

### 5.1. Configurar ZRAM (Swap comprimido en RAM sin desgastar el SSD)

```bash
sudo apt install systemd-zram-generator -y

sudo tee /etc/systemd/zram-generator.conf > /dev/null << 'EOF'
[zram0]
zram-size = ram / 2
compression-algorithm = zstd
swap-priority = 100
fs-type = swap
EOF

sudo systemctl daemon-reload
sudo systemctl start systemd-zram-setup@zram0.service

```

### 5.2. Parámetros del Kernel (`sysctl.d`)

Crea un archivo dedicado para optimizar el uso de ZRAM y activar el algoritmo **TCP BBR**:

```bash
sudo tee /etc/sysctl.d/99-server-optimizations.conf > /dev/null << 'EOF'
# Optimización de memoria para ZRAM (prioriza comprimir en RAM antes que ir al SSD)
vm.swappiness = 150
vm.vfs_cache_pressure = 50
vm.page-cluster = 0

# Optimización de Red (TCP BBR + Fast Open + Buffers)
net.core.default_qdisc = fq
net.ipv4.tcp_congestion_control = bbr
net.ipv4.tcp_fastopen = 3
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216
net.ipv4.tcp_rmem = 4096 87380 16777216
net.ipv4.tcp_wmem = 4096 65536 16777216
EOF

sudo sysctl --system

```

---

## 6. Protección y Rendimiento del SSD (256 GB)

### 6.1. Habilitar TRIM semanal

```bash
sudo systemctl enable --now fstrim.timer

```

### 6.2. Reducir escrituras de metadatos (`noatime`)

Por defecto, cada vez que Linux lee un archivo (ej. Nginx sirviendo tu app en Angular o Docker leyendo imágenes), actualiza en el disco la fecha exacta de esa lectura (_access time_ o `atime`). Esto genera miles de pequeñas escrituras constantes que desgastan innecesariamente tu SSD de 156 GB. Añadir `noatime` desactiva este registro.

**Paso 1: Respaldar el archivo de configuración**
Un error aquí puede evitar que Ubuntu arranque correctamente. Crea una copia de seguridad:

```bash
sudo cp /etc/fstab /etc/fstab.bak

```

**Paso 2: Identificar tu configuración actual**
Revisa cómo está montado tu disco:

```bash
cat /etc/fstab

```

Busca la línea que tenga una barra solitaria `/` en la segunda columna (esa es tu partición principal). Fíjate en la cuarta columna (las opciones), que generalmente dice `defaults` o `errors=remount-ro`.

**Paso 3: Añadir la regla `noatime**`
Edita el archivo:

```bash
sudo nano /etc/fstab

```

Agrega `,noatime` (separado por coma y **sin espacios**) en la cuarta columna de la línea de la partición `/`.

Ejemplos de cómo debería quedar según tu caso:

- Si decía `defaults`: `UUID=tu-uuid-original / ext4 defaults,noatime 0 1`
- Si decía `errors=remount-ro`: `UUID=tu-uuid-original / ext4 noatime,errors=remount-ro 0 1`
- (Guarda con `Ctrl + O`, pulsa `Enter` y sal con `Ctrl + X`)\*.

* Ejemplo de como podría estar actualmente:

```bash
# /etc/fstab: static file system information.
#
# Use 'blkid' to print the universally unique identifier for a
# device; this may be used with UUID= as a more robust way to name devices
# that works even if disks are added and removed. See fstab(5).
#
# <file system> <mount point>   <type>  <options>       <dump>  <pass>
# / was on /dev/sda2 during curtin installation
/dev/disk/by-uuid/65d0f382-3177-46d4-8d64-151b34e1fec4 / ext4 defaults 0 1
# /boot/efi was on /dev/sda1 during curtin installation
/dev/disk/by-uuid/C2F7-F2B4 /boot/efi vfat defaults 0 1
/swap.img       none    swap    sw      0       0
```

- Ejemplo de como debe quedar:

```bash
# /etc/fstab: static file system information.
#
# Use 'blkid' to print the universally unique identifier for a
# device; this may be used with UUID= as a more robust way to name devices
# that works even if disks are added and removed. See fstab(5).
#
# <file system> <mount point>   <type>  <options>       <dump>  <pass>
# / was on /dev/sda2 during curtin installation
/dev/disk/by-uuid/65d0f382-3177-46d4-8d64-151b34e1fec4 / ext4 defaults,noatime 0 1
# /boot/efi was on /dev/sda1 during curtin installation
/dev/disk/by-uuid/C2F7-F2B4 /boot/efi vfat defaults 0 1
/swap.img        none    swap    sw      0       0
```

**Paso 4: Validar y aplicar sin reiniciar**
Aplica el cambio en caliente y asegúrate de que no haya errores de sintaxis:

```bash
# Verificar que la sintaxis de fstab es correcta
sudo findmnt --verify

# Aplicar el cambio inmediatamente
sudo mount -o remount /

# Confirmar que 'noatime' ya está activo (deberías verlo en la salida)
findmnt -no OPTIONS /

```

### 6.3. Limitar tamaño de logs del sistema (`journald`)

Evita que los registros del sistema consuman gigabytes del SSD:

```bash
sudo mkdir -p /etc/systemd/journald.conf.d
sudo tee /etc/systemd/journald.conf.d/size-limit.conf > /dev/null << 'EOF'
[Journal]
SystemMaxUse=100M
RuntimeMaxUse=50M
EOF

sudo systemctl restart systemd-journald

```

---

## 7. Optimización de Docker, Node.js y Angular

### 7.1. Configurar el demonio de Docker (`/etc/docker/daemon.json`)

Limita el tamaño de los logs de los contenedores y desactiva `userland-proxy` para ahorrar RAM y CPU en el enrutamiento de red:

```bash
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json > /dev/null << 'EOF'
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  },
  "userland-proxy": false,
  "live-restore": true,
  "storage-driver": "overlay2"
}
EOF

sudo systemctl restart docker

```

### 7.2. Buenas prácticas en tus `docker-compose.yml`

1. **Angular:** Compila siempre la app en un _multi-stage build_ y sírvela con `nginx:alpine` (~8 MB RAM). Nunca ejecutes `ng serve` en el servidor.
2. **Node.js:** Usa imágenes `node:22-alpine` (o LTS vigente), define `NODE_ENV=production` y limita la memoria de V8 para evitar que un pico sature los 8 GB del equipo:

```yaml
services:
  backend-node:
    image: node:22-alpine
    restart: unless-stopped
    environment:
      - NODE_ENV=production
      - NODE_OPTIONS=--max-old-space-size=512
    deploy:
      resources:
        limits:
          memory: 768M
          cpus: "1.5"

  frontend-angular:
    image: nginx:alpine
    restart: unless-stopped
    deploy:
      resources:
        limits:
          memory: 64M
          cpus: "0.5"
```

---

## 8. Reinicio y Validación Final

Reinicia el servidor para aplicar todos los cambios de kernel y hardware:

```bash
sudo reboot

```

Tras volver a conectar por SSH, verifica que todo esté activo:

```bash
# 1. Verificar ZRAM activo
zramctl

# 2. Verificar TCP BBR activo
sysctl net.ipv4.tcp_congestion_control

# 3. Verificar consumo en reposo y estados de energía C6/C7 del i5
sudo turbostat --Summary --quiet --show PkgWatt,CorWatt,Busy%,Bzy_MHz,Pkg%pc2,Pkg%pc3,Pkg%pc6,Pkg%pc7 --interval 2

```
