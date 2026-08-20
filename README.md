# 🏠 Homelab — Servidor Ubuntu Bare-Metal con Hardening de Seguridad

[![OS](https://img.shields.io/badge/OS-Ubuntu%20Server-E95420?logo=ubuntu&logoColor=white)](https://ubuntu.com/server)
[![SSH](https://img.shields.io/badge/SSH-ED25519%20%2B%20Zero%20Trust-black?logo=openssh&logoColor=white)](#-fase-2-aprovisionamiento-y-hardening-core)
[![Firewall](https://img.shields.io/badge/Firewall-UFW-orange)](#-fase-2-aprovisionamiento-y-hardening-core)
[![Storage](https://img.shields.io/badge/Storage-LVM-blue)](#-fase-2-aprovisionamiento-y-hardening-core)
[![Docker](https://img.shields.io/badge/Contenedores-Docker%20%2B%20Compose-2496ED?logo=docker&logoColor=white)](#-fase-3-optimización-de-hardware-y-orquestación)
[![Pi-hole](https://img.shields.io/badge/DNS%20Sinkhole-Pi--hole-red?logo=pihole&logoColor=white)](#-fase-4-despliegue-de-dns-sinkhole-pi-hole-e-iac)
[![Status](https://img.shields.io/badge/Estado-En%20producción-brightgreen)]()

Documentación técnica del despliegue de un servidor doméstico (**homelab**) desde cero: recuperación de un equipo con Windows 10 inoperativo, instalación bare-metal de Ubuntu Server en modo headless, y hardening de seguridad aplicando principios de **Zero Trust** (autenticación por clave pública, sin acceso root, firewall restrictivo). Este repositorio es la base sobre la que se construirá una arquitectura modular de servicios (contenerización, self-hosting, etc.).

---

## 📑 Tabla de Contenidos

- [Resumen del Entorno](#-resumen-del-entorno)
- [Arquitectura y Topología](#-arquitectura-y-topología)
- [Fase 1: Rescate de Datos y Preparación](#-fase-1-rescate-de-datos-y-preparación)
- [Fase 2: Aprovisionamiento y Hardening Core](#-fase-2-aprovisionamiento-y-hardening-core)
- [Fase 3: Optimización de Hardware y Orquestación](#-fase-3-optimización-de-hardware-y-orquestación)
- [Fase 4: Despliegue de DNS Sinkhole (Pi-hole) e IaC](#-fase-4-despliegue-de-dns-sinkhole-pi-hole-e-iac)
- [Stack Tecnológico](#-stack-tecnológico)
- [Decisiones de Diseño](#-decisiones-de-diseño)
- [Roadmap](#-roadmap)
- [Autor](#-autor)

---

## 🖥️ Resumen del Entorno

| Componente | Detalle |
|---|---|
| **Hardware** | Lenovo Yoga — Intel i3-6100 @ 2.30GHz, 8GB RAM |
| **Sistema Operativo** | Ubuntu Server (bare-metal, modo headless) |
| **Red** | IP estática en `192.168.100.X`, conexión a la puerta de enlace vía infraestructura PLC |
| **Acceso remoto** | SSH sobre clave pública ED25519, sin contraseñas ni acceso root |
| **Firewall** | UFW con política *default deny incoming*, único puerto expuesto: `22/tcp` |

---

## 🌐 Arquitectura y Topología

El servidor está conectado a la red local mediante una **IP estática** (`192.168.100.X`), evitando la asignación dinámica por DHCP para garantizar un punto de acceso administrativo predecible e inmutable. La conexión hacia la puerta de enlace predeterminada se realiza a través de infraestructura **PLC (Power Line Communication)**, aprovechando el cableado eléctrico existente en lugar de tendido de red dedicado.

---

## 🔧 Fase 1: Rescate de Datos y Preparación

Antes de intervenir el sistema, se priorizó la recuperación segura de la información del equipo original (Windows 10, inoperativo).

- **Creación de Live USB**: imagen de Ubuntu Desktop flasheada con [Rufus](https://rufus.ie/), usando esquema de particiones **GPT** para compatibilidad con arranque **UEFI** nativo.
- **Auditoría de BIOS**: desactivación de *Secure Boot* y *Fast Boot* para permitir el arranque desde medios extraíbles, y habilitación de **Intel VT-x** para dar soporte a la futura contenerización de servicios.
- **Bypass y rescate de datos**: montaje de las unidades de almacenamiento originales desde el entorno live y extracción segura de documentos críticos hacia almacenamiento externo, antes de cualquier formateo.

---

## 🔐 Fase 2: Aprovisionamiento y Hardening Core

### Instalación y almacenamiento

Formateo completo del disco y despliegue limpio de **Ubuntu Server**, utilizando **LVM (Logical Volume Manager)** para permitir el redimensionado y la escalabilidad del almacenamiento en caliente sin necesidad de reparticionar.

### Red

Abandono del DHCP en favor de una **IP estática** y puerta de enlace configuradas manualmente, garantizando que el punto de acceso administrativo no cambie entre reinicios.

### Identidad y acceso (Zero Trust)

1. **Generación de claves**: par de claves criptográficas de curva elíptica **ED25519** generado en la máquina administradora (Windows 11), descartando deliberadamente RSA por su mayor coste computacional y menor eficiencia frente a curvas elípticas modernas.
2. **Hardening de SSH**:
   - Inyección manual de la clave pública en el servidor y ajuste estricto de permisos sobre `~/.ssh` (evitando que `sshd` rechace la clave por permisos demasiado abiertos).
   - Modificación de `sshd_config` aplicando políticas Zero Trust: `PermitRootLogin no` y `PasswordAuthentication no`, forzando la autenticación exclusivamente por clave pública.

### Firewall perimetral (UFW)

Activación de UFW con política de **denegación global por defecto** (`default deny incoming`) y apertura exclusiva del puerto **22/tcp**, minimizando la superficie de ataque a un único servicio expuesto y auditado.

---

## ⚙️ Fase 3: Optimización de Hardware y Orquestación

### Configuración headless (Lid Switch)

Modificación del gestor de sesiones **systemd**, editando `/etc/systemd/logind.conf` y estableciendo la directiva `HandleLidSwitch=ignore`. Esto evita la suspensión del sistema al cerrar la tapa física del equipo, garantizando disponibilidad **24/7** y reduciendo el consumo energético frente a mantener la pantalla activa.

### Despliegue de Docker Engine

Instalación del motor de contenedores y el plugin **Docker Compose** desde los repositorios oficiales, verificando las llaves **GPG** antes de la instalación. Se mantiene una postura de seguridad de **mínimo privilegio**, requiriendo el uso explícito de `sudo` para la administración de contenedores en lugar de añadir el usuario al grupo `docker` sin restricciones.

---

## 🕳️ Fase 4: Despliegue de DNS Sinkhole (Pi-hole) e IaC

### Infraestructura como Código (IaC) y gestión de secretos

Inicialización de repositorio Git local implementando un `.gitignore` estricto para aislar credenciales. Las variables de entorno reales se mantienen en archivos `.env` locales, subiendo al control de versiones únicamente plantillas neutras (`.env.example`), evitando así la exposición accidental de secretos.

### Apertura de puertos (UFW)

Reconfiguración del cortafuegos para permitir tráfico entrante en los puertos **53/tcp** y **53/udp** (resolución DNS), así como **80/tcp** (acceso al panel de administración HTTP).

### Troubleshooting

| Problema | Causa raíz | Solución |
|---|---|---|
| **Conflicto de puerto DNS** (`address already in use` en el 53) | El *Stub Listener* interno de Ubuntu ocupaba el puerto 53 | Desactivación en `/etc/systemd/resolved.conf` (`DNSStubListener=no`) y regeneración del enlace simbólico de resolución del sistema para liberar el bindeo al contenedor |
| **Inyección de credenciales fallida** | Parsing incorrecto de caracteres especiales en el archivo `.env` por parte de Docker Compose | Actualización en caliente con `docker exec -it pihole pihole setpassword` para forzar la escritura directa en la base de datos interna de Pi-hole |
| **Fuga de DNS (DNS Leak)** — el panel de Pi-hole registraba 0 peticiones | El módulo de protección web del antivirus (Norton) secuestraba el tráfico DNS a nivel de núcleo, forzándolo hacia `8.8.8.8` | Rollback de la configuración TCP/IPv4 del cliente Windows a la espera de desinstalar el software antivirus conflictivo |

> El diagnóstico se realizó mediante `nslookup`, confirmando que las resoluciones no pasaban por el sinkhole a pesar de la configuración correcta del cliente — un recordatorio de que el enrutamiento DNS puede secuestrarse por debajo de la capa de aplicación.

---

## 🧰 Stack Tecnológico

| Categoría | Herramienta |
|---|---|
| Sistema Operativo | Ubuntu Server |
| Gestión de almacenamiento | LVM |
| Imagen de arranque | Rufus (GPT / UEFI) |
| Criptografía de acceso | ED25519 (OpenSSH) |
| Firewall | UFW |
| Virtualización (preparado) | Intel VT-x |
| Contenerización | Docker Engine + Docker Compose |
| DNS Sinkhole / Ad-blocking | Pi-hole |
| Gestión de sesión | systemd (`logind.conf`, `resolved.conf`) |
| Control de versiones / IaC | Git + `.gitignore` + `.env.example` |

---

## 🧭 Decisiones de Diseño

| Decisión | Alternativa descartada | Motivo |
|---|---|---|
| IP estática vía asignación manual | DHCP | Acceso administrativo predecible e inmutable |
| ED25519 | RSA | Mayor seguridad con claves más cortas y mejor rendimiento |
| Autenticación solo por clave pública | Contraseñas | Elimina el vector de ataque por fuerza bruta |
| UFW *default deny* | Reglas permisivas | Minimizar superficie de ataque desde el diseño |
| LVM | Particionado tradicional | Escalabilidad de almacenamiento sin downtime |
| `HandleLidSwitch=ignore` | Adaptador de corriente + pantalla siempre activa | Disponibilidad 24/7 con menor consumo energético |
| `sudo` explícito para Docker | Usuario en grupo `docker` | Principio de menor privilegio; evita escalada de privilegios accidental |
| Secretos en `.env` + `.gitignore` | Credenciales hardcodeadas en el repo | Infraestructura como Código segura, sin exponer datos sensibles |

---

## 🗺️ Roadmap

- [x] Despliegue de servicios en contenedores (Docker + Compose)
- [x] DNS Sinkhole / bloqueo de publicidad a nivel de red (Pi-hole)
- [ ] Reverse proxy con TLS (Nginx / Caddy + Let's Encrypt)
- [ ] Monitorización (Prometheus + Grafana)
- [ ] Backups automatizados y cifrados
- [ ] VPN de acceso (WireGuard) como capa adicional a SSH

---

## 👤 Autor

**Eric** — Estudiante de Ingeniería Informática (Tecnologías de la Información), Universidad de Alicante.

> Este repositorio documenta el cimiento sobre el que se construirá la arquitectura modular completa del homelab.
