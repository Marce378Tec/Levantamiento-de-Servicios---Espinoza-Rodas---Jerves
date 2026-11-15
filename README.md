# Práctica 3: La línea de comando para levantar servicios en red

**Asignatura:** Sistemas Operativos I (Linux) | NRC 4292 - TICS-00087
**Docente:** Ing. Jonnathan F. Nivicela A.
**Periodo académico:** Septiembre 2026 - Febrero 2027
**Laboratorio:** C-007, Campus Balzay
**Estudiantes:** Carlos Jerves y Marcelino Espinoza

---

## Descripción

Este repositorio es un **Runbook técnico**: documenta cómo se despliegan, se aseguran y se validan servicios de red sobre **CentOS Stream** en un entorno virtualizado (VirtualBox, red en modo Puente). Cada servicio se asegura con **Firewalld** y **SELinux**.

## Servicios implementados

| # | Servicio | Herramienta | Puerto | Documentación |
|---|---|---|---|---|
| 1 | Acceso remoto | SSH | 22/TCP | [01-SSH](01-SSH.md) |
| 2 | Sincronización horaria | Chrony (NTP) | 123/UDP | [02-NTP](02-NTP.md) |
| 3 | Transferencia de archivos | vsftpd (FTP) | 20-21/TCP | [03-FTP](03-FTP.md) |
| 4 | Servidor web | Apache (HTTP) | 80/TCP, 443/TCP | [04-HTTP](04-HTTP.md) |

## Topología general

```
                        Red Bridge
[ Windows + MobaXterm ] ------------> [ CentOS Stream ]
   (cliente)                            ├─ SSH     (22/TCP)
                                        ├─ NTP     (123/UDP)
                                        ├─ FTP     (20-21/TCP)
                                        └─ HTTP    (80/TCP)
```

## Estructura de cada página

Todas las páginas siguen el mismo orden:

1. **Arquitectura y Requisitos**
2. **Despliegue y Configuración**
3. **Aseguramiento Seguridad SELinux/Firewall**
4. **Pruebas de Aceptación y Troubleshooting**



## Referencia

[1] J. V. Hidalgo Guijarro, G. G. Arcos Ponce y J. A. Naranjo Cedeño, *Configuración y administración avanzada sobre el Sistema Operativo CentOS 7.0*. Universidad Politécnica Estatal del Carchi, 2022. doi: 10.32645/9789942914804.
