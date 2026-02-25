# Manual completo: Ubuntu + OpenClaw seguro y accesible desde internet (costo casi cero)

## 0) Objetivo

Tener una laptop con Ubuntu nuevo dedicada a OpenClaw, con acceso remoto desde cualquier lugar, pero solo para ti y con el menor costo posible.

Este manual prioriza:

- Seguridad real (sin puertos publicos abiertos en el router).
- Acceso privado con credenciales y MFA.
- Operacion simple y mantenible.
- Costo mensual ~0 (excepto tu futuro plan de modelo/LLM).

---

## 1) Arquitectura recomendada (la mas segura y barata)

**Recomendacion principal:** no publicar puertos al internet publico.

Usa esta arquitectura:

1. Laptop Ubuntu dedicada a OpenClaw.
2. OpenClaw corriendo como servicio local.
3. Acceso remoto por **Tailscale** (red privada WireGuard, plan personal gratuito).
4. SSH y dashboard solo por la red privada de Tailscale.
5. Router sin port-forwarding, sin DMZ, sin UPnP.

Con esto, la laptop esta "accesible desde internet", pero no expuesta de forma abierta.

---

## 2) Preparacion inicial de Ubuntu (dia 0)

## 2.1 Durante instalacion de Ubuntu

- Usa Ubuntu LTS reciente (ideal: 24.04 LTS).
- Activa cifrado de disco completo (LUKS) en el instalador.
- Crea un usuario administrador unico (ejemplo: `openclawadmin`).
- Usa password largo (minimo 16 caracteres, mejor frase de paso).

## 2.2 Actualiza e instala base del sistema

```bash
sudo apt update && sudo apt full-upgrade -y
sudo apt install -y curl git ca-certificates jq ufw fail2ban unattended-upgrades openssh-server gnupg
sudo systemctl enable --now ssh
sudo systemctl enable --now unattended-upgrades
```

Opcional pero recomendado:

```bash
sudo apt autoremove -y
```

---

## 3) Endurecimiento base de acceso SSH

> Importante: primero configura tu llave SSH y valida acceso antes de desactivar password por SSH.

## 3.1 Genera llave SSH en tu equipo cliente (tu laptop personal principal)

```bash
ssh-keygen -t ed25519 -a 100 -f ~/.ssh/openclaw_laptop -C "openclaw-laptop"
```

## 3.2 Copia llave publica al servidor Ubuntu (en red local al inicio)

```bash
ssh-copy-id -i ~/.ssh/openclaw_laptop.pub openclawadmin@IP_LOCAL_UBUNTU
```

## 3.3 Verifica login por llave

```bash
ssh -i ~/.ssh/openclaw_laptop openclawadmin@IP_LOCAL_UBUNTU
```

## 3.4 Endurece SSH (`/etc/ssh/sshd_config`)

Edita:

```bash
sudo nano /etc/ssh/sshd_config
```

Asegura estas lineas (ajusta usuario):

```text
PermitRootLogin no
PasswordAuthentication no
KbdInteractiveAuthentication no
PubkeyAuthentication yes
X11Forwarding no
AllowUsers openclawadmin
```

Valida y reinicia SSH:

```bash
sudo sshd -t
sudo systemctl restart ssh
```

---

## 4) Firewall correcto (UFW)

1) Bloquear todo ingreso por defecto.
2) Permitir SSH temporal para no bloquearte.
3) Luego restringir SSH solo a interfaz Tailscale.

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow OpenSSH
sudo ufw enable
sudo ufw status verbose
```

Despues de configurar Tailscale (seccion 5), restringe SSH:

```bash
sudo ufw delete allow OpenSSH
sudo ufw allow in on tailscale0 to any port 22 proto tcp
sudo ufw status numbered
```

---

## 5) Acceso remoto privado con Tailscale (plan gratuito)

## 5.1 Instalar Tailscale en la laptop servidor

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up --ssh
tailscale status
```

## 5.2 Instalar Tailscale en tus dispositivos cliente

Instala Tailscale en tu otra laptop, desktop o celular y entra con tu cuenta.

## 5.3 Asegurar cuenta y politica de acceso

- Activa MFA en tu proveedor de identidad (Google/GitHub/Microsoft, etc).
- Mantiene solo tus dispositivos autorizados en el tailnet.
- En ACL de Tailscale, deja acceso solo a tu usuario.

Ejemplo de ACL minima (conceptual):

```json
{
  "acls": [
    {
      "action": "accept",
      "src": ["tu_correo@dominio.com"],
      "dst": ["openclaw-laptop:22,3000"]
    }
  ]
}
```

> Si no usaras dashboard remoto, deja solo `:22`.

---

## 6) Instalacion oficial de OpenClaw en Ubuntu

Segun instalador oficial de OpenClaw, se requiere Node.js 22+.

## 6.1 Opcion recomendada (script oficial)

```bash
curl -fsSL https://openclaw.ai/install.sh | bash
```

Si quieres instalar sin asistente interactivo inicial:

```bash
curl -fsSL https://openclaw.ai/install.sh | bash -s -- --no-onboard
```

## 6.2 Opcion manual (si prefieres control total)

```bash
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash -
sudo apt-get install -y nodejs
sudo npm install -g openclaw@latest
openclaw onboard --install-daemon
```

## 6.3 Validacion basica

```bash
node -v
npm -v
openclaw doctor
openclaw status
```

Si no encuentra `openclaw`, revisa PATH:

```bash
npm prefix -g
echo "$PATH"
```

Si hace falta:

```bash
echo 'export PATH="$(npm prefix -g)/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
```

---

## 7) Manejo seguro de credenciales de OpenClaw

No guardes API keys en el historial shell ni en repositorios.

## 7.1 Archivo de entorno protegido

```bash
sudo install -d -m 700 /etc/openclaw
sudo nano /etc/openclaw/openclaw.env
sudo chmod 600 /etc/openclaw/openclaw.env
```

Contenido ejemplo:

```bash
OPENCLAW_API_KEY=PEGA_AQUI_TU_API_KEY
```

## 7.2 Inyectar variables al servicio systemd

Primero identifica el servicio (puede ser de sistema o de usuario):

```bash
systemctl list-unit-files --type=service | rg -i openclaw
systemctl --user list-unit-files --type=service | rg -i openclaw
```

Luego define el nombre real de unidad, por ejemplo:

- `openclaw.service`
- `openclaw-gateway.service`

Luego crea override (ajusta nombre real):

```bash
sudo systemctl edit NOMBRE_SERVICIO_OPENCLAW
```

Agrega:

```ini
[Service]
EnvironmentFile=/etc/openclaw/openclaw.env
```

Aplica cambios (sistema):

```bash
sudo systemctl daemon-reload
sudo systemctl restart NOMBRE_SERVICIO_OPENCLAW
sudo systemctl status NOMBRE_SERVICIO_OPENCLAW
```

Si tu unidad es de usuario, usa:

```bash
systemctl --user daemon-reload
systemctl --user restart NOMBRE_SERVICIO_OPENCLAW
systemctl --user status NOMBRE_SERVICIO_OPENCLAW
```

---

## 8) Uso remoto desde cualquier lugar (solo tu)

## 8.1 Conectarte por SSH (red privada Tailscale)

```bash
ssh -i ~/.ssh/openclaw_laptop openclawadmin@NOMBRE_O_IP_TAILSCALE
```

## 8.2 Usar dashboard sin exponer puertos publicos

Haz tunel SSH desde tu equipo cliente:

```bash
ssh -i ~/.ssh/openclaw_laptop -L 3333:127.0.0.1:3000 openclawadmin@NOMBRE_O_IP_TAILSCALE
```

Luego abre en tu navegador local:

```text
http://127.0.0.1:3333
```

Si tu dashboard usa otro puerto, reemplaza `3000`.

---

## 9) Hardening extra recomendado (alto valor, costo 0)

## 9.1 Fail2ban

```bash
sudo systemctl enable --now fail2ban
sudo fail2ban-client status
```

## 9.2 Actualizaciones automaticas

```bash
sudo dpkg-reconfigure --priority=low unattended-upgrades
```

## 9.3 Reducir superficie de ataque

- Desactiva UPnP en el router.
- No abras puertos 22/80/443 al internet publico.
- Si no usas Bluetooth en esa laptop, desactivalo.
- Mantiene solo software necesario para OpenClaw.

---

## 10) Backup y recuperacion

Minimo recomendado:

- Backup semanal cifrado de:
  - `~/.openclaw`
  - `/etc/openclaw`
  - `/etc/ssh`
- Destino: disco USB externo cifrado (costo unico, sin mensualidad).

Comando rapido de backup (tar cifrado con gpg simetrico):

```bash
sudo tar -czf - /home/openclawadmin/.openclaw /etc/openclaw /etc/ssh | gpg -c -o /ruta_usb/backup_openclaw_$(date +%F).tar.gz.gpg
```

---

## 11) Checklist final de seguridad (debe quedar en verde)

- [ ] Ubuntu LTS actualizado.
- [ ] Disco cifrado.
- [ ] `PermitRootLogin no`.
- [ ] `PasswordAuthentication no`.
- [ ] UFW activo con politica deny incoming.
- [ ] SSH permitido solo en `tailscale0`.
- [ ] Tailscale activo con MFA.
- [ ] Sin port-forwarding en router.
- [ ] OpenClaw funcionando (`openclaw doctor` OK).
- [ ] API keys fuera de historial y con permisos `600`.
- [ ] Backup semanal probado.

---

## 12) Costos estimados

- Ubuntu: **$0**
- OpenClaw (software): **$0**
- Tailscale plan personal: **$0**
- Seguridad base (UFW/fail2ban/unattended-upgrades): **$0**
- Costo variable real: **tu plan de modelo/LLM** (cuando lo actives)

---

## 13) Runbook rapido de operacion diaria

En la laptop servidor:

```bash
openclaw status
sudo systemctl status NOMBRE_SERVICIO_OPENCLAW
tailscale status
sudo ufw status verbose
```

Logs de OpenClaw:

```bash
sudo journalctl -u NOMBRE_SERVICIO_OPENCLAW -n 100 --no-pager
```

Actualizacion mensual recomendada:

```bash
sudo apt update && sudo apt full-upgrade -y
npm install -g openclaw@latest
sudo systemctl restart NOMBRE_SERVICIO_OPENCLAW
```

---

## 14) Nota final importante

Si quieres el mejor balance seguridad/costo:

- **No abras puertos del router**.
- Usa **Tailscale + MFA + SSH con llave**.
- Mantiene OpenClaw y Ubuntu siempre al dia.

Con eso tienes una laptop OpenClaw usable desde cualquier lugar, pero con acceso restringido a tus credenciales.
