# Tutorial OpenVPN

Este tutorial muestra paso a paso la configuración y despliegué de un servidor OPENVPN sobre Debian 10.

Para esto, presentamos dos opciones:
1. Existe un `servidor CA` y un `servidor VPN`
    + Es una buena configuración dado que la CA no habita en el mismo server que la VPN, por lo que si alguien llega a vulnerar el servidor VPN no se comprometé directamente la CA.
    + Requiere dos máquinas
1. Existe `un solo servidor` que alberga ambos servicios
    + Requiere un solo servidor

## Opción de 2 servidores

+ [OpenVPN_AND_CA](https://github.com/elcaza/tutorial_openvpn/tree/master/OpenVPN_AND_CA). 

## Opción de 1 servidor

+ [OpenVPN_ONE_Server](https://github.com/elcaza/tutorial_openvpn/tree/master/OpenVPN_ONE_Server). 


# Referencias

Este es un tutorial basado en el artículo de [Digital Ocean](https://www.digitalocean.com/community/tutorials/como-configurar-un-servidor-de-openvpn-en-ubuntu-18-04-es). 

Sin embargo, la versión que se muestra en este repositorio contiene notas adicionales.