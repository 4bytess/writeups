# MIRAME - DOCKERLABS

## INFORMACIÓN Y DESPLIEGUE DE LA MÁQUINA

Esta máquina es de [DockerLabs](https://dockerlabs.es)

Para descargarla vas a [DockerLabs](https://dockerlabs.es) y buscas la máquina Mirame. Abres el enlace de mega, descargas el zip, lo descomprimes y ejecutas `bash ./auto_deploy.sh ./mirame.tar`. De esta forma se ejecutará el contendor de docker de la máquina.

### descripción

- Dificultad: fácil
- Sistema Operativo: Linux
- Autor: maciiii___

## RECONOCIMIENTO DE RED

### conectividad y SO

Antes de comenzar, comprobamos que tenemos conectividad con la máquina usando la herramienta ping: `ping -c 1 172.17.0.2`.
> `-c 1`: solo enviar 1 paquete ICMP

![image](https://github.com/user-attachments/assets/fc6c2d67-43fc-46ad-aaad-9b91530ca829)

Vemos que recibimos respuesta, así que tenemos conectividad. Como el TTL de la respuesta es 64, muy probablemente el Sistema Operativo de la máquina sea Linux.

### puertos y servicios

Hacemos un escaneo de todo el rango de puertos con nmap: `nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 172.17.0.2 -oG tcp_ports`.
> `-p-`: todo el rango de puertos

> `--open`: solo reportar los puertos abiertos

> `-sS`: usar el método de escaneo SYN, que es más sigiloso y rápido que el normal. Requiere privilegios a nivel de sistema para enviar raw packets

> `--min-rate 5000` enviar como mínimo 5000 paquetes por segundo

> `-vvv` triple verbose. Nada más descubrir un puerto lo muestra por pantalla, también muestra más información de lo normal sobre el escaneo

> `-n` no hacer resolución DNS

> `-Pn` no hacer host discovery

> `-oG tcp_ports` exportar el output en formato grepeable al archivo `tcp_ports`

#### puertos abiertos

**22 - TCP - SSH**

**80 - TCP - HTTP**

Hacemos un escaneo de servicios y versiones con nmap sobre los puertos 22 y 80: `nmap -p22,80 -sCV 172.17.0.2 -oN tcp_ports_targeted`.
> `-p22,80` hacer el escaneo sobre los puertos 22 y 80.

> `-sCV` escanear el servicio y la versión (`sV`) y lanzar los scripts de reconocimiento por defecto (`sC`)

> `-oN tcp_ports_targeted` exportar el output en formato nmap al archivo `tcp_ports_targeted`

### RECONOCIMIENTO WEB

Primero desde consola hacemos `whatweb http://172.17.0.2`.

![image](https://github.com/user-attachments/assets/88dc408e-30f1-47fb-8fba-b80e3414a213)

Vemos que el título nos avisa de que hay una página de login.

Cargamos el servidor web desde el navegador:

![image](https://github.com/user-attachments/assets/a7b5851d-8364-45fc-803f-c9be1ed057dd)

Vemos el panel de login. Si probamos a poner una comilla, sale un error de la base de datos MariaDB:

![image](https://github.com/user-attachments/assets/ef94c9b3-65bd-4397-9521-16b16916c4ef)

Esto nos indica que el panel es vulnerable a SQLi.

### EXPLOTACIÓN WEB

Si probamos a poner en el campo de usuario `admin` y en el de contraseña `' or 1=1-- -` conseguimos bypassear el login, y nos redirige a `page.php`. Pero no nos asigna ninguna cookie de sesión, por lo que podemos acceder a `page.php` cuando queramos. Podemos seguir explotando la SQLi para dumpear la información útil que haya dentro del servidor MariaDB. Nos creamos un script en python3 que automatize el dumpeado de todos los datos: [script_SQLi](https://github.com/4bytess/dockerlabs-scripts/tree/main/mirame/SQLi). Lo ejecutamos:

![image](https://github.com/user-attachments/assets/977fe507-d8e7-43bb-a2eb-de538ea06f25)

Vemos varias credenciales almacenadas en la tabla usuarios de la base de datos users. Si probamos todas las credenciales en el panel de login web, todas simplemente nos redirigen a `page.php`. Como hay una que se llama directoriotravieso, podemos probar a ver si hay un directorio llamado así en la web:

![image](https://github.com/user-attachments/assets/f830a41b-4b85-4189-803e-0620441ea7ac)

Vemos que existe. Tiene solo una imagen JPG:

![image](https://github.com/user-attachments/assets/5d342819-f5a5-4d3e-a1bb-6a3fadc8d170)

### ESTEGANOGRAFÍA Y ZIP

En sí, la imagen no tiene a simple vista información interesante. Pero podemos descargarla con `wget http://172.17.0.2/directoriotravieso/miramebien.jpg`. Una vez la hemos descargado, intentamos sacar información escondida de la imagen: `steghide extract -sf miramebien.jpg`:

![image](https://github.com/user-attachments/assets/06c0e40c-20cd-44ed-bf96-1d50e3e9ac9b)

Pero nos pide una contraseña. Podemos usar stegcracker para hacer fuerza bruta de contraseñas. `stegcracker ./miramebien.jpg` (stegcracker usa el rockyou.txt por defecto):

![image](https://github.com/user-attachments/assets/ee246490-1c7d-4ba3-bdea-bbc81c14c954)

Encuntra la contraseña: chocolate. Ahora hacemos `steghide extract -sf ./miramebien.jpg` usando esa contraseña:

![image](https://github.com/user-attachments/assets/b63feda7-4eac-48d0-bbac-5efb6fbb9f02)

Y al usar esa contraseña se crea un archivo zip llamado ocultito.zip. Si lo intentamos descomprimir nos pide contraseña. Podemos hacer fuerza bruta usando [zip_bruteforce.sh](https://github.com/4bytess/zip_bruteforce): `zip_bruteforce.sh /usr/share/wordlists/rockyou.txt ./ocultito.zip`:

![image](https://github.com/user-attachments/assets/d8ad3ca2-4130-4b4e-9123-bc1cb25cf75b)

Y nos encuentra la contraseña: stupid1. Si ahora descomprimimos el zip con esta contraseña, se nos crea otro archivo llamado secret.txt, que contiene las credenciales carlos:carlitos. Si las probamos por ssh funcionan.

### ESCALADA DE PRIVILEGIOS (ROOT)

Siendo el usuario carlos, podemos buscar desde la raíz archivos con permisos SUID: `find / -perm -4000 2>/dev/null`:

![image](https://github.com/user-attachments/assets/de8ba7c8-381b-4222-b18c-c8094e5ecbae)

Vemos que el binario `/usr/bin/find`, con el propio que hemos hecho este escaneo, tiene permisos SUID. Entonces con `find` podemos listar todos los directorios y archivos del sistema, podemos hacer `find /root` y ver el contenido. Pero lo más crítico es que podemos ganar una consola como root, ya que este comando tiene el parámetro `-exec` con el que podemos ejecutar comandos. Para ganar una bash como root hacemos `find . -exec /bin/bash -p \; -quit`:

![image](https://github.com/user-attachments/assets/1ab5e7c6-10e5-4597-b883-05e0c5bca169)

Y ganamos acceso como root a la máquina Mirame.