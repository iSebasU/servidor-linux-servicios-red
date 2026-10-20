# Problemas encontrados y soluciones

Registro de las fallas que aparecieron durante el montaje del servidor y cómo se diagnosticaron.

| # | Problema | Síntoma | Causa | Solución |
|---|---|---|---|---|
| 1 | No se podían instalar paquetes | `apache2 : Depende: apache2-bin (= 2.4.41-4ubuntu3.23) pero no va a instalarse` | Índice de paquetes desactualizado: el espejo de descarga no respondía | Ver problema 2 y luego `sudo apt-get update` |
| 2 | Espejo de Ubuntu caído | `apt-get update` mostraba `Err: ... co.archive.ubuntu.com ... Conexión rehusada` | El espejo colombiano rechazaba conexiones | Cambiar al servidor principal: `sudo sed -i 's\|http://co.archive.ubuntu.com\|http://archive.ubuntu.com\|g' /etc/apt/sources.list` |
| 3 | DHCP en `failed` tras instalar | `isc-dhcp-server.service ... failed` | Es normal: no tenía subred ni interfaz configuradas | Configurar `/etc/default/isc-dhcp-server` y `/etc/dhcp/dhcpd.conf` |
| 4 | DHCP seguía fallando | `status=1/FAILURE` | La interfaz (`enp0s8`) y la subred (`192.168.10.0`) no coincidían con la red real (`enp0s3`, `192.168.1.0`) | Usar la interfaz real y declarar la subred donde está la IP del servidor |
| 5 | DHCP caía al reiniciar la VM | `failed` después de cada arranque o cambio de adaptador | El servicio arrancaba antes de que la interfaz tuviera IP | Override de systemd con `After=network-online.target` y `Restart=on-failure` |
| 6 | Archivo de configuración vacío | nano abría un archivo nuevo: `El directorio «/etc/defaul» no existe` | Error de escritura en la ruta | Usar la ruta correcta; autocompletar con **Tab** |
| 7 | No se podía guardar en nano | `El fichero no es de escritura` | Archivo abierto sin permisos de administrador | Abrir con `sudo nano ...` |
| 8 | Guest Additions no instalaban | `Please install the gcc make perl packages` | Faltaban compiladores para construir los módulos | `sudo apt-get install gcc make perl build-essential dkms linux-headers-$(uname -r)` y `sudo /sbin/rcvboxadd setup` |
| 9 | `sudo` rechazaba la contraseña | `Lo siento, pruebe otra vez` al pegar bloques | Las líneas pegadas se tomaban como intentos de contraseña | Autenticarse primero con `sudo -v` y luego pegar el bloque |
| 10 | Errores en la guía del DNS | Registro SOA con `.edu.co.ni.`; `resolv.conf` apuntando a `192.168.1.1` | Errores del documento | Se quitó `.ni`; el DNS se entrega al cliente por DHCP con la IP correcta `192.168.1.10` |
| 11 | La consola de XP "no dejaba escribir" | Prompt `>` después de `nslookup` | `nslookup` sin argumentos abre el modo interactivo | Escribir el nombre en el `>` o salir con `exit`; usar `nslookup www.udea.edu.co` |
| 12 | Archivo descargado por FTP no aparecía | No estaba en Mis documentos | `get` guarda en la carpeta actual de CMD (`C:\Documents and Settings\Administrador`) | Buscar ahí o usar `lcd C:\` antes del `get` |
| 13 | Comando `echo` fallaba en XP | `"XP" no se reconoce como un comando` | El comando largo se partió en dos líneas | Escribir el comando corto y en una sola línea |
| 14 | FTP se quedaba pegado en `dir` | Sin respuesta después de `200 PORT command successful` | El cliente de XP usa modo activo y el firewall de Windows bloquea la conexión de datos | Desbloquear o desactivar el firewall de XP |
| 15 | Riesgo con el adaptador NAT | El cliente podía recibir DNS de VirtualBox (`10.0.2.x`) | Dos adaptadores de red entregando configuración | Deshabilitar el adaptador NAT antes de las pruebas |

## Lecciones

- Leer el mensaje de error completo (`journalctl -u <servicio>`) antes de cambiar configuraciones.
- La interfaz, la IP del servidor y la subred declarada deben coincidir.
- Validar antes de reiniciar: `named-checkconf`, `named-checkzone`.
- Respaldar cada archivo antes de editarlo (`.original`).
- Probar siempre desde el cliente y con el servicio apagado y encendido.
