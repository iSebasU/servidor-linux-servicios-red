# Guía de instalación y pruebas

Paso a paso para montar los cuatro servicios en un servidor Ubuntu 20.04 y comprobarlos desde un cliente de la misma red.

## Requisitos

| Elemento | Valor usado |
|---|---|
| Servidor | Ubuntu 20.04, IP fija `192.168.1.10/24`, interfaz `enp0s3` |
| Cliente | Cualquier equipo con IP automática (se probó con Windows XP) |
| Red | Red interna de VirtualBox (`intnet`), mismo nombre en ambas VM |
| Puerta de enlace | `192.168.1.1` |

Instalar los paquetes (con internet, antes de pasar a red interna):

```bash
sudo apt-get update
sudo apt-get install isc-dhcp-server bind9 apache2 vsftpd
```

Comandos de control, iguales para los cuatro servicios (`isc-dhcp-server`, `bind9`, `apache2`, `vsftpd`):

| Acción | Comando |
|---|---|
| Estado | `systemctl status <servicio>` |
| Encender / apagar | `sudo systemctl start <servicio>` / `sudo systemctl stop <servicio>` |
| Reiniciar tras cambios | `sudo systemctl restart <servicio>` |
| Arrancar con el sistema | `sudo systemctl enable <servicio>` |

---

## 1. DHCP

Asigna automáticamente IP, máscara, puerta de enlace y DNS a los clientes (UDP 67/68), mediante el proceso **DORA**: Discover, Offer, Request y Acknowledge.

### Configuración

1. Interfaz de escucha: copiar `config/dhcp/isc-dhcp-server` a `/etc/default/isc-dhcp-server`.
   ```
   INTERFACESv4="enp0s3"
   ```
2. Subred: copiar `config/dhcp/dhcpd.conf` a `/etc/dhcp/dhcpd.conf`.
   ```
   subnet 192.168.1.0 netmask 255.255.255.0 {
     range 192.168.1.50 192.168.1.100;
     option domain-name-servers 192.168.1.10, 8.8.8.8;
     option routers 192.168.1.1;
     option broadcast-address 192.168.1.255;
     default-lease-time 600;
     max-lease-time 7200;
   }
   ```
3. Arranque robusto: copiar `config/dhcp/override.conf` a `/etc/systemd/system/isc-dhcp-server.service.d/override.conf`. Hace que el servicio espere a la red y se reintente si falla al arrancar.
4. Aplicar:
   ```bash
   sudo systemctl daemon-reload
   sudo systemctl restart isc-dhcp-server
   ```

### Pruebas desde el cliente

```
ipconfig /release
ipconfig /renew
ipconfig /all
```

Esperado: IP entre `192.168.1.50` y `192.168.1.100`, servidor DHCP `192.168.1.10` y DNS `192.168.1.10`. En el servidor, los préstamos quedan en `/var/lib/dhcp/dhcpd.leases`.

---

## 2. DNS

Traduce nombres a IP (zona directa) e IP a nombres (zona inversa) con `bind9`, en el puerto 53. Dominio: `udea.edu.co`.

### Configuración

1. Declaración de zonas: copiar `config/dns/named.conf.local` a `/etc/bind/named.conf.local`.
2. Zona directa: copiar `config/dns/db.udea.edu.co.zone` a `/var/cache/bind/`.
3. Zona inversa: copiar `config/dns/db.192.168.1` a `/var/cache/bind/`.
4. Validar y aplicar:
   ```bash
   sudo named-checkconf
   sudo named-checkzone udea.edu.co /var/cache/bind/db.udea.edu.co.zone
   sudo named-checkzone 1.168.192.in-addr.arpa /var/cache/bind/db.192.168.1
   sudo systemctl restart bind9
   ```
   `named-checkconf` sin salida y `OK` en ambas zonas indican que no hay errores.

| Registro | Tipo | Valor |
|---|---|---|
| `udea.edu.co` | NS | `ns.udea.edu.co` |
| `udea.edu.co` | MX 10 | `mail.udea.edu.co` |
| `ns`, `mail`, `www` | A | `192.168.1.10` |
| `10` (192.168.1.10) | PTR | `ns` / `mail` / `www.udea.edu.co` |

### Pruebas desde el cliente

```
nslookup www.udea.edu.co
nslookup mail.udea.edu.co
nslookup 192.168.1.10
```

Esperado: los nombres resuelven a `192.168.1.10` y la IP devuelve un nombre de `udea.edu.co`.

---

## 3. Servidor web (Apache)

Entrega páginas por HTTP (TCP 80) desde `/var/www/html`.

### Configuración

```bash
sudo cp /var/www/html/index.html /var/www/html/index.html.original
sudo cp web/index.html /var/www/html/index.html
```

No requiere reinicio: Apache lee el archivo en cada petición.

### Pruebas desde el cliente

| Prueba | Acción | Esperado |
|---|---|---|
| Por IP | `http://192.168.1.10` | Carga el sitio |
| Por nombre (DNS + Apache) | `http://www.udea.edu.co` | Carga el sitio |
| Página inexistente | `http://www.udea.edu.co/noexiste.html` | Error 404 |
| Puerto | `telnet 192.168.1.10 80` | Conecta |

Recargar con **Ctrl + F5** para evitar la caché del navegador.

---

## 4. FTP (vsftpd)

Transferencia de archivos entre cliente y servidor: control por TCP 21 y una conexión de datos aparte.

### Configuración

1. Copiar `config/ftp/vsftpd.conf` a `/etc/vsftpd.conf`. Opciones clave:

   | Opción | Función |
   |---|---|
   | `anonymous_enable=NO` | Exige usuario y contraseña |
   | `local_enable=YES` | Permite usuarios del sistema |
   | `write_enable=YES` | Permite subir, borrar y renombrar |
   | `chroot_local_user=YES` | Encierra a cada usuario en su carpeta |
   | `allow_writeable_chroot=YES` | Permite el chroot con carpeta escribible |
   | `ftpd_banner=...` | Mensaje de bienvenida |

2. Crear un usuario dedicado y reiniciar:
   ```bash
   sudo adduser ftpudea
   sudo systemctl restart vsftpd
   ```

### Pruebas desde el cliente

```
ftp 192.168.1.10
ftp> dir
ftp> get archivo_del_servidor.txt
ftp> put archivo_del_cliente.txt
ftp> bye
```

Esperado: banner de bienvenida, `230 Login successful` y `226 Transfer complete`. En el servidor, `ls -l /home/ftpudea` muestra el archivo subido, con permiso `-rw-------` por el `local_umask=077` de Ubuntu.

Si `dir` se queda detenido, el firewall del cliente está bloqueando la conexión de datos (modo activo).

---

## 5. Comprobar que la respuesta viene del servidor

Para cada servicio: probar con el servicio encendido, detenerlo (`sudo systemctl stop <servicio>`), repetir la prueba y verificar que falla, y volver a encenderlo.

| Servicio | Con el servicio detenido |
|---|---|
| DHCP | El cliente no obtiene IP |
| DNS | `nslookup` agota el tiempo de espera |
| Apache | El navegador muestra error de conexión |
| FTP | `ftp 192.168.1.10` no conecta |

---

## 6. Cómo se integran

```
Cliente enciende         → DHCP le asigna IP y le indica el DNS 192.168.1.10
Cliente pide un nombre   → DNS responde 192.168.1.10
Cliente abre el sitio    → Apache entrega la página
Cliente transfiere       → FTP con el usuario ftpudea
```

Si algo falla, consulta [`problemas-y-soluciones.md`](problemas-y-soluciones.md).
