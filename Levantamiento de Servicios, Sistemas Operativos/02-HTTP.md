# 02-HTTP: Servidor web Apache (httpd) con página personalizada

## 01 Arquitectura y Requisitos

### Descripción
Se despliega un servidor web **Apache (httpd)** en CentOS Stream. Se modifica la página de prueba predeterminada para que muestre un título personalizado, los nombres de los estudiantes y una imagen (`ucuencatec.png`).

### Topología

<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/97e905c8-d0df-4364-8bdd-3cc4b6246c78" />


### Requisitos

| Requisito | Detalle |
|---|---|
| Sistema operativo | CentOS Stream (VirtualBox, adaptador en modo Puente/Bridge) |
| Paquete | `httpd` |
| Página de prueba | `/usr/share/testpage/index.html` |
| Imagen | Un archivo de imagen (ejemplo: `ucuencatec.png`) |
| Permisos | Usuario con `sudo` |
| Cliente | Navegador web o `curl` |

---

## 02 Despliegue y Configuración

### 2.1 Instalar e iniciar Apache

```bash
sudo systemctl status httpd
```
Revisa si el servicio existe y si está activo.

```bash
sudo dnf install -y httpd
```
Instala Apache (solo si no estaba instalado). La opción `-y` acepta la instalación automáticamente.

```bash
sudo systemctl start httpd
```
Inicia el servicio.

```bash
sudo systemctl enable httpd
```
Hace que Apache inicie automáticamente al encender el servidor.

```bash
sudo systemctl status httpd
```
Confirma que el servicio quedó en estado `active (running)`.

### 2.2 Respaldar la página de prueba

```bash
sudo cp /usr/share/testpage/index.html /usr/share/testpage/index.html.backup
```
Crea una copia de seguridad de la página original.

```bash
ls -l /usr/share/testpage/index.html
```
Muestra los permisos y el dueño del archivo.

### 2.3 Cambiar el título de la página

```bash
sudo nano /usr/share/testpage/index.html
```
Abre la página en el editor. Para cambiar el título:

1. Buscar la línea `<title>HTTP Server Test Page</title>` (en nano: `Ctrl+W`, escribir `HTTP Server Test Page` y `Enter`).
2. Cambiar el texto por el título personalizado, por ejemplo:

```html
<title>Práctica 3 - Carlos Jerves y Marcelino Espinoza</title>
```

> Si no se ve el símbolo `<`, buscar la línea `/html`, que está cerca de la línea que buscamos.

3. Guardar y salir: `Ctrl+O`, `Enter`, `Ctrl+X`.

```bash
sudo systemctl restart httpd
```
Reinicia Apache para aplicar el cambio.

### 2.4 Agregar la imagen

```bash
sudo cp /ruta/de/la/imagen/ucuencatec.png /usr/share/testpage/
```
Copia la imagen a la carpeta de la página. Se debe cambiar `ucuencatec.png` por el nombre y formato reales de la imagen.

```bash
sudo chmod 644 /usr/share/testpage/ucuencatec.png
```
Da permiso de lectura a todos (dueño: leer y escribir; grupo y otros: solo leer).

```bash
sudo restorecon -v /usr/share/testpage/ucuencatec.png
```
Restaura el contexto de SELinux correcto, para que Apache pueda leer la imagen.

```bash
sudo nano /usr/share/testpage/index.html
```
Abre la página de nuevo. Buscar la línea `<a class="navbar-brand"...>` (en nano: `Ctrl+W` y escribir `<a class`) y agregar la imagen dentro de esa etiqueta. Debe quedar parecida a esta:

```html
<a class="navbar-brand" href="#"><img src="ucuencatec.png" alt="Logo" height="40"> Carlos Jerves y Marcelino Espinoza</a>
```

Guardar con `Ctrl+O`, `Enter`, `Ctrl+X` y reiniciar:

```bash
sudo systemctl restart httpd
```

---

## 03 Aseguramiento Seguridad SELinux/Firewall

### 3.1 Firewalld

```bash
sudo firewall-cmd --permanent --add-service=http
```
Abre el servicio HTTP (puerto 80) de forma permanente.

```bash
sudo firewall-cmd --reload
```
Recarga el firewall para aplicar la regla.

```bash
sudo firewall-cmd --list-all
```
Verifica que `http` aparezca en la lista de servicios.

### 3.2 SELinux

| Elemento | Valor |
|---|---|
| Contexto requerido para archivos web | `httpd_sys_content_t` |
| Comando para restaurarlo | `sudo restorecon -v <archivo>` |
| Comando para revisarlo | `ls -Z /usr/share/testpage/` |

El contexto correcto evita errores `403 Forbidden`. SELinux se deja en modo `Enforcing`.

### 3.3 Matriz de puertos

| Puerto | Protocolo | Uso | Regla |
|---|---|---|---|
| 80 | TCP | HTTP | `--add-service=http` |
| 443 | TCP | HTTPS (opcional) | `--add-service=https` (solo si se usa SSL/TLS) |

### 3.4 Medidas de seguridad aplicadas
- Solo se abre el puerto 80 en el firewall.
- Archivos con permiso `644`: lectura para todos, escritura solo para el dueño.
- Contexto SELinux correcto en los archivos web.
- Copia de seguridad de la página original.

---

## 04 Pruebas de Aceptación y Troubleshooting

### 4.1 Casos de prueba

| # | Prueba | Comando / Acción | Resultado esperado |
|---|---|---|---|
| 1 | Servicio activo | `sudo systemctl status httpd` | `active (running)` |
| 2 | Puerto escuchando | `sudo ss -tlnp \| grep :80` | httpd escuchando en el puerto 80 |
| 3 | Firewall | `sudo firewall-cmd --list-all` | `http` en la lista de servicios |
| 4 | Contexto SELinux | `ls -Z /usr/share/testpage/ucuencatec.png` | Contexto `httpd_sys_content_t` |
| 5 | Código de estado HTTP | `curl -I http://localhost` | `HTTP/1.1 200 OK` |
| 6 | Acceso desde otro equipo | Abrir `http://IP_DEL_CENTOS` en el navegador de Windows | Se ve la página con el título, los nombres y la imagen |
| 7 | Imagen disponible | `curl -I http://localhost/ucuencatec.png` | `HTTP/1.1 200 OK` |

> 📸 **Evidencia:**

<img width="1301" height="808" alt="caphttpnyi" src="https://github.com/user-attachments/assets/51ecd874-733b-4ecd-9d7b-984fba1bb9b0" />


### 4.2 Plan de contingencia / Troubleshooting

#### Error 1: `403 Forbidden`
- **Causa:** el archivo no tiene permiso de lectura o tiene un contexto SELinux incorrecto.
- **Solución:** ejecutar `sudo chmod 644 <archivo>` y `sudo restorecon -v <archivo>`. Para ver los bloqueos de SELinux: `sudo ausearch -m avc -ts recent`.

#### Error 2: La página no abre desde otro equipo
- **Causa:** el firewall bloquea el puerto 80, o la máquina virtual no está en modo Bridge.
- **Solución:** ejecutar `sudo firewall-cmd --permanent --add-service=http` y `sudo firewall-cmd --reload`. Confirmar que la VM esté en modo Puente y que la IP sea la correcta (`ip a`).

#### Error 3: La imagen no aparece o los cambios no se ven
- **Causa:** el nombre o la extensión de la imagen no coincide con el de la etiqueta `<img>`, no se reinició Apache, o el navegador guardó la página anterior en caché.
- **Solución:** revisar el nombre exacto con `ls -l /usr/share/testpage/`, ejecutar `sudo systemctl restart httpd` y recargar el navegador con `Ctrl+F5`.

#### Cuidados adicionales
- Si Apache no inicia, revisar los errores con `sudo journalctl -u httpd` o `sudo apachectl configtest`.
- Si la imagen sigue sin cargar desde `/usr/share/testpage/`, copiarla a `/var/www/html/` (directorio web raíz) y aplicar de nuevo `chmod 644` y `restorecon`.
- Después de cada cambio en los archivos, reiniciar el servicio.
