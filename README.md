# Lab-Vpn-Wireguard-Linux
## Laboratorio hecho en GNS3 de implementación VPN Wireguard en Ubuntu 20.04
![wireguard](https://user-images.githubusercontent.com/88456338/129817228-e0bef97f-d088-4b67-b82d-404dff316be6.png)

WireGuard VPN es una aplicación software completamente gratuita que nos permitirá establecer túneles VPN. Este completo software incorpora todos los protocolos de comunicación y criptografía necesarios, para levantar una red privada virtual entre varios clientes y un servidor. 
Características de WireGuard VPN

WireGuard VPN es un software para crear una red privada virtual (VPN) extremadamente sencilla de configurar, muy rápida (más rápida que IPsec y OpenVPN) y que utiliza la criptografía más moderna por defecto, sin necesidad de seleccionar entre diferentes algoritmos de cifrado simétrico, asimétrico y de hashing. El objetivo de WireGuard VPN es convertirse en un estándar, y que más usuarios domésticos y empresas comiencen a utilizarlo, en lugar de usar IPsec o el popular OpenVPN.  Se ejecuta sobre el protocolo UDP, por lo tanto es considerado un túnel de capa 3.

## Topologia de la red de laboratorio GNS3
![topologia](https://user-images.githubusercontent.com/88456338/129826735-4ce0056f-678e-4f70-bca1-fffe3519ebfd.png)

La topología de este laboratorio VPN está compuesta por la "parte" rosada que simula ser Internet, con sus respectivas Ip's públicas y los extremos verde y azul, que simulan ser redes lan que salen a Internet a través de NAT. La LAN azul tiene como borde un Router y un Windows 10 cliente. Del otro lado, la LAN verde tiene como Router un Linux, que también será el servidor VPN.

## Prueba de conectividad

Como es de esperar, desde la LAN azul no hago ping hacia la LAN verde, ni tampoco puedo acceder a sus recursos. Para lograr dicha conectividad se necesita una VPN, qué es lo que voy a configurar

### Ping desde el Windows 10 en la lan AZUL a Internet:

![capture-20210818-003211](https://user-images.githubusercontent.com/88456338/129832678-1ea791b3-7ee0-4635-a18f-0fd7f1f28878.png)

Hacia internet hay conectividad, pero no hacia la otra lan

![capture-20210818-003415](https://user-images.githubusercontent.com/88456338/129832772-024291fa-1de1-4df5-a9d9-121b0c0c792e.png)

### Ping desde el VPC de la lan VERDE a Internet

![capture-20210818-010231](https://user-images.githubusercontent.com/88456338/129835214-84a66a55-cd5e-43b4-a54c-57c1db25b8f2.png)


Este VPC sale a internet a través del Linux, que está siendo utilizado como Router, para habilitar esto hay que ingresar el siguiente comando 
`sysctl -w net.ipv4.ip_forward=1` y también tener habilitado NAT, para que todo lo que salga por la interface enp0s3 salga enmascarada, para esto se utiliza el siguiente comando 
`iptables -t nat -A POSTROUTING -o enp0s3 -j MASQUERADE`

## Servidor Ubuntu 20.04 VPN

Desde el kernel 5.4 Wireguard se encuentra en los repositorios de Linux en su versión estable. Concretamente en la versión 20.04 de Ubuntu ya disponemos de este kernel. Para instalar Wireguard, se ejecuta el siguiente comando `sudo apt install wireguard`


Ya con Wireguard instalado procederemos a preparar la interfaz. Para ello nos moveremos al directorio /etc/wireguard:

`cd /etc/wireguard/`

Aquí generaremos nuestro par de claves pública y privada de la siguiente manera:

`wg genkey | tee servidor_private.key | wg pubkey > servidor_public.key`

![capture-20210818-011041](https://user-images.githubusercontent.com/88456338/129835686-c6cec783-fe7c-492d-ba81-294a7f166913.png)

### Crearemos el archivo de configuración:

`touch wg0.conf`

Desde la ruta en la que estamos /etc/wireguard, vamos a copiar y a pegar desde la línea de comandos nuestra clave privada

`cat servidor_private.key >> wg0.conf`

![capture-20210818-011617](https://user-images.githubusercontent.com/88456338/129836100-2b30618c-6481-4183-897e-b489d0ade5d5.png)

### Edición el archivo wg0.conf

Procedemos a editar el archivo de configuración. Utilizaré vim

`vim wg0.conf`

Se abrirá el editor y nuestra clave privada ya está dentro, la pegamos con el comando `cat`. Ahora editaremos el archivo y lo dejaremos así:
~~~
[Interface]

Address = 10.0.0.1
PrivateKey = Pegada_anteriormente
ListenPort = 51820
PostUp = iptables -A FORWARD -i %i -j ACCEPT; iptables -A FORWARD -o %i -j ACCEPT; iptables -t nat -A POSTROUTING -o enp0s3 -j MASQUERADE
PostDown = iptables -D FORWARD -i %i -j ACCEPT; iptables -D FORWARD -o %i -j ACCEPT; iptables -t nat -D POSTROUTING -o enp0s3 -j MASQUERADE
~~~

- “Address” es la dirección en la VPN, se puede poner cualquier direccion, siempre que no esté ocupada. Esta dirección es la que crea la red VPN, cada cliente deberá tener una distinta es decir, si el servidor es la 10.0.0.1 el cliente será la 10.0.0.2.
- “PrivateKey” es la clave privada, ya estaba copiada y pegada previamente.
- “ListenPort” es el puerto donde va a trabajar Wireguard. Importante, por defecto 51820 UDP es el puerto en el que trabaja esta VPN, pero puede ser cualquier otro. 
- “PostUp y PostDown” son las reglas del firewall. En este punto hay que fijarse bien que interface es la que estamos utilizando, en este caso es enp0s3

![capture-20210818-014358](https://user-images.githubusercontent.com/88456338/129838329-10cbb2d7-6a18-43f4-a71c-8244fa498046.png)


### Activacion de Wireguard para que inicie con el sistema, si hay un error en el archivo de configuracion, nos indicará en este punto

`systemctl enable wg-quick@wg0`

`systemctl start wg-quick@wg0`

`systemctl status wg-quick@wg0`

![capture-20210818-014056](https://user-images.githubusercontent.com/88456338/129838246-ed313917-b644-47af-a4e5-fec419e36728.png)


Necesitamos tener habilitado el forwarding:

`sysctl -w net.ipv4.ip_forward=1`


## Configuracion en el cliente Windows 10

Desde nuestro equipo Windows, nos vamos a la web de Wireguard y descargamos el programa para Windows. Lo instalamos y le damos permisos de administrador. A continuación en **Add Tunnel** presionamos sobre **Add empty tunnel** y rellenamos. Las claves pública y privada ya nos las autogenera el propio programa. Tenemos que poner la llave publica del servidor, quedando asi:

![capture-20210818-021907](https://user-images.githubusercontent.com/88456338/129841597-d1f94f80-96ab-4c56-b7dc-b14383910b96.png)


## Configurando el servidor para añadir el cliente Windows.

Nuevamente dentro de nuestro servidor Linux y en la ruta `/etc/wireguard/` vamos a modificar el archivo `wg0.conf`. Así:

Añadimos debajo de *Interface* el apartado *Peer* y queda de esta manera:
~~~
[Interface]
Address = 10.0.0.1
PrivateKey = Aquí va nuestra clave privada
ListenPort = 51820
PostUp = iptables -A FORWARD -i %i -j ACCEPT; iptables -A FORWARD -o %i -j ACCEPT; iptables -t nat -A POSTROUTING -o enp0s3 -j MASQUERADE
PostDown = iptables -D FORWARD -i %i -j ACCEPT; iptables -D FORWARD -o %i -j ACCEPT; iptables -t nat -D POSTROUTING -o enp0s3 -j MASQUERADE

[Peer]
Publickey = LA CLAVE PÚBLICA QUE GENERÓ TU CLIENTE WINDOWS
AllowedIPs = 10.0.0.2/32
PersistentKeepAlive = 25
~~~

El archivo `wg0.conf` quedaría de esta forma, con el peer agregado

![capture-20210818-022246](https://user-images.githubusercontent.com/88456338/129841870-70153334-16a2-42a4-87ae-ed8443cd8e42.png)

## Activacion de la VPN y prueba de conectividad

En el equipo Windows, le damos a Activate y probamos la VPN

![capture-20210818-022504](https://user-images.githubusercontent.com/88456338/129842046-b73f6ac4-32a5-4e55-80b9-00f2fbb0df83.png)

