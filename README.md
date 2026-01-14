PASO 1: INFRAESTRUCTURA DOCKER (NGINX + SFTP)

OBJETIVO DEL PASO
Montar una infraestructura `inmutable` con Docker Compose que replique el escenario del enunciado: un servicio web (Nginx) y un servicio de transferencia (SFTP) donde lo que se suba por SFTP sea visible inmediatamente desde el navegador, sin reiniciar nada.

POR QUE SE HACE ASI (DECISIONES TECNICAS)
- Dos contenedores separados: se usa un contenedor para Nginx (servir la web) y otro para SFTP (subida de ficheros), porque el ejercicio pide un `Servicio Web (Nginx)` y un `Servicio de Transferencia (SFTP)`.
- Puertos mapeados 80->8080 y 22->2222: el enunciado exige que el puerto 80 del contenedor Nginx se mapee al 8080 del host, y el puerto 22 del contenedor SFTP se mapee al 2222 del host.
- Volumen compartido (la clave del ejercicio): ambos contenedores comparten un volumen/carpeta para que Nginx sirva el mismo contenido que el usuario sube por SFTP, cumpliendo que lo subido por FileZilla sea visible inmediatamente en el navegador.
- Rutas internas: Nginx suele servir desde `/usr/share/nginx/html` y el ejercicio indica que hay que `conectar` las rutas internas de ambos contenedores usando el mismo volumen.

ARCHIVOS IMPLICADOS
- docker-compose.yml: define los 2 servicios, sus puertos, y el volumen compartido.
- default.conf (opcional en este paso, recomendado): fuerza que Nginx use como root el directorio del volumen compartido, alineado con el enfoque de inyectar configuracion en Docker (mas importante en el paso de HTTPS, pero util desde el principio).

EVIDENCIAS (CAPTURAS) PARA ESTE PASO
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