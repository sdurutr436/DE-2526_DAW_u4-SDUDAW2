# PASO 1: INFRAESTRUCTURA DOCKER (NGINX + SFTP)

## OBJETIVO DEL PASO
Montar una infraestructura `inmutable` con Docker Compose que replique el escenario del enunciado: un servicio web (Nginx) y un servicio de transferencia (SFTP) donde lo que se suba por SFTP sea visible inmediatamente desde el navegador, sin reiniciar nada.

### POR QUE SE HACE ASI (DECISIONES TECNICAS)
- Dos contenedores separados: se usa un contenedor para Nginx (servir la web) y otro para SFTP (subida de ficheros), porque el ejercicio pide un `Servicio Web (Nginx)` y un `Servicio de Transferencia (SFTP)`.
- Puertos mapeados 80->8080 y 22->2222: el enunciado exige que el puerto 80 del contenedor Nginx se mapee al 8080 del host, y el puerto 22 del contenedor SFTP se mapee al 2222 del host.
- Volumen compartido (la clave del ejercicio): ambos contenedores comparten un volumen/carpeta para que Nginx sirva el mismo contenido que el usuario sube por SFTP, cumpliendo que lo subido por FileZilla sea visible inmediatamente en el navegador.
- Rutas internas: Nginx suele servir desde `/usr/share/nginx/html` y el ejercicio indica que hay que `conectar` las rutas internas de ambos contenedores usando el mismo volumen.

### ARCHIVOS IMPLICADOS
- docker-compose.yml: define los 2 servicios, sus puertos, y el volumen compartido.
- default.conf (opcional en este paso, recomendado): fuerza que Nginx use como root el directorio del volumen compartido, alineado con el enfoque de inyectar configuracion en Docker (mas importante en el paso de HTTPS, pero util desde el principio).

### EVIDENCIAS (CAPTURAS) PARA ESTE PASO
Estas capturas encajan con la checklist de evidencias (especialmente para demostrar contenedores activos y mapeo de puertos).

- ![Contenedores docker lanzados](img/docker-ps-contenedores.png)
- ![Index básico para demostrar que la confgiruación funciona](img/pagina-nginx-basica-demostracion.png)

Código usado:

``docker-compose.yml``

```bash
services:
  web:
    image: nginx:alpine
    container_name: p41_web
    ports:
      - "8080:80"
    volumes:
      - ./www:/usr/share/nginx/html:ro
      - ./default.conf:/etc/nginx/conf.d/default.conf:ro
    depends_on:
      - sftp

  sftp:
    image: atmoz/sftp:alpine
    container_name: p41_sftp
    ports:
      - "2222:22"
    # usuario:pass:uid:gid:directorio
    command: "daw:1234:1001:1001:upload"
    volumes:
      # El contenedor SFTP escribe en /home/daw/upload
      - ./www:/home/daw/upload
```

``default.conf``

```bash
server {
  listen 80;
  server_name localhost;

  root /usr/share/nginx/html;
  index index.html;

  location / {
    try_files $uri $uri/ =404;
  }
}
```

# PASO 2: TRANSFERENCIA SFTP (FILEZILLA) Y DESPLIEGUE DE CONTENIDO

## OBJETIVO DEL PASO
En este paso se realiza el despliegue de los ficheros de la web usando SFTP con FileZilla, conectando contra el contenedor SFTP expuesto en el puerto 2222. El objetivo es demostrar que los ficheros subidos por SFTP aparecen inmediatamente servidos por Nginx en el puerto 8080 gracias al volumen compartido entre ambos contenedores

## POR QUE SE HACE ASI (DECISIONES TÉCNICAS)
- Se usa SFTP (y no FTP) porque SFTP es el protocolo de transferencia seguro sobre SSH, y el enunciado lo recomienda frente a FTP por seguridad.
- Se usa un cliente gráfico (FileZilla) porque el enunciado permite clientes como FileZilla (que ha costado descargarlo porque la página mandaba a descargar versiones promo que Windows decia que eran virus) o WinSCP para realizar la transferencia.
- Se suben los ficheros al directorio ``upload`` del usuario SFTP, ya que en el contenedor ``atmoz/sftp`` el usuario queda chroot en su home y el directorio de escritura habitual es un subdirectorio (mi caso ``upload``).
- Gracias al volumen compartido, lo que se escribe en /home/daw/upload dentro del contenedor SFTP se refleja en la carpeta ./www del host, que es la misma que Nginx sirve desde /usr/share/nginx/html.

## CONFIGURACIÓN EN FILEZILLA
En el *Gestor de sitios* en Filezilla se crea una entrada con:
- Protocolo: ``SFTP - SSH File Transfer Protocol``
- Servidor: ``localhost`` por ser mi caso.
- Puerto: ``2222`` configurado por mi caso.
- Modo de acceso: ``Normal``.
- Usuario: ``daw`` configurado para mi caso.
- Contrseña: ``1234``.

![Configuración en FileZilla](img/configuracion-filezilla.png)

## DESPLIEGUE DE LAS DOS WEBS (SEGUN LA PRÁCTICA)
La práctica exige desplegar dos aplicaciones web estáticas:

1. Web principal (en carpeta raiz):
- Repositorio desde el que se saca: ``https://github.com/cloudacademy/static-website-example``
- Ubicación en el servidor: raiz del sitio.
- En FileZilla: subir el contenido del repositorio a ``/upload`` (no subir la carpeta contenedora, sino lo que hay dentro y dejar en la raiz en index.html)
- Comprobación de los pasos dados:

Carpeta con el contenido, se debe hacer click derecho sobre el contenido en el hueco de abajo a la izquierda, señalando todos los archivos, y pulsar en subir:

![Contenido de la carpeta de static-webstie-example](img/contenido-localhost-pagina1.png)

Una vez subido, se verá como queda en el hueco de abajo a la derecha. Luego abriendo ``localhost:8080``:

![Página 1 funcionando: static-website-example](img/contenido-localhost-pagina1.png)

2. Web secundaria (subcarpeta /reloj):
- Repositorio del que se saca: 'https://github.com/ArchiDep/static-clock-website'
- Ubicación en el servidor: ``/reloj``
- En FileZilla: se crea primero la carpeta en la raiz llamada ``/reloj`` y se sube dentro el contenido del repositorio del reloj (deja su ``index.html`` en la raiz de ``/reloj`` lo cual crea esa ruta).
- Comprobación de los pasos dados:

![Carpeta reloj creada](img/reloj-desde-filezilla.png)

Una vez creada, clicamos en ella para que sea la nueva carpeta donde se suben cosas. Mismo sistema de la página 1, localizar la carpeta, ver sus archivos, seleccionar, clic derecho, subir.

![Pagina reloj desde filezilla](img/pagina-reloj-desde-filezilla.png)

Luego entramos en el navegador a la ruta: ``localhost:8080/reloj`` y vemos:

![Pagina reloj funcionando en localhost:8080/reloj](img/reloj-funcionando-8080.png)

Que funciona perfectamente.

# PASO 3 FINAL: CHECKLIST, EVIDENCIAS Y ENTREGA

## OBJETIVO
Completar la entrega de la practica adjuntando evidencias (capturas) y el codigo final de configuracion (docker-compose.yml y default.conf), marcando con '✅' cada requisito cumplido.

## QUE SE ENTREGA
- README.md / memoria con:
  - Checklist marcada.
  - Capturas como evidencia de cada punto.
  - Codigo completo de 'docker-compose.yml' y 'default.conf'.
- (Opcional) carpeta del proyecto con:
  - docker-compose.yml
  - default.conf
  - certs/ (nginx-selfsigned.crt y nginx-selfsigned.key)
  - www/ (web principal y /reloj)

CHECKLIST

FASE 1: INSTALACION Y CONFIGURACION

[✅] 1) Servicio Nginx activo con ``docker compose ps``. Alpine no trae ``service``.
![Servicio activo con docker compose ps](img/img-checklist/comprobacion-docker-compose-ps.png)

[✅] 2) Configuracion cargada.
![Configuracion cargada](img/img-checklist/comprobacion-listado-conf.png)

[✅] 3) Resolucion de nombres (/etc/hosts).

![Comandos de Windows para la resolución de nombres](img/img-checklist/comandos-para-añadir-p41.local.png)

![Acceso a través de p41 hosts](img/img-checklist/acceso-a-traves-de-p41-hosts.png)

[✅] 4) Contenido web principal visible (CloudAcademy) en vez de la pagina por defecto.

![Pagina CloudAcademy en 8080](img/img-checklist/demostracion-pagina-8080.png)

FASE 2: TRANSFERENCIA SFTP (FILEZILLA)

[✅] 5) Conexion SFTP exitosa.
![Conexion filezilla correcta](img/img-checklist/filezilla-conexion-correcta.png)

[✅] 6) Permisos de escritura OK (sin 'Permission denied').
![Pagina de Reloj añadida con permisos de escritura](img/pagina-reloj-desde-filezilla.png)

FASE 3: INFRAESTRUCTURA DOCKER
[✅] 7) Contenedores activos simultaneamente (Nginx + SFTP) y puertos mapeados.

El ejemplo de recreación ya demuestra que los contenedores están activos simultaneamente
![Contenedores funcionando simultaneamente](img/recreacion-contenedores-docker.png)

[✅] 8) Persistencia (volumen compartido): lo que se sube por SFTP se ve en la web.
Existen varias capturas a lo largo del documento que ya lo demuestran.

[✅] 9) Despliegue multi-sitio: reloj en subcarpeta.
![Reloj funcionando desde subcarpeta (se demuestra por la ruta)](img/reloj-desde-filezilla.png)

FASE 4: SEGURIDAD HTTPS

[✅] 10) Cifrado SSL: el servidor responde por HTTPS.
Se demuestra en varias de las capturas del resto del documento.

[✅] 11) Redireccion forzada: HTTP redirige a HTTPS (301).
Se demuestra en una captura del resto del documento

CODIGO FINAL

```bash
services:
  web:
    image: nginx:alpine
    container_name: p41_web
    ports:
      - '8080:80'
      - '8443:443'
    volumes:
      - ./www:/usr/share/nginx/html:ro
      - ./default.conf:/etc/nginx/conf.d/default.conf:ro
      - ./certs/nginx-selfsigned.crt:/etc/ssl/certs/nginx-selfsigned.crt:ro
      - ./certs/nginx-selfsigned.key:/etc/ssl/private/nginx-selfsigned.key:ro
    depends_on:
      - sftp

  sftp:
    image: atmoz/sftp:alpine
    container_name: p41_sftp
    ports:
      - '2222:22'
    command: 'daw:1234:1001:1001:upload'
    volumes:
      - ./www:/home/daw/upload
```

```bash
server {
  listen 80;
  server_name localhost;

  location / {
    return 301 https://$host:8443$request_uri;
  }
}

server {
  listen 443 ssl;
  server_name localhost;

  ssl_certificate     /etc/ssl/certs/nginx-selfsigned.crt;
  ssl_certificate_key /etc/ssl/private/nginx-selfsigned.key;

  root /usr/share/nginx/html;
  index index.html;

  location / {
    try_files $uri $uri/ =404;
  }
}
```

ARBOLEADO FINAL
```bash
PS C:\Users\Usu2S\Documents\DAW_2\Despliegue\DE-2526_DAW_u4-SDUDAW2> tree /f
Listado de rutas de carpetas
El número de serie del volumen es 6263-D144
C:.
│   default.conf
│   docker-compose.yml
│   README.md
│   
├───certs
│       nginx-selfsigned.crt
│       nginx-selfsigned.key
│       
├───img
│   │   conexion-certificado-indica-no-seguro.png
│   │   conexion-existosa-filezilla.png
│   │   configuracion-filezilla.png
│   │   contenido-localhost-pagina1.png
│   │   demostracion-pagina-8080.png
│   │   docker-ps-contenedores.png
│   │   evidencia-301.png
│   │   generacion_certificado_nginx.png
│   │   pagina-nginx-basica-demostracion.png
│   │   pagina-reloj-desde-filezilla.png
│   │   paginas-estaticas-descomprimidas-localizadas.png
│   │   paginas-zip-desktop.png
│   │   recreacion-contenedores-docker.png
│   │   reloj-desde-filezilla.png
│   │   reloj-funcionando-8080.png
│   │   
│   └───img-checklist
│           acceso-a-traves-de-p41-hosts.png
│           comandos-para-añadir-p41.local.png
│           comprobacion-docker-compose-ps.png
│           comprobacion-listado-conf.png
│           demostracion-pagina-8080.png
│           filezilla-conexion-correcta.png
│
└───www
    │   index.html
    │   LICENSE.MD
    │   README.MD
    │
    ├───assets
    │   ├───css
    │   │       font-awesome.min.css
    │   │       ie9.css
    │   │       main.css
    │   │       noscript.css
    │   │
    │   ├───fonts
    │   │       fontawesome-webfont.eot
    │   │       fontawesome-webfont.svg
    │   │       fontawesome-webfont.ttf
    │   │       fontawesome-webfont.woff
    │   │       fontawesome-webfont.woff2
    │   │       FontAwesome.otf
    │   │
    │   ├───js
    │   │       jquery.min.js
    │   │       main.js
    │   │       skel.min.js
    │   │       util.js
    │   │
    │   └───sass
    │       │   ie9.scss
    │       │   main.scss
    │       │   noscript.scss
    │       │
    │       ├───base
    │       │       _page.scss
    │       │       _typography.scss
    │       │
    │       ├───components
    │       │       _box.scss
    │       │       _button.scss
    │       │       _form.scss
    │       │       _icon.scss
    │       │       _image.scss
    │       │       _list.scss
    │       │       _table.scss
    │       │
    │       ├───layout
    │       │       _bg.scss
    │       │       _footer.scss
    │       │       _header.scss
    │       │       _main.scss
    │       │       _wrapper.scss
    │       │
    │       └───libs
    │               _functions.scss
    │               _mixins.scss
    │               _skel.scss
    │               _vars.scss
    │
    ├───error
    │       index.html
    │
    ├───images
    │       bg.jpg
    │       overlay.png
    │       pic01.jpg
    │       pic02.jpg
    │       pic03.jpg
    │
    └───reloj
            index.html
            README.md
            script.js
            style.css
```

JUSTIFICACION
- Se usa SFTP (cliente FileZilla) para transferir ficheros al contenedor SFTP expuesto en el puerto 2222.
- Nginx sirve el contenido del volumen compartido por HTTP (8080) y por HTTPS (8443) con certificado autofirmado.

LISTADO DE PROBLEMAS ENFRENTADOS

- `403 / Nginx no servia el contenido` al principio porque no apuntaba a un `index.html` porque no se creó (pero si apuntaba a la base de Nginx)
- `FileZilla incorrecto / instalador bloqueado`: el instalador que se descarga `sponsored` lo que Windows lo bloqueaba. Costó encontrar el FileZilla correcto.
- `Dudas operativas al subir con FileZilla`: confusión sobre si había que "arrastrar" la carpeta o su contenido en lugar de añadirlos de forma correcta.
- `Problemas editando el host de Windows`: se abria el bloc de notas de W11 pero no guardaba a pesar de hacer la apertura como administrador. Se hizo la edición por comandos.
- `Certificados inválidos`: no se generaron de forma correcta al principio y se tuvo que hacer por comandos.
- `Dificultad para ver el 301` porque no estaba marcada la opción de `Preserve logs` en el navegador.