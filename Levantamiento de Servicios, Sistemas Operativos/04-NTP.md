# 04-NTP: Sincronización horaria con Chrony

## 01 Arquitectura y Requisitos

### Descripción
Se configura **chronyd** en CentOS Stream como servidor de hora (NTP). El equipo Windows, con la hora desconfigurada a propósito, se conecta a CentOS y recupera la hora correcta. Tener la hora exacta es importante para los registros de auditoría y para los certificados SSL/TLS.

### Topología

<img width="1536" height="1024" alt="image" src="https://github.com/user-attachments/assets/1b2b4854-6c32-4e73-b1d4-9c80b8ce71df" />

### Requisitos

| Requisito | Detalle |
|---|---|
| Servidor | CentOS Stream con `chronyd` (VirtualBox, modo Puente/Bridge) |
| Cliente | Windows con el servicio `w32time` |
| Red | CentOS y Windows en la misma red (se ven entre sí) |
| Permisos | `sudo` en CentOS y CMD como administrador en Windows |
| Puerto | 123/UDP abierto en CentOS |

---

## 02 Despliegue y Configuración

### 2.1 Configurar el servidor (CentOS)

```bash
ip a
```
Muestra las interfaces de red. Se anota la IP del CentOS (por ejemplo `192.168.0.X`), porque el cliente Windows la necesita.

```bash
sudo nano /etc/chrony.conf
```
Abre el archivo de configuración de chrony. Al final del archivo se agregan estas dos líneas:

```
allow 192.168.0.100
local stratum 10
```

| Directiva | Función |
|---|---|
| `allow 192.168.0.100` | Permite consultas de hora desde esa IP (la IP de nuestro Windows). Se puede usar una red completa, por ejemplo `192.168.0.0/24` |
| `local stratum 10` | El servidor da su propia hora aunque no tenga Internet. Así Windows puede sincronizarse igual |

```bash
sudo systemctl restart chronyd
```
Reinicia el servicio para aplicar los cambios.

```bash
sudo systemctl enable chronyd
```
Hace que el servicio inicie automáticamente al encender el servidor.

### 2.2 Configurar el cliente (Windows)

1. Abrir el menú Inicio y escribir `cmd`.
2. Clic derecho sobre **Símbolo del sistema** → **Ejecutar como administrador**.
3. Pegar este comando, reemplazando `IP_DEL_CENTOS` por la IP anotada con `ip a`:

```cmd
sc config w32time start= auto & net start w32time & w32tm /config /manualpeerlist:"IP_DEL_CENTOS,0x8" /syncfromflags:manual /update & w32tm /resync /rediscover & w32tm /query /status
```

| Parte del comando | Función |
|---|---|
| `sc config w32time start= auto` | El servicio de hora de Windows inicia automáticamente |
| `net start w32time` | Inicia el servicio ahora |
| `w32tm /config /manualpeerlist:"IP,0x8"` | Define el servidor CentOS como fuente de hora (`0x8` = modo cliente) |
| `/syncfromflags:manual` | Usa solo el servidor definido a mano |
| `w32tm /resync /rediscover` | Fuerza la sincronización inmediata |
| `w32tm /query /status` | Muestra el estado de la sincronización |

Para repetir la sincronización y revisar el estado:

```cmd
w32tm /resync
w32tm /query /status
```

---

## 03 Aseguramiento Seguridad SELinux/Firewall

### 3.1 Firewalld

```bash
sudo firewall-cmd --add-service=ntp --permanent
```
Abre el servicio NTP (puerto 123/UDP) de forma permanente.

```bash
sudo firewall-cmd --reload
```
Recarga el firewall para aplicar la regla.

```bash
sudo firewall-cmd --list-all
```
Verifica que `ntp` aparezca en la lista de servicios.

### 3.2 SELinux
`chronyd` ya tiene su propia política de SELinux (`chronyd_t`). **No se necesita ningún booleano adicional** para este servicio. SELinux se deja en modo `Enforcing`.

### 3.3 Matriz de puertos

| Puerto | Protocolo | Uso | Regla |
|---|---|---|---|
| 123 | UDP | Sincronización de hora (NTP) | `--add-service=ntp` |

### 3.4 Medidas de seguridad aplicadas
- Solo se permiten consultas desde la IP (o red) indicada en `allow`.
- Solo se abre el puerto 123/UDP, ningún otro.
- El servicio queda habilitado con `systemctl enable`.

---

## 04 Pruebas de Aceptación y Troubleshooting

### 4.1 Casos de prueba

| # | Prueba | Comando / Acción | Resultado esperado |
|---|---|---|---|
| 1 | Servicio activo | `sudo systemctl status chronyd` | `active (running)` |
| 2 | Firewall | `sudo firewall-cmd --list-all` | `ntp` en la lista de servicios |
| 3 | Fuentes de hora (CentOS) | `chronyc sources` | Lista de fuentes de hora |
| 4 | Estado del reloj (CentOS) | `chronyc tracking` | Muestra el estrato y el desfase |
| 5 | Clientes conectados (CentOS) | `sudo chronyc clients` | Aparece la IP del Windows |
| 6 | Sincronización (Windows) | `w32tm /resync` | `El comando se completó correctamente` |
| 7 | Origen de la hora (Windows) | `w32tm /query /status` | Línea **Origen (Source)** con la IP del CentOS |
| 8 | Hora igual en ambos equipos | Comparar la hora de Windows con `date` en CentOS | La hora coincide |

> 📸 **Evidencia:**
>  <img width="442" height="475" alt="image" src="https://github.com/user-attachments/assets/29b711a4-4511-4f2f-8639-c51b1a557c7e" />


### 4.2 Plan de contingencia / Troubleshooting

#### Error 1: Windows no sincroniza (`Se agotó el tiempo de espera` / sin respuesta)
- **Causa:** el firewall de CentOS bloquea el puerto 123/UDP.
- **Solución:** ejecutar `sudo firewall-cmd --add-service=ntp --permanent` y `sudo firewall-cmd --reload`. También revisar que las dos máquinas se vean con `ping IP_DEL_CENTOS`.

#### Error 2: El servidor ignora la consulta del cliente
- **Causa:** la IP del Windows no coincide con la directiva `allow` (la IP cambió, o se usa DHCP).
- **Solución:** revisar la IP con `ipconfig` en Windows, corregir `allow` en `/etc/chrony.conf` (o usar la red completa `192.168.0.0/24`) y reiniciar con `sudo systemctl restart chronyd`.

#### Error 3: El Origen (Source) sigue en `Local CMOS Clock`
- **Causa:** el servicio `w32time` no está iniciado, o el CMD no se abrió como administrador.
- **Solución:** abrir CMD como administrador, volver a pegar el comando de la sección 2.2 y ejecutar `w32tm /resync /rediscover`.

#### Cuidados adicionales
- Una hora desfasada puede romper certificados SSL/TLS y mezclar el orden de los logs de auditoría.
- Si la IP del CentOS cambia, hay que actualizar `manualpeerlist` en Windows.
- Después de cada cambio en `/etc/chrony.conf`, reiniciar `chronyd`.
