# LOS 40 LADRONES - DOCKERLABS

## INFORMACIÓN Y DESPLIEGUE DE LA MÁQUINA

Esta máquina es de [DockerLabs](https://dockerlabs.es)

Para descargarla vas a [DockerLabs](https://dockerlabs.es) y buscas la máquina Los 40 Ladrones. Abres el enlace de mega, descargas el zip, lo descomprimes y ejecutas `bash ./auto_deploy.sh ./los40ladrones.tar`. De esta forma se ejecutará el contendor de docker de la máquina.

### descripción

- Dificultad: fácil
- Sistema Operativo: Linux
- Autor: firstatack

## RECONOCIMIENTO DE RED

Primero comprobamos la conectividad con la máquina objetivo: `ping -c 1 172.17.0.2`
> `-c 1` solo enviar un paquete ICMP

![image](https://github.com/user-attachments/assets/14b1c16b-ae8a-4c77-a8f1-f754e2c228e1)

Vemos que nos llega la respuesta, por lo que tenemos conectividad. Además el TTL es de 64 por lo que la máquina usa Linux.

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

**80 - TCP - HTTP**

Hacemos un escaneo de servicios y versiones con nmap sobre el puerto abierto: `nmap -p80 -sCV 172.17.0.2 -oN tcp_ports_targeted`.
> `-p80` hacer el escaneo sobre el puerto 80.

> `-sCV` escanear el servicio y la versión (`sV`) y lanzar los scripts de reconocimiento por defecto (`sC`)

> `-oN tcp_ports_targeted` exportar el output en formato nmap al archivo `tcp_ports_targeted`

## RECONOCIMIENTO WEB

Primero desde consola lanzamos `whatweb http://172.17.0.2`:

![image](https://github.com/user-attachments/assets/0ace851e-fdd1-4561-869e-531e0952ddf1)

No vemos nada interesante a parte de la versión de Apache, así que pasamos a ver el servidor web desde el navegador:

![image](https://github.com/user-attachments/assets/8d4494f2-5a0d-45f0-8e9f-4d8b87ddf09a)

Vemos el archivo de Apache2 por defeco de Ubuntu. Pasamos a hacer fuzzing con `gobuster dir -u http://172.17.0.2 -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x txt,bak,php,py,php.bak`:
> `dir` usar el modo de fuzzing de directorios

> `-u http.//172.17.0.2/` indicamos la URL para fuzzear

> `-w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt` indicamos el diccionario para fuzzear

> `-x txt,bak,php,py,php.bak` por cada palabra del diccionario probamos todas estas extensiones de archivo

![image](https://github.com/user-attachments/assets/bcab162b-c5a2-436d-b50d-6f78491ec31c)

Encontramos el archivo qdefense.txt. Si lo cargamos en el navegador vemos esto:

![image](https://github.com/user-attachments/assets/53378837-1059-40e9-a45d-0f9688a802b8)

Esto nos da mucha información. Nos da un posible usuario llamado toctoc, y además nos da la información de un posible código de port knocking, como el usuario se llama toctoc y hay 3 números que podrían ser puertos. 

El port knocking es una técnica en la cual un firewall establece una secuencia de puertos, y si alguien se intenta conectar en orden a esos puertos preestablecidos, se abre un puerto que antes estaba cerrado. 

## INTRUSIÓN

Podemos probar a hacer port knocking con esa secuencia que hemos visto en el archivo, usanos la herramienta `knock`: `knock 172.17.0.2 7000 8000 9000`. Después de ejecutar el comando hacemos otro escaneo de todo el rango de puertos para ver si se ha abierto algún puerto: `nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 172.17.0.2`:

![image](https://github.com/user-attachments/assets/23f5ec78-5d57-4246-a076-a9680935ed91)

Y vemos que se ha abierto el puerto 22 del ssh. Ahora podemos probar fuerza bruta con el posible usuario toctoc: `hydra -l toctoc -P /usr/share/wordlists/rockyou.txt ssh://172.17.0.2`:
> `-l toctoc` usuario toctoc

> `-P /usr/share/wordlists/rockyou.txt` especificamos el diccionario

![image](https://github.com/user-attachments/assets/f2765b32-f961-4f09-b2c9-b3cd1d76cdbc)

Vemos que consigue sacara la contraseña: kittycat. Entonces nos conectamos por ssh: `ssh toctoc@172.17.0.2` y de contraseña kittycat.

## ESCALADA DE PRIVILEGIOS

### root

Siendo toctoc, hacemos `sudo -l` para listar nuestros privilegios especiales de sudo:

![image](https://github.com/user-attachments/assets/04ec7b1f-3dee-4f51-8358-9abd220ad4b5)

Podemos ejecutar `/opt/bash` como cualquier usuario. Parece ser una copia de `/bin/bash`, así que probamos a hacer `sudo /opt/bash` y ganamos una bash como root directamente:

![image](https://github.com/user-attachments/assets/b4a5f12c-0ad9-436e-a4f1-d5e230ab0e93)

Y ya somos root en la máquina Los 40 Ladrones.