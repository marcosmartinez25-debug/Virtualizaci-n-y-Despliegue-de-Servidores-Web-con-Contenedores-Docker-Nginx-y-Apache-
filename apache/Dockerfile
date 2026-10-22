FROM httpd:2.4-alpine

LABEL maintainer="DevOps Junior - 2 DAW" \
      description="Backend Apache aislado tras proxy inverso Nginx"

# Hardening base de la configuracion principal
COPY conf/hardening-base.conf /usr/local/apache2/conf/extra/hardening-base.conf

RUN set -eux; \
    sed -i \
      -e 's|^#\(LoadModule remoteip_module .*\)|\1|' \
      -e 's|^#\(LoadModule headers_module .*\)|\1|' \
      -e 's|^#\(LoadModule rewrite_module .*\)|\1|' \
      -e 's|^#\(LoadModule unique_id_module .*\)|\1|' \
      -e 's|^#\(LoadModule reqtimeout_module .*\)|\1|' \
      /usr/local/apache2/conf/httpd.conf; \
    printf '\nIncludeOptional conf/extra/hardening-base.conf\nIncludeOptional conf/extra/sitio-privado.conf\n' \
      >> /usr/local/apache2/conf/httpd.conf; \
    httpd -t

# Apache escucha en 80 SOLO dentro de la red interna de Docker sin que este puerto sea expuesto al exterior. El proxy inverso Nginx se encargara de exponer el puerto 80 al exterior.
EXPOSE 80

HEALTHCHECK --interval=15s --timeout=5s --start-period=10s --retries=3 \
  CMD wget -q --spider http://127.0.0.1/ || exit 1