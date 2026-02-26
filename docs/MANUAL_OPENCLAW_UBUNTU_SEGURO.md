# Manual personalizado: OpenClaw local en laptop Ubuntu (multiusuario, seguro, costo casi cero)

## 0) Perfil y objetivo de este manual

Este documento esta adaptado a tu escenario real:

- Laptop Lenovo Ideapad (Core i3, 12 GB RAM, SSD 480 GB, HDD 1 TB).
- Instalacion limpia desde cero.
- OpenClaw correra localmente en esa laptop.
- Uso por varias personas via canales de mensajeria (WhatsApp/Telegram).
- Laptop encendida 24/7, con recuperacion automatica tras reinicio.
- Acceso remoto de administracion desde laptop/cel/tablet.
- Presupuesto de infraestructura mensual: idealmente $0.
- Cerebro/modelo: API externa (equilibrio costo/calidad).

---

## 1) Decisiones tecnicas recomendadas para tu caso

1. **Ubuntu recomendado:** 24.04 LTS.
2. **Sin puertos abiertos en router:** no usar port forwarding, DMZ ni UPnP.
3. **Administracion remota segura:** Tailscale gratis + MFA + SSH con llave.
4. **Operacion multiusuario:** por canales OpenClaw (Telegram/WhatsApp), no por cuentas SSH.
5. **Arranque automatico:** daemon de OpenClaw en systemd + `linger` activo.
6. **Backups semanales:** automaticos al HDD de 1 TB.

---

## 2) Instalacion limpia de Ubuntu (dia 0)

## 2.1 Durante el instalador

- Elige **Ubuntu 24.04 LTS**.
- Activa cifrado de disco completo (LUKS) para el SSD.
- Crea usuario admin unico, por ejemplo `openclawadmin`.
- Usa password fuerte (ideal 16+ caracteres).

## 2.2 Primer arranque: paquetes base

```bash
sudo apt update && sudo apt full-upgrade -y
sudo apt install -y curl git ca-certificates jq ufw fail2ban unattended-upgrades openssh-server gnupg rsync
sudo systemctl enable --now ssh
sudo systemctl enable --now fail2ban
sudo systemctl enable --now unattended-upgrades
sudo apt autoremove -y
```

---

## 3) Seguridad base del sistema

## 3.1 SSH por llave (sin password)

En tu equipo cliente (desde donde administras):

```bash
ssh-keygen -t ed25519 -a 100 -f ~/.ssh/openclaw_laptop -C "openclaw-laptop"
```

Copia llave al servidor (en red local al inicio):

```bash
ssh-copy-id -i ~/.ssh/openclaw_laptop.pub openclawadmin@IP_LOCAL_UBUNTU
```

Prueba acceso:

```bash
ssh -i ~/.ssh/openclaw_laptop openclawadmin@IP_LOCAL_UBUNTU
```

## 3.2 Endurecer SSH

```bash
sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config.bak.$(date +%F)
sudo nano /etc/ssh/sshd_config
```

Deja estas directivas:

```text
PermitRootLogin no
PasswordAuthentication no
KbdInteractiveAuthentication no
PubkeyAuthentication yes
X11Forwarding no
AllowUsers openclawadmin
```

Valida y aplica:

```bash
sudo sshd -t
sudo systemctl restart ssh
```

## 3.3 Firewall UFW

Primero habilita reglas base:

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow OpenSSH
sudo ufw enable
sudo ufw status verbose
```

Luego, cuando Tailscale ya este operativo (seccion 4), restringe SSH a esa interfaz:

```bash
sudo ufw delete allow OpenSSH
sudo ufw allow in on tailscale0 to any port 22 proto tcp
sudo ufw status numbered
```

---

## 4) Acceso remoto seguro y gratis (Tailscale)

### Que es Tailscale en una linea
Es una red privada cifrada (WireGuard) que te deja entrar a tu laptop desde cualquier lugar sin exponer puertos publicos.

## 4.1 Instalar y activar

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up --ssh
tailscale status
tailscale ip -4
```

## 4.2 Seguridad de cuenta (MFA)

- Inicia sesion en Tailscale con tu cuenta principal (Google/GitHub/Microsoft).
- Asegura que esa cuenta ya tenga 2FA/MFA activa (tu caso: si).
- Elimina de Tailscale cualquier dispositivo que ya no uses.

## 4.3 Acceso remoto desde cualquier dispositivo

Instala Tailscale en tu laptop, celular y tablet.  
Para administrar la laptop OpenClaw:

```bash
ssh -i ~/.ssh/openclaw_laptop openclawadmin@NOMBRE_O_IP_TAILSCALE
```

---

## 5) Instalar OpenClaw localmente en Ubuntu

Requisito oficial actual: Node.js 22+ (el instalador oficial lo maneja).

## 5.1 Instalacion recomendada (oficial)

```bash
curl -fsSL https://openclaw.ai/install.sh | bash
```

Si quieres relanzar configuracion:

```bash
openclaw onboard
```

## 5.2 Decisiones dentro del wizard (recomendadas para tu caso)

Cuando ejecutes onboarding, usa:

- Modo: **Local**
- Proveedor modelo: **API externa** (tu plan elegido)
- Workspace: default (`~/.openclaw/workspace`) o ruta dedicada
- Gateway bind: **127.0.0.1** (no publico)
- Tailscale exposure: **Off** (administracion remota ya va por SSH/Tailscale)
- Canales: Telegram y/o WhatsApp
- Daemon: **instalar y habilitar**

## 5.3 Verificacion

```bash
openclaw doctor
openclaw status
openclaw channels list
```

---

## 6) Arranque automatico y recuperacion tras reinicio

Tu requisito: que al encender la laptop todo vuelva automaticamente.

## 6.1 Identificar unidad systemd de OpenClaw

```bash
systemctl --user list-unit-files --type=service | rg -i openclaw
```

Toma el nombre que aparezca (ejemplo: `openclaw.service`) y reemplaza `NOMBRE_UNIDAD`.

## 6.2 Habilitar autoarranque

```bash
systemctl --user enable --now NOMBRE_UNIDAD
sudo loginctl enable-linger "$USER"
systemctl --user status NOMBRE_UNIDAD
```

`enable-linger` evita que el servicio dependa de una sesion grafica activa.

## 6.3 Prueba real

```bash
sudo reboot
```

Despues de reiniciar:

```bash
openclaw status
systemctl --user status NOMBRE_UNIDAD
```

---

## 7) Multiusuario por mensajeria (sin abrir puertos del router)

## 7.1 Telegram (recomendado para iniciar rapido)

1. Crea bot con **@BotFather** y guarda el token.
2. Configura canal Telegram en OpenClaw (wizard o config).
3. Politica recomendada al inicio:
   - `dmPolicy: "pairing"`
   - `groupPolicy: "allowlist"`
4. Arranca gateway (si no esta en daemon):

```bash
openclaw gateway
```

5. Aprobar solicitudes de usuarios:

```bash
openclaw pairing list telegram
openclaw pairing approve telegram CODIGO
```

Nota: no uses webhook publico al inicio; evita exponer endpoints.

## 7.2 WhatsApp (soportado por WhatsApp Web / Baileys)

1. Login del canal por QR:

```bash
openclaw channels login --channel whatsapp
```

2. Politica recomendada al inicio:
   - `dmPolicy: "pairing"`
   - `groupPolicy: "allowlist"`
3. Aprobar usuarios:

```bash
openclaw pairing list whatsapp
openclaw pairing approve whatsapp CODIGO
```

4. Para produccion estable, es mejor usar numero dedicado para OpenClaw.

## 7.3 Politica recomendada para tus primeros 30 dias

- Semana 1-2: `pairing` (control de entrada por aprobacion manual).
- Semana 3+: migrar a `allowlist` con usuarios/numeros autorizados.
- No compartir credenciales SSH con participantes; solo usar canales.

---

## 8) Operacion 24/7 y mantenimiento

## 8.1 Comandos de salud

```bash
openclaw status
openclaw channels status
tailscale status
sudo ufw status verbose
```

Logs:

```bash
systemctl --user status NOMBRE_UNIDAD
journalctl --user -u NOMBRE_UNIDAD -n 200 --no-pager
```

## 8.2 Actualizacion mensual recomendada

```bash
sudo apt update && sudo apt full-upgrade -y
npm install -g openclaw@latest
systemctl --user restart NOMBRE_UNIDAD
```

---

## 9) Backups automaticos semanales al HDD (1 TB)

Objetivo: recuperar rapido si hay corrupcion o error humano.

## 9.1 Script de backup

```bash
sudo tee /usr/local/sbin/openclaw-backup.sh > /dev/null <<'EOF'
#!/usr/bin/env bash
set -euo pipefail

OPENCLAW_USER="openclawadmin"
SRC_HOME="/home/${OPENCLAW_USER}/.openclaw"
TARGET_BASE="/mnt/lab-hdd/backups/openclaw"
STAMP="$(date +%F_%H-%M-%S)"
DEST="${TARGET_BASE}/${STAMP}"

mkdir -p "${DEST}"

if [ -d "${SRC_HOME}" ]; then
  tar -czf "${DEST}/home-openclaw.tgz" -C "/home/${OPENCLAW_USER}" ".openclaw"
fi

if [ -d "/etc/ssh" ]; then
  tar -czf "${DEST}/etc-ssh.tgz" -C /etc ssh
fi

if [ -d "/etc/openclaw" ]; then
  tar -czf "${DEST}/etc-openclaw.tgz" -C /etc openclaw
fi

find "${TARGET_BASE}" -mindepth 1 -maxdepth 1 -type d -mtime +45 -exec rm -rf {} +
EOF

sudo chmod 700 /usr/local/sbin/openclaw-backup.sh
```

Si tu HDD esta montado en otra ruta, cambia `TARGET_BASE`.

## 9.2 Timer semanal con systemd

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
sudo systemctl list-timers | rg openclaw-backup
```

---

## 10) Checklist final (debe quedar en verde)

- [ ] Ubuntu 24.04 LTS instalado limpio y actualizado.
- [ ] Disco principal cifrado.
- [ ] OpenClaw instalado y funcional (`openclaw doctor` y `openclaw status`).
- [ ] Daemon de OpenClaw habilitado y probado tras reboot.
- [ ] Tailscale activo con MFA.
- [ ] SSH solo por llave y solo por `tailscale0`.
- [ ] Sin puertos abiertos en router (sin DMZ, sin UPnP).
- [ ] Canales con `pairing` o `allowlist` (no `open` en produccion).
- [ ] Backup semanal automatico activo en HDD.

---

## 11) Costos esperados

- Ubuntu: **$0**
- OpenClaw software: **$0**
- Tailscale plan personal: **$0**
- Infra de red extra: **$0**
- Costo variable: **tu plan de API externa**

---

## 12) Preguntas finales para dejar este manual 100% cerrado

1. Quieres arrancar primero con **Telegram**, **WhatsApp**, o ambos al mismo tiempo?
2. Para WhatsApp, usaras numero dedicado para OpenClaw o tu numero personal?
3. Quieres que te deje una plantilla exacta de `allowlist` inicial (nombres y telefonos)?
4. Que proveedor/modelo API usaras primero para balance costo/calidad?
