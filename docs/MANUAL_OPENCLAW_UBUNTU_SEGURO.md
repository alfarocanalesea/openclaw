# Manual definitivo: OpenClaw en laptop Ubuntu local, seguro, 24/7, paso a paso

Este manual esta escrito para ejecutarse de inicio a fin sin suposiciones.
Incluye:

- instalacion limpia de Ubuntu en laptop local
- creacion/verificacion de usuario no-root
- hardening de SSH/firewall
- acceso remoto seguro sin puertos publicos
- instalacion de OpenClaw
- arranque automatico tras reinicio
- canales multiusuario (Telegram y WhatsApp)
- backup semanal automatico al HDD

---

## 0) Perfil objetivo (tu caso)

- Equipo: Lenovo Ideapad, Core i3, 12 GB RAM, SSD 480 GB, HDD 1 TB.
- OpenClaw corre localmente en esa laptop.
- Uso por varias personas via mensajeria (Telegram/WhatsApp).
- Administracion remota por ti desde laptop/cel/tablet.
- Encendido 24/7.
- Costo de infraestructura mensual: ~0.

---

## 1) Arquitectura final recomendada (segura y barata)

1. Ubuntu 24.04 LTS en SSD.
2. OpenClaw instalado localmente y ejecutandose como servicio.
3. Administracion remota por Tailscale (red privada cifrada).
4. SSH solo con llave, sin password.
5. UFW bloqueando todo ingreso salvo SSH por `tailscale0`.
6. Router sin puertos abiertos, sin DMZ, sin UPnP.

Resultado: accesible desde internet para ti, sin exponer la laptop de forma publica.

---

## 2) Variables que usaras (edita una vez)

Usaremos estos nombres para que el manual sea copi/pega:

```bash
export LAB_USER="openclawadmin"
export LAB_HOSTNAME="openclaw-lab"
export BACKUP_MOUNT="/mnt/lab-hdd"
```

Si decides otro usuario o hostname, cambia solo estas variables y adapta comandos.

Para no perderlas tras reinicio, guardalas en `~/.bashrc`:

```bash
cat <<'EOF' >> ~/.bashrc
export LAB_USER="openclawadmin"
export LAB_HOSTNAME="openclaw-lab"
export BACKUP_MOUNT="/mnt/lab-hdd"
EOF
source ~/.bashrc
```

---

## FASE A - Instalacion limpia de Ubuntu

## 3) Instalar Ubuntu 24.04 LTS desde cero

### 3.1 Preparar USB booteable

1. Descarga ISO oficial Ubuntu 24.04 LTS desde ubuntu.com.
2. Crea USB booteable (balenaEtcher o Rufus).
3. Arranca laptop desde USB (menu boot de BIOS/UEFI).

### 3.2 Instalador Ubuntu (opciones recomendadas)

En el asistente:

1. Idioma y teclado.
2. Instalacion normal o minima (ambas validas; minima reduce ruido).
3. Red Wi-Fi conectada.
4. Tipo instalacion: manual y usa **solo el SSD de 480 GB** para Ubuntu.
   - No toques el HDD de 1 TB en este paso.
5. Activa cifrado de disco (LUKS) para el SSD.
6. Crea usuario **NO ROOT**:
   - Nombre de usuario: `openclawadmin` (o el que elijas)
   - Password fuerte 16+ caracteres
7. Completa instalacion y reinicia.

### 3.3 Verificacion post-instalacion (obligatoria)

Inicia sesion con tu usuario y ejecuta:

```bash
whoami
id -u
groups
```

Esperado:
- `whoami` muestra tu usuario (ej. `openclawadmin`)
- `id -u` debe ser distinto de `0`
- debes pertenecer al grupo `sudo`

Si no estas en sudo, corrige con:

```bash
sudo usermod -aG sudo "$USER"
newgrp sudo
```

---

## 4) Crear usuario no-root (solo si no existe)

Si por algun motivo no tienes usuario no-root admin, crea uno:

```bash
sudo adduser "$LAB_USER"
sudo usermod -aG sudo "$LAB_USER"
id "$LAB_USER"
```

Verifica que el UID no sea 0 y que tenga grupo sudo.

Cambiate al nuevo usuario:

```bash
su - "$LAB_USER"
```

---

## 5) Bloquear uso directo de root (obligatorio)

Ubuntu normalmente trae root bloqueado por defecto, pero validalo:

```bash
sudo passwd -l root
sudo passwd -S root
```

Esperado: estado `L` (locked).

---

## 6) Actualizar base del sistema

```bash
sudo apt update && sudo apt full-upgrade -y
sudo apt install -y curl git ca-certificates jq ufw fail2ban unattended-upgrades openssh-server gnupg rsync
sudo systemctl enable --now ssh
sudo systemctl enable --now fail2ban
sudo systemctl enable --now unattended-upgrades
sudo timedatectl set-ntp true
sudo apt autoremove -y
```

Verifica:

```bash
systemctl is-active ssh
systemctl is-active fail2ban
systemctl is-active unattended-upgrades
timedatectl status | rg "NTP service|System clock synchronized"
```

---

## 7) Configurar hostname fijo del laboratorio

```bash
sudo hostnamectl set-hostname "$LAB_HOSTNAME"
hostnamectl
```

Opcional: reinicia para limpiar caches de red/nombre.

---

## FASE B - Disco de backup (HDD 1 TB)

## 8) Montar HDD para backups automaticos

### 8.1 Identificar el HDD correcto

```bash
lsblk -o NAME,SIZE,FSTYPE,TYPE,MOUNTPOINT,UUID
```

Identifica el disco/particion de 1 TB (ejemplo comun: `/dev/sdb1`).

### 8.2 Formatear SOLO si esta vacio o no lo necesitas

**Advertencia:** este comando borra el contenido de la particion objetivo.

```bash
sudo mkfs.ext4 /dev/sdb1
```

Si ya tiene datos utiles, salta este paso.

### 8.3 Crear punto de montaje y montar

```bash
sudo mkdir -p "$BACKUP_MOUNT"
sudo mount /dev/sdb1 "$BACKUP_MOUNT"
df -h | rg "$BACKUP_MOUNT"
```

### 8.4 Montaje persistente en reinicio (fstab)

Obtiene UUID:

```bash
sudo blkid /dev/sdb1
```

Agrega entrada en `/etc/fstab` (reemplaza UUID real):

```bash
echo 'UUID=REEMPLAZA_UUID_REAL  /mnt/lab-hdd  ext4  defaults,nofail  0  2' | sudo tee -a /etc/fstab
sudo mount -a
df -h | rg "/mnt/lab-hdd"
```

---

## FASE C - Acceso remoto seguro (sin puertos publicos)

## 9) Configurar SSH con llave (sin password)

### 9.1 Generar llave en tu equipo cliente (tu laptop personal)

En el cliente:

```bash
ssh-keygen -t ed25519 -a 100 -f ~/.ssh/openclaw_laptop -C "openclaw-lab-admin"
```

### 9.2 Copiar llave al servidor (inicialmente por IP local)

En cliente:

```bash
ssh-copy-id -i ~/.ssh/openclaw_laptop.pub "$LAB_USER@IP_LOCAL_UBUNTU"
```

### 9.3 Probar acceso por llave

```bash
ssh -i ~/.ssh/openclaw_laptop "$LAB_USER@IP_LOCAL_UBUNTU"
```

Si entra sin pedir password del usuario remoto, esta correcto.

## 10) Endurecer SSH server

En la laptop OpenClaw:

```bash
sudo mkdir -p /etc/ssh/sshd_config.d
sudo tee /etc/ssh/sshd_config.d/99-openclaw-hardening.conf > /dev/null <<'EOF'
PermitRootLogin no
PasswordAuthentication no
KbdInteractiveAuthentication no
PubkeyAuthentication yes
X11Forwarding no
AllowUsers openclawadmin
EOF
```

Si tu usuario no es `openclawadmin`, edita esa linea:

```bash
sudo sed -i "s/^AllowUsers.*/AllowUsers $USER/" /etc/ssh/sshd_config.d/99-openclaw-hardening.conf
```

Validar y recargar:

```bash
sudo sshd -t
sudo systemctl restart ssh
sudo systemctl status ssh --no-pager
```

## 11) Instalar Tailscale (gratis) para administracion remota privada

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up --ssh
tailscale status
tailscale ip -4
```

Anota el nombre o IP de Tailscale que aparezca.

### 11.1 Configurar MFA (obligatorio)

1. Entra a panel de Tailscale.
2. Verifica que tu proveedor de identidad tenga 2FA/MFA activo.
3. Revoca dispositivos viejos/no usados.

### 11.2 Prueba SSH por Tailscale

Desde cliente:

```bash
ssh -i ~/.ssh/openclaw_laptop "$LAB_USER@NOMBRE_O_IP_TAILSCALE"
```

`NOMBRE_O_IP_TAILSCALE` lo obtienes en la laptop OpenClaw con:

```bash
tailscale status
tailscale ip -4
```

## 12) UFW estricto (solo SSH por Tailscale)

Ejecuta en la laptop OpenClaw:

```bash
sudo ufw --force reset
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow in on tailscale0 to any port 22 proto tcp
sudo ufw --force enable
sudo ufw status verbose
```

No abras puertos 22/80/443 en router.

---

## 13) Endurecer router (muy importante)

En la configuracion del router de casa:

- desactiva UPnP
- desactiva DMZ
- elimina cualquier port-forwarding hacia la laptop OpenClaw

Con Tailscale no necesitas abrir puertos publicos.

---

## FASE D - Instalacion OpenClaw local

## 14) Instalar OpenClaw en Ubuntu

### 14.1 Instalacion oficial

```bash
curl -fsSL https://openclaw.ai/install.sh | bash -s -- --no-onboard
```

### 14.2 Ejecutar onboarding + daemon

```bash
openclaw onboard --install-daemon
```

En el wizard usa:

- mode: local
- provider: API externa (tu plan)
- gateway bind: 127.0.0.1
- gateway auth: token habilitado
- tailscale exposure: off
- canales: telegram/whatsapp segun plan
- daemon: install/enable

## 15) Verificar instalacion OpenClaw

```bash
node -v
npm -v
openclaw --version
openclaw doctor
openclaw status
```

Si `openclaw` no existe en PATH:

```bash
echo 'export PATH="$(npm prefix -g)/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
openclaw --version
```

---

## 16) Garantizar arranque automatico tras reboot

### 16.1 Detectar unidad systemd de OpenClaw

```bash
systemctl --user list-unit-files --type=service | rg -i openclaw
```

Guarda la unidad principal:

```bash
export OPENCLAW_UNIT="$(systemctl --user list-unit-files --type=service | awk '/openclaw/ {print $1; exit}')"
echo "$OPENCLAW_UNIT"
```

Si sale vacio, repite:

```bash
openclaw onboard --install-daemon
```

### 16.2 Habilitar y persistir sin sesion grafica

```bash
systemctl --user enable --now "$OPENCLAW_UNIT"
sudo loginctl enable-linger "$USER"
systemctl --user status "$OPENCLAW_UNIT" --no-pager
```

### 16.3 Evitar suspension accidental (laptop 24/7)

```bash
sudo sed -i 's/^#HandleLidSwitch=.*/HandleLidSwitch=ignore/' /etc/systemd/logind.conf
sudo sed -i 's/^#HandleLidSwitchExternalPower=.*/HandleLidSwitchExternalPower=ignore/' /etc/systemd/logind.conf
sudo systemctl restart systemd-logind
```

### 16.4 Prueba final de reinicio

```bash
sudo reboot
```

Al volver:

```bash
openclaw status
systemctl --user is-active "$OPENCLAW_UNIT"
```

Esperado: `active`.

---

## FASE E - Canales multiusuario (Telegram + WhatsApp)

## 17) Telegram paso a paso

1. En Telegram, abre `@BotFather`.
2. Ejecuta `/newbot`.
3. Guarda `BOT_TOKEN`.
4. Configura Telegram en OpenClaw (comandos exactos):

```bash
read -rsp "Pega BOT_TOKEN: " BOT_TOKEN; echo
openclaw config set channels.telegram.enabled true
openclaw config set channels.telegram.botToken "$BOT_TOKEN"
openclaw config set channels.telegram.dmPolicy pairing
openclaw config set channels.telegram.groupPolicy allowlist
openclaw config get channels.telegram.dmPolicy
openclaw config get channels.telegram.groupPolicy
unset BOT_TOKEN
```

Si tu version no soporta `config set`, usa asistente interactivo:

```bash
openclaw config
```

5. Reinicia servicio OpenClaw para aplicar cambios:

```bash
systemctl --user restart "$OPENCLAW_UNIT"
```

Inicia/verifica gateway:

```bash
openclaw status
openclaw channels status
```

Aprobar usuarios entrantes:

```bash
openclaw pairing list telegram
openclaw pairing approve telegram CODIGO
```

No habilites webhook publico al inicio.

## 18) WhatsApp paso a paso

Define politicas de acceso:

```bash
openclaw config set channels.whatsapp.dmPolicy pairing
openclaw config set channels.whatsapp.groupPolicy allowlist
openclaw config get channels.whatsapp.dmPolicy
openclaw config get channels.whatsapp.groupPolicy
```

Si `config set` no funciona, usa:

```bash
openclaw config
```

Login por QR:

```bash
openclaw channels login --channel whatsapp
```

Verificar estado:

```bash
openclaw channels status
```

Aplica cambios de servicio:

```bash
systemctl --user restart "$OPENCLAW_UNIT"
```

Politica inicial recomendada:
- `dmPolicy: pairing`
- `groupPolicy: allowlist`

Aprobar usuarios:

```bash
openclaw pairing list whatsapp
openclaw pairing approve whatsapp CODIGO
```

Recomendacion: usar numero dedicado para OpenClaw cuando pases a uso estable.

## 19) Regla operativa multiusuario recomendada

- Semana 1-2: pairing (aprobacion manual).
- Semana 3+: migrar a allowlist fija.
- Nunca compartir acceso SSH con participantes.
- Los participantes usan solo Telegram/WhatsApp.

---

## FASE F - Backups automaticos semanales

## 20) Script de backup al HDD

```bash
sudo tee /usr/local/sbin/openclaw-backup.sh > /dev/null <<'EOF'
#!/usr/bin/env bash
set -euo pipefail

OPENCLAW_USER="openclawadmin"
TARGET_BASE="/mnt/lab-hdd/backups/openclaw"
STAMP="$(date +%F_%H-%M-%S)"
DEST="${TARGET_BASE}/${STAMP}"

mkdir -p "${DEST}"

if [ -d "/home/${OPENCLAW_USER}/.openclaw" ]; then
  tar -czf "${DEST}/home-openclaw.tgz" -C "/home/${OPENCLAW_USER}" ".openclaw"
fi

if [ -d "/etc/ssh" ]; then
  tar -czf "${DEST}/etc-ssh.tgz" -C /etc ssh
fi

if [ -f "/etc/ssh/sshd_config.d/99-openclaw-hardening.conf" ]; then
  cp -a "/etc/ssh/sshd_config.d/99-openclaw-hardening.conf" "${DEST}/"
fi

find "${TARGET_BASE}" -mindepth 1 -maxdepth 1 -type d -mtime +45 -exec rm -rf {} +
EOF

sudo chmod 700 /usr/local/sbin/openclaw-backup.sh
```

Si tu usuario no es `openclawadmin`, edita el script:

```bash
sudo sed -i "s/^OPENCLAW_USER=.*/OPENCLAW_USER=\"$USER\"/" /usr/local/sbin/openclaw-backup.sh
```

## 21) Programar backup semanal

```bash
sudo tee /etc/systemd/system/openclaw-backup.service > /dev/null <<'EOF'
[Unit]
Description=Weekly OpenClaw backup

[Service]
Type=oneshot
ExecStart=/usr/local/sbin/openclaw-backup.sh
EOF
```

```bash
sudo tee /etc/systemd/system/openclaw-backup.timer > /dev/null <<'EOF'
[Unit]
Description=Run OpenClaw backup weekly

[Timer]
OnCalendar=Sun *-*-* 03:00:00
Persistent=true

[Install]
WantedBy=timers.target
EOF
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now openclaw-backup.timer
sudo systemctl start openclaw-backup.service
sudo systemctl list-timers | rg openclaw-backup
```

Verifica archivos backup:

```bash
ls -lah /mnt/lab-hdd/backups/openclaw
```

---

## FASE G - Operacion diaria y mantenimiento

## 22) Comandos de salud diaria

```bash
openclaw status
openclaw channels status
tailscale status
sudo ufw status verbose
systemctl --user status "$OPENCLAW_UNIT" --no-pager
```

Logs OpenClaw:

```bash
journalctl --user -u "$OPENCLAW_UNIT" -n 200 --no-pager
```

## 23) Actualizacion mensual recomendada

```bash
sudo apt update && sudo apt full-upgrade -y
npm install -g openclaw@latest
systemctl --user restart "$OPENCLAW_UNIT"
openclaw doctor
```

---

## FASE H - Troubleshooting rapido

## 24) Error: no puedes entrar por SSH

1. Entra localmente en la laptop.
2. Verifica servicio:

```bash
sudo systemctl status ssh --no-pager
sudo sshd -t
```

3. Verifica UFW:

```bash
sudo ufw status verbose
ip -br a | rg tailscale0
```

## 25) Error: OpenClaw no levanta tras reboot

```bash
systemctl --user status "$OPENCLAW_UNIT" --no-pager
journalctl --user -u "$OPENCLAW_UNIT" -n 200 --no-pager
sudo loginctl show-user "$USER" | rg Linger
```

Si `Linger=no`:

```bash
sudo loginctl enable-linger "$USER"
systemctl --user restart "$OPENCLAW_UNIT"
```

## 26) Error: `openclaw` no encontrado

```bash
npm prefix -g
echo "$PATH"
echo 'export PATH="$(npm prefix -g)/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
```

---

## 27) Prueba de aceptacion final (debe quedar todo OK)

Ejecuta:

```bash
whoami
id -u
sudo passwd -S root
tailscale status
sudo ufw status verbose
openclaw doctor
openclaw status
openclaw channels status
systemctl --user is-active "$OPENCLAW_UNIT"
sudo systemctl is-enabled openclaw-backup.timer
```

Criterio de aprobado:

- usuario actual no es root (`id -u` != 0)
- root bloqueado (`L`)
- tailscale activo
- ufw activo con SSH solo en tailscale0
- openclaw en estado sano
- unidad systemd activa
- timer de backup habilitado

---

## 28) Costos

- Ubuntu: $0
- OpenClaw software: $0
- Tailscale personal: $0
- Infra adicional: $0
- costo variable: tu API externa (modelo)

---

## 29) Recomendacion operativa final

Para tu laboratorio:

1. administra solo por Tailscale + SSH llave.
2. no abras puertos del router.
3. usuarios finales solo por Telegram/WhatsApp.
4. backups semanales obligatorios.
5. actualizacion mensual obligatoria.
