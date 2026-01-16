# DESPLIEGUE — Evidencias y respuestas

Este documento recopila todas las evidencias y respuestas de la practica.

---

## Parte 1 — Evidencias minimas

### Fase 1: Instalacion y configuracion

1) Servicio Nginx activo
- Que demuestra: El contenedor de Nginx esta corriendo correctamente
- Comando: `docker compose ps`
- Evidencia: [contenedorLevantado.png](evidencias/contenedorLevantado.png)

2) Configuracion cargada
- Que demuestra: Nginx ha cargado la configuracion sin errores
- Comando: `docker compose exec web-server nginx -t`
- Evidencia: [configuracionCargada.png](evidencias/configuracionCargada.png)

3) Resolucion de nombres
- Que demuestra: El dominio personalizado miweb.local resuelve a localhost
- Evidencia: [host_personalizado.png](evidencias/host_personalizado.png)

4) Contenido Web
- Que demuestra: La web principal se sirve correctamente en /
- Evidencia: [miweb.local.png](evidencias/miweb.local.png)

### Fase 2: Transferencia SFTP (Filezilla)

5) Conexion SFTP exitosa
- Que demuestra: FileZilla conecta al servidor SFTP en puerto 2222
- Evidencia: [filezilla.png](evidencias/filezilla.png)

6) Permisos de escritura
- Que demuestra: Se pueden subir archivos al volumen compartido via SFTP
- Evidencia: [filezilla.png](evidencias/filezilla.png)

### Fase 3: Infraestructura Docker

7) Contenedores activos
- Que demuestra: Los servicios web-server y sftp-server estan Up con puertos mapeados
- Comando: `docker compose ps`
- Evidencia: [i-01-compose-ps.png](evidencias/i-01-compose-ps.png)

8) Persistencia (Volumen compartido)
- Que demuestra: El volumen web_data es compartido entre nginx y sftp
- Evidencia: [docker_compose.png](evidencias/docker_compose.png)

9) Despliegue multi-sitio
- Que demuestra: La aplicacion reloj funciona en /reloj
- Evidencia: [clock.png](evidencias/clock.png)

### Fase 4: Seguridad HTTPS

10) Cifrado SSL
- Que demuestra: El sitio se sirve por HTTPS con certificado autofirmado
- Evidencia: [certificado.png](evidencias/certificado.png)

11) Redireccion forzada
- Que demuestra: HTTP redirige a HTTPS con codigo 301
- Evidencia: [f-02-301-network.png](evidencias/f-02-301-network.png)

---

## Parte 2 — Evaluacion RA2 (a–j)

### a) Parametros de administracion
- Respuesta:

| Directiva | Que controla | Configuracion incorrecta y efecto | Como comprobarlo |
|-----------|--------------|-----------------------------------|------------------|
| `worker_processes` | Numero de procesos worker. `auto` detecta CPUs | `worker_processes 1;` en servidor con 8 CPUs limita rendimiento | `grep worker_processes /etc/nginx/nginx.conf` |
| `worker_connections` | Conexiones simultaneas por worker (defecto: 1024) | `worker_connections 10;` rechaza conexiones en produccion | `grep worker_connections /etc/nginx/nginx.conf` |
| `access_log` | Ubicacion del registro de accesos HTTP | `access_log off;` impide analisis de trafico | `ls -lh /var/log/nginx/access.log` |
| `error_log` | Ubicacion y nivel de logging de errores | `error_log debug;` en produccion genera logs enormes | `tail /var/log/nginx/error.log` |
| `keepalive_timeout` | Tiempo de conexion keep-alive (defecto: 65s) | `keepalive_timeout 300;` agota recursos del servidor | `nginx -T \| grep keepalive` |
| `include` | Incluye archivos de configuracion adicionales | Olvidar `include /etc/nginx/conf.d/*.conf;` ignora virtual hosts | `ls -la /etc/nginx/conf.d/` |
| `gzip` | Compresion gzip de respuestas HTTP | `gzip off;` desperdicia ancho de banda | `curl -I -H "Accept-Encoding: gzip"` |

Cambio aplicado: Se modifico `keepalive_timeout` de 65 a 30 segundos en `conf/default.conf`:
```nginx
server {
    listen 443 ssl;
    keepalive_timeout  30;
}
```

- Evidencias:
  - [nginxconfDirectiva.png](evidencias/nginxconfDirectiva.png)
  - [02_nginx-t.png](evidencias/02_nginx-t.png)
  - [configuracionCargada.png](evidencias/configuracionCargada.png)

### b) Ampliacion de funcionalidad + modulo investigado
- Opcion elegida: B1 (Gzip)
- Respuesta:

Archivo `conf/gzip.conf`:
```nginx
gzip on;
gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/xml+rss text/javascript;
gzip_comp_level 5;
gzip_vary on;
```

Montaje en docker-compose.yml:
```yaml
volumes:
  - ./conf/gzip.conf:/etc/nginx/conf.d/gzip.conf
```

- Evidencias:
  - [b1-01_gzipconf.png](evidencias/b1-01_gzipconf.png)
  - [b1-02_compose_volume_gzip.png](evidencias/b1-02_compose_volume_gzip.png)
  - [b1-03_nginx-t.png](evidencias/b1-03_nginx-t.png)
  - [b1-04_curl_gzip.png](evidencias/b1-04_curl_gzip.png)

#### Modulo investigado: ngx_http_headers_more_module
- Para que sirve: Permite manipular cabeceras HTTP de peticiones y respuestas de forma avanzada. Puede establecer, eliminar y modificar cabeceras, a diferencia del modulo estandar que solo permite `add_header`.
- Como se instala/carga:
  - Compilacion: `./configure --add-module=/path/to/headers-more-nginx-module`
  - Modulo dinamico: `load_module modules/ngx_http_headers_more_filter_module.so;`
  - Paquete Debian/Ubuntu: `apt-get install libnginx-mod-http-headers-more-filter`
- Fuente(s):
  - https://github.com/openresty/headers-more-nginx-module
  - https://www.nginx.com/resources/wiki/modules/headers_more/

### c) Sitios virtuales / multi-sitio
- Respuesta:

Multi-sitio implementado:
- Web principal en `/`
- Web secundaria en `/reloj`

Diferencia entre multi-sitio por path y por nombre:
El multi-sitio por PATH utiliza diferentes rutas URL dentro del mismo dominio, configurado mediante bloques `location` en el mismo `server`. El multi-sitio por NOMBRE utiliza diferentes dominios configurados mediante bloques `server` con diferentes `server_name`.

Otros tipos de multi-sitio:
1. Por puerto: Diferentes aplicaciones en distintos puertos (80, 8080, 3000)
2. Por IP: Cada server block escucha en una IP especifica del servidor

Configuracion aplicada:
```nginx
root /usr/share/nginx/html;

location / {
    try_files $uri $uri/ =404;
}

location /reloj {
    alias /usr/share/nginx/html/reloj;
    index index.html;
    try_files $uri $uri/ /reloj/index.html;
}
```

- Evidencias:
  - [c1-01-root.png](evidencias/c1-01-root.png)
  - [clock.png](evidencias/clock.png)
  - [c1-03-defaultconf.png](evidencias/c1-03-defaultconf.png)

### d) Autenticacion y control de acceso
- Respuesta:

Configuracion de /admin:
```nginx
location /admin/ {
    alias /usr/share/nginx/html/admin/;
    index index.html;
    auth_basic "Area Restringida";
    auth_basic_user_file /etc/nginx/.htpasswd;
}
```

- Evidencias:
  - [d-01-admin-html.png](evidencias/d-01-admin-html.png)
  - [d-02-defaultconf_auth.png](evidencias/d-02-defaultconf_auth.png)
  - [d-03curl-401.png](evidencias/d-03curl-401.png)
  - [d-04curl-200.png](evidencias/d-04curl-200.png)

### e) Certificados digitales
- Respuesta:

Que es .crt y .key:
- .crt (Certificado): Contiene la clave publica y la informacion del certificado (dominio, organizacion, validez, emisor). Se puede compartir publicamente.
- .key (Clave privada): Contiene la clave privada asociada. NUNCA debe compartirse. Se usa para descifrar datos encriptados con la clave publica.

Por que -nodes en laboratorio:
La opcion `-nodes` indica que no se cifre la clave privada con contrasena. En laboratorio permite que Nginx inicie automaticamente sin introducir contrasena. En produccion se recomienda cifrar la clave privada.

- Evidencias:
  - [e-01-ls-certs.png](evidencias/e-01-ls-certs.png)
  - [e-02-compose-certs.png](evidencias/e-02-compose-certs.png)
  - [e-03-defaultconf-ssl.png](evidencias/e-03-defaultconf-ssl.png)

### f) Comunicaciones seguras
- Respuesta:

Configuracion de dos server blocks:

Server Block puerto 80 (redireccion):
```nginx
server {
    listen 80;
    server_name localhost;
    return 301 https://$host:8443$request_uri;
}
```

Server Block puerto 443 (servicio HTTPS):
```nginx
server {
    listen 443 ssl;
    ssl_certificate /etc/ssl/certs/nginx-selfsigned.crt;
    ssl_certificate_key /etc/ssl/private/nginx-selfsigned.key;
}
```

Por que dos server blocks:
Se usan dos server blocks para separar responsabilidades: el bloque HTTP (80) solo redirige a HTTPS, garantizando que todas las peticiones se cifren. El bloque HTTPS (443) sirve el contenido real.

- Evidencias:
  - [f-01-https.png](evidencias/f-01-https.png)
  - [f-02-301-network.png](evidencias/f-02-301-network.png)

### g) Documentacion
- Respuesta:

Arquitectura:
- Servicios: web-server (Nginx), sftp-server (atmoz/sftp)
- Puertos: 80:80, 8080:80 (HTTP), 8443:443 (HTTPS), 2222:22 (SFTP)
- Volumenes: web_data, conf/default.conf, conf/gzip.conf, certificados SSL

Configuracion Nginx:
- Ubicacion: practica-nginx/conf/default.conf
- Server blocks: HTTP (80) redirige, HTTPS (443) sirve
- Root: /usr/share/nginx/html
- Locations: /, /reloj, /admin/

Seguridad:
- Certificados SSL autofirmados
- HTTPS en puerto 8443
- Redirect HTTP a HTTPS con 301
- Compresion Gzip activada
- Autenticacion HTTP Basic en /admin/

- Evidencias: enlaces a todas las capturas en este documento

### h) Ajustes para implantacion de apps
- Respuesta:

Despliegue de /reloj:
Al desplegar una aplicacion en un subdirectorio como `/reloj`, las rutas relativas (`./style.css`) funcionan correctamente. Las rutas absolutas desde raiz (`/style.css`) fallarian porque buscarian en la raiz del servidor. La configuracion usa `alias` para mapear `/reloj` a la carpeta fisica correspondiente.

Problema tipico de permisos SFTP:
Al subir archivos via SFTP, pueden crearse con permisos restrictivos causando errores 403. Solucion: Configurar usuario SFTP con UID correcto en docker-compose.yml y ajustar permisos con chmod/chown tras la subida.

- Evidencias:
  - [c1-01-root.png](evidencias/c1-01-root.png)
  - [clock.png](evidencias/clock.png)

### i) Virtualizacion en despliegue
- Respuesta:

Instalacion nativa en SO:
- Nginx instalado directamente (`apt install nginx`)
- Configuracion en `/etc/nginx/` modificada directamente
- Cambios persistentes en el filesystem
- Dificulta portabilidad y puede generar conflictos

Contenedor efimero + volumenes:
- Nginx en contenedor Docker aislado
- Configuracion montada desde el host mediante volumenes
- Contenedor efimero: puede destruirse y recrearse sin perder configuracion
- Portabilidad, aislamiento, versionado facil, reproducibilidad

- Evidencias:
  - [i-01-compose-ps.png](evidencias/i-01-compose-ps.png)

### j) Logs: monitorizacion y analisis
- Respuesta:

Generacion de trafico:
```powershell
1..20 | ForEach-Object { curl.exe -s http://localhost:8080/ > $null }
1..10 | ForEach-Object { curl.exe -s http://localhost:8080/no-existe-$_ > $null }
```

Monitorizacion en tiempo real:
```bash
docker compose logs -f web-server
```

Extraccion de metricas:
```bash
# Top URLs
docker exec mi-servidor-nginx sh -c 'awk "{print $7}" /var/log/nginx/access.log | sort | uniq -c | sort -nr | head'

# Codigos de estado
docker exec mi-servidor-nginx sh -c 'awk "{print $9}" /var/log/nginx/access.log | sort | uniq -c | sort -nr | head'

# URLs con 404
docker exec mi-servidor-nginx sh -c 'awk "$9==404 {print $7}" /var/log/nginx/access.log | sort | uniq -c | sort -nr | head'
```

- Evidencias:
  - [j-01-logs-follow.png](evidencias/j-01-logs-follow.png)

---

## Checklist final

### Parte 1
- [x] 1) Servicio Nginx activo
- [x] 2) Configuracion cargada
- [x] 3) Resolucion de nombres
- [x] 4) Contenido Web
- [x] 5) Conexion SFTP exitosa
- [x] 6) Permisos de escritura
- [x] 7) Contenedores activos
- [x] 8) Persistencia (Volumen compartido)
- [x] 9) Despliegue multi-sitio (/reloj)
- [x] 10) Cifrado SSL
- [x] 11) Redireccion forzada (301)

### Parte 2 (RA2)
- [x] a) Parametros de administracion
- [x] b) Ampliacion de funcionalidad + modulo investigado
- [x] c) Sitios virtuales / multi-sitio
- [x] d) Autenticacion y control de acceso
- [x] e) Certificados digitales
- [x] f) Comunicaciones seguras
- [x] g) Documentacion
- [x] h) Ajustes para implantacion de apps
- [x] i) Virtualizacion en despliegue
- [x] j) Logs: monitorizacion y analisis
