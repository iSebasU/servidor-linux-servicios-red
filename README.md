# Servidor Linux con servicios de red

Proyecto final del curso **Administración de Servidores Linux** – Universidad Católica Luis Amigó.

Servidor **Ubuntu 20.04** que ofrece cuatro servicios de red (DHCP, DNS, servidor web y FTP) a un cliente **Windows XP**, ambos en una red interna de VirtualBox. Todas las pruebas se realizaron desde el cliente.

## Topología

```
┌──────────────────────────┐      Red interna       ┌──────────────────────────┐
│  Servidor Ubuntu 20.04   │     192.168.1.0/24     │    Cliente Windows XP    │
│  IP fija 192.168.1.10    │◄──────────────────────►│  IP por DHCP (.50–.100)  │
│  DHCP · DNS · Apache · FTP│                        │                          │
└──────────────────────────┘                        └──────────────────────────┘
```

## Servicios

| Servicio | Paquete | Puerto | Origen | Modificación |
|---|---|---|---|---|
| DHCP | `isc-dhcp-server` | UDP 67/68 | Guía del curso | Rango `192.168.1.50 – 192.168.1.100` y arranque automático robusto |
| DNS | `bind9` | 53 | Guía del curso | Dominio `udea.edu.co`, zona directa e inversa, correcciones a la guía |
| Web | `apache2` | TCP 80 | Elección propia | Sitio web propio con explicaciones y simuladores |
| FTP | `vsftpd` | TCP 20/21 | Elección propia | Usuario dedicado, `chroot` y banner propio |

## Estructura del repositorio

```
├── config/
│   ├── dhcp/      Configuración del servidor DHCP
│   ├── dns/       Zonas y declaración de bind9
│   └── ftp/       Configuración de vsftpd
├── web/           Sitio publicado por Apache
└── docs/
    ├── guia-instalacion-y-pruebas.md  Configuración y pruebas paso a paso
    ├── problemas-y-soluciones.md  Fallas encontradas y cómo se resolvieron
    └── img/                       Capturas de las pruebas
```

### Ubicación de cada archivo en el servidor

| Archivo del repositorio | Ruta en el servidor |
|---|---|
| `config/dhcp/isc-dhcp-server` | `/etc/default/isc-dhcp-server` |
| `config/dhcp/dhcpd.conf` | `/etc/dhcp/dhcpd.conf` |
| `config/dhcp/override.conf` | `/etc/systemd/system/isc-dhcp-server.service.d/override.conf` |
| `config/dns/named.conf.local` | `/etc/bind/named.conf.local` |
| `config/dns/db.udea.edu.co.zone` | `/var/cache/bind/db.udea.edu.co.zone` |
| `config/dns/db.192.168.1` | `/var/cache/bind/db.192.168.1` |
| `config/ftp/vsftpd.conf` | `/etc/vsftpd.conf` |
| `web/index.html` | `/var/www/html/index.html` |

## Direccionamiento

| Elemento | Valor |
|---|---|
| Red | `192.168.1.0/24` |
| Servidor | `192.168.1.10` (fija) |
| Puerta de enlace | `192.168.1.1` |
| Rango DHCP | `192.168.1.50 – 192.168.1.100` |
| DNS entregado | `192.168.1.10`, `8.8.8.8` |
| Dominio | `udea.edu.co` |

## Cómo replicarlo

1. Instalar los paquetes en Ubuntu 20.04:
   ```bash
   sudo apt-get update
   sudo apt-get install isc-dhcp-server bind9 apache2 vsftpd
   ```
2. Configurar IP fija `192.168.1.10/24` en la interfaz de la red interna (`enp0s3`).
3. Copiar cada archivo de `config/` y `web/` a su ruta (tabla anterior).
4. Crear el usuario FTP: `sudo adduser ftpudea`.
5. Reiniciar los servicios:
   ```bash
   sudo systemctl daemon-reload
   sudo systemctl restart isc-dhcp-server bind9 apache2 vsftpd
   ```

## Pruebas desde el cliente

| Servicio | Prueba en Windows XP | Resultado esperado |
|---|---|---|
| DHCP | `ipconfig /release` · `ipconfig /renew` · `ipconfig /all` | IP del rango y servidor DHCP `192.168.1.10` |
| DNS | `nslookup www.udea.edu.co` · `nslookup 192.168.1.10` | Resolución directa e inversa |
| Apache | `http://www.udea.edu.co` | Carga el sitio del proyecto |
| FTP | `ftp 192.168.1.10` → `get` / `put` | `226 Transfer complete` |

Cada servicio se probó también **apagado y encendido** (`systemctl stop` / `start`) para demostrar que las respuestas provienen del servidor.

El paso a paso completo está en [`docs/guia-instalacion-y-pruebas.md`](docs/guia-instalacion-y-pruebas.md) y las fallas resueltas en [`docs/problemas-y-soluciones.md`](docs/problemas-y-soluciones.md).

## Créditos

Basado en la guía *Memorias del curso: Administración de Servidores Linux* (prof. Eder Acevedo, actualizada por Bayron Ospina), con modificaciones propias.

**Autor:** Sebastian Jaramillo Benitez – Ingeniería de Sistemas, Universidad Católica Luis Amigó.
