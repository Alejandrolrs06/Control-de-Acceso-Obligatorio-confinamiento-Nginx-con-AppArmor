# Confinamiento de Nginx con AppArmor en Kali Linux

## 1. Objetivo
Crear y ajustar un perfil de AppArmor para confinar el servidor web Nginx, validando su funcionamiento en modo de aprendizaje (complain) y aplicándolo finalmente en modo de bloqueo (enforce).

## 2. Entorno Utilizado
* **Sistema Operativo:** Kali Linux
* **Servidor Web:** Nginx 1.30.4
* **Herramientas:** AppArmor 4.x (apparmor-utils)
* **Virtualización:** VirtualBox (SSH habilitado para comodidad).

## 3. Configuración Inicial de Nginx
Primero instalé Nginx con el comando `sudo apt install nginx` y comprobé que funcionaba, para ello hice una petición al servidor local con `curl -I http://localhost` y me respondió con un `200 OK` que significa que todo va bien.

![Captura 1 - Configuración inicial de Nginx](img1.png)

## 4. Creación y Ajuste del Perfil

**El problema:** Lo normal es usar una herramienta automática llamada `aa-genprof` que crea el perfil ella sola. Pero en Kali Linux nos dio un error: `Can't find system log "/var/log/syslog"`. Esto pasa porque Kali guarda los registros de otra manera y esa herramienta no lo encuentra.

![Captura 2 - Error de aa-genprof](img2.png)

**La solución:** Como la automática falló, creé el archivo del perfil a mano en `/etc/apparmor.d/usr.sbin.nginx`. Ahí le dije qué carpetas puede leer (como `/etc/nginx/` o `/var/www/html/`) y cuáles puede escribir (como los logs en `/var/log/nginx/`).

![Captura 3 - Archivo del perfil en nano](img3.png)

Luego lo cargué con `sudo apparmor_parser -r`. Al cargarlo me salió un aviso de `unknown option (audit)` pero era solo un aviso porque antes habíamos tocado un archivo de configuración; no afectó en nada y el perfil se cargó bien en modo "aprendizaje" (complain).

![Captura 4 - Carga del perfil y modo complain](img4.png)

## 5. Evidencias de Registros en Modo Complain
Puse el perfil en modo aprendizaje con `sudo aa-complain /usr/sbin/nginx`. Luego hice varias peticiones al servidor (`curl` y `restart`) y revisé los registros con `journalctl`. Vi líneas como esta: `apparmor="ALLOWED" ... name="/etc/ssl/kali.cnf"`. Esto demuestra que AppArmor estaba vigilando a Nginx y apuntando lo que hacía (leer un certificado de seguridad) pero sin bloquearlo, que es justo lo que hace este modo.

![Captura 5 - Registros en modo complain](img5.png)

## 6. Perfil Final en Modo Enforce
Una vez revisado todo, lo pasé al modo de bloqueo real con `sudo aa-enforce /usr/sbin/nginx`. El sistema me confirmó: `Estableciendo /usr/sbin/nginx al modo forzar`.

![Captura 6 - Modo enforce](img6.png)

## 7. Comprobación Tras el Confinamiento
Para asegurarme de que no habíamos roto nada, volví a hacer un `curl -I http://localhost` y me siguió dando `200 OK`. Además, con `systemctl status nginx` vi que el servicio seguía `active (running)` (funcionando perfectamente) con sus procesos trabajando.

![Captura 7 - Comprobación final](img7.png)

## 8. Reflexión sobre Accesos Permitidos
Para que Nginx pueda trabajar sin chocar con la seguridad, tuvimos que darle permiso para:
* **Leer:** la carpeta de configuración (`/etc/nginx/`) y la de las páginas web (`/var/www/html/`).
* **Leer y Escribir:** la carpeta de los logs (`/var/log/nginx/`) para apuntar quién entra, y el archivo del PID (`/run/nginx.pid`).
* **Capacidades especiales:** `net_bind_service` (para poder usar el puerto 80) y `setuid`/`setgid` (para cambiar al usuario `www-data`).

Le hemos dado solo lo mínimo indispensable para que funcione. Si un hacker entrara no podría usar Nginx para leer otros archivos del sistema o ejecutar comandos peligrosos.

## 9. Diferencias entre Complain y Enforce
* **Complain (Aprendizaje):** Si Nginx hace algo que no está en la lista, el sistema solo lo apunta en el registro (como un vigilante que toma notas), pero lo deja pasar. Sirve para probar y ajustar el perfil.
* **Enforce (Estricto):** Si Nginx hace algo que no está permitido, el sistema lo bloquea directamente y además lo apunta en el registro. Es el modo definitivo para proteger el servidor de verdad.
