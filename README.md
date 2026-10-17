# Gaming LAN Bridge con ZeroTier + MOON

<img width="1666" height="521" alt="image" src="https://github.com/user-attachments/assets/9b56a115-e5c0-4698-a7b4-174e77c34630" />


Guía para montar una red virtual de juegos en LAN mediante ZeroTier autoalojado, un controlador propio y un MOON propio.

> **Importante:** esta guía documenta una configuración concreta que se probó en un entorno particular. Adapta interfaces, direcciones, rutas y reglas de firewall a tu red. Haz una copia de seguridad de la configuración de red antes de cambiar Netplan: un error puede dejar la VM inaccesible por SSH.

## Objetivo

Crear una red LAN virtual de capa 2 (Layer 2) para juegos y servicios, permitiendo:

- LAN Discovery en juegos compatibles.
- Acceso a servidores dedicados.
- Acceso a recursos de la VLAN de gaming.
- Acceso a servicios como FileBrowser, WoW y SMB.
- Usar un controlador propio y un MOON propio, sin depender de ZeroTier Central para la administración habitual.
- Incorporar los clientes que permita la capacidad del servidor y la configuración; no existe una garantía universal de «clientes ilimitados».

## Arquitectura de referencia

| Componente | Valor de ejemplo |
|---|---|
| Sistema del servidor | Ubuntu Server 26.04 |
| IP del servidor en la VLAN de gaming | `192.168.40.253` |
| VLAN de gaming | `192.168.40.0/24` |
| Gateway de la VLAN | `192.168.40.1` (OPNsense, según la red de ejemplo) |
| Nombre de la red ZeroTier | `anzonynet-lan-gaming` |
| Network ID | `tu_network_id` |
| Interfaz física de ejemplo | `ens18` |
| Bridge Linux | `br0` |
| Puerto ZeroTier habitual | UDP `9993` |

**Sustituye los valores de ejemplo por los de tu instalación.** `tu_network_id` es un marcador de posición: usa el Network ID real de la red creada en tu controlador. Del mismo modo, sustituye `ip_publica` por la IP pública o el endpoint público que corresponda.

## 1. Instalar ZeroTier

En Ubuntu Server:

```bash
curl -s https://install.zerotier.com | sudo bash
```

Comprueba el estado:

```bash
sudo zerotier-cli info
sudo systemctl status zerotier-one
```

> El comando de instalación descarga y ejecuta un script remoto con privilegios. Si prefieres revisar el procedimiento antes de ejecutarlo, consulta la documentación oficial de ZeroTier.

## 2. Instalar y configurar el controlador ZeroTier One

ZeroTier One no es únicamente un cliente: el servicio incluye la implementación del controlador autoalojado. Al instalar el paquete `zerotier-one`, queda disponible la API local; las redes que crees desde esa API o desde ztncui se gestionan en el propio servidor.

Comprueba el identificador del nodo:

```bash
sudo zerotier-cli info
```

La salida contiene un identificador de nodo de 10 caracteres. El Network ID de las redes creadas por el controlador se basa en el identificador del nodo controlador.

La API de administración local suele escuchar en `127.0.0.1:9993` y utiliza un token secreto. Puedes consultar el token con:

```bash
sudo cat /var/lib/zerotier-one/authtoken.secret
```

**No publiques ese token ni lo incluyas en GitHub.** Permite administrar el servicio ZeroTier local. La API debe permanecer accesible solo desde el propio servidor salvo que se haya diseñado y protegido expresamente otro acceso.

Los datos del controlador se almacenan en:

```text
/var/lib/zerotier-one/controller.d/
```

Cuando se crea una red desde ztncui, la aplicación utiliza la API local de ZeroTier One para crearla y administrarla. No hay que instalar un segundo servicio independiente llamado `zerotier-controller`.

Documentación oficial: [ZeroTier Network Controller](https://docs.zerotier.com/controller/) y [ZeroTier One Service API](https://docs.zerotier.com/api-service/).

## 3. Instalar ztncui (interfaz web)

[ztncui](https://github.com/key-networks/ztncui) es una interfaz web para gestionar un controlador ZeroTier autoalojado. Debe instalarse en el mismo servidor que ZeroTier One, o configurarse con acceso seguro a la API del controlador.

> **Nota de compatibilidad:** las instrucciones de instalación desde el código fuente del repositorio de ztncui mencionan Node.js 14 y dependencias nativas de Argon2. Node.js 14 está obsoleto. Antes de instalar en una versión moderna de Ubuntu, revisa las instrucciones de paquetes DEB/RPM del proyecto o sus opciones de contenedor; no asumas que el método antiguo con Node.js 14 es la mejor opción para una instalación nueva.

### 3.1 Instalar desde el código fuente

El README del proyecto describe el siguiente método general. Los requisitos exactos pueden variar según la versión de ztncui, Node.js y Ubuntu:

```bash
sudo apt update
sudo apt install -y git build-essential g++ npm
sudo npm install -g node-gyp
git clone https://github.com/key-networks/ztncui.git
cd ztncui/src
npm install
```

Si `npm install` falla por incompatibilidad de Node.js o de las dependencias nativas, detente y utiliza un método de instalación compatible publicado por el proyecto. No fuerces una instalación con dependencias sin soporte.

### 3.2 Configurar el acceso a la API de ZeroTier One

ztncui necesita leer el token de la API local de ZeroTier One. El repositorio admite un archivo `.env` en el directorio raíz de ztncui con una variable `ZT_TOKEN`.

Desde el directorio raíz del repositorio, crea el archivo:

```bash
cd ~/ztncui
sudo cat /var/lib/zerotier-one/authtoken.secret
```

Copia el token manualmente y crea `.env` con este formato:

```dotenv
ZT_TOKEN=PEGA_AQUI_EL_TOKEN_LOCAL
NODE_ENV=production
```

Protege el archivo porque contiene una credencial administrativa:

```bash
chmod 600 .env
```

Asegúrate de que el usuario que ejecuta ztncui pueda leerlo. No publiques `.env`, no lo añadas al repositorio y no pegues su contenido en incidencias públicas. Si el proyecto se instala con un paquete o contenedor, sigue el método de configuración correspondiente a ese método de instalación en lugar de mezclarlo con esta instalación desde código fuente.

Por defecto, la API de ZeroTier One se encuentra en `localhost:9993`; el proyecto permite cambiar la dirección mediante `ZT_ADDR` si fuera necesario.

### 3.3 Preparar la contraseña inicial y arrancar ztncui

El README del repositorio fuente indica copiar el archivo de contraseña inicial:

```bash
cp -v etc/default.passwd etc/passwd
```

Después, desde el directorio `src`, arranca la aplicación para probarla:

```bash
cd ~/ztncui/src
npm start
```

La instalación descrita por el README escucha por defecto en el puerto TCP `3000`, normalmente solo en localhost. Para una prueba local, abre `http://localhost:3000` desde el propio servidor o utiliza un túnel SSH desde otro equipo. No expongas una interfaz de administración sin cifrado directamente a Internet.

El README histórico indica credenciales iniciales `admin` / `password` para determinadas instalaciones de ztncui. **Si tu versión utiliza esas credenciales, cámbialas inmediatamente en el primer acceso** y crea una cuenta administrativa propia. Comprueba el comportamiento de la versión que realmente instales.

### 3.4 Arranque automático

El proyecto describe PM2 como una opción para mantener ztncui ejecutándose tras cerrar la sesión. Si has instalado la aplicación desde el código fuente y has comprobado que funciona:

```bash
sudo npm install -g pm2
cd ~/ztncui/src
pm2 start bin/www --name ztncui
pm2 startup
```

Ejecuta el comando adicional que PM2 te muestre para configurar el inicio automático y después guarda la lista de procesos:

```bash
pm2 save
pm2 status
```

Si instalaste ztncui mediante un paquete DEB/RPM o contenedor, sigue el método de servicio y actualización de ese paquete; no combines automáticamente métodos de arranque diferentes.

### 3.5 Acceso remoto seguro

Opciones recomendadas:

- Acceder mediante un túnel SSH.
- Publicar la interfaz detrás de un proxy inverso con HTTPS y controles de acceso.
- Mantenerla solo en una red administrativa de confianza.

No configures `HTTP_ALL_INTERFACES=yes` ni abras el puerto TCP `3000` a Internet sin HTTPS y controles de acceso. El token de ZeroTier y la interfaz de administración deben tratarse como credenciales privilegiadas.

Una vez iniciada la sesión, utiliza ztncui para crear una red, configurar el rango de IP, las rutas y autorizar miembros. Los detalles específicos de esta red aparecen en el siguiente apartado.

## 4. Crear y configurar la red en el controlador

Crea la red en tu controlador ZeroTier autoalojado y anota el Network ID. En esta guía se representa como:

```text
tu_network_id
```

En la configuración de la red:

### IPv4 Auto Assign

Activa la asignación automática de direcciones IPv4.

### Assignment Pools

Configura un rango de direcciones para los clientes que no entre en conflicto con el DHCP de OPNsense ni con las direcciones estáticas. Por ejemplo, el rango `192.168.40.121`–`192.168.40.160` solo es válido si está libre y es apropiado para tu red.

### Routes

Configura la ruta de la red local que quieres alcanzar:

```text
192.168.40.0/24
```

La configuración original de esta instalación indicaba añadirla **sin gateway**. No introduzcas `via x.x.x.x` ni la IP del servidor como gateway en ese campo, salvo que tu diseño de red requiera expresamente una ruta distinta.

### Controller

En la instalación de referencia, el controlador se configuró con la dirección `192.168.40.253`, la IP de la VM. Ajusta este valor al diseño real de tu controlador.

### Autorizar clientes

Los clientes que se unan a la red pueden necesitar autorización desde el controlador, según la configuración que hayas implementado.

## 5. Crear un bridge Linux

Instala las herramientas:

```bash
sudo apt update
sudo apt install bridge-utils -y
```

En la configuración original se creaba un bridge llamado `br0` y se añadía la interfaz física `ens18`:

```bash
sudo brctl addbr br0
sudo brctl addif br0 ens18
```

> **Aviso:** estos comandos crean un bridge temporal y pueden entrar en conflicto con una configuración de Netplan que ya gestione `br0`. Para una configuración persistente, aplica la configuración de Netplan del siguiente apartado en lugar de depender únicamente de estos comandos manuales.

## 6. Configurar Netplan

Ejemplo de configuración final utilizada como referencia. Guarda una copia del YAML actual antes de modificarlo. Comprueba que la interfaz, la dirección MAC, la IP y el gateway coinciden con tu instalación.

Archivo habitual: `/etc/netplan/` (el nombre concreto puede variar).

```yaml
network:
  version: 2
  renderer: networkd

  ethernets:
    ens18:
      dhcp4: false
      dhcp6: false

  bridges:
    br0:
      interfaces:
        - ens18

      addresses:
        - 192.168.40.253/24

      routes:
        - to: default
          via: 192.168.40.254

      nameservers:
        addresses:
          - 1.1.1.1
          - 9.9.9.9

      dhcp4: false
      dhcp6: false

      macaddress: bc:24:11:c4:74:d5
```

**Revisa especialmente el gateway:** la arquitectura de referencia menciona `192.168.40.1`, mientras que la configuración Netplan original utiliza `192.168.40.254`. No son el mismo valor. Configura el gateway que realmente corresponda a tu red antes de aplicar los cambios.

Aplica la configuración desde la consola de la VM, si es posible:

```bash
sudo netplan generate
sudo netplan try
```

Si la conectividad funciona, confirma los cambios cuando Netplan lo solicite. Después puedes usar `sudo netplan apply` si es necesario.

## 7. Script para ajustar el bridge de ZeroTier

Archivo:

```text
/usr/local/bin/ztbridge.sh
```

Contenido de la configuración original:

```bash
#!/bin/bash

sleep 10

ZT_IF=$(ip link | grep zt | head -1 | cut -d: -f2 | tr -d ' ')

ip link set "$ZT_IF" master br0
ip link set "$ZT_IF" mtu 1500

ip link set "$ZT_IF" promisc on
ip link set ens18 promisc on
ip link set br0 promisc on

echo 0 > /sys/class/net/br0/bridge/multicast_snooping 2>/dev/null || true

ip addr flush dev "$ZT_IF"
```

Crea el archivo y dale permisos:

```bash
sudo nano /usr/local/bin/ztbridge.sh
sudo chmod +x /usr/local/bin/ztbridge.sh
```

> **Nota:** este script selecciona la primera interfaz cuyo nombre contiene `zt`. Si hay varias interfaces ZeroTier, verifica que selecciona la correcta. Antes de usarlo en producción, conviene añadir comprobaciones para evitar ejecutar `ip link set` con una variable vacía si ZeroTier aún no ha creado la interfaz.

## 8. Crear el servicio systemd

Archivo:

```text
/etc/systemd/system/ztbridge.service
```

Contenido:

```ini
[Unit]
Description=ZeroTier Bridge Fix
After=zerotier-one.service

[Service]
Type=oneshot
ExecStart=/usr/local/bin/ztbridge.sh

[Install]
WantedBy=multi-user.target
```

Activa el servicio:

```bash
sudo systemctl daemon-reload
sudo systemctl enable ztbridge
sudo systemctl start ztbridge
```

Comprueba su estado:

```bash
sudo systemctl status ztbridge
```

## 9. Abrir el puerto UDP en OPNsense

En la configuración de ejemplo se creó una regla de redirección de puerto (Port Forward) en la WAN:

- Protocolo: UDP.
- Puerto de destino: `9993`.
- IP de destino: `192.168.40.253`.

Ajusta la regla a la IP real del servidor y a tu política de firewall. Comprueba también las reglas asociadas que permitan el tráfico necesario. No publiques servicios adicionales si no son necesarios.

## 10. Crear el MOON

Un MOON permite configurar un nodo ZeroTier como punto de encuentro (moon). La IP pública y el puerto deben ser alcanzables desde los clientes que deban utilizarlo.

### 8.1 Obtener la identidad pública

En el servidor:

```bash
sudo cat /var/lib/zerotier-one/identity.public
```

Copia la identidad pública completa. No publiques `identity.secret` ni compartas archivos privados de identidad.

### 8.2 Generar el archivo de configuración

```bash
cd /tmp
sudo zerotier-idtool initmoon /var/lib/zerotier-one/identity.public > moon.json
```

Edita `moon.json`:

```bash
sudo nano /tmp/moon.json
```

Busca `stableEndpoints` y configura el endpoint público de tu servidor. Ejemplo con marcador de posición:

```json
"stableEndpoints": [
  "ip_publica/9993"
]
```

Sustituye `ip_publica` por la IP pública real o el endpoint adecuado. Si tu IP pública cambia, tendrás que actualizar el endpoint y regenerar/distribuir la configuración MOON según corresponda.

### 8.3 Generar e instalar el archivo MOON

Desde el directorio donde está `moon.json`:

```bash
sudo zerotier-idtool genmoon moon.json
```

El comando genera un archivo `.moon`. Cópialo al directorio de moons:

```bash
sudo mkdir -p /var/lib/zerotier-one/moons.d/
sudo cp ./*.moon /var/lib/zerotier-one/moons.d/
sudo systemctl restart zerotier-one
```

Comprueba:

```bash
sudo zerotier-cli listmoons
```

## 11. Obtener el ID del MOON y configurar Orbit

El comando `orbit` necesita **dos valores diferentes**: el ID del MOON y el ID del nodo ZeroTier que actúa como MOON. No uses el Network ID de la red en lugar de estos identificadores.

### 9.1 Consultar el ID del nodo

En el servidor MOON:

```bash
sudo zerotier-cli info
```

La salida suele incluir una línea parecida a:

```text
200 info a1b2c3d4e5 ONLINE
```

El identificador de 10 caracteres (`a1b2c3d4e5` en este ejemplo) es el **Node ID** del servidor. Usa el valor real que te muestre tu sistema.

### 9.2 Consultar el ID del MOON

Después de generar el archivo `.moon`, puedes consultar su contenido para localizar el identificador del MOON. El archivo suele tener un nombre con un identificador hexadecimal; comprueba la estructura real del archivo generado y la documentación de la versión de ZeroTier instalada para identificar el **Moon ID** correcto. No des por hecho que el Network ID de tu red o el Node ID son el Moon ID.

### 9.3 Ejecutar Orbit en cada cliente que deba utilizar el MOON

En cada cliente ZeroTier correspondiente, ejecuta el comando con los identificadores reales:

```bash
sudo zerotier-cli orbit <moon_id> <moon_seed>
```

- `<moon_id>`: identificador del MOON.
- `<moon_seed>`: Node ID del servidor que actúa como MOON.

En algunas configuraciones los dos identificadores pueden parecer similares, pero debes obtener ambos de los datos generados por ZeroTier, no copiarlos de la red ZeroTier.

Comprueba el resultado:

```bash
sudo zerotier-cli listmoons
sudo zerotier-cli peers
```

Si necesitas deshacer la asociación con un MOON, consulta primero la documentación de la versión de ZeroTier instalada para usar el comando de desorbit correspondiente.

## 12. Unir clientes a la red

Instala ZeroTier en cada cliente y únete a la red:

```bash
sudo zerotier-cli join tu_network_id
```

En Windows, los comandos y permisos pueden variar; puedes usar la interfaz de ZeroTier o ejecutar el CLI como administrador.

Autoriza el cliente en el controlador si es necesario. Comprueba que recibe una dirección IP correcta y que puede alcanzar los equipos/servicios previstos.

## 13. Comandos útiles para diagnóstico

```bash
# Estado del cliente ZeroTier
sudo zerotier-cli info

# Peers
sudo zerotier-cli peers

# Redes
sudo zerotier-cli listnetworks

# MOONs
sudo zerotier-cli listmoons

# Estado de los enlaces del bridge
bridge link

# Interfaces y direcciones
ip a

# Escucha UDP en el puerto 9993
sudo ss -lunp | grep 9993

# Servicio ZeroTier
sudo systemctl status zerotier-one

# Servicio del bridge
sudo systemctl status ztbridge
```

## 14. Resultados de las pruebas originales

En la instalación documentada se indicó que funcionaban:

- Half-Life.
- Half-Life Dedicated Server.
- Blood: Fresh Supply.
- FileBrowser.
- Servidores dedicados.
- Acceso a los recursos previstos de la VLAN de gaming.
- LAN Discovery en juegos compatibles.

Estos resultados corresponden a las pruebas realizadas en esa instalación y no garantizan el mismo comportamiento en todas las redes, sistemas o versiones.

## 15. Limitaciones observadas

En las pruebas originales, algunos juegos antiguos no anunciaban correctamente las partidas mediante broadcast:

- Quake II.
- Quake III Arena.
- Unreal Tournament clásico.

En esos casos, se observó que conectarse directamente por IP funcionaba:

```text
Connect by IP
```

## 16. Seguridad y mantenimiento

- No subas claves privadas, secretos, credenciales ni copias de seguridad sensibles al repositorio.
- Antes de hacer público el repositorio, revisa los archivos y el historial de Git.
- Usa marcadores como `ip_publica` y `tu_network_id` en la documentación pública.
- Limita las reglas de firewall a los puertos y destinos necesarios.
- Mantén Ubuntu y ZeroTier actualizados.
- Documenta las IP, rutas e interfaces propias de tu entorno en un archivo local excluido de Git.
- Revisa la configuración del bridge después de actualizaciones del sistema o de ZeroTier.

## Referencias

- [Documentación oficial de ZeroTier](https://docs.zerotier.com/)
- [ZeroTier en GitHub](https://github.com/zerotier/ZeroTierOne)
