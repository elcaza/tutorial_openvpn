# Tutorial OpenVPN
Creando un servidor OpenVPN sobre Debian 10.

Este es un tutorial basado en en el artículo de [Digital Ocean](https://www.digitalocean.com/community/tutorials/como-configurar-un-servidor-de-openvpn-en-ubuntu-18-04-es). Sin embargo, este tutorial contiene algunas notas adicionales.	

# Creando nuestro porpio servidor vpn

OpenVPN es una solución de capa de conexión segura (SSL) de funciones completas y de código abierto que cuenta con una amplia variedad de configuraciones. A través de este tutorial, configurará un servidor de OpenVPN en un servidor Debian 10. 

# Prerrequisitos
Vamos a necesitar una máquina
+ Máquina OpenVPN (Debian 10)
	+ OpenVPN
	+ CA
+ Clientes (Opcionales)
	+ (Debian 10)

# Paso 1: Instalar OpenVPN y EasyRSA

## Desde la carpeta OpenVPN

Instalamos openvpn

```
sudo apt update
sudo apt -y install openvpn
```

Descargamos easy-rsa desde su repositorio en github

```
wget -P ~/ https://github.com/OpenVPN/easy-rsa/releases/download/v3.0.4/EasyRSA-3.0.4.tgz
cd ~
tar xvf EasyRSA-3.0.4.tgz
```

Creamos una carpeta para la CA y otra para el servidor OPENVPN. También eliminamos el tar.

```
cp -r EasyRSA-3.0.4 CA_EasyRSA-3.0.4
mv EasyRSA-3.0.4 VPN_EasyRSA-3.0.4
rm EasyRSA-3.0.4.tgz
```

# Paso 2: Configurar las variables de EasyRSA y crear la CA

## Desde la carpeta CA

```
cd ~/CA_EasyRSA-3.0.4/
cp vars.example vars
vim vars
```
Descomentamos las siguientes líneas y personalizamos nuestros valores

```conf
set_var EASYRSA_REQ_COUNTRY    "US"
set_var EASYRSA_REQ_PROVINCE   "California"
set_var EASYRSA_REQ_CITY       "San Francisco"
set_var EASYRSA_REQ_ORG        "Copyleft Certificate Co"
set_var EASYRSA_REQ_EMAIL      "me@example.net"
set_var EASYRSA_REQ_OU         "My Organizational Unit"
```

Ejecutamos el script `easyrsa` llamando a la función `init-pki`
```
./easyrsa init-pki

```

+ output
+ Note: using Easy-RSA configuration from: ./vars
+ init-pki complete; you may now create a CA or requests.
+ Your newly created PKI dir is: /home/user/EasyRSA-3.0.4/pki


Luego, ejecute la secuencia de comandos `easyrsa` nuevamente, seguida de la opción `build-ca`. Con esto, se crearán la CA y dos archivos importantes, **ca.crt** y **ca.key**, que representarán los lados públicos y privados de un certificado SSL.

+ ca.crt 
	+ Es el archivo de certificado público de CA que usan, en el contexto de OpenVPN, el servidor y el cliente para informarse entre sí que son parte de la misma red de confianza y no atacantes desconocidos. Por este motivo, su servidor y todos sus clientes necesitarán una copia del archivo ca.crt.
+ ca.key 
	+ Es la clave privada que la máquina CA usa para firmar claves y certificados para servidores y clientes. Si un atacante logra acceder a su CA, y con ello a su archivo ca.key, podrá firmar solicitudes de certificados y acceder a su VPN, lo que inhabilitará su seguridad. Esta es la razón por la cual su archivo ca.key deberá estar **únicamente** en su máquina de CA. A su vez, lo ideal sería que su máquina de CA estuviera desconectada cuando no firme solicitudes de certificados como medida de seguridad adicional.
+ Si no desea que se le solicite una contraseña cada vez que interactúe con su CA, puede ejecutar el comando build-ca con la opción nopass, de la siguiente forma:

```
./easyrsa build-ca nopass
```
En el resultado, se le solicitará confirmar el nombre común de su CA:

+ El nombre común es el que se usa para hacer referencia a esta máquina en el contexto de la autoridad de certificación. Puede ingresar cualquier secuencia de caracteres para el nombre común de la CA. No obstante, para hacerlo más simple, presione ENTER para aceptar el nombre predeterminado.
+ Con esto, su CA quedará configurada y lista para comenzar a firmar solicitudes de certificado.

Los archivos importantes se encuentran en:
+ EasyRSA-3.0.4/pki/ca.crt
+ EasyRSA-3.0.4/pki/private/ca.key

# Paso 3: Crear los archivos de certificado, clave y cifrado del servidor OPENVPN

Ahora que ya tiene lista una CA, puede generar una solicitud de clave y certificado privados desde su servidor y luego transferirla a su CA para que la firme y cree el certificado solicitado. También puede crear algunos archivos adicionales que se usan durante el proceso de cifrado.

## Desde la carpeta OpenVPN

Creamos una pki

```
cd ~/VPN_EasyRSA-3.0.4/
./easyrsa init-pki
```

Luego, ejecutamos la secuencia de comandos easyrsa nuevamente, esta vez con la opción gen-req seguida de un nombre común para la máquina. Este nombre, una vez más, puede ser cualquiera, aunque puede ser útil que sea descriptivo. A lo largo de este tutorial, el nombre común del servidor OpenVPN será simplemente “server”. Asegúrese de incluir también la opción nopass. Si no lo hace, se protegerá con contraseña el archivo de solicitud, lo que puede generar problemas de permisos más adelante:

Nota: si elige otro nombre que no sea “server”, deberá modificar algunas de las instrucciones a continuación. Por ejemplo, al copiar los archivos generados al directorio /etc/openvpn, deberá sustituir los nombres que correspondan. También deberá modificar el archivo /etc/openvpn/server.conf más adelante para señalar los archivos .crt y .key correctos.

```
./easyrsa gen-req server nopass
```

Con esto se crearán una clave privada para el servidor y un archivo de solicitud de certificado llamado server.req. Copie la clave del servidor al directorio /etc/openvpn/

```
sudo cp ~/VPN_EasyRSA-3.0.4/pki/private/server.key /etc/openvpn/
```

Pasamos a firmar nuestro .req

## Desde la carpeta CA
Usando nuevamente la secuencia de comandos easyrsa, importe el archivo server.req y siga la ruta de este con su nombre común:
+ Nota: Importamos el *.req bajo el nombre "server"

```
cd ~/CA_EasyRSA-3.0.4/
./easyrsa import-req ~/VPN_EasyRSA-3.0.4/pki/reqs/server.req server
```

Luego, firme la solicitud ejecutando la secuencia `easyrsa` con la opción `sign-req` ,seguida del tipo de solicitud y el nombre común. El tipo de solicitud puede ser `client` o `server`. Por ello, para la solicitud de certificado del servidor de OpenVPN asegúrese de usar el tipo de solicitud server:

```
./easyrsa sign-req server server
```

Confirmamos escribiendo `yes` 
+ Si cifró su clave de CA, se le solicitará ingresar la contraseña en este punto.
+ Certificate created at: /home/user/CA_EasyRSA-3.0.4/pki/issued/server.crt

Transferimos nuestro certificado recien firmado y el certificado de la CA a la carpeta de configuración OpenVPN

```
sudo cp pki/issued/server.crt /etc/openvpn/
sudo cp pki/ca.crt /etc/openvpn/

```

## Desde la carpeta OpenVPN

Luego, diríjase a su directorio EasyRSA:
Desde ahí, cree una clave segura Diffie-Hellman para usarla durante el intercambio de claves escribiendo:

```
cd ~/VPN_EasyRSA-3.0.4/
./easyrsa gen-dh
```

Esta operación puede tardar unos minutos. Una vez que se complete, genere una firma HMAC para fortalecer las capacidades de verificación de integridad TLS del servidor:
+ (OUTPUT): DH parameters of size 2048 created at /home/user/VPN_EasyRSA-3.0.4/pki/dh.pem

Ejecutamos

```
sudo openvpn --genkey --secret ta.key

```

Cuando el comando se aplique, copie los dos nuevos archivos a su directorio /etc/openvpn/:

```
sudo cp ta.key /etc/openvpn/
sudo cp pki/dh.pem /etc/openvpn/
```

Con esto, se generarán todos los archivos de certificados y claves necesarios para su servidor. Ya está listo para crear los certificados y las claves correspondientes que usará su máquina cliente para acceder a su servidor de OpenVPN.

# Paso 4: Generar un par de certificado y clave de cliente

Aunque puede generar una solicitud de claves y certificados privados en su máquina cliente y luego enviarla a la CA para que la firme, en esta guía se describe un proceso para generar la solicitud de certificado en el servidor. El beneficio de esto es que podemos crear una secuencia de comandos que generará de manera automática archivos de configuración que contienen las claves y los certificados necesarios. Esto le permite evitar la transferencia de claves, certificados y archivos de configuración a los clientes y optimiza el proceso para unirse a la VPN.

Generaremos un par individual de clave y certificado de cliente para esta guía. Si tiene más de un cliente, puede repetir este proceso para cada uno. Tenga en cuenta que deberá pasar un valor de nombre único a la secuencia de comandos para cada cliente. En este tutorial, el primer par de certificado y clave se denominará *“client1”*.

Comience por crear una estructura de directorios dentro de su directorio de inicio para almacenar los archivos de certificado y clave de cliente.
+ Debido a que almacenará los pares de certificado y clave de sus clientes y los archivos de configuración en este directorio, debe bloquear sus permisos ahora como medida de seguridad:

## Desde la carpeta OpenVPN

```
mkdir -p ~/client-configs/keys
chmod -R 700 ~/client-configs

```

Luego, diríjase al directorio EasyRSA y ejecute la secuencia de comandos `​​​​​​easyrsa` con las opciones `gen-req` y `nopass`, junto con el `nombre común` para el cliente:

```
cd ~/VPN_EasyRSA-3.0.4/
./easyrsa gen-req client1 nopass
```

+ (OUTPUT):
	+ keypair and certificate request completed. Your files are:
	+ req: /home/user/VPN_EasyRSA-3.0.4/pki/reqs/client1.req
	+ key: /home/user/VPN_EasyRSA-3.0.4/pki/private/client1.key


copie el archivo client1.key al directorio /client-configs/keys/ que creó antes.
Luego, transfiera el archivo client1.req a su carpeta de CA



```
cp pki/private/client1.key ~/client-configs/keys/
```

Luego, transfiera el archivo client1.req a su carpeta de CA

```
cp ~/VPN_EasyRSA-3.0.4/pki/reqs/client1.req ~/VPNuser@$IP_CA:/tmp
```

## Desde la carpeta CA

Luego firme la solicitud como lo hizo en el caso del servidor en el paso anterior. Esta vez, asegúrese de especificar el tipo de solicitud `client`:

Con esto, se creará un archivo de certificado de cliente llamado client1.crt. Transfiera este archivo de vuelta a la carpeta VPN servidor:

```
cd ~/CA_EasyRSA-3.0.4
./easyrsa import-req ~/VPN_EasyRSA-3.0.4/pki/reqs/client1.req client1
./easyrsa sign-req client client1
cp ~/CA_EasyRSA-3.0.4/pki/issued/client1.crt ~/client-configs/keys/
```

## Desde la carpeta OpenVPN
Copie también los archivos `ca.crt` y `ta.key` al directorio `/client-configs/keys/`:


```
cd ~/VPN_EasyRSA-3.0.4
sudo cp ta.key ~/client-configs/keys/
sudo cp /etc/openvpn/ca.crt ~/client-configs/keys/
```

Con esto, se generarán los certificados y las claves de su servidor y cliente, y se almacenarán en los directorios correspondientes de su servidor. Aún quedan algunas acciones que se deben realizar con estos archivos, pero se realizarán más adelante. Por ahora, puede comenzar a configurar OpenVPN en su servidor.

# Paso 5: Configurar el servicio de OpenVPN

Ahora que se generaron los certificados y las claves de su cliente y servidor, puede comenzar a configurar el servicio de OpenVPN para que use estas credenciales.

Comience copiando un archivo de configuración de OpenVPN de muestra al directorio de configuración y luego extráigalo para usarlo como base para su configuración:

## Desde la carpeta OPENVPN

```
cd ~/VPN_EasyRSA-3.0.4
sudo cp /usr/share/doc/openvpn/examples/sample-config-files/server.conf.gz /etc/openvpn/
sudo gzip -d /etc/openvpn/server.conf.gz
```

Abrimos el archivo de configuración
```
sudo vim /etc/openvpn/server.conf
```

Busque la directiva tls-auth para encontrar la sección HMAC. Los comentarios de esta línea no deberían existir, pero si esto no sucede elimine “;” para quitar los comentarios:

```
tls-auth ta.key 0 # This file is secret
```


Luego, busque las líneas `cipher` con comentarios para encontrar la sección de cifrado. El código `AES-256-CBC` ofrece un buen nivel de cifrado y cuenta con buen respaldo. Una vez más, no debería haber comentarios para esta línea, pero si esto no sucede simplemente elimine el “;” que la precede

Y debajo, agregue una directiva `auth` para seleccionar el algoritmo de codificación de mensajes `HMAC`. `SHA256` es una buena opción:

```
cipher AES-256-CBC
auth SHA256
```

Luego, encuentre la línea que contenga la directiva dh que define los parámetros Diffie-Hellman. Debido a algunos cambios recientes realizados en EasyRSA, el nombre de archivo de la clave Diffie-Hellman puede ser distinto del que figura en el ejemplo del archivo de configuración del servidor. Si es necesario, cambie el nombre de archivo que aparece eliminando 2048 para que coincida con la clave que generó en el paso anterior:
+ Antes `dh dh2048.pem`
+ Ahora `dh dh.pem`

```
dh dh.pem
```
 
Por último, busque los ajustes `user` y `group`, y elimine “;” al inicio de cada uno para quitar los comentarios de estas líneas:

```
user nobody
group nogroup
```

Los cambios realizados al archivo de muestra server.conf hasta el momento son necesarios para que OpenVPN funcione. Los cambios mencionados a continuación son opcionales, aunque también se necesitan para muchos casos de uso comunes.


# Opcionales ===================================

### (Opcional) Aplicar cambios de DNS para redireccionar todo el tráfico a través de la VPN

Con los ajustes anteriores, se creará la conexión de VPN entre las dos máquinas, pero no se forzarán conexiones para usar el túnel. Si desea usar la VPN para dirigir todo su tráfico, probablemente le convenga aplicar los ajustes de sistemas de nombre de domino (DNS) a las computadoras clientes.

Para habilitar esta funcionalidad, debe cambiar algunas directivas del archivo `server.conf`. Encuentre la sección `redirect-gateway` y elimine el punto y coma, “`;`”, del inicio de la línea `redirect-gateway` para quitar los comentarios:

## Desde la carpeta OPENVPN

```
sudo vim /etc/openvpn/server.conf
```

Tras la edición queda de la siguiente manera

```
push "redirect-gateway def1 bypass-dhcp"
```

Debajo de esto, encontrará la sección dhcp-option. Nuevamente, elimine “;” del inicio de ambas líneas para quitar los comentarios:

```
push "dhcp-option DNS 208.67.222.222"
push "dhcp-option DNS 208.67.220.220"
```

Esto ayudará a los clientes a configurar de nuevo sus ajustes de DNS para usar el túnel de la VPN como puerta de enlace predeterminada.

### (Opcional) Ajustar el puerto y el protocolo

Por defecto, el servidor de OpenVPN usa el puerto `1194` y el protocolo `UDP` para aceptar las conexiones de los clientes. Si necesita usar un puerto diferente debido a restricciones de los entornos de red que sus clientes puedan emplear, puede cambiar la opción port. *Si no aloja contenido web en su servidor de OpenVPN*, el puerto 443 es una opción común, ya que se suele permitir en las reglas de firewall.

*Si no tiene necesidad de usar un puerto y protocolo distintos, es mejor dejar estos dos ajustes como sus valores predeterminados.*

## Desde la carpeta OPENVPN

```
sudo vim /etc/openvpn/server.conf
```

```
# Optional!
;port 1194
port 443
```

Algunas veces, el protocolo se limita a ese puerto también. Si esto sucede, cambie proto de UDP a TCP:

```
# Optional!
proto tcp
;proto udp
```

Si **cambia** el protocolo a TCP, deberá cambiar el valor de la directiva `explicit-exit-notify` de `1` a `0`, ya que solo UDP la usa. Si no lo hace al usar TCP, se producirán errores al iniciar el servicio de OpenVPN:

```
# Optional!
;explicit-exit-notify 1
explicit-exit-notify 0
```

*Si no tiene necesidad de usar un puerto y protocolo distintos, es mejor dejar estos dos ajustes como sus valores predeterminados.*

### (Opcional) Apuntar a credenciales no predeterminadas

## Desde la carpeta OPENVPN

```
sudo vim /etc/openvpn/server.conf
```

Si anteriormente seleccionó un nombre distinto durante el comando ./ build-key-server, modifique las líneas cert y key que ve para apuntar a los archivos .crt y .key adecuados. Si usó el nombre predeterminado, “server”, esto ya está correctamente configurado:

```
cert server.crt
key server.key
```

# Opcionales ===================================

Luego de revisar y aplicar los cambios necesarios a la configuración de su servidor de OpenVPN para sus necesidades de uso específicas, puede comenzar a aplicar algunos cambios a la conexión de su servidor.

# Paso 6: Ajustar la configuración de redes del servidor

Hay algunos aspectos de la configuración de redes del servidor que deben modificarse para que OpenVPN pueda dirigir el tráfico de manera correcta a través de la VPN. El primero es el *enrutamiento de IP*, un método para determinar a dónde se debe dirigir el tráfico de IP. Esto es esencial para la funcionalidad de VPN que proporcionará su servidor.

Para ajustar el enrutamiento de IP predeterminado de su servidor, modifique el archivo `/etc/sysctl.conf`:

## Desde la carpeta OPENVPN

```
sudo vim /etc/sysctl.conf
```

Dentro de este, busque la línea con comentarios que configura `net.ipv4.ip_forward`. Elimine el carácter ****`“#”` del inicio de la línea para quitar los comentairos de este ajuste:

```
net.ipv4.ip_forward=1
```

Guarde y cierre el archivo cuando termine.

Para leer el archivo y modificar los valores de la sesión actual, escriba lo siguiente:

```
sudo sysctl -p
```

+ Mostrará net.ipv4.ip_forward = 1

## Desde la carpeta OPENVPN

### Si hay firewall

### Si no, saltar al paso 7

Si siguió la guía de instalación inicial para servidores de Ubuntu 18.04 mencionada en los requisitos previos, debería tener un firewall UFW establecido. 

Independientemente de que use el firewall para bloquear el tráfico no deseado (algo que debe hacer casi siempre), a los efectos de esta guía necesitará un firewall para manipular parte del tráfico que ingresa al servidor. 

Algunas de las reglas del firewall deben modificarse para permitir el enmascaramiento, un concepto de iptables que proporciona traducción de direcciones de red (NAT) dinámica sobre la marcha para dirigir de manera correcta las conexiones de los clientes.

Antes de abrir el archivo de configuración de firewall para agregar las reglas de enmascaramiento, primero debe encontrar la interfaz de red pública de su máquina. Para hacer esto, escriba lo siguiente:

```
ip route | grep default
```

+ Output
	+ `default via 192.168.100.1 dev eth0 proto dhcp src 192.168.100.177 metric 100 `


Su interfaz pública es la secuencia que se halla en el resultado de este comando después de la palabra `“dev”`. Por ejemplo, en este resultado se muestra la interfaz denominada `eth0`, que aparece resaltada a continuación:

Una vez que tenga la interfaz asociada con su ruta predeterminada, abra el archivo /etc/ufw/before.rules para agregar la configuración pertinente:

```
sudo vim /etc/ufw/before.rules
```


Las reglas de UFW suelen agregarse usando el comando `ufw`. Sin embargo, las reglas enumeradas en el archivo `before.rules` se leen e implementan antes de que se carguen las reglas de UFW. En la parte superior del archivo, agregue las líneas resaltadas a continuación. Con esto, se establecerá la política predeterminada de la cadena `POSTROUTING` en la tabla `nat` y se enmascarará el tráfico que provenga de la VPN. Recuerde reemplazar `wlp11s0​​​` en la línea `-A POSTROUTING`siguiente por la interfaz que encontró en el comando anterior:


```
# START OPENVPN RULES
# NAT table rules
*nat
:POSTROUTING ACCEPT [0:0] 
# Allow traffic from OpenVPN client to wlp11s0 (change to the interface you discovered!)
-A POSTROUTING -s 10.8.0.0/8 -o eth0 -j MASQUERADE
COMMIT
# END OPENVPN RULES

# Don't delete these required lines, otherwise there will be errors
```

Guarde y cierre el archivo cuando termine.

Luego, debe indicarle a UFW que permita también los paquetes reenviados de modo predeterminado. Para hacer esto, abra el archivo `/etc/default/ufw`:

```
sudo vim /etc/default/ufw
```

Dentro de este, encuentre la directiva `DEFAULT_FORWARD_POLICY` y cambie el valor de `DROP` a `ACCEPT`:

```
DEFAULT_FORWARD_POLICY="ACCEPT"
```

Guarde y cierre el archivo cuando termine.

Luego, modifique el firewall para permitir el tráfico hacia OpenVPN. Si no cambió el puerto y el protocolo en el archivo `/etc/openvpn/server.conf`, deberá abrir el tráfico UDP al puerto `1194`. Si modificó el puerto o el protocolo, sustituya los valores que seleccionó aquí.

En caso de que se haya olvidado de agregar el puerto SSH al seguir el tutorial de los requisitos previos, agréguelo aquí también:

```
sudo ufw allow 1194/udp
sudo ufw allow OpenSSH
```

Luego de agregar esas reglas, deshabilite y vuelva a habilitar UFW para reiniciarlo y cargue los cambios de todos los archivos que haya modificado:

```
sudo ufw disable
sudo ufw enable
```

Su servidor quedará, así, configurado para manejar de manera correcta el tráfico de OpenVPN.


# Paso 7: Iniciar y habilitar el servicio de OpenVPN

Finalmente, ya está listo para iniciar el servicio de OpenVPN en su servidor. Esto se hace mediante la utilidad systemctl de systemd.

Inicie el servidor de OpenVPN especificando el nombre de su archivo de configuración como una variable de instancia después del nombre del archivo de unidad de systemd. El archivo de configuración de su servidor se llama /etc/openvpn/server.conf. Por lo tanto, debe agregar @server  al final de su archivo de unidad cuando lo llame:“”

Luego de ello comprobamos que el servicio se haya iniciado correctamente escribiendo lo siguiente:
+ Esperamos un Active: active (running)

```
sudo systemctl start openvpn@server
sudo systemctl status openvpn@server
```

También puede controlar que la interfaz tun0 de OpenVPN esté disponible escribiendo lo siguiente:

```
ip addr show tun0
```

Luego de iniciar el servicio, habilítelo para que se cargue de manera automática en el inicio:

```
sudo systemctl enable openvpn@server
```

Su servicio de OpenVPN quedará, así, configurado y en funcionamiento. Sin embargo, para comenzar a usarlo debe crear primero un archivo de configuración para la máquina cliente. En el tutorial ya se explicó la forma crear pares de certificado y clave para clientes, y en el siguiente paso demostraremos la forma de crear una infraestructura que generará archivos de configuración de clientes fácilmente.

# Paso 8: Crear la infraestructura de configuración de clientes

Es posible que se deban crear archivos de configuración para clientes de OpenVPN, ya que todos los clientes deben tener su propia configuración y alinearse con los ajustes mencionados en el archivo de configuración del servicio. 

En este paso, en lugar de detallarse el proceso para escribir un único archivo de configuración que solo se pueda usar en un cliente, se describe un proceso para crear una infraestructura de configuración de cliente que puede usar para generar archivos de configuración sobre la marcha. 

Primero creará un archivo de configuración “de base” y luego una secuencia de comandos que le permitirá generar archivos de configuración, certificados y claves de clientes exclusivos según sea necesario.

Comience creando un nuevo directorio en el que almacenará archivos de configuración de clientes dentro del directorio client-configs creado anteriormente:

## Desde la carpeta OPENVPN

```
cd ~/VPN_EasyRSA-3.0.4
mkdir -p ~/client-configs/files
```

Luego, copie un archivo de configuración de cliente de ejemplo al directorio client-configs para usarlo como su configuración de base:

```
cp /usr/share/doc/openvpn/examples/sample-config-files/client.conf ~/client-configs/base.conf
```

Editamos el archivo que acabamos de copiar:

```
vim ~/client-configs/base.conf

```

Dentro de este, ubique la directiva `remote`. Esto dirige al cliente a la dirección de su servidor de OpenVPN; la dirección IP pública de su servidor de OpenVPN. Si decidió cambiar el puerto en el que el servidor de OpenVPN escucha, también deberá cambiar `1194` por el puerto seleccionado:
+ Pones la ip de tu servidor y el puerto
+ Puedes poner más de una

```
# The hostname/IP and port of the server.
# You can have multiple remote entries
# to load balance between the servers.

;remote your_server_ip 1194
remote 192.168.100.197 1194
```

Asegúrese de que el protocolo coincida con el valor que usa en la configuración del servidor:

```
proto udp
```

Luego, elimine los comentarios de las directivas user y group quitando “;” al inicio de cada línea:

```
# Downgrade privileges after initialization (non-Windows only)
user nobody
group nogroup
```

Encuentre las directivas que establecen ca, cert y key. Elimine los comentarios de estas directivas, ya que pronto agregará los certificados y las claves dentro del archivo:

```
# SSL/TLS parms.
# See the server config file for more
# description.  It's best to use
# a separate .crt/.key file pair
# for each client.  A single ca
# file can be used for all clients.
ca ca.crt
cert client.crt
key client.key
```

De manera similar, elimine los comentarios de la directiva tls-auth, ya que agregará ta.key directamente al archivo de configuración de cliente:

```
# If a tls-auth key is used on the server
# then every client must also have the key.
tls-auth ta.key 1
```

Refleje los ajustes de `cipher` y `auth` establecidos en el archivo /etc/openvpn/server.conf:
+ Nota: Seguimos editando `~/client-configs/base.conf`
	+ Solamente estamos tomando como referencias los ajustes establecidos anteriormente en el archivo server.conf

```
cipher AES-256-CBC
auth SHA256
```

Luego, agregue la directiva `key-direction` en algún lugar del archivo. Es *necesario que* fije el valor “`1`” para esta, a fin de que la VPN funcione de manera correcta en la máquina cliente:

```
##########################################
# Custom config

# Key Direction
key-direction 1
```

Por último, agregue algunas líneas **no incluidas**. Aunque puede incluir estas directivas en todos los archivos de configuración de clientes, *solo debe habilitarlas para clientes Linux que incluyan un archivo* `/etc/openvpn/update-resolv-conf`. Esta secuencia de comandos usa la utilidad resolvconf para actualizar la información de DNS para clientes Linux.

 
```
# Only if you linux client have /etc/openvpn/update-resolv-conf

# script-security 2
# up /etc/openvpn/update-resolv-conf
# down /etc/openvpn/update-resolv-conf
```

*Si su cliente tiene Linux instalado y un archivo /etc/openvpn/update-resolv-conf, elimine los comentarios de estas líneas del archivo de configuración de cliente luego de que se haya generado.*

Guarde y cierre el archivo cuando termine.

### Automatización

A continuación, cree una secuencia de comandos simple que compile su configuración de base con el certificado, la clave y los archivos de cifrado pertinentes, y luego ubique la configuración generada en el directorio `~/client-configs/files`. 

Abra un nuevo archivo llamado `make_config.sh` en el directorio `~/client-configs`:

```
vim ~/client-configs/make_config.sh
```

*Contenido*

```shell
#!/bin/bash

# First argument: Client identifier
if [ -z "${1}" ]
	then
		echo "Debes pasar un argumento con el identificador del cliente"
		exit
fi

KEY_DIR=./keys/
OUTPUT_DIR=./files/
BASE_CONFIG=./base.conf

cat ${BASE_CONFIG} \
	<(echo -e '<ca>') \
	${KEY_DIR}ca.crt \
	<(echo -e '</ca>\n<cert>') \
	${KEY_DIR}${1}.crt \
	<(echo -e '</cert>\n<key>') \
	${KEY_DIR}${1}.key \
	<(echo -e '</key>\n<tls-auth>') \
	${KEY_DIR}ta.key \
	<(echo -e '</tls-auth>')  \
	> ${OUTPUT_DIR}${1}.ovpn
```

Le damos permisos

```
chmod 700 ~/client-configs/make_config.sh
```

Esta secuencia de comandos realizará una copia del archivo base.conf que creó, recopilará todos los archivos de certificados y claves que haya confeccionado para su cliente, extraerá el contenido de estos y los anexará a la copia del archivo de configuración de base, y exportará todo este contenido a un nuevo archivo de configuración de cliente. Esto significa que se evita la necesidad de administrar los archivos de configuración, certificado y clave del cliente por separado, y que toda la información necesaria se almacena en un solo lugar. El beneficio de esto es que, si alguna vez necesita agregar un cliente más adelante, puede simplemente ejecutar esta secuencia de comandos para crear de manera rápida el archivo de configuración y asegurarse de que toda la información importante se almacene en una sola ubicación de acceso sencillo.

Tenga en cuenta que siempre que agregue un nuevo cliente, deberá generar claves y certificados nuevos para poder ejecutar esta secuencia de comandos y generar su archivo de configuración. Podrá practicar con este comando en el siguiente paso.

# Paso 9: Generar las configuraciones de clientes

Si siguió la guía, creó un certificado y una clave de cliente llamados `client1.crt` y `client1.key`, respectivamente, en el paso 4. Puede generar un archivo de configuración para estas credenciales si se dirige al directorio `~/client-configs` y ejecuta la secuencia de comandos que realizó al final del paso anterior:


```
cd ~/client-configs
sudo ./make_config.sh client1
```

Con esto, se creará un archivo llamado client1.ovpn en su directorio ~/client-configs/files:

Debe transferir este archivo al dispositivo que planee usar como cliente. Por ejemplo, puede ser su computadora local o un dispositivo móvil.

Si bien las aplicaciones exactas empleadas para lograr esta transferencia dependerán del sistema operativo de su dispositivo y sus preferencias personales, un método seguro y confiable consiste en usar el protocolo de transferencia de archivos SSH (SFTP ) o la copia segura (SCP) en el backend. Con esto se transportarán los archivos de autenticación de VPN de su cliente a través de una conexión cifrada.

Aquí se ofrece un comando SFTP de muestra que usa el ejemplo client1.ovpn y que usted puede ejecutar desde su computadora local (macOS o Linux). Dispone el archivo .ovpn en su directorio de inicio:

## Desde la Máquina cliente
Nos conectaremos con las credenciales de la *máquina OPENVPN* y descargaremos el archivo `client1.ovpn`.
```
sftp user@192.168.100.197:client-configs/files/client1.ovpn ~/

```

# Paso 10: Instalar la configuración de cliente

## Linux

Si usa Linux, dispone de varias herramientas según su distribución. En su entorno de escritorio o gestor de ventanas también pueden incluirse utilidades de conexión.

Sin embargo, el método de conexión más universal consiste en simplemente usar el software OpenVPN.

Debian
```
sudo apt update
sudo apt install openvpn
```

Centos (Solamente si es centos)
```
sudo yum install epel-release
sudo yum install openvpn
```

Verifique si su distribución incluye una secuencia de comandos `/etc/openvpn/update-resolv-conf`:

```
ls /etc/openvpn
```

Si pudo encontrar un archivo `update-resolv-conf`, elimine los comentarios de las tres líneas que agregó para modificar los ajustes de DNS:

```
vim client1.ovpn
```

Descomente las líneas para que quede de la siguiente manera.
```
script-security 2
up /etc/openvpn/update-resolv-conf
down /etc/openvpn/update-resolv-conf
```

Si usa CentOS, cambie la directiva `group` de `nogroup` a `nobody` para que coincidan los grupos de distribución disponibles:

```
group nobody
```

Guarde y cierre el archivo.

Ahora, podrá conectarse a la VPN simplemente apuntando el comando openvpn hacia el archivo de configuración de cliente:

```
sudo openvpn --config client1.ovpn
```

## Otras plataformas


Ver el el paso 10 de:

https://www.digitalocean.com/community/tutorials/como-configurar-un-servidor-de-openvpn-en-ubuntu-18-04-es

# Paso 11: Probar la conexión de su VPN (opcional)


# Paso 12: Revocar certificados de clientes

# Extra
Para configurar más clientes, solo debe seguir los `pasos` `4` y `9` a `11` para cada dispositivo adicional. Para rechazar el acceso de los clientes, siga el `paso 12`.

# Referencias

Este es un tutorial basado en el artículo de [Digital Ocean](https://www.digitalocean.com/community/tutorials/como-configurar-un-servidor-de-openvpn-en-ubuntu-18-04-es). 

Sin embargo, la versión que se muestra en este repositorio contiene notas adicionales.