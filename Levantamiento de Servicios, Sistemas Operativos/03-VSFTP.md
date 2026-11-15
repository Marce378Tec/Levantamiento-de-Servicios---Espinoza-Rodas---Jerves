# 03-FTP: Servidor FTP con vsftpd (usuario sin shell)

## 01 Arquitectura y Requisitos

### Descripción
Se despliega un servidor FTP con **vsftpd** en CentOS Stream. El acceso se hace con un usuario local dedicado (`usuario_ftp`) que **no tiene shell** (`/sbin/nologin`). Así puede transferir archivos, pero no puede abrir una terminal en el servidor.

### Topología

<img width="1672" height="941" alt="image" src="https://github.com/user-attachments/assets/9bd069dd-529d-4435-8ebd-02cf5d3a89d1" />


### Requisitos

| Requisito | Detalle |
|---|---|
| Sistema operativo | CentOS Stream (VirtualBox, adaptador en modo Puente/Bridge) |
| Paquete | `vsftpd` |
| Cliente | MobaXterm (sesión FTP/SFTP) |
| Permisos | Usuario con `sudo` |
| Red | La VM y el cliente deben verse en la misma red |

---

## 02 Despliegue y Configuración

### 2.1 Instalar e iniciar el servicio

```bash
sudo dnf install vsftpd -y
```
Instala el paquete vsftpd. La opción `-y` acepta la instalación automáticamente.

```bash
sudo systemctl status vsftpd
```
Muestra si el servicio está activo o apagado.

```bash
sudo systemctl start vsftpd
```
Inicia el servicio (solo si estaba apagado).

```bash
sudo systemctl enable vsftpd
```
Hace que el servicio arranque automáticamente al encender el servidor.

### 2.2 Respaldar y editar la configuración

```bash
sudo cp /etc/vsftpd/vsftpd.conf /etc/vsftpd/vsftpd.conf.backup
```
Crea una copia de seguridad del archivo original. Si algo falla, se puede restaurar.

```bash
sudo nano /etc/vsftpd/vsftpd.conf
```
Abre el archivo de configuración. Las directivas deben quedar así:

```ini
# Deshabilitar el acceso anónimo
anonymous_enable=NO

# Permitir el acceso a usuarios locales del sistema
local_enable=YES

# Permitir escritura (subida de archivos)
write_enable=YES

# Enjaular al usuario en su directorio personal (quitar el "#")
chroot_local_user=YES

# Permitir escritura en la raíz del chroot (obligatorio con chroot_local_user=YES)
allow_writeable_chroot=YES

# Omitir la validación de shell (necesario al usar /sbin/nologin)
check_shell=NO

# Escuchar en IPv4 (quitar el "#" y poner YES)
listen=YES

# Desactivar IPv6
listen_ipv6=NO
```

| Directiva | Función |
|---|---|
| `anonymous_enable=NO` | Bloquea el acceso sin usuario ni contraseña |
| `local_enable=YES` | Permite iniciar sesión con usuarios del sistema |
| `write_enable=YES` | Permite subir, modificar y crear carpetas |
| `chroot_local_user=YES` | El usuario solo ve su carpeta personal |
| `allow_writeable_chroot=YES` | Evita el error de chroot con carpeta escribible |
| `check_shell=NO` | vsftpd no revisa `/etc/shells`, necesario con `/sbin/nologin` |
| `listen=YES` | vsftpd corre como servicio independiente (IPv4) |
| `listen_ipv6=NO` | No usa IPv6 (no se pueden activar los dos a la vez) |

### 2.3 Crear el usuario sin shell

```bash
sudo useradd -m -s /sbin/nologin usuario_ftp
```
Crea el usuario con carpeta personal (`-m`) y sin acceso a terminal (`-s /sbin/nologin`).

```bash
sudo passwd usuario_ftp
```
Asigna una contraseña segura al usuario.

### 2.4 Revisar la lista de usuarios bloqueados

```bash
sudo nano /etc/vsftpd/user_list
```
Si `usuario_ftp` aparece en esta lista, **hay que quitarlo**. Con la configuración por defecto, los usuarios de esta lista no pueden entrar por FTP.

### 2.5 Ajustar PAM (si el login falla)

```bash
sudo nano /etc/pam.d/vsftpd
```
Buscar la línea con `pam_shells.so` y comentarla con `#`:

```
#auth required pam_shells.so
```
Esta línea rechaza a los usuarios con shell no válido (como `/sbin/nologin`).

### 2.6 Carpeta de destino compartida (usuario FTP y usuario de escritorio)

Con este paso, los archivos que se suben desde MobaXterm caen en una carpeta que también se puede abrir desde el escritorio de CentOS. En los comandos, `usuario_normal` es el usuario de la sesión de escritorio y `usuario_ftp` es el usuario FTP.

**1. Crear la carpeta de destino** (desde la sesión del usuario normal):

```bash
mkdir -p /home/usuario_normal/Archivos_FTP
```
Crea la carpeta donde llegarán los archivos. La opción `-p` evita errores si ya existe.

**2. Cambiar la carpeta de inicio del usuario FTP:**

```bash
sudo usermod -d /home/usuario_normal/Archivos_FTP usuario_ftp
```
Cambia el "hogar" de `usuario_ftp`. Al conectarse, entra directo a `Archivos_FTP` en lugar de su carpeta vacía.

**3. Configurar los permisos compartidos:**

```bash
sudo chown -R usuario_normal:usuario_ftp /home/usuario_normal/Archivos_FTP
```
El dueño es el usuario normal y el grupo es `usuario_ftp`, así los dos tienen acceso.

```bash
sudo chmod 2775 /home/usuario_normal/Archivos_FTP
```
Da permisos completos al dueño y al grupo (`775`). El `2` inicial (*setgid*) hace que los archivos nuevos hereden el grupo de la carpeta.

```bash
sudo chmod 711 /home/usuario_normal
```
Permite que el usuario FTP **pase** por la carpeta personal hasta llegar a `Archivos_FTP`, sin poder ver su contenido.

**4. Permitir que el grupo edite los archivos subidos:**

```bash
sudo nano /etc/vsftpd/vsftpd.conf
```
Agregar o cambiar esta línea:

```ini
local_umask=002
```
Por defecto, vsftpd crea los archivos subidos sin permiso de escritura para el grupo. Con `002`, el usuario normal puede **abrir y editar** los archivos, no solo verlos.

### 2.7 Reiniciar el servicio

```bash
sudo systemctl restart vsftpd
```
Aplica todos los cambios de configuración.

---

## 03 Aseguramiento Seguridad SELinux/Firewall

### 3.1 Firewalld

```bash
sudo firewall-cmd --permanent --add-service=ftp
```
Abre el servicio FTP (puerto 21) de forma permanente.

```bash
sudo firewall-cmd --reload
```
Recarga el firewall para aplicar la regla.

```bash
sudo firewall-cmd --list-all
```
Verifica que `ftp` aparezca en la lista de servicios permitidos.

### 3.2 SELinux

```bash
sudo setsebool -P ftpd_full_access on
```
Activa el permiso para que vsftpd pueda leer y escribir en los directorios de los usuarios. La opción `-P` lo deja permanente después de reiniciar.

### 3.3 Matriz de puertos

| Puerto | Protocolo | Uso | Regla |
|---|---|---|---|
| 21 | TCP | Control FTP | `--add-service=ftp` |
| 20 | TCP | Datos (modo activo) | Incluido en el servicio `ftp` |
| Rango pasivo | TCP | Datos (modo pasivo) | Opcional, ver nota |

> **Nota (modo pasivo):** si se necesita FTP pasivo, se define un rango en `vsftpd.conf` (por ejemplo `pasv_min_port=30000` y `pasv_max_port=30100`) y se abre en el firewall con `sudo firewall-cmd --permanent --add-port=30000-30100/tcp`.

### 3.4 Medidas de seguridad aplicadas

- Acceso anónimo deshabilitado.
- Usuario sin shell (`/sbin/nologin`): no puede abrir la terminal si le roban la contraseña.
- Usuario enjaulado (chroot): solo ve su directorio personal.
- IPv6 desactivado para reducir la superficie de ataque.
- Carpeta compartida con permisos `2775`: solo el dueño y el grupo pueden escribir.
- Carpeta personal con `711`: el usuario FTP solo puede pasar por ella, no listar su contenido.

---

## 04 Pruebas de Aceptación y Troubleshooting

### 4.1 Casos de prueba

| # | Prueba | Comando / Acción | Resultado esperado |
|---|---|---|---|
| 1 | Servicio activo | `sudo systemctl status vsftpd` | `active (running)` |
| 2 | Puerto escuchando | `sudo ss -tlnp \| grep :21` | vsftpd escuchando en el puerto 21 |
| 3 | Firewall | `sudo firewall-cmd --list-all` | `ftp` en la lista de servicios |
| 4 | Conexión FTP | MobaXterm → sesión FTP con la IP del CentOS y `usuario_ftp` | Conexión exitosa |
| 5 | Subida de archivo | Arrastrar un archivo al panel remoto | Archivo subido sin errores |
| 6 | Descarga de archivo | Descargar un archivo al equipo local | Archivo descargado |
| 7 | Sin acceso a terminal | `ssh usuario_ftp@IP_CENTOS` | `This account is currently not available.` |
| 8 | Carpeta de destino | Abrir MobaXterm de nuevo y conectarse | La sesión inicia dentro de `Archivos_FTP` |
| 9 | Archivo visible en el escritorio | Subir un archivo y abrir `Archivos_FTP` en CentOS | El archivo aparece en la carpeta |
| 10 | Permisos del archivo subido | `ls -l /home/usuario_normal/Archivos_FTP` | Grupo `usuario_ftp` y permiso de escritura para el grupo (`-rw-rw-r--`) |

> 📸 **Evidencia:**

> <img width="1600" height="852" alt="image" src="https://github.com/user-attachments/assets/74a529a7-1277-42ca-9758-c365a99be39d" />


### 4.2 Plan de contingencia / Troubleshooting

#### Error 1: `530 Login incorrect` (no deja iniciar sesión)
- **Causa:** el usuario está en `/etc/vsftpd/user_list`, o `pam_shells.so` rechaza el shell `/sbin/nologin`.
- **Solución:** quitar al usuario de `user_list`, comentar `pam_shells.so` en `/etc/pam.d/vsftpd`, confirmar `check_shell=NO` y reiniciar con `sudo systemctl restart vsftpd`.

#### Error 2: `500 OOPS: vsftpd: refusing to run with writable root inside chroot()`
- **Causa:** el usuario está enjaulado y su carpeta raíz tiene permiso de escritura.
- **Solución:** agregar `allow_writeable_chroot=YES` en `vsftpd.conf` y reiniciar el servicio.

#### Error 3: `550 Permission denied` o bloqueo al subir archivos
- **Causa:** SELinux bloquea a vsftpd, o falta `write_enable=YES`.
- **Solución:** ejecutar `sudo setsebool -P ftpd_full_access on` y revisar `write_enable=YES`. Para ver los bloqueos: `sudo ausearch -m avc -ts recent`.

#### Cuidados adicionales
- Si la conexión da *timeout*, revisar que `ftp` esté abierto en el firewall y que la VM esté en modo **Bridge**.
- Si el modo pasivo se queda colgado, abrir el rango de puertos pasivos en `vsftpd.conf` y en el firewall.
- Si MobaXterm no entra a `Archivos_FTP` o da `550`, revisar que `chmod 711` esté aplicado en la carpeta personal del usuario normal.
- Si MobaXterm sigue mostrando la carpeta anterior, cerrar la sesión y abrirla de nuevo para que actualice la ruta.
- Si el usuario normal no puede editar un archivo subido, revisar que `local_umask=002` esté en `vsftpd.conf`.
- Después de cada cambio en `vsftpd.conf`, reiniciar el servicio.

#### Cuidados adicionales
- Si la conexión da *timeout*, revisar que `ftp` esté abierto en el firewall y que la VM esté en modo **Bridge**.
- Si el modo pasivo se queda colgado, abrir el rango de puertos pasivos en `vsftpd.conf` y en el firewall.
- Después de cada cambio en `vsftpd.conf`, reiniciar el servicio.
