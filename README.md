# Práctica: Levantamiento de servicios

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
| 1 | Acceso remoto | SSH | 22/TCP | [01-SSH](Levantamiento%20de%20Servicios,%20Sistemas%20Operativos/01-SSH.md) |
| 2 | Servidor web | Apache (HTTP) | 80/TCP, 443/TCP | [02-HTTP](Levantamiento%20de%20Servicios,%20Sistemas%20Operativos/02-HTTP.md) |
| 3 | Transferencia de archivos | vsftpd (FTP) | 20-21/TCP | [03-VSFTP](Levantamiento%20de%20Servicios,%20Sistemas%20Operativos/03-VSFTP.md) |
| 4 | Sincronización horaria | Chrony (NTP) | 123/UDP | [04-NTP](Levantamiento%20de%20Servicios,%20Sistemas%20Operativos/04-NTP.md) |

## Topología general

<img width="2172" height="724" alt="image" src="https://github.com/user-attachments/assets/d95a01c7-44bf-4d3e-b109-c7da87fb07dc" />


## Estructura de cada página

Todas las páginas siguen el mismo orden:

1. **Arquitectura y Requisitos**
2. **Despliegue y Configuración**
3. **Aseguramiento Seguridad SELinux/Firewall**
4. **Pruebas de Aceptación y Troubleshooting**

## Referencia

[1] J. V. Hidalgo Guijarro, G. G. Arcos Ponce y J. A. Naranjo Cedeño, *Configuración y administración avanzada sobre el Sistema Operativo CentOS 7.0*. Universidad Politécnica Estatal del Carchi, 2022. doi: 10.32645/9789942914804.
