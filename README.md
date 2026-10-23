# Despliegue de Nginx y Apache con Docker

Práctica del módulo **Despliegue de aplicaciones web (0614)** — 2º DAW

Autor: Marcos Martínez Cossío

---

## De qué va esto

La idea es montar dos servidores web en contenedores separados, cada uno haciendo una cosa distinta, en vez de tener uno solo que lo haga todo:

- **Nginx** está delante. Es el único que se puede tocar desde fuera. Sirve una web estática y, cuando le piden el otro dominio, le pasa la petición a Apache (esto se llama proxy inverso).
- **Apache** está detrás, escondido. No tiene ningún puerto abierto hacia mi ordenador, así que no se puede entrar a él directamente. Solo Nginx puede hablar con él. Dentro tiene una zona `/privado/` que solo se abre si la petición trae una cabecera concreta.

Lo monto todo con Docker Compose para que levantarlo sea un solo comando.

---

## Cómo está montado

```
                     MI ORDENADOR (Windows 11 + Docker Desktop)
                                    |
                           http://localhost:8080
                                    |
    +===============================v==============================+
    |          red_frontal  -  bridge  172.28.0.0/24               |
    |                                                              |
    |   +------------------------------------------------------+   |
    |   |  edge_nginx          nginx:1.27-alpine               |   |
    |   |  172.28.0.10  :80    <-- EL UNICO PUERTO ABIERTO     |   |
    |   |                                                      |   |
    |   |  www.consultora.local  -> web estatica (la sirve el) |   |
    |   |  app.consultora.local  -> se la pasa a Apache -----+ |   |
    |   |  cualquier otro nombre -> corta la conexion (444)  | |   |
    |   +----------------------------------------------------|-+   |
    +========================================================|=====+
                                                             |
    +========================================================|=====+
    |     red_interna  -  bridge  172.29.0.0/24  INTERNAL     |     |
    |        (esta red no tiene salida a internet)            |     |
    |                              172.29.0.10 ---------------+     |
    |                                    |  X-Forwarded-For         |
    |                                    v                          |
    |   +------------------------------------------------------+   |
    |   |  backend_apache      httpd:2.4-alpine                |   |
    |   |  172.29.0.20  :80    <-- SIN PUERTO EN EL HOST       |   |
    |   |                                                      |   |
    |   |  /          -> publico                               |   |
    |   |  /privado/  -> pide X-Auth-Token: DAW2026            |   |
    |   +------------------------------------------------------+   |
    +==============================================================+
```

Lo importante del dibujo: **hay dos redes, no una**. Nginx está en las dos (en una habla con mi PC y en la otra con Apache), y Apache solo está en la de dentro. Esa red interna la he marcado como `internal`, que quiere decir que ni entra ni sale nada de ahí hacia internet.

---

## Archivos del proyecto

```
.
├── docker-compose.yml          # Levanta los dos contenedores y crea las redes
├── .gitignore                  # Para que los logs no suban al repo
├── .gitattributes              # Fuerza finales de linea LF (trabajo en Windows)
├── README.md
│
├── nginx/
│   ├── conf.d/
│   │   ├── 00-hardening.conf   # Ajustes de seguridad generales
│   │   └── default.conf        # Los dos dominios y el proxy inverso
│   ├── html/
│   │   ├── index.html
│   │   └── estilos.css
│   └── logs/
│
└── apache/
    ├── Dockerfile              # Enciende solo los modulos que necesito
    ├── conf/
    │   ├── hardening-base.conf # Ajustes de seguridad generales
    │   └── sitio-privado.conf  # El sitio, /privado/ y los logs
    ├── html/
    │   ├── index.html
    │   ├── estilos.css
    │   └── privado/
    │       └── index.html
    └── logs/
```

Dos detalles que no son porque sí:

**Por qué el de Nginx se llama `00-hardening.conf`.** Nginx carga los archivos de `conf.d` por orden alfabético. En ese archivo defino cosas (el `map` y los formatos de log) que luego uso en `default.conf`, y si se cargara después petaría al arrancar. Con el `00-` me aseguro de que va primero.

**Por qué Apache tiene Dockerfile y Nginx no.** A Apache tengo que activarle módulos que vienen apagados en la imagen oficial (sobre todo `mod_remoteip`, que es el que saca la IP real del cliente). Eso hay que hacerlo construyendo la imagen. Nginx me vale tal cual viene, solo le monto la configuración.

---

## Cómo desplegarlo

### Lo que hace falta

Windows 11 con Docker Desktop (yo uso el backend de WSL2), Git y un editor.

### 1. Los dominios

Los dos dominios me los he inventado, no existen en internet, así que hay que decirle al ordenador a dónde apuntan. Abro PowerShell **como administrador** y ejecuto:

```powershell
Add-Content -Path C:\Windows\System32\drivers\etc\hosts -Value "`n127.0.0.1`twww.consultora.local`n127.0.0.1`tapp.consultora.local"
ipconfig /flushdns
```

Esto añade dos líneas al archivo `hosts`, que es una lista que mira Windows antes de preguntar a ningún DNS.

> Si no se puede tocar ese archivo (por permisos del equipo), se puede saltar este paso y usar `curl --resolve` en las pruebas, que manda la misma cabecera `Host` y funciona igual.

### 2. Levantarlo

```bash
git clone <url-del-repo>
cd <carpeta-del-repo>

docker compose config --quiet    # comprueba que el compose no tiene errores
docker compose up -d --build     # construye Apache y levanta todo
docker compose ps                # ver que los dos estan healthy
```

### 3. Comandos del día a día

```bash
docker compose logs -f nginx-edge    # ver los logs en directo
docker compose restart nginx-edge    # reiniciar uno
docker compose down                  # parar y borrar los contenedores
docker compose down -v --rmi local   # borrar tambien imagenes y volumenes
```

---

## Comprobar que funciona

### Los dos dominios responden, y cada uno por su servidor

```bash
curl -I http://www.consultora.local:8080/
curl -I http://app.consultora.local:8080/
```

<!-- Los dos comandos curl -I seguidos, que se vea la cabecera X-Servido-Por -->
![Los dos dominios respondiendo](capturas/01-ambos-dominios.png)

Para distinguir quién contesta, le he metido a cada servidor una cabecera propia:

| Dominio | Quién responde | Cabecera `X-Servido-Por` |
|---|---|---|
| `www.consultora.local` | Nginx (web estática) | `nginx-edge` |
| `app.consultora.local` | Apache (a través del proxy) | `apache-backend` y `nginx-edge -> apache-backend` |
| cualquier otro nombre | nadie, corta la conexión | — |

En el segundo salen las dos cabeceras: la de Apache (que es quien genera la página) y la de Nginx (que es quien la reenvía). Eso demuestra que la petición ha pasado por los dos.

También se puede probar el caso raro:

```bash
curl -I http://localhost:8080/
```

Da error de "respuesta vacía" porque le he puesto a Nginx un bloque por defecto que **corta la conexión con un 444** si el nombre que piden no es ninguno de los dos míos. Así nadie puede entrar poniendo la IP a pelo ni ir probando nombres a ver qué sale.

### Apache no tiene ningún puerto abierto

```bash
docker compose ps
docker compose port apache-backend 80
```

<!-- El docker compose ps (columna de puertos de Apache vacia) y el port fallando -->
![Apache sin puertos publicados](capturas/02-aislamiento.png)

El segundo comando **falla a propósito** (devuelve `invalid IP:0`), y eso es justo lo que quiero demostrar: Apache no publica nada hacia mi ordenador.

En el `ps` la diferencia se ve comparando las dos líneas:

```
edge_nginx       ...   0.0.0.0:8080->80/tcp     <- publicado al host
backend_apache   ...   80/tcp                   <- solo escucha por dentro
```

La flecha `->` es lo que indica que un puerto está abierto hacia fuera. Apache pone `80/tcp` a secas, que solo significa "este contenedor escucha en el 80 dentro de la red de Docker".

En el compose, a Apache le he puesto `expose` en vez de `ports`. `expose` solo deja constancia de que el contenedor escucha en el 80 por dentro, pero no abre nada hacia fuera.

### La zona `/privado/` solo se abre con la cabecera

```bash
# Sin cabecera
curl -I http://app.consultora.local:8080/privado/

# Con la cabecera mal
curl -I -H "X-Auth-Token: INCORRECTO" http://app.consultora.local:8080/privado/

# Con la cabecera bien
curl -I -H "X-Auth-Token: DAW2026" http://app.consultora.local:8080/privado/
```

<!-- Las tres peticiones seguidas: 403, 403 y 200 -->
![Endpoint restringido](capturas/03-privado.png)

Lo que he comprobado:

| Lo que mando | Lo que responde |
|---|---|
| nada | `403` |
| `X-Auth-Token: INCORRECTO` | `403` |
| `X-Auth-Token:` vacía | `403` |
| `X-Auth-Token: daw2026` (en minúsculas) | `403` |
| `X-Auth-Token: DAW2026` | `200` |
| un `DELETE` a cualquier sitio | `403` |

Las minúsculas también dan 403 porque la comparación distingue mayúsculas, el token tiene que ir clavado.

### Los logs de Apache guardan la IP del que entra de verdad

```bash
grep curl apache/logs/app_access.log | tail -5
```

<!-- El contenido de app_access.log, que se vean las dos primeras columnas -->
![Logs con la IP real](capturas/04-logs-ip-real.png)

Cada línea empieza con **dos IPs**:

```
172.28.0.1 172.29.0.10 - - [22/Oct/2026:23:56:27 +0000] "GET / HTTP/1.1" 200 1122 "-" "curl/8.21.0" ...
|--------| |----------|
     |           |
     |           +-- la IP de quien se conecta a Apache, o sea Nginx
     +-------------- la IP real del que ha pedido la página
```

Esto es lo que pide el enunciado y es lo que más me costó. El problema es que, como Apache está detrás del proxy, la conexión que le llega siempre viene de Nginx. Si no haces nada, **todas las líneas del log tendrían la IP de Nginx** y el registro no serviría para nada: no sabrías nunca quién ha entrado.

Lo que lo arregla es `mod_remoteip`. Nginx le manda a Apache una cabecera `X-Forwarded-For` con la IP del cliente, y ese módulo la lee y la pone como IP buena. Yo registro las dos a propósito, para poder enseñar la diferencia.

### De paso, no te puedes colar falseando la IP

Se podría pensar que, si Apache se fía de esa cabecera, cualquiera puede mandarla a mano y aparecer en el log con la IP que quiera:

```bash
curl -H "X-Forwarded-For: 8.8.8.8" http://app.consultora.local:8080/
```

Pero no cuela:

```
172.28.0.1 172.29.0.10 ... "GET / HTTP/1.1" 200 1122 "8.8.8.8" "curl/8.21.0" ...
|--------|                                            |-------|
  IP real                                      lo que intento colar
```

Funciona así: Nginx coge lo que manda el cliente y le **añade detrás** la IP real que él ha visto. Y en Apache solo me he fiado de la red interna (`172.29.0.0/24`), así que al recorrer la lista descarta únicamente los saltos de los que me fío y se queda con la que puso Nginx. La falsa queda apuntada en el log, pero nunca se usa como identidad.

---

## Decisiones que he tomado

### Las redes

| Qué he hecho | Por qué |
|---|---|
| Dos redes en vez de una | Separo el tráfico de mi PC a Nginx del de Nginx a Apache |
| La de dentro con `internal: true` | Se queda sin salida: Apache no puede ni salir a internet ni ser alcanzado desde fuera |
| Apache con `expose` y sin `ports` | `expose` no abre nada en el host, solo documenta el puerto interno |
| IP fijas y subredes puestas a mano | Así puedo decirle a Apache de qué rango se fía, y el despliegue sale igual siempre |
| `depends_on` con `service_healthy` | Nginx no arranca hasta que Apache responde, así no salen 502 al principio |
| `no-new-privileges` | Evita que dentro del contenedor se pueda escalar a root por un binario con setuid |

### Seguridad

En **Nginx**:

- `server_tokens off` — la cabecera `Server` pasa de `nginx/1.27.x` a solo `nginx`. Parece una tontería pero es lo primero que mira un escáner para saber qué versión atacar.
- El bloque por defecto con `444`, que ya he explicado arriba.
- Cabeceras: `X-Content-Type-Options`, `X-Frame-Options`, `Referrer-Policy`, `Content-Security-Policy` y `Permissions-Policy`.
- `limit_req` y `limit_conn` para que nadie me reviente a peticiones desde una misma IP.
- `proxy_hide_header X-Powered-By` para no chivar qué hay detrás.

En **Apache**:

- `ServerTokens Prod` y `ServerSignature Off`, misma idea que arriba.
- `TraceEnable Off`, que desactiva el método TRACE.
- `Options -Indexes` para que no se vea el listado de archivos de una carpeta sin index.
- Bloqueo de archivos tipo `.env`, `.bak`, `.sql`, que si se cuelan ahí son un regalo.
- Solo permito `GET`, `POST` y `HEAD`.
- En el Dockerfile activo los módulos **uno a uno** en vez de encenderlos todos: cada módulo cargado es una cosa más que te pueden atacar.

Y la configuración y el contenido van montados en **solo lectura** (`:ro`) en los dos. Lo único que se puede escribir son las carpetas de logs, porque ahí sí tienen que escribir los servidores.

### Los logs y el RGPD

Una IP es un dato personal, así que no se puede ir guardando alegremente. He hecho tres cosas:

**Dos registros en vez de uno.** Uno con la IP entera, para si pasa algo y hay que investigar, y otro con el último número borrado, para mirar estadísticas sin guardar datos de nadie:

```
app_access.log        10.10.10.5     <- IP entera
app_access_rgpd.log   10.10.10.0     <- el ultimo trozo a cero
```

**En el anonimizado guardo menos cosas.** Le he quitado el navegador y el referer, porque con eso también se puede identificar a alguien aunque no tengas su IP.

**Los logs no suben al repositorio.** Están en el `.gitignore`. Si los subiera a GitHub estaría publicando direcciones IP de gente, y eso sí que es un problema de verdad.

> Lo suyo sería borrar el log con IP entera cada 30 días. El anonimizado se puede guardar el tiempo que haga falta porque ya no identifica a nadie.

---

## Problemas que me he encontrado

### 1. `/privado/` se abría sin la cabecera

**Qué pasaba.** Tenía la restricción puesta en un bloque `<Directory>` y daba `200` igual, sin mandar nada.

**Por qué.** Apache mira las secciones en este orden: primero `<Directory>`, luego `<Files>` y al final `<Location>`. Y la última machaca lo de antes. Yo tenía un `<Location "/">` más abajo para limitar los métodos HTTP, y sin querer estaba cargándose la restricción del `/privado/`.

**Cómo lo arreglé.** Puse el control de la cabecera también en un `<Location>`, pero **después** del de los métodos, para que sea lo último que se mira:

```apache
<Location "/">                       <!-- primero esto -->
    <LimitExcept GET POST HEAD>
        Require all denied
    </LimitExcept>
</Location>

<Location "/privado">                <!-- y despues esto -->
    <RequireAll>
        Require expr %{HTTP:X-Auth-Token} == 'DAW2026'
    </RequireAll>
</Location>
```

### 2. Apache seguía guardando la IP de Nginx

**Qué pasaba.** Tenía `mod_remoteip` activado, la cabecera `X-Forwarded-For` llegaba bien, y aun así en el log salía la IP de Nginx.

**Por qué.** Tenía las líneas de `RemoteIP` puestas fuera del `<VirtualHost>`, arriba del todo. Ese módulo guarda su configuración por servidor y **no la hereda** de ahí hacia los sitios virtuales, así que era como si no hubiera puesto nada.

**Cómo lo arreglé.** Meterlas dentro del `<VirtualHost>` y ya.

### 3. Seguía sin salir la IP real, pero solo con IPs privadas

**Qué pasaba.** Con las líneas ya dentro, funcionaba si probaba con una IP de internet, pero con las IP de mi red (las que empiezan por 172 o 10) seguía saliendo la de Nginx.

**Por qué.** Estaba usando `RemoteIPTrustedProxy`, y resulta que esa directiva **se salta las IP privadas** de la cabecera. Como se le acaba la lista, se queda con la IP de la conexión, o sea la de Nginx. Lo probé para asegurarme:

| Directiva | IP que mando | IP que guarda |
|---|---|---|
| `RemoteIPTrustedProxy` | `203.0.113.77` (pública) | `203.0.113.77` ✔ |
| `RemoteIPTrustedProxy` | `172.28.0.1` (privada) | `127.0.0.1` ✘ |
| `RemoteIPInternalProxy` | `172.28.0.1` (privada) | `172.28.0.1` ✔ |
| `RemoteIPInternalProxy` | `203.0.113.77` (pública) | `203.0.113.77` ✔ |

Y claro, en mi caso el cliente soy yo desde mi propio ordenador, que llego con una IP privada. Con la otra directiva no me habría salido nunca.

**Cómo lo arreglé.** Cambiar a `RemoteIPInternalProxy`, que acepta las dos. Y no pierdo seguridad, porque sigo fiándome solo de la red interna, así que lo de falsear la IP sigue sin colar.