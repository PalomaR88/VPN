# VPN: Virtual Private Network
## Configuración de openVPN basada en SSL/TLS
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

### Creación del escenario
Se han configurado dos máquinas, una como cliente y otra como servidor. El servidor con dos redes, una de ellas interna que comparte con el cliente, que se llamará cliente-int. Esta es la configuración de [Vagrantfile](https://github.com/PalomaR88/VPN/blob/master/Vagrantfile).

Además, se configura otro cliente que será quien se conectará a la VPN, que se llamará cliente-ext con la siguiente configuración en el Vagrantfile:
~~~
Vagrant.configure("2") do |config|

  config.vm.define :cliente1 do |cliente1|
    cliente1.vm.box = "debian/buster64"
    cliente1.vm.hostname = "Cliente1"
    cliente1.vm.network :public_network,:bridge=>"enp2s0"
  end
end
~~~


### Servidor
##### Instalación de OpenVPN
~~~
vagrant@servidor:~$ sudo apt install openvpn
~~~

##### Creación de claves
Para la creación de claves se va a seguir el tutorial de la práctica sobre [certificados digitales](https://github.com/PalomaR88/Certificados_digitales-HTTPS-/blob/master/Certificados-digitales-HTTPS.md#tarea-1-certificado-autofirmado). Se debe crear una clave para la entidad certificadora y un certificado para esta. Además, una clave y un certificado para la VPN firmado por la autoridad cerfificadora.

También hay que crear la clave Diffie-Hellman con el parámetro **dhparam**:
~~~
vagrant@servidor:/etc/openvpn$ sudo openssl dhparam -out dhparams.pem 4096
~~~

##### Configuración en el servidor
Se activa el bit de forward:
~~~
root@servidor:/home/vagrant# echo 1 > /proc/sys/net/ipv4/ip_forward
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
ca /etc/openvpn/ca.cert.pem

#Certificado local
cert /etc/openvpn/serverVPN.crt 

#Clave privada local
key /etc/openvpn/serverVPN.key

#Activar la compresión LZO
comp-lzo

#Detectar caídas de la conexión
keepalive 10 60

#Nivel de información
verb 3
~~~

Para firmar .csr del cliente se utiliza el siguiente comando:
~~~
root@servidor:/vagrant# openssl x509 -req -in VPN_Alejandro.csr -CA /root/ca/certs/ca.cert.pem -CAkey /root/ca/private/ca.key.pem -CAcreateserial -out VPN_Alejandro.crt
Signature ok
subject=C = ES, ST = Sevilla, L = Dos Hermanas, O = IES Gonzalo Nazareno, CN = Alejandro Morales Gracia
Getting CA Private Key
Enter pass phrase for /root/ca/private/ca.key.pem:
~~~

Se configura el fichero **/etc/openvpn/contra.txt** para contener la contraseña de nuestra clave privada y en servidor.conf se añade:
~~~
askpass contra.txt
~~~

Además, se descomenta la siguiente línea del fichero **/etc/default/openvpn**:
~~~
AUTOSTART="all"
~~~

Y se reinicia el servicio:
~~~
root@servidor:/etc/openvpn# systemctl start openvpn@servidor
~~~

### Cliente-ext
Se instala openvpn y se crean los datos iguales que la autoridad certificadora
~~~
Country Name (2 letter code) [AU]:ES
State or Province Name (full name) [Some-State]:Sevilla
Locality Name (eg, city) []:Dos Hermanas
Organization Name (eg, company) [Internet Widgits Pty Ltd]:IES Gonzalo Nazareno
Organizational Unit Name (eg, section) []:Paloma 
Common Name (e.g. server FQDN or YOUR name) []:Alejandro Morales Gracia
Email Address []:ale95mogra@gmail.com	
~~~

Pasamos el fichero `VPN_Alejandro.csr` a la Autoridad Certificadora y esperamos a que nos devuelva el certificado firmado y el certificado de CA.

Ademas vamos a mover los certificados y la clave privada a la ruta `/etc/openvpn/client`

Una vez que tenemos los dos certificados, tenemos que hacer el fichero de configuración del cliente, donde vamos a indicar la interfaz `tun`, la dirección a la que se tiene que conectar y los certificados.

~~~
nano VPN.conf
~~~

~~~
dev tun
remote 172.22.0.56
ifconfig 10.99.99.0 255.255.255.0
pull
tls-client
ca /etc/openvpn/client/ca.cert.pem
cert /etc/openvpn/client/VPN_Alejandro.crt
key /etc/openvpn/client/VPN_Alejandro.key
comp-lzo
keepalive 10 60
verb 3
askpass contra.txt
~~~

Se crea el fichero contra.txt donde se introduce la contraseña y se descomenta la siguiente línea del fichero **/etc/default/openvpn**:
~~~
AUTOSTART="all"
~~~

Tras reiniciar el servicio se comprueba que se ha añadido el tun:
~~~
ip a
    1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
        link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
        inet 127.0.0.1/8 scope host lo
           valid_lft forever preferred_lft forever
        inet6 ::1/128 scope host 
           valid_lft forever preferred_lft forever
    2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
        link/ether 08:00:27:8d:c0:4d brd ff:ff:ff:ff:ff:ff
        inet 10.0.2.15/24 brd 10.0.2.255 scope global dynamic eth0
           valid_lft 85611sec preferred_lft 85611sec
        inet6 fe80::a00:27ff:fe8d:c04d/64 scope link 
           valid_lft forever preferred_lft forever
    3: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
        link/ether 08:00:27:b8:70:44 brd ff:ff:ff:ff:ff:ff
        inet 172.22.0.163/16 brd 172.22.255.255 scope global dynamic eth1
           valid_lft 1029sec preferred_lft 1029sec
        inet6 fe80::a00:27ff:feb8:7044/64 scope link 
           valid_lft forever preferred_lft forever
    4: tun0: <POINTOPOINT,MULTICAST,NOARP,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UNKNOWN group    default qlen 100
        link/none 
        inet 10.99.99.6 peer 10.99.99.5/32 scope global tun0
           valid_lft forever preferred_lft forever
        inet6 fe80::9e0e:4d6c:ad95:ab79/64 scope link stable-privacy 
           valid_lft forever preferred_lft forever
~~~

### Cliente-int
Se crea una nueva tabla de enrutamiento para este cliente:
~~~
sudo ip r del default
sudo ip r add default via 192.168.100.20
~~~

### Comprobación
Desde cliente-ext realizamos un ping a la dirección `192.168.100.2` que es la red interna del servidor.
~~~
ping 192.168.100.2
    PING 192.168.100.2 (192.168.100.2) 56(84) bytes of data.
    64 bytes from 192.168.100.2: icmp_seq=1 ttl=64 time=1.28 ms
    64 bytes from 192.168.100.2: icmp_seq=2 ttl=64 time=1.43 ms
    64 bytes from 192.168.100.2: icmp_seq=3 ttl=64 time=1.34 ms
    64 bytes from 192.168.100.2: icmp_seq=4 ttl=64 time=0.970 ms
    64 bytes from 192.168.100.2: icmp_seq=5 ttl=64 time=1.39 ms
    ^C
    --- 192.168.100.2 ping statistics ---
    5 packets transmitted, 5 received, 0% packet loss, time 12ms
    rtt min/avg/max/mdev = 0.970/1.280/1.425/0.166 ms
~~~



## Site to site
Una red con un router, otra red con otro router y se van a conectar los dos equipos a la VPN.

Configura una conexión VPN sitio a sitio entre dos equipos del cloud:

- Cada equipo estará conectado a dos redes, una de ellas en común
- Para la autenticación de los extremos se usarán obligatoriamente certificados digitales, que se generarán utilizando openssl y se almacenarán en el directorio /etc/openvpn, junto con con los parámetros Diffie-Helman y el certificado de la propia Autoridad de Certificación.
- Se utilizarán direcciones de la red 10.99.99.0/24 para las direcciones virtuales de la VPN.
- Tras el establecimiento de la VPN, una máquina de cada red detrás de cada servidor VPN debe ser capaz de acceder a una máquina del otro extremo.

Documenta el proceso detalladamente.

### Presparación del escenario
Se añade una nueva máquina cliente en Vagrantfile que comparta red con la máquina cliente-ext que se ha usado en el ejercicio anterior:
~~~
# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|

  config.vm.define :vpnserv do |vpnserv|
    vpnserv.vm.box = "debian/buster64"
    vpnserv.vm.hostname = "servidor"
    vpnserv.vm.network :public_network,:bridge=>"enp2s0"
    vpnserv.vm.network :private_network, ip: "192.168.200.1", virtualbox__intnet:"mired1"
  end
  config.vm.define :vpnsts do |vpnsts|
    vpnsts.vm.box = "debian/buster64"
    vpnsts.vm.hostname = "cliente"
    vpnsts.vm.network :private_network, ip: "192.168.200.2", virtualbox__intnet:"mired1"
  end
end
~~~

### Servidor
El fichero servidor.conf se configura de la siguiente forma:
~~~
dev tun
ifconfig 10.99.99.1 10.99.99.2
router 192.168.100.0 255.255.255.0
tls-server
dh /etc/openvpn/dhparams.pem
ca /etc/openvpn/ca.cert.pem
cert /etc/openvpn/serverVPN.crt
key /etc/openvpn/serverVPN.key
comp-lzo
keepalive 10 60
log /var/log/server.log
verb 6
askpass contra.txt
~~~

Y se crea el tunel:
~~~
root@servidor:/etc/openvpn# systemctl restart openvpn
~~~

Se comprueba que se ha creado correctamente:
~~~
vagrant@servidor:~$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 08:00:27:8d:c0:4d brd ff:ff:ff:ff:ff:ff
    inet 10.0.2.15/24 brd 10.0.2.255 scope global dynamic eth0
       valid_lft 86258sec preferred_lft 86258sec
    inet6 fe80::a00:27ff:fe8d:c04d/64 scope link 
       valid_lft forever preferred_lft forever
3: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 08:00:27:8b:69:ac brd ff:ff:ff:ff:ff:ff
    inet 192.168.43.202/24 brd 192.168.43.255 scope global dynamic eth1
       valid_lft 3472sec preferred_lft 3472sec
    inet6 fe80::a00:27ff:fe8b:69ac/64 scope link 
       valid_lft forever preferred_lft forever
4: eth2: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 08:00:27:c4:00:bf brd ff:ff:ff:ff:ff:ff
    inet 192.168.100.1/24 brd 192.168.100.255 scope global eth2
       valid_lft forever preferred_lft forever
    inet6 fe80::a00:27ff:fec4:bf/64 scope link 
       valid_lft forever preferred_lft forever
7: tun0: <POINTOPOINT,MULTICAST,NOARP,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UNKNOWN group default qlen 100
    link/none 
    inet 10.99.99.1 peer 10.99.99.2/32 scope global tun0
       valid_lft forever preferred_lft forever
    inet6 fe80::7b3:e9b8:59fb:925/64 scope link stable-privacy 
       valid_lft forever preferred_lft forever
~~~


### Servidor/Cliente1
Con los ficheros .key, .crt y .pem correspondientes en la máquina, los mismo que se usaron para el ejercicio anterior, se crea el fichero **/etc/openvpn/cliente.conf**:
Antes de configurar el cliente, el servidor tiene que haberle pasado al cliente los fichero VPN_Alejandro.key, VPN_Alejandro.crt y ca.cert.pem como en la anterior práctica.
~~~
dev tun
ifconfig 10.99.99.2 10.99.99.1
remote 172.22.0.56
route 192.168.100.0 255.255.255.0
tls-client
ca /etc/openvpn/client/ca.cert.pem
cert /etc/openvpn/client/VPN_Alejandro.crt
key /etc/openvpn/client/VPN_Alejandro.key
comp-lzo
keepalive 10 60
log /var/log/host.log
verb 6
askpass pass.txt
~~~

Se activa el bit de forward para dejar pasar el tráfico al Cliente1:
~~~
root@servidor:/home/vagrant# echo 1 > /proc/sys/net/ipv4/ip_forward
~~~

Y se reinicia el servicio
~~~
sudo systemctl restart openvpn
~~~

Comprobación:
~~~
ip a
    1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
        link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
        inet 127.0.0.1/8 scope host lo
           valid_lft forever preferred_lft forever
        inet6 ::1/128 scope host 
           valid_lft forever preferred_lft forever
    2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
        link/ether 08:00:27:8d:c0:4d brd ff:ff:ff:ff:ff:ff
        inet 10.0.2.15/24 brd 10.0.2.255 scope global dynamic eth0
           valid_lft 84637sec preferred_lft 84637sec
        inet6 fe80::a00:27ff:fe8d:c04d/64 scope link 
           valid_lft forever preferred_lft forever
    3: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
        link/ether 08:00:27:b8:70:44 brd ff:ff:ff:ff:ff:ff
        inet 172.22.2.123/16 brd 172.22.255.255 scope global dynamic eth1
           valid_lft 8996sec preferred_lft 8996sec
        inet6 fe80::a00:27ff:feb8:7044/64 scope link 
           valid_lft forever preferred_lft forever
    4: eth2: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
        link/ether 08:00:27:7c:e2:d3 brd ff:ff:ff:ff:ff:ff
        inet 192.168.200.1/24 brd 192.168.200.255 scope global eth2
           valid_lft forever preferred_lft forever
        inet6 fe80::a00:27ff:fe7c:e2d3/64 scope link 
           valid_lft forever preferred_lft forever
    21: tun0: <POINTOPOINT,MULTICAST,NOARP,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UNKNOWN group default qlen 100
        link/none 
        inet 10.99.99.2 peer 10.99.99.1/32 scope global tun0
           valid_lft forever preferred_lft forever
        inet6 fe80::83d:90fb:c65e:93d6/64 scope link stable-privacy 
           valid_lft forever preferred_lft forever

ip r
    default via 10.0.2.2 dev eth0 
    10.0.2.0/24 dev eth0 proto kernel scope link src 10.0.2.15 
    10.99.99.1 dev tun0 proto kernel scope link src 10.99.99.2 
    172.22.0.0/16 dev eth1 proto kernel scope link src 172.22.2.123 
    192.168.200.0/24 dev eth2 proto kernel scope link src 192.168.200.1 
~~~

Se hace ping en desde el cliente/servidor al servidor y cliente contrarios:
~~~
vagrant@Servidor/Cliente1:/etc/openvpn$ ping 10.99.99.1
    PING 10.99.99.1 (10.99.99.1) 56(84) bytes of data.
    64 bytes from 10.99.99.1: icmp_seq=1 ttl=64 time=2.85 ms
    64 bytes from 10.99.99.1: icmp_seq=2 ttl=64 time=3.33 ms
    64 bytes from 10.99.99.1: icmp_seq=3 ttl=64 time=4.24 ms
    ^C
    --- 10.99.99.1 ping statistics ---
    3 packets transmitted, 3 received, 0% packet loss, time 5ms
    rtt min/avg/max/mdev = 2.851/3.473/4.240/0.576 ms

vagrant@Servidor/Cliente1:/etc/openvpn$ ping 192.168.100.2
    PING 192.168.100.2 (192.168.100.2) 56(84) bytes of data.
    64 bytes from 192.168.100.2: icmp_seq=1 ttl=63 time=3.37 ms
    64 bytes from 192.168.100.2: icmp_seq=2 ttl=63 time=8.10 ms
    64 bytes from 192.168.100.2: icmp_seq=3 ttl=63 time=20.4 ms
    ^C
    --- 192.168.100.2 ping statistics ---
    3 packets transmitted, 3 received, 0% packet loss, time 4ms
    rtt min/avg/max/mdev = 3.367/10.631/20.428/7.191 ms
~~~

### Cliente
Se modifica la tabla de enrutamiento:
~~~
vagrant@cliente:~$ sudo ip r del default
vagrant@cliente:~$ sudo ip r add default via 192.168.100.1
~~~


### Comprobación
**Desde la máquina servidor** 
- A la máquina cliente/servidor:
~~~
vagrant@servidor:/etc/openvpn$ ping 192.168.200.1
PING 192.168.200.1 (192.168.200.1) 56(84) bytes of data.

64 bytes from 192.168.200.1: icmp_seq=1 ttl=64 time=57.1 ms
64 bytes from 192.168.200.1: icmp_seq=2 ttl=64 time=51.6 ms
64 bytes from 192.168.200.1: icmp_seq=3 ttl=64 time=56.8 ms
^C
--- 192.168.200.1 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 5ms
rtt min/avg/max/mdev = 51.572/55.153/57.053/2.548 ms
~~~

- Al cliente de la máquina cliente/servidor:
~~~
vagrant@servidor:/etc/openvpn$ ping 192.168.200.2
PING 192.168.200.2 (192.168.200.2) 56(84) bytes of data.
64 bytes from 192.168.200.2: icmp_seq=1 ttl=63 time=56.6 ms
64 bytes from 192.168.200.2: icmp_seq=2 ttl=63 time=47.9 ms
^C
--- 192.168.200.2 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 7ms
rtt min/avg/max/mdev = 47.893/52.252/56.612/4.365 ms
~~~

**Desde el cliente**
- A la máquina cliente/servidor:
~~~
vagrant@cliente:~$ ping 192.168.200.1
PING 192.168.200.1 (192.168.200.1) 56(84) bytes of data.
64 bytes from 192.168.200.1: icmp_seq=1 ttl=63 time=13.1 ms
^C64 bytes from 192.168.200.1: icmp_seq=2 ttl=63 time=13.2 ms

--- 192.168.200.1 ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 398ms
rtt min/avg/max/mdev = 13.116/13.159/13.202/0.043 ms
~~~

- Al cliente de la máquina cliente/servidor:
~~~
vagrant@cliente:~$ ping 192.168.200.2
PING 192.168.200.2 (192.168.200.2) 56(84) bytes of data.
64 bytes from 192.168.200.2: icmp_seq=1 ttl=62 time=15.1 ms
64 bytes from 192.168.200.2: icmp_seq=2 ttl=62 time=103 ms
64 bytes from 192.168.200.2: icmp_seq=3 ttl=62 time=319 ms
^C
--- 192.168.200.2 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 16ms
rtt min/avg/max/mdev = 15.119/145.758/318.835/127.571 ms
~~~
