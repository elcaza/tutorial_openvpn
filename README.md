# Tutorial OpenVPN

Este tutorial muestra paso a paso la configuración y despliegué de un servidor OPENVPN sobre Debian 10.

OpenVPN trabaja con certificados, por lo que es necesario crear una CA (Certificate Authority). Hay dos principales formas de crear nuestros servicio. 

1. Dos servidiores (Server CA y Server VPN)
    + Es una buena configuración dado que la CA no habita en el mismo servidor que la VPN, por lo que si alguien llega a vulnerar el servidor VPN no se comprometé directamente la CA.
    + Requiere dos máquinas
1. Un solo servidor que alberga ambos servicios
    + Requiere un solo servidor
    + Se puede proteger la CA mediante un usuario y comprimiendo la carpeta con opciones de password para mantenerla segura.

## Opción de 2 servidores

+ [OpenVPN_AND_CA](https://github.com/elcaza/tutorial_openvpn/tree/master/OpenVPN_AND_CA). 

## Opción de 1 servidor

+ [OpenVPN_ONE_Server](https://github.com/elcaza/tutorial_openvpn/tree/master/OpenVPN_ONE_Server). 

# Comandos útiles para la transferencia de archivos

Login al servidor con llaves .pem

```
ssh -i "server_vpn.pem" user@IP_OR_DNS
```

Login al servidor ssh

```
ssh user@IP_OR_DNS
```

Descarga de archivos mediante llave .pem

```
sftp -i "server_vpn.pem" user@IP_OR_DNS:ruta_al/archivo.ovpn ~/destino_de_descarga
```

Subir archivos mediante  scp 

```
scp ruta_archivo_/client1.ovpn user@IP_OR_DNS:~/ruta_destino
```


# Referencias

Este es un tutorial basado en el artículo de [Digital Ocean](https://www.digitalocean.com/community/tutorials/como-configurar-un-servidor-de-openvpn-en-ubuntu-18-04-es). 

Sin embargo, la versión que se muestra en este repositorio contiene notas adicionales.