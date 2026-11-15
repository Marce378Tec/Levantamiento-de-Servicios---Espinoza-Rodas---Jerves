# 01-SSH: Acceso remoto seguro con SSH (cambio de puerto)

## 01 Arquitectura y Requisitos

### Descripción
Se configura el servicio **SSH (sshd)** en CentOS Stream para la administración remota. Por seguridad, se cambia el puerto por defecto (22) a un puerto personalizado (ejemplo: **2222**). Para que el cambio funcione, el puerto nuevo se registra en **SELinux** y se abre en **Firewalld**.

### Topología

<img width="1774" height="887" alt="image" src="https://github.com/user-attachments/assets/45efd416-a8a7-48e6-aa46-744aa192e374" />


### Requisitos

| Requisito | Detalle |
|---|---|
| Sistema operativo | CentOS Stream (VirtualBox, adaptador en modo Puente/Bridge) |
| Servicio | `sshd` (OpenSSH Server) |
| Paquete adicional | `policycoreutils-python-utils` (incluye el comando `semanage`) |
| Cliente | MobaXterm |
| Permisos | Usuario con `sudo` |
| Puerto elegido | 2222/TCP (ejemplo, se puede usar otro) |

---

## 02 Despliegue y Configuración

### 2.1 Revisar el servicio

```bash
sudo systemctl status sshd
```
Muestra si el servicio SSH está activo (`active (running)`).

### 2.2 Respaldar la configuración

```bash
sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config.backup
```
Crea una copia de seguridad del archivo original. Si algo falla, se puede restaurar.

### 2.3 Cambiar el puerto

```bash
sudo nano /etc/ssh/sshd_config
```
Abre el archivo de configuración de SSH. Buscar la línea `#Port 22`, quitar el símbolo `#` y cambiar el número:

```
Port 2222
```

Guardar y salir: `Ctrl+O`, `Enter`, `Ctrl+X`.

> ⚠️ No cerrar la sesión SSH actual hasta confirmar que el puerto nuevo funciona. Así no se pierde el acceso al servidor.

### 2.4 Aplicar los cambios

```bash
sudo systemctl restart sshd
```
Reinicia el servicio para que use el puerto nuevo. Este paso se hace **después** de configurar SELinux y el firewall (sección 03).

```bash
sudo systemctl status sshd
```
Confirma que el servicio quedó activo con el puerto nuevo.

---

## 03 Aseguramiento Seguridad SELinux/Firewall

### 3.1 SELinux: registrar el puerto nuevo

SELinux solo permite que `sshd` use puertos autorizados. Si se cambia el puerto sin registrarlo, el servicio no inicia.

```bash
sudo dnf install -y policycoreutils-python-utils
```
Instala `semanage`, la herramienta para administrar políticas de SELinux.

```bash
sudo semanage port -a -t ssh_port_t -p tcp 2222
```
Agrega el puerto 2222/TCP al tipo `ssh_port_t`, que es el tipo de puerto permitido para SSH.

| Opción | Función |
|---|---|
| `-a` | Agrega una regla nueva |
| `-t ssh_port_t` | Tipo de puerto de SSH |
| `-p tcp` | Protocolo TCP |
| `2222` | Puerto nuevo (ejemplo) |

### 3.2 Firewalld

```bash
sudo firewall-cmd --permanent --add-port=2222/tcp
```
Abre el puerto 2222/TCP de forma permanente.

```bash
sudo firewall-cmd --reload
```
Recarga el firewall para aplicar la regla (también sirve `sudo systemctl reload firewalld`).

```bash
sudo firewall-cmd --list-ports
```
Verifica que `2222/tcp` aparezca en la lista de puertos abiertos.

### 3.3 Matriz de puertos

| Puerto | Protocolo | Uso | Regla |
|---|---|---|---|
| 2222 | TCP | SSH (puerto personalizado) | `--add-port=2222/tcp` |
| 22 | TCP | SSH por defecto | Servicio `ssh` de Firewalld (se puede cerrar al terminar las pruebas) |

### 3.4 Medidas de seguridad aplicadas
- Puerto cambiado: se reducen los intentos automáticos de ataque al puerto 22.
- Puerto registrado en SELinux (`ssh_port_t`), sin desactivar SELinux.
- Solo se abre en el firewall el puerto necesario.
- Copia de seguridad de `sshd_config` antes de editar.

---

## 04 Pruebas de Aceptación y Troubleshooting

### 4.1 Casos de prueba

| # | Prueba | Comando / Acción | Resultado esperado |
|---|---|---|---|
| 1 | Servicio activo | `sudo systemctl status sshd` | `active (running)` |
| 2 | Puerto escuchando | `sudo ss -tlnp \| grep 2222` | `sshd` escuchando en el puerto 2222 |
| 3 | Regla de SELinux | `sudo semanage port -l \| grep ssh` | `ssh_port_t` con `tcp 2222, 22` |
| 4 | Firewall | `sudo firewall-cmd --list-ports` | `2222/tcp` en la lista |
| 5 | Conexión por MobaXterm | Sesión SSH con la IP del CentOS y puerto `2222` | Inicia sesión correctamente |
| 6 | Conexión por terminal | `ssh -p 2222 usuario@IP_DEL_CENTOS` | Pide la contraseña y entra |

> 📸 **Evidencia:** agregar aquí las capturas de pantalla de cada prueba, incluida la sesión de MobaXterm con el puerto nuevo.

### 4.2 Plan de contingencia / Troubleshooting

#### Error 1: `Connection refused` o `Connection timed out` al conectarse
- **Causa:** el firewall bloquea el puerto nuevo, o el cliente sigue usando el puerto 22.
- **Solución:** ejecutar `sudo firewall-cmd --permanent --add-port=2222/tcp` y `sudo firewall-cmd --reload`. En MobaXterm, cambiar el puerto de la sesión a 2222.

#### Error 2: `sshd` no inicia después de cambiar el puerto
- **Causa:** SELinux bloquea el puerto porque no se registró con `semanage`.
- **Solución:** ejecutar `sudo semanage port -a -t ssh_port_t -p tcp 2222` y luego `sudo systemctl restart sshd`. Para ver el bloqueo: `sudo ausearch -m avc -ts recent`.

#### Error 3: El servicio falla por un error en `sshd_config`
- **Causa:** error de escritura en el archivo, o la línea `Port` sigue comentada con `#`.
- **Solución:** revisar el archivo con `sudo sshd -t` (muestra el error y la línea). Si no se puede arreglar, restaurar la copia: `sudo cp /etc/ssh/sshd_config.backup /etc/ssh/sshd_config`.

#### Cuidados adicionales
- Mantener abierta una sesión SSH mientras se prueba el puerto nuevo, para no perder el acceso.
- Si el puerto 2222 ya funciona, se puede cerrar el puerto 22 con `sudo firewall-cmd --permanent --remove-service=ssh` y `sudo firewall-cmd --reload`.
- Después de cada cambio en `sshd_config`, reiniciar el servicio.
