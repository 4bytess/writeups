# DARKWEB - DOCKERLABS

## INFORMACIÓN Y DESPLIEGUE DE LA MÁQUINA

Esta máquina es de [DockerLabs](https://dockerlabs.es)

Para descargarla vas a [DockerLabs](https://dockerlabs.es) y buscas la máquina darkweb. Abres el enlace de mega, descargas el zip, lo descomprimes y ejecutas `bash ./auto_deploy.sh ./darkweb.tar`. De esta forma se ejecutará el contendor de docker de la máquina.

### descripción

- Dificultad: difícil
- Sistema Operativo: linux
- Autor: d1se0

## ESQUEMA

![darkweb](https://github.com/user-attachments/assets/42247b9d-2666-4db2-b370-dbb457082104)

## ENUMERACIÓN

### conectividad y SO

Primero comprobamos la conectividad con la máquina:

```python
ping -c 1 172.17.0.2
```

![image](https://github.com/user-attachments/assets/35938246-7515-4671-a156-07d90b5070aa)

Recibimso respuesta. Como el TTL es de 64, sabemos que la máquina usa Linux.

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

![image](https://github.com/user-attachments/assets/42d88979-8129-4c01-a5db-7cc0bd742821)

#### puertos abiertos

**22 - TCP - SSH**

**139 - TCP - SMB**

**445 - TCP - SMB**

Ahora hacemos un escaneo de versiones y servicios sobre estos tres puertos:

```python
nmap -p22,139,445 -sCV 172.17.0.2 -oN targeted
```

> `-p22,139,445` hacer el escaneo solo sobre los puertos 22, 139 y 445

> `-sCV` lanzar escaneo de versiones y scripts estándar de reconocimiento

> `-oN targeted` exportar el output al archivo `targeted`, en formato nmap

![image](https://github.com/user-attachments/assets/8a6d0c39-2276-43e8-b533-45893282deaa)

### puerto 445

Listamos los recursos de red compartidos por el servicio SMB, sin autenticación:

```python
smbclient -L //172.17.0.2 -N
```

![image](https://github.com/user-attachments/assets/d98586c3-5284-49e9-8d87-a16da65444d5)

Vemos un recurso interesante llamado `darkshare`. Nos intentamos conectar sin autenticar:

```python
smbclient //172.17.0.2/darkshare -N
```

![image](https://github.com/user-attachments/assets/e862c5b0-a9dd-46de-a9e7-3ef4b7db8df5)

Vemos todos esos archivos de texto. Nos los descargamos con el comando `get` y los vamos inspeccionando en local.

El archivo `ilegal.txt` es el único con información interesante:

![image](https://github.com/user-attachments/assets/42f23bc5-6e66-41c9-aca0-9ef34773f037)

Parece tener un texto cifrado. Abajo del texto pone "usa 5, tu me entiendes". Esto podría hacer referencia al cifrado ROT5, en el que se aplica una rotación de 5 caracteres al texto. 'a' corresponde a 'f', 'b' corresponde a 'g'... Creamos un script sencillo en python3 para poder descifrar ROT5:

```python
#!/usr/bin/python3

def descifrar(texto):

    abecedario = ['a', 'b', 'c', 'd', 'e', 'f', 'g', 'h', 'i', 'j', 'k', 'l', 'm', 'n', 'o', 'p', 'q', 'r', 's', 't', 'u', 'v', 'w', 'x', 'y', 'z']

    texto_descifrado = ""

    for caracter in texto:

        if caracter == " ":
            texto_descifrado += " "
            continue

        if caracter in abecedario:

            caracter_descifrado = abecedario[abecedario.index(caracter) - 5]

            texto_descifrado += caracter_descifrado

            continue

        texto_descifrado += caracter

    print(texto_descifrado)

def main():

    print("Ingresa el texto codificado:")
    texto_cifrado = input("")

    descifrar(texto_cifrado.lower())

if __name__ == '__main__':

    main()
```

Ahora desciframos el texto cifrado:

![image](https://github.com/user-attachments/assets/b8ad1529-8905-4831-928b-6120a1d17fec)

### dominio onion

Se nos revela un dominio onion. Para conectarnos tendremos que hacer uso de la red TOR. Esto se puede hacer de manera muy simple con el navegador Brave:

![image](https://github.com/user-attachments/assets/5730554f-38a2-49fe-a0df-1f62856e17aa)

Le damos a "Nueva conexión Tor para este sitio". Luego esperamos a que se conecte a TOR, y cargamos el dominio:

![image](https://github.com/user-attachments/assets/f1fd7785-c28f-4c67-9064-4dc294fd48a0)

Vemos una página con varias categorías. Ninguna funciona excepto en la que pone "Access the Dark Web". Si clicamos, nos lleva a un `darkweb.html`:

![image](https://github.com/user-attachments/assets/aa4a10f4-1d8f-4716-b285-b4c02c0d2ab8)

No hay nada interesante, pero en el apartado "Hidden Marketplace", hay un producto llamado "Confidential List's Passwords", que nos lleva a un diccionario con varias contraseñas:

![image](https://github.com/user-attachments/assets/30939f96-d7de-4141-afb4-7349f99d6d2f)

## INTRUSIÓN

Vemos que en todas estas contraseñas encontramos varias veces la palabra "dark", además que está relacionada con la temática de la web. Podemos hacer un ataque de fuerza bruta por SSH sobre el posible usuario dark, usando el diccionario que acabamos de encontrar:

```python
hydra -l dark -P password_list.txt ssh://172.17.0.2
```

![image](https://github.com/user-attachments/assets/d607352b-7dd2-46bd-8348-83e0efa90655)

Al parecer, el usuario dark existe, y su contraseña es "oniondarkgood". Nos conectamos:

```python
ssh dark@172.17.0.2
```

## ESCALADA DE PRIVILEGIOS

### root

Siendo el usuario dark, ejecutamos `sudo -l` para ver nuestros privilegios de sudo:

![image](https://github.com/user-attachments/assets/d9a9fcc8-b697-4131-875b-36e5967aade6)

Podemos ejecutar como cualquier usuario, sin usar contraseña, el script `/home/dark/hidden.py`. Miramos a ver el código de este script:

![image](https://github.com/user-attachments/assets/7490d092-59c7-45d7-b361-44b1ccc4f538)

Al parecer se ejecuta el script en bash `/usr/local/bin/Update.sh`. Ejecutamos `ls -l /usr/local/bin/Update.sh`:

![image](https://github.com/user-attachments/assets/b72bd1d8-8086-40a6-936a-c8f9ff5dca23)

El usuario dark es el propietario del script, así que podemos controlar lo que el script de python ejecuta, y como lo podemos ejecutar con sudo, tenemos una forma de ejecutar comandos como root. Ya por defecto el contenido del script en bash es `chmod u+s /bin/bash`, así que simplemente ejecutamos el script en python con sudo:

```python
sudo /home/dark/hidden.py
```

![image](https://github.com/user-attachments/assets/9772eca5-2f56-4067-83ee-7ba9a29e2eb6)

Vemos que la bash ahora es SUID. Ejecutamos `bash -p`:

![image](https://github.com/user-attachments/assets/6f0af47b-276b-4edf-80df-38a52ae3d964)

Y ganamos una bash como root en la máquina Darkweb.