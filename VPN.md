# VPN: Virtual Private Network
### Configuración de openVPN basada en SSL/TLS
#### Acceso remoto
Un cliente se conecta a la VPN.

Configura una conexión VPN de acceso remoto entre dos equipos:
- Uno de los dos equipos (el que actuará como servidor) estará conectado a dos redes
- Para la autenticación de los extremos se usarán obligatoriamente certificados digitales, que se generarán utilizando openssl y se almacenarán en el directorio /etc/openvpn, junto con los parámetros Diffie-Helman y el certificado de la propia Autoridad de Certificación.
- Se utilizarán direcciones de la red 10.99.99.0/24 para las direcciones virtuales de la VPN. La dirección 10.99.99.1 se asignará al servidor VPN.
- Los ficheros de configuración del servidor y del cliente se crearán en el directorio /etc/openvpn de cada máquina, y se llamarán servidor.conf y cliente.conf respectivamente. La configuración establecida debe cumplir los siguientes aspectos:
- El demonio openvpn se manejará con systemctl.
- Se debe configurar para que la comunicación esté comprimida.
- La asignación de direcciones IP será dinámica.
- Existirá un fichero de log en el equipo.
- Se mandarán a los clientes las rutas necesarias.
- Tras el establecimiento de la VPN, la máquina cliente debe ser capaz de acceder a una máquina que esté en la otra red a la que está conectado el servidor.

Documenta el proceso detalladamente. Explica los distintos parámetros de los ficheros de configuración. ¿Qué es el parámetro Diffie-Helman?

### Preparación de la máquina
Se ha configurado una máquina que actuará como servidor y que tiene dos redes:
~~~
    vpnserv.vm.network :public_network,:bridge=>"enp2s0"
    vpnserv.vm.network :private_network, ip: "192.168.100.1", virtualbox__intnet:"mired1"
~~~

Se instala OpenVPN:
~~~
vagrant@servidor:~$ sudo apt install openvpn easy-rsa
~~~

### Creación de claves
Para la creación de claves se va a seguir el tutorial de la práctica sobre [certificados digitales](https://github.com/PalomaR88/Certificados_digitales-HTTPS-/blob/master/Certificados-digitales-HTTPS.md#tarea-1-certificado-autofirmado). Se debe crear una clave para la entidad certificadora y un certificado para esta. Además, una clave y un certificado para la VPN firmado por la autoridad cerfificadora.

También hay que crear la clave Diffie-Hellman con el parámetro **dhparam**:
~~~
vagrant@servidor:/etc/openvpn$ sudo openssl dhparam -out dhparams.pem 4096
~~~

Se edita el fichero **/etc/openvpn/servidor.com**:
~~~
#Dispositivo de túnel
dev tun
    
#Direcciones IP virtuales
server 10.99.99.0 255.255.255.0 

#subred local
push "route 192.168.100.0 255.255.255.0"

# Rol de servidor
tls-server

#Parámetros Diffie-Hellman
dh /etc/openvpn/dhparams.pem

#Certificado de la CA
ca /root/ca/certs/ca.cert.pem

#Certificado local
cert /root/ca/serverVPN.crt 

#Clave privada local
key /root/ca/serverVPN.key

#Activar la compresión LZO
comp-lzo

#Detectar caídas de la conexión
keepalive 10 60

#Nivel de información
verb 3
~~~




Y se edita el fichero de la siguiente forma:
~~~
port 1194 
proto udp
dev tun
ca /etc/openvpn/ca.crt
cert /etc/openvpn/ServerVPN.crt
key /etc/openvpn/ServerVPN.key
dh /etc/openvpn/dh2048.pem
server 10.99.99.0 255.255.255.0
ifconfig-pool-persist ipp.txt
push "route 192.168.100.0 255.255.255.0"
keepalive 10 120
comp-lzo
persist-key
persist-tun
status openvpn-status.log
log /var/log/openvpn.log
verb 4
~~~

Se activa el IP forwarding:
~~~
root@servidor:/var/log/openvpn# echo 1 > /proc/sys/net/ipv4/ip_forward
~~~

Y se activa la siguiente regla de IP tables:
~~~
iptables -A POSTROUTING -t nat -s 10.99.99.0/24 -o eth1 -j MASQUERADE
~~~

### Entidad certificadora

~~~
root@servidor:/etc/openvpn# ls
ca.key.pem  paloma.iesgn.org.key  serverVPN.cert.pem  VNS.conf
client	    server		  servidor.com	      vpn.conf
easy-rsa    server.conf		  update-resolv-conf
root@servidor:/etc/openvpn# openssl genrsa 4096 > VPN_Paloma.key
Generating RSA private key, 4096 bit long modulus (2 primes)
.............................................................................................................................++++
..........................................................................................................++++
e is 65537 (0x010001)
root@servidor:/etc/openvpn# chmod 600 VPN_Paloma.key 
root@servidor:/etc/openvpn# openssl req -new -key 
ca.key.pem            server/               update-resolv-conf
client/               server.conf           VNS.conf
easy-rsa/             serverVPN.cert.pem    vpn.conf
paloma.iesgn.org.key  servidor.com          VPN_Paloma.key
root@servidor:/etc/openvpn# openssl req -new -key VPN_Paloma.key -out VPN_Paloma.csr
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [AU]:ES
State or Province Name (full name) [Some-State]:Sevilla
Locality Name (eg, city) []:Dos Hermanas
Organization Name (eg, company) [Internet Widgits Pty Ltd]:IES Gonzalo Nazareno
Organizational Unit Name (eg, section) []:Paloma
Common Name (e.g. server FQDN or YOUR name) []:Paloma R.
Email Address []:palomagarciacampon08@gmail.com

Please enter the following 'extra' attributes
to be sent with your certificate request
A challenge password []:paloma
An optional company name []:
~~~

Se crea el fichero **/etc/openvpn/servidor.com** con la siguiente configuración:
~~~
#Dispositivo de túnel
dev tun
    
#Direcciones IP virtuales
server 10.99.99.0 255.255.255.0 

#subred local
push "route 192.168.100.0 255.255.255.0"

# Rol de servidor
tls-server

#Parámetros Diffie-Hellman
dh /etc/openvpn/dh1024.pem

#Certificado de la CA
ca /etc/openvpn/ca.cert.pem

#Certificado local
cert /etc/openvpn/server.crt

#Clave privada local
key /etc/openvpn/server.key

#Activar la compresión LZO
comp-lzo

#Detectar caídas de la conexión
keepalive 10 60

#Nivel de información
verb 3
~~~

Para crear el certificado se va a seguir la guía de la Práctica de [HTTPS](https://github.com/PalomaR88/Certificados_digitales-HTTPS-/blob/master/Certificados-digitales-HTTPS.md#tarea-1-certificado-autofirmado).









#### Site to site
Una red con un router, otra red con otro router y se van a conectar los dos equipos a la VPN.

Configura una conexión VPN sitio a sitio entre dos equipos del cloud:

- Cada equipo estará conectado a dos redes, una de ellas en común
- Para la autenticación de los extremos se usarán obligatoriamente certificados digitales, que se generarán utilizando openssl y se almacenarán en el directorio /etc/openvpn, junto con con los parámetros Diffie-Helman y el certificado de la propia Autoridad de Certificación.
- Se utilizarán direcciones de la red 10.99.99.0/24 para las direcciones virtuales de la VPN.
- Tras el establecimiento de la VPN, una máquina de cada red detrás de cada servidor VPN debe ser capaz de acceder a una máquina del otro extremo.

Documenta el proceso detalladamente.

