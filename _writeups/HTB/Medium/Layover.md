---
layout: default
title: "Layover"
date: 2026-10-17
---
![](/assets/Pasted%20image%2020261017084241.png)

## Resumen Ejecutivo

Layover es una máquina Medium Linux que simula un entorno corporativo de aeropuerto con una red WiFi interna. El acceso inicial llega por RDP a un jumpbox, desde donde se conecta a una red WiFi simulada (mac80211\_hwsim) para capturar credenciales de un usuario en texto claro via sniffing pasivo. Con esas credenciales se explota una vulnerabilidad de inyección de comportamiento en Craft CMS 5.9.8 que permite RCE sin ser administrador. El movimiento lateral requiere descifrar una contraseña cifrada con la clave de seguridad del CMS. La escalada a root se consigue abusando de CVE-2026-34990, una vulnerabilidad en CUPS 2.4.16 que permite a cualquier usuario local capturar un token de autenticación del demonio de impresión y usarlo para escribir archivos arbitrarios como root — en este caso, un fragmento de sudoers.

---

￼## 1\. Reconocimiento

**Objetivo de esta fase:** Determinar qué servicios están expuestos en el target para priorizar vectores de ataque.

￼### 1.1 Puertos abiertos

El target expone únicamente dos puertos. Esto es inusual — la mayoría de máquinas HTB tienen más superficie — y es un hint de que el acceso inicial no va por la vía convencional de explotar un servicio web directamente.

```
Puerto 22  — SSH (OpenSSH 9.6p1 Ubuntu)
Puerto 3389 — RDP/xrdp (no requiere NLA)
```

El puerto 3389 corresponde a **RDP** (Remote Desktop Protocol), el protocolo de escritorio remoto de Microsoft adoptado también en Linux via `xrdp`. La ausencia de **NLA** (Network Level Authentication) significa que el servidor acepta la conexión antes de pedir credenciales — esto simplifica la explotación y es considerado una misconfiguration en entornos de producción.

￼### 1.2 Acceso inicial via RDP

La descripción de la máquina provee credenciales iniciales: `contractor / Contractor2026!`. La autenticación SSH con estas credenciales falla porque `sshd` tiene deshabilitada la autenticación por contraseña para este usuario. El acceso correcto es via RDP:

```
rdesktop -u contractor -p 'Contractor2026!' -g 1280x1024 <IP>
```

**rdesktop** es un cliente RDP open source para Linux. Al conectar, se obtiene un escritorio XFCE del sistema `airside-ws01`, que resulta ser un contenedor LXD (identificable por la interfaz `eth0@if11`, que indica una interfaz virtual de red dentro de un namespace).
