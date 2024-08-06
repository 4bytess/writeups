# INJECTION - DOCKERLABS

## INFORMACIÓN Y DESPLIEGUE DE LA MÁQUINA

Esta máquina es de [DockerLabs](https://dockerlabs.es)

Para descargarla vas a [DockerLabs](https://dockerlabs.es) y buscas la máquina Injection. Abres el enlace de mega, descargas el zip, lo descomprimes y ejecutas `bash ./auto_deploy.sh ./injection.tar`. De esta forma se ejecutará el contendor de docker de la máquina.

### descripción

- Dificultad: muy fácil
- Sistema Operativo: Linux
- Autor: El Pingüino de Mario

## RECONOCIMIENTO DE RED

### conectividad y SO

Comprobamos la conectividad con la máquina: `ping -c 1 172.17.0.2`.
> `-c 1`: solo enviar 1 paquete ICMP

![image](https://github.com/user-attachments/assets/53747492-e791-4a6c-9b42-ae246908f258)

Vemos que nos llega la respuesta de la máquina, con un TTL de 64 por lo que su SO es Linux.

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


Hacemos un escaneo de servicios y versiones con nmap sobre los puertos abiertos: `nmap -p22,80 -sCV 172.17.0.2 -oN tcp_ports_targeted`.
> `-p22,80` hacer el escaneo sobre el puerto 22 y el 80.

> `-sCV` escanear el servicio y la versión (`sV`) y lanzar los scripts de reconocimiento por defecto (`sC`)

> `-oN tcp_ports_targeted` exportar el output en formato nmap al archivo `tcp_ports_targeted`

## RECONOCIMIENTO WEB

Primero lanzamos desde la consola el comando `whatweb http://172.17.0.2`
![image](https://github.com/user-attachments/assets/e1a8597a-9606-4235-afa9-2dae86bfdb69)

Vemos que es un Apache 2.4.52. También vemos un campo de contraseña y una cookie de sesión, lo que significa que debe haber una base de datos y un panel de login.
En la raíz de la web solo está el archivo de Apache2 de Ubuntu de configuración.
Probamos a hacer un fuzzing básico con gobuster: `gobuster dir -u http://172.17.0.2/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x txt,php,py`
> `dir` fuzzing de directorios

> `-u http://172.17.0.2` URL

> `-w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt` diccionario

> `-x txt,php,py` fuzzeamos las extensiones .txt, .php y .py.

Gobuster encuentra los archivos `index.php` y `config.php`. En el index.php hay un panel de login:
![image](https://github.com/user-attachments/assets/0dde8c96-a12a-4bc4-b9ad-d2155b2d2905)

Al poner una comilla en el campo de usuario, sale un error de MariaDB. Esto nos da la información de que el motor de base de datos que se usa es MariaDB, y de que el panel de login es vulnerable a inyección SQL.

## EXPLOTACIÓN WEB

Para bypassear el panel de login, probamos a poner de usuario `admin' or 1=1-- -`, ya que el usuario admin suele ser un nombre común para administradores. Y de contraseña ponemos cualquier cosa.
Al hacer esto, se consigue bypassear el panel de login ya que la query de SQL del servidor pasa de ser así:
`login where user_password == login_password`
a ser así:
`login where user_passsword == login_password ' or 1=1-- -`
de forma que siempre devuelve True.

Una vez logeados vemos esto:

![image](https://github.com/user-attachments/assets/cba5b31a-5c44-489f-ae00-bc9b5d5302a3)

Obtenemos unas credenciales que serían dylan:KJSDFG789FGSDF78. Esto es muy útil ya que el puerto 22 del SSH está abierto, así que se puede probar a conectarse: `ssh dylan@172.17.0.2` y de contraseña "KJSDFG789FGSDF78".

Y estamos dentro:

![image](https://github.com/user-attachments/assets/fc0b5cce-da31-4922-a463-66178e7ae291)

## ESCALADA DE PRIVILEGIOS

Buscamos archivos con permisos SUID desde la raíz del sistema: `find / -perm -4000 2>/dev/null`.
> `-perm -4000` buscar archivos con permisos SUID

> `2>/dev/null` redirigir el los errores al `/dev/null` para no mostrarlos y eliminar el ruido.

Vemos los siguientes archivos:

![image](https://github.com/user-attachments/assets/33d11eea-c946-4dd7-9af5-4e8ac1c26d07)

Podemos ir buscando en [gtfobins](https://gtfobins.github.io) cada binario y ver si se puede escalar privilegios si es SUID.
El binario `/usr/bin/env` es una vía para ganar privilegios, ya que con él se pueden ejecutar comandos, de forma que como es SUID y su propietario es root, los ejecutamos como root. Entonces para ganar una bash como root hacemos `/usr/bin/env /bin/bash -p`.
> `-p` lanzar la bash de forma privilegiada y tomar el UID de root en vez de seguir con el de dylan.

Y así ya tenemos una bash como root:

![image](https://github.com/user-attachments/assets/2badfe77-7e7f-4a8f-8567-9a873f71777c)