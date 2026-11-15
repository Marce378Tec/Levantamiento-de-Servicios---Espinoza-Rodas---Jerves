
# Servidor SSH 


## 01. Arquitectura y Requisitos

• Arquitectura: Administración remota cifrada mediante protocolo SSH sobre red en modo Puente (Bridged), migrando del puerto estándar TCP 22 al puerto personalizado TCP 2222 para mitigar ataques automatizados.
• Requisitos: Servicio openssh-server, cliente externo MobaXterm y utilidades de gestión de políticas de seguridad.
bash
#Instalación de herramienta requerida para gestionar puertos en SELinux
sudo dnf install -y policycoreutils-python-utils
Usa el código con precaución.

## 02. Despliegue y Configuración

• Procedimiento: Verificación del estado inicial, respaldo del archivo maestro de configuración y modificación de la directiva de puerto.
bash
#1. Verificar estado inicial del servicio
sudo systemctl status sshd

#2. Crear copia de seguridad de la configuración original
sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config.backup

#3. Editar el archivo de configuración
sudo nano /etc/ssh/sshd_config
Usa el código con precaución.
Modificación interna en sshd_config: Descomentar (quitar #) y cambiar el puerto a Port 2222 (Opcional recomendado: cambiar PermitRootLogin no). Guardar con Ctrl+O, Enter y salir con Ctrl+X.

## 03. Aseguramiento (Seguridad, SELinux y Firewall)

• Políticas: Etiquetado del nuevo puerto en el contexto de SELinux, habilitación de tráfico entrante en el firewall y cierre del puerto por defecto antes de aplicar cambios.
bash
#1. Autorizar el puerto 2222 para SSH en SELinux
sudo semanage port -a -t ssh_port_t -p tcp 2222

#2. Abrir el puerto 2222 en el firewall y cerrar el puerto 22 original
sudo firewall-cmd --permanent --add-port=2222/tcp
sudo firewall-cmd --permanent --remove-service=ssh

#3. Recargar firewall y verificar puertos activos
sudo firewall-cmd --reload
sudo firewall-cmd --list-ports

#4. Reiniciar el servicio SSH para aplicar todos los cambios
sudo systemctl restart sshd
Usa el código con precaución.

## 04. Pruebas de Aceptación y Troubleshooting

• Validación: Confirmación local de que el demonio escucha en el puerto 2222 y prueba de inicio de sesión remoto desde MobaXterm.
bash
#Verificar que el servicio levantó sin errores en el puerto 2222
sudo systemctl status sshd
Usa el código con precaución.
• Troubleshooting (Resolución de problemas):
	• Error de permiso al reiniciar sshd: Ocurre si se reinicia el servicio antes de ejecutar el comando semanage port. Se soluciona aplicando la regla de SELinux y volviendo a reiniciar sshd.
	• Bloqueo de sesión: Mantener siempre abierta la terminal SSH original mientras se prueba la nueva conexión en una segunda ventana de MobaXterm.
