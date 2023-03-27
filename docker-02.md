---
marp: true
title: Apuntes IISS 2021
description: Apuntes de Implementación e Implantación de Sistemas Software, curso 2021/22
---

<!-- size: 16:9 -->
<!-- theme: default -->

<!-- paginate: false -->

<!-- headingDivider: 1 -->

<style>
h1 {
  text-align: center;
}
img[alt~="center"] {
  display: block;
  margin: 0 auto;
}
</style>

# DOCKER - Parte 2

![width:480 center](img/docker-010.png)

---

## Recordatorio...


`docker ps` Contenedores creados en ejecución

`docker ps -a` Contenedores creados (incluye los que están parados)

`docker container ls`  (con o sin -a)

`docker run NOMBRE_IMAGEN` Crear un contenedor (si no tiene la imagen en local, la descarga)

`docker image ls` Ver las imágenes disponibles en local 


Docker da un nombre aleatorio a los contenedores si no lo elegimos nosotros. Todos los contenedores tienen un ID

```bash
$ docker ps -a
CONTAINER ID   IMAGE                    COMMAND                  CREATED         STATUS                      PORTS     NAMES
c8bb700e5f39   hello-world              "/hello"                 5 seconds ago   Exited (0) 4 seconds ago              busy_haibt
efe18fd91e7b   dodero/myserver:latest   "/bin/sh -c 'apache2…"   20 hours ago    Exited (130) 20 hours ago             myserver
```

---

`docker rm busy_haibt` Para borrar contenedores

`docker rm c8b` (no hace falta el ID completo)

Elegir el nombre del contenedor (debe ser único en local):

`docker run --name=hola hello-world`

Crear un contenedor asignando un puerto random en local:

`docker run -d -P --name=apache-server bitnami/apache`

`docker ps -a` (para ver puerto asignado)

Ejecutar terminal en el contenedor creado:

`docker exec -it apache-server /bin/bash`

---

Construir una imagen nueva

```bash
mkdir mi-apache
nano index.html
mv index.html mi-apache
nano Dockerfile
```

Dockerfile:
```docker
FROM bitnami/apache
COPY index.html /opt/bitnami/apache/htdocs/index.html
```

```bash
docker build -t usuario/mi-apache .
docker run -d -P --name=mi-apache-1 usuario/mi-apache
```

Subir nueva imagen a Docker Hub:

`docker push usuario/mi-apache`

---

## Directorios enlazados y volúmenes

Para compartir información con Docker cuando necesitamos:

- Compartir un __directorio__ entre múltiple contenedores.
- Compartir __un directorio__ entre el host y un contenedor.
- Compartir __un fichero__ entre el host y un contenedor.

Opciones:

- __Directorios enlazados__ *(bind):* la información se guarda fuera de Docker en el host local. Esta opción es mejor para datos no generados por los contenedores.
- __Volúmenes__: la información se guarda usando Docker. Mejor para datos generados por los propios contenedores.

---

## Directorios enlazados (bind)

Características:

- Permiten  __persistencia__  de la información del contenedor.
- Se montan en un  __path específico de la máquina local__  (fuera de Docker).
- Permiten  __borrar los contenedores sin perder la información__ .

---

### Ejemplo con directorios enlazados (bind) (1)

Crear directorio local `p02` para compartir:

`mkdir /Users/Usuario/p02`

Ejecutar el contenedor:

```bash
docker run -d -P --name=apache-bind-1 \
  --mount type=bind,source=`pwd`/p02,target=/app bitnami/apache
```

Comprobar los puertos a los que se ha asignado el 8080 (http) y 8443 (https) del Servidor Apache configurado en el contenedor:

```bash
$ docker ps -a
CONTAINER ID   IMAGE           COMMAND                 CREATED        STATUS         PORTS                                              NAMES
...
bcaae8157e45   bitnami/apache  "/opt/bitnami/script…"  3 minutes ago  Up 3 minutes   0.0.0.0:55004->8080/tcp, 0.0.0.0:55003->8443/tcp   apache-bind-1
...
```

---

Editar el fichero con _nano_ (u otro editor)

`nano /Users/Usuario/p02/index.html`

```html
<!DOCTYPE html>
<html lang="es">
<head>
  <meta charset="utf-8">
  <title>Apache en Docker</title>
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
</head>
<body>
  <h1>Hola Usuario! :)</h1>
</body>
</html>
```


---

Abrir en el host las URL http://localhost:55004/ y https://localhost:55003/

![width:550](img/iiss-docker-021.png) ![width:550](img/iiss-docker-022.png)

Si eliminamos el contenedor, no perdemos el contenido de la web:


```bash
docker stop apache-bind-1
docker rm apache-bind-1
docker run -d -P --name=apache-bind-2 \
  --mount type=bind,source=`pwd`/p02,target=/app bitnami/apache
```

---

## Volúmenes

### Características

- Permiten persistencia de información del contenedor
- Se montan en un _path_ específico del contenedor
- El acceso a la información sólo puede realizarse a través de Docker


### Ejemplo con volúmenes (1)

Creando un volumen para la web

```
docker run -d -P --name=apache-volume-1 \
   --mount type=volume,source=vol-apache,target=/app bitnami/apache
```

Comprobar el puerto asignado con `docker ps -a `

---

Acceso al volumen

`docker volume ls`

`docker exec -it apache-volume-1 /bin/bash`

Comprobar que `/app` está vacía y salir con `exit`

Actualizamos el contenido del volumen

`nano index.html`

`docker cp index.html apache-volume-1:app/`

Acceso de nuevo al volumen y comprobar que está el archivo `index.html`

`docker exec -it apache-volume-1 /bin/bash`


---

Acceso al contenido de la web

![width:550](img/iiss-docker-023.png)

Compartir volumen con otro contenedor

```
docker run -d -P --name=apache-volume-2 \
   --mount type=volume,source=vol-apache,target=/app bitnami/apache
```

---

Borrando volumen

```
docker stop apache-volume-2
docker rm apache-volume-2
```

`docker volume remove vol-apache`

```
Error response from daemon: remove vol-apache: volume is in use - [4a4794f86...3c8]
```

Reintentar borrado:

```
docker stop apache-volume-1
docker rm apache-volume-1
docker rm apache-volume-2
docker volume rm vol-apache
```

Comprobar que se ha borrado con `docker volume ls`

---

##  Configuraciones de red

- Permiten comunicación entre todos los contenedores pertenecientes a una red a través del nombre del contenedor
- Permiten aislamiento con respecto a otros contenedores
- Un contenedor puede pertenecer a varias redes

---

###  Tipos de redes

Ver las redes existentes: `docker network ls`

```bash
$ docker network ls
NETWORK ID     NAME      DRIVER    SCOPE
21c353b4b7a5   bridge    bridge    local
7a7abec15748   host      host      local
40157ecf9fcf   none      null      local
```

- **host** representa la red del propio equipo y haría referencia a `eth0`
- **bridge** representa la red `docker0` y a ella se conectan todos los contenedores por defecto 
- **none** significa que el contenedor no se incluye en ninguna red. Si verificamos esto con el comando `ifconfig` dentro del contenedor, veríamos que sólo tiene la interfaz de loopback `lo`

Saber qué contenedores usan un tipo de red: `docker network inspect bridge`

---

###  Ejemplo de configuración de red (1)

WordPress empaquetado en un contenedor:

https://github.com/bitnami/bitnami-docker-wordpress

Configuración manual de la red:

https://github.com/bitnami/bitnami-docker-wordpress#using-the-docker-command-line

Crear una nueva red:

`docker network create wordpress-network`

Crear primer volumen para persistencia de MariaDB:

`docker volume create --name mariadb_data`

Crear segundo volumen para persistencia de WordPress:

`docker volume create --name wordpress_data`

---

Crear primer contenedor que usa la red (base de datos `mariadb`):

```bash
docker run -d --name mariadb --env ALLOW_EMPTY_PASSWORD=yes \
  --env MARIADB_USER=bn_wordpress \
  --env MARIADB_DATABASE=bitnami_wordpress \
  --network wordpress-network \
  --volume mariadb_data:/bitnami/mariadb \
  bitnami/mariadb:latest
```

Crear segundo contenedor que usa la red (servidor `wordpress`):

```bash
docker run -d --name wordpress -p 8080:8080 -p 8443:8443 \
  --env ALLOW_EMPTY_PASSWORD=yes \
  --env WORDPRESS_DATABASE_USER=bn_wordpress \
  --env WORDPRESS_DATABASE_NAME=bitnami_wordpress \
  --network wordpress-network \
  --volume wordpress_data:/bitnami/wordpress \
  bitnami/wordpress:latest
```

---

![width:600 center](img/iiss-docker-024.png)

---

## Tarea: Práctica 2

Subir un documento PDF con los comandos realizados para resolver los siguientes ejercicios propuestos. Incluir también la respuesta a las preguntas realizadas y capturas donde se requiera mostrar en el navegador el correcto funcionamiento de los contenedores.

---

### Parte 1

1. Crear volumen compartido `volumenDocker`
2. Crear un contenedor de Nginx que use el volumen `volumenDocker`.
3. Modifique el contenido del fichero `index.html` incluyendo un saludo personal en lugar del texto por defecto.
4. Cree un segundo contenedor que también use el volumen `volumenDocker`.
5. Compruebe que puede acceder a `localhost:80` (primer contenedor) y `localhost:81` (segundo contenedor) y ver el contenido de `index.html`.

---

### Parte 2

1. Crear una nueva red `redDocker`.
2. Crear un contenedor de Ubuntu `Ubuntu1`.
3. Crear un contenedor de Ubuntu `Ubuntu2`.
4. Conectar `Ubuntu1` a la red `redDocker`.
5. Intentar hacer ping a `Ubuntu1` desde `Ubuntu2`. ¿Funciona? ¿Por qué?.
6. Conectar `Ubuntu2` a la red `redDocker`.
7. Intentar de nuevo hacer ping a `Ubuntu1` desde `Ubuntu2`. ¿Funciona? ¿Por qué?.

---

## Task 2

Upload a PDF document with the commands to be run to solve the following tasks. Answer also the proposed questions and include screenshots where it is required to check the correct functioning of containers.

---

###  Part 1

1. Create shared volume `DockerVolume`
2. Create a Nginx container that uses the `DockerVolume` volume.
3. Update the `index.html` file contents by including a personal greeting instead of the default message.
4. Create a second container that uses the `DockerVolume` volume.
5. Check that you can access `localhost:80` (first container) and `localhost:81` (second container) and see the content of `index.html`.

---

### Part 2

1. Create a new `DockerNetwork` network.
2. Create an Ubuntu container `Ubuntu1`.
3. Create an Ubuntu container `Ubuntu2`.
4. Connect `Ubuntu1` to the `DockerNetwork` network.
5. Try to _ping_ from `Ubuntu1` to `Ubuntu2`. Does it work? Why?
6. Connect `Ubuntu2` to the `DockerNetwork` network.
7. Try to _ping_ from `Ubuntu1` to `Ubuntu2` again. Does it work? Why?

 - ## PARTE 1
 ## Procedimiento para crear y ejecutar un contenedor Docker con Nginx y un volumen

En este procedimiento, se explicará cómo crear un contenedor Docker con Nginx y un volumen. Para ello, se utilizarán los siguientes comandos en una terminal:
1. Crear un volumen Docker con el siguiente comando:

```bash
docker volume create volumenDocker
```

Este comando creará un volumen de nombre "volumenDocker" que será utilizado para almacenar los datos del contenedor.

2. Crear un archivo Dockerfile con el siguiente comando:

```bash
touch Dockerfile
code ./Dockerfile
```


Este comando creará un archivo Dockerfile en la carpeta actual y abrirá el archivo en el editor de texto VS Code para editarlo.
3. Editar el archivo Dockerfile con el siguiente contenido:

```dockerfile
FROM nginx:latest

# Copiar el archivo index.html a la carpeta de documentos de Nginx
RUN echo '<!DOCTYPE html><html><head><meta charset="UTF-8"><title>Francisco López Guerrero. 2023</title><style>body {background-color: black;}</style></head><body><h1 style="color: white; text-align: center;">Francisco López Guerrero. 2023</h1><img src="https://www.aquariumcostadealmeria.com/wp-content/uploads/2019/04/shutterstock_256752703.jpg" alt="Imagen de un cangrejo."></body></html>' > /usr/share/nginx/html/index.html

# Exponer el puerto 80 para que el servidor web sea accesible desde el exterior
EXPOSE 80
```



Este archivo Dockerfile indicará a Docker cómo construir el contenedor y configurar Nginx para servir el archivo index.html en el puerto 81.

4. Construya la imagen de Docker utilizando el siguiente comando:

```bash
docker build -t nginx1 .
```
5. Ejecute el contenedor de Docker utilizando el siguiente comando, lo que asignará el puerto 80 del host al puerto 80 del contenedor y usará la imagen de nginx1 que acabamos de construir:

```bash
docker run -p 80:80 nginx 1-v volumenDocker:/usr/share/nginx/html nginx
```
6. Abra otra terminal y ejecute el siguiente comando para ejecutar un segundo contenedor de Docker que usará el volumen personalizado que acabamos de crear:

```bash
docker run -d -p 81:80 --name nginx-container2 -v volumenDocker:/usr/share/nginx/html nginx
```

Este comando creará un segundo contenedor de Docker utilizando la imagen de nginx y el volumen personalizado que acabamos de crear. Asignará el puerto 81 del host al puerto 80 del contenedor y lo nombrará como nginx-container2.
Enlace a mi imagen de DockerHub: https://hub.docker.com/repository/docker/craboftw/nginx1/general

¡Listo! Ahora tiene dos contenedores de Docker en ejecución: uno que utiliza la imagen de nginx1 y otro que utiliza el volumen personalizado que
network error

+ ## PARTE 2
 ## Procedimiento para crear una red y conectar dos contenedores Docker
1. Crear una nueva red de Docker utilizando el siguiente comando:

```bash
docker network create DockerNetwork
```
2. Crear un contenedor de Ubuntu llamado Ubuntu1 utilizando el siguiente comando:

```bash
docker run -it --name Ubuntu1 ubuntu /bin/bash
```
3. Crear otro contenedor de Ubuntu llamado Ubuntu2 utilizando el siguiente comando:

```bash
docker run -it --name Ubuntu2 ubuntu /bin/bash
```
4. Conectar el contenedor Ubuntu1 a la red DockerNetwork utilizando el siguiente comando:

```bash
docker network connect DockerNetwork Ubuntu1
```
5. Intenta hacer ping desde Ubuntu1 a Ubuntu2. Para ello desde una tercera terminal podemos averiguar la ip haciendo los siguientes comando

```bash
$ docker inspect   --format '{{ .NetworkSettings.IPAddress }}' Ubuntu1
```
Nos devuelve la siguiente ip en mi caso:
172.17.0.2
```bash
$ docker inspect   --format '{{ .NetworkSettings.IPAddress }}' Ubuntu2
```
Nos devuelve la siguiente ip en mi caso: 172.17.0.3
y luego desde la terminal de Ubuntu1 instalamos ping y hacemos ping a Ubuntu2

```bash
apt-get update -y
apt-get install -y iputils-ping

ping 172.17.0.2
```
```bash
From 172.17.0.2 icmp_seq=1 Destination Host Unreachable
```


Esto no funcionará, ya que Ubuntu2 no está conectado a la red DockerNetwork. Nos dice que que la ip es inalcanzable.

6. Conectar el contenedor Ubuntu2 a la red DockerNetwork utilizando el siguiente comando:

```bash
docker network connect DockerNetwork Ubuntu2
```
7. Intenta hacer ping desde Ubuntu1 a Ubuntu2 nuevamente utilizando el siguiente comando:

```bash
ping 172.17.0.2
```
```bash
64 bytes from 172.17.0.2: icmp_seq=1 ttl=64 time=0.322 ms
```


Ahora debería funcionar, ya que ambos contenedores están conectados a la misma red de Docker.
