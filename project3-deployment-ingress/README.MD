He tenido problemas para conseguir esto con mi WSL  + Docker Desktop. 

Aqui los pasos y causas , hasta la solucion

A continuación tienes un paso a paso bastante detallado para diagnosticar y resolver por qué, aún ejecutando minikube tunnel, tu Ingress no responde en http://rollingdemo.local. El objetivo es que verifiques cada punto y veas exactamente dónde está la traba.

Resumen rápido: Con WSL2 + Docker Desktop + Minikube, la IP 192.168.49.x normalmente no es alcanzable sin un túnel. Usar minikube tunnel debería exponer la misma IP que te da minikube ip, y así (con un /etc/hosts que asocie rollingdemo.local a esa IP) poder hacer curl http://rollingdemo.local. Pero a veces entran en juego otros factores (sudo no concedido, conflicto en el puerto 80, firewall, etc.).

1. Confirmar que “rolllingdemo.local” resuelve a 192.168.49.2 en WSL
En tu consola de WSL, haz:
bash
Copiar código
cat /etc/hosts | grep rollingdemo.local
Debe aparecer la línea:
lua
Copiar código
192.168.49.2    rollingdemo.local
Verifica que efectivamente resuelva a esa IP:
bash
Copiar código
getent hosts rollingdemo.local
# o
ping -c 2 rollingdemo.local
Si no te da respuesta, algo está mal con /etc/hosts.
Si responde a 192.168.49.2, vamos bien al menos en DNS.
2. Revisar si el “tunnel” está realmente activo y autorizado
Cuando corres:

bash
Copiar código
minikube tunnel
verás algo como:

rust
Copiar código
❗  The service/ingress rolling-ing requires privileged ports to be exposed: [80 443]
🔑  sudo permission will be asked for it.
🏃  Starting tunnel for service rolling-ing.
A veces WSL pedirá la contraseña de sudo (o un sudo prompt) para poder bindear los puertos 80/443 en la interfaz “del host”. Asegúrate de escribir bien tu contraseña y no cerrar esa ventana.

2.1. Confirmar que realmente está escuchando
Abre otra terminal de WSL (deja la de minikube tunnel corriendo) y ejecuta:

bash
Copiar código
sudo lsof -i -P -n | grep LISTEN | grep -E '80|443'
Si no aparece nada escuchando en :80 o :443 en nombre de minikube o algo similar, entonces es que el túnel no se pudo levantar.
Si sí aparece un proceso de vpnkit / minikube / “something” en *:80, significa que está escuchando.
A veces, si Docker Desktop también ocupa el puerto 80, puede haber conflicto. Revisa que ningún otro servicio (Apache, NGINX local, etc.) esté ya en el puerto 80.

3. Verificar la conexión al Ingress dentro de WSL
Sigue con la consola donde ya tienes minikube tunnel activo y haz:

bash
Copiar código
curl -v -H "Host: rollingdemo.local" http://127.0.0.1
¿Por qué 127.0.0.1 en lugar de rollingdemo.local?

Porque con minikube tunnel, en muchos casos minikube expone el Ingress/LoadBalancer en localhost:80 (o en la IP que minikube crea internamente).
Importante: El Ingress hace host-based routing usando el encabezado Host: rollingdemo.local.
Si no incluyes -H "Host: rollingdemo.local", es probable que NGINX no encuentre la regla y te devuelva 404 Not Found (el “default backend”).
Si ese comando (con -H) te devuelve 200 OK y muestra “VERSION 1”, significa que el túnel sí está funcionando y que la petición está entrando al Ingress.

Si obtienes curl: (7) Connection refused o Timeout, tu túnel no está realmente en marcha.
Si obtienes 404 Not Found, el Ingress está recibiendo el tráfico, pero no coincide con la regla de host. Asegúrate de poner -H "Host: rollingdemo.local".
3.1. Luego probar con curl -v http://rollingdemo.local
Si la línea en /etc/hosts es 192.168.49.2 rollingdemo.local y el túnel está redirigiendo 192.168.49.2:80 → contenedor, teóricamente también debería funcionar. Pero a veces en WSL2 la ruta a 192.168.49.2 difiere de la que se configura en localhost.

Si te vuelve a dar “Connection timed out”, puede indicar que la red de WSL no está permitiendo llegar a 192.168.49.2 aunque el túnel esté abierto.
Una prueba rápida:

bash
Copiar código
ping -c 2 192.168.49.2
Si ni siquiera ping contesta, es muy probable que no haya NAT hacia esa IP.
En algunos setups de WSL, localhost sí funciona, pero 192.168.49.2 no.
Así que, en entornos WSL, es frecuente que “localhost” sea la forma en que el túnel funciona. Esto es lo que produce el famoso nginx 404 al no matchear el host, a menos que agregues la cabecera Host: rollingdemo.local.

4. “Pero me abre el navegador en http://localhost/ y veo 404 Not Found, ¿por qué?”
Porque tu Ingress tiene la siguiente regla en ingress.yaml:

yaml
Copiar código
spec:
  rules:
    - host: rollingdemo.local
      http:
        paths:
        - path: /
          pathType: Prefix
          backend:
            service:
              name: rolling-svc
              port:
                number: 80
El NGINX Ingress sólo atiende la ruta / si el Host HTTP es rollingdemo.local.
Cuando abres http://localhost/ en el navegador, tu navegador manda Host: localhost. El Ingress no tiene ninguna regla para ese host → responde con el backend default (404).

Soluciones para ver algo en http://localhost/ directamente:

Opción rápida: En lugar de usar host: rollingdemo.local en tu Ingress, omitir la sección host: y poner:

yaml
Copiar código
spec:
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: rolling-svc
            port:
              number: 80
Entonces cualquier host (localhost, 127.0.0.1, etc.) matchea y te servirá la app.
Pero pierdes la funcionalidad de host-based routing.

O bien: Mantener host: rollingdemo.local, y cuando quieras probar con http://localhost, añadir manualmente -H "Host: rollingdemo.local" en el curl, o en tu navegador hacer un override del Host. (En un navegador normal, eso es más complicado sin plugin, por eso preferimos /etc/hosts).

Seguir usando rollingdemo.local en la barra de direcciones del navegador, y en el /etc/hosts (o en Windows C:\Windows\System32\drivers\etc\hosts) poner 127.0.0.1 rollingdemo.local.

Entonces http://rollingdemo.local y Host: rollingdemo.local coinciden, y el Ingress hará match.
¡Y ojo!, a veces es mejor apuntar a 127.0.0.1 (en vez de 192.168.49.2) si el túnel está realmente escuchando en localhost.
5. Chequeos de firewall y puertos
Si todo lo anterior no funciona y sigues con “timeout”, revisa:

¿Está Docker Desktop u otro servicio usando ya el puerto 80 en WSL?
sudo netstat -tunlp | grep :80
¿Le diste efectivamente el sudo password al minikube tunnel?
Si el túnel no logra binding a 80, no va a enrutarte nada.
Firewall en WSL: Normalmente, en Ubuntu dentro de WSL no hay ufw activo por defecto, pero revisa con:
sudo ufw status
Si está “active”, prueba sudo ufw allow 80 o desactívalo para test.
Versión de minikube y driver:
Asegúrate de tener algo relativamente actualizado: minikube version
Confirma que tu driver sea docker (que es lo usual con Docker Desktop).
El contenedor NGINX Ingress ya vimos que está Running y no hay errores en logs.
6. Resumen del plan de ataque
Ejecutar minikube tunnel en una consola. Concede permisos de sudo cuando lo pida.

Ver si tienes 2 opciones para “a dónde” conectar:

Opción A:

/etc/hosts → 127.0.0.1 rollingdemo.local
Abrir http://rollingdemo.local/ en WSL o en Windows.
Con host: rollingdemo.local, el Ingress hará match.
Si ves “404 Not Found”, puede ser que tu navegador esté usando Host: localhost, no “rollingdemo.local”. Asegúrate que la barra de direcciones ponga “rollingdemo.local”.
Opción B:

/etc/hosts → 192.168.49.2 rollingdemo.local
Pero en WSL2 a veces no hay ruta a 192.168.49.2 aunque el túnel esté activo.
Revisa con ping 192.168.49.2.
A menudo la forma más fiable es apuntar a localhost.
Test con curl -v -H "Host: rollingdemo.local" http://127.0.0.1 (o con la IP a la que tengas en hosts).

Si aquí obtienes 200 OK y “VERSION 1”, el Ingress está funcionando.
Si a pesar de todo curl se cuelga (“Connection timed out”):

Comprueba que no tengas un firewall local.
Comprueba que tu minikube tunnel no se esté cerrando por error.
Observa si minikube tunnel muestra algún log adicional o error.
7. ¿Por qué me abrió un navegador en “http://localhost/” con 404?
Porque minikube tunnel a veces, al exponer un Service/Ingress en puerto 80, abre tu navegador por defecto a http://localhost. Pero no coincide el Host con rollingdemo.local.
Por ende, el Ingress no matchea y sale el 404.
Es un comportamiento normal. Suele ser confuso, pero no indica error real en el Ingress, sólo que el Host es distinto.
8. Conclusión
El Ingress en sí está bien configurado (lo demuestran los logs y la respuesta 200 que se ve en el log).
El problema es red/puerto en WSL2: por defecto, 192.168.49.x no es accesible desde tu host sin un túnel que escuche en un puerto privilegiado (80).
Debes asegurarte de que minikube tunnel obtenga los permisos sudo y logre binding a ese puerto. Luego usar un host y una IP que apunte realmente al túnel (muchas veces, 127.0.0.1 rollingdemo.local es más sencillo).
Si ingresas a http://localhost sin Host: rollingdemo.local, es normal ver 404, pues tu Ingress rules.host no coincide con “localhost”.
Con todo esto verificado, deberías poder (1) encender minikube tunnel, (2) apuntar a http://rollingdemo.local (que resuelve a 127.0.0.1 o a la IP que te funcione) y ver “VERSION 1” sin problemas. ¡Éxitos!






Tú dijiste:
ernesto@ECM:~/kubernetes/k8s-learning/project3-deployment-ingress$ minikube tunnel
✅  Tunnel successfully started

📌  NOTE: Please do not close this terminal as this process must stay alive for the tunnel to be accessible ...

❗  The service/ingress rolling-ing requires privileged ports to be exposed: [80 443]
🔑  sudo permission will be asked for it.
🏃  Starting tunnel for service rolling-ing.

Aqui dice algo de que requiere permisos, pero nunca me los pide en ningun lado

ernesto@ECM:~/kubernetes/k8s-learning$ curl -I http://rollingdemo.local
curl: (28) Failed to connect to rollingdemo.local port 80 after 131869 ms: Connection timed out
ernesto@ECM:~/kubernetes/k8s-learning$ curl -I http://rollingdemo.local
curl: (28) Failed to connect to rollingdemo.local port 80 after 133234 ms: Connection timed out
ernesto@ECM:~/kubernetes/k8s-learning$ cat /etc/hosts | grep rollingdemo.local
192.168.49.2    rollingdemo.local
ernesto@ECM:~/kubernetes/k8s-learning$ getent hosts rollingdemo.local
192.168.49.2    rollingdemo.local
ernesto@ECM:~/kubernetes/k8s-learning$ ping -c 2 rollingdemo.local
PING rollingdemo.local (192.168.49.2) 56(84) bytes of data.

--- rollingdemo.local ping statistics ---
2 packets transmitted, 0 received, 100% packet loss, time 1014ms

ernesto@ECM:~/kubernetes/k8s-learning$ ping -c 2 rollingdemo.local
PING rollingdemo.local (192.168.49.2) 56(84) bytes of data.

--- rollingdemo.local ping statistics ---
2 packets transmitted, 0 received, 100% packet loss, time 1061ms

ernesto@ECM:~/kubernetes/k8s-learning$


ernesto@ECM:~$ sudo lsof -i -P -n | grep LISTEN | grep -E '80|443'
ssh       45890            root    4u  IPv6 11115050      0t0  TCP [::1]:80 (LISTEN)
ssh       45890            root    5u  IPv4 11115051      0t0  TCP 127.0.0.1:80 (LISTEN)
ssh       45890            root    6u  IPv6 11115054      0t0  TCP [::1]:443 (LISTEN)
ssh       45890            root    7u  IPv4 11115055      0t0  TCP 127.0.0.1:443 (LISTEN)
ernesto@ECM:~$



ernesto@ECM:~$ curl -v -H "Host: rollingdemo.local" http://127.0.0.1
*   Trying 127.0.0.1:80...
* Connected to 127.0.0.1 (127.0.0.1) port 80 (#0)
> GET / HTTP/1.1
> Host: rollingdemo.local
> User-Agent: curl/7.81.0
> Accept: */*
>
* Mark bundle as not supporting multiuse
< HTTP/1.1 200 OK
< Date: Mon, 06 Jan 2025 22:53:26 GMT
< Content-Type: text/plain; charset=utf-8
< Content-Length: 10
< Connection: keep-alive
< X-App-Name: http-echo
< X-App-Version: 0.2.3
<
VERSION 1
* Connection #0 to host 127.0.0.1 left intact
ernesto@ECM:~$

ernesto@ECM:~$ minikube ip
192.168.49.2
ernesto@ECM:~$
ernesto@ECM:~/kubernetes/k8s-learning$ ping -c 2 192.168.49.2
PING 192.168.49.2 (192.168.49.2) 56(84) bytes of data.

--- 192.168.49.2 ping statistics ---
2 packets transmitted, 0 received, 100% packet loss, time 1031ms

ernesto@ECM:~/kubernetes/k8s-learning$
ernesto@ECM:~$ sudo netstat -tunlp | grep :80
tcp        0      0 127.0.0.1:80            0.0.0.0:*               LISTEN      45890/ssh
tcp6       0      0 ::1:80                  :::*                    LISTEN      45890/ssh
ernesto@ECM:~$
ernesto@ECM:~$ curl -v http://rollingdemo.local
*   Trying 192.168.49.2:80...
* connect to 192.168.49.2 port 80 failed: Connection timed out
* Failed to connect to rollingdemo.local port 80 after 129330 ms: Connection timed out
* Closing connection 0
curl: (28) Failed to connect to rollingdemo.local port 80 after 129330 ms: Connection timed out


ernesto@ECM:~$ sudo ufw status
Status: inactive
ernesto@ECM:~$ 

ernesto@ECM:~$ minikube version
minikube version: v1.34.0
commit: 210b148df93a80eb872ecbeb7e35281b3c582c61
ernesto@ECM:~$
ChatGPT

Pensamiento durante 17 segundos
Viendo tu salida de comandos, está clarísimo que en tu entorno WSL no tienes ruta hacia la IP 192.168.49.2. Por eso:

El ping a 192.168.49.2 da 100% de pérdida (no hay respuesta).
El curl http://rollingdemo.local (que resuelve a 192.168.49.2) te da timeout.
Sin embargo, el Ingress sí está vivo y expuesto en localhost:80 vía minikube tunnel, tal como lo demuestras al hacer:

bash
Copiar código
curl -v -H "Host: rollingdemo.local" http://127.0.0.1
# ...
< HTTP/1.1 200 OK
< ...
VERSION 1
Eso confirma que el Ingress Controller está recibiendo la petición (con ese Host: rollingdemo.local) y te responde “VERSION 1”.

¿Por qué no te pide sudo y no ves 192.168.49.2 abierto?
El minikube tunnel en WSL a veces se monta en localhost (127.0.0.1)

En tu netstat se ve:
markdown
Copiar código
ssh    45890  root  4u  IPv6 ...
    LISTEN  127.0.0.1:80
Esto sugiere que el proceso del túnel está “ocupando” el puerto 80 en localhost (posiblemente a través de una conexión SSH interna).
No siempre se ve el prompt de sudo si ya está usando un mecanismo de reenvío de puertos en WSL.
WSL no tiene ruta a 192.168.49.2

Por defecto, la interfaz 192.168.49.x en minikube (con Docker Desktop) no es alcanzable directamente desde WSL2; solo un “virtual switch” la ve internamente.
El tunnel te abre localhost:80, no la IP 192.168.49.2.
¿Cómo acceder con “rollingdemo.local” sin usar -H "Host:..." manualmente?
Si deseas que curl http://rollingdemo.local/ (o tu navegador dentro de WSL) funcione automáticamente, tienes que apuntar rollingdemo.local a 127.0.0.1 en tu /etc/hosts, en lugar de (o además de) 192.168.49.2. Por ejemplo:

bash
Copiar código
sudo nano /etc/hosts

# Quita / comenta la línea con 192.168.49.2
# 192.168.49.2 rollingdemo.local

# Agrega:
127.0.0.1 rollingdemo.local
Luego:

bash
Copiar código
curl -v http://rollingdemo.local
El DNS resolverá rollingdemo.local → 127.0.0.1.
minikube tunnel está escuchando en 127.0.0.1:80.
El request tendrá Host: rollingdemo.local.
El Ingress matchea esa regla y devuelve “VERSION 1”.
Con eso ya no necesitas usar -H "Host: rollingdemo.local" cada vez.

Resumen rápido
Razón del fallo: En WSL2 no puedes llegar a 192.168.49.2, aunque sea la “IP de minikube”. El minikube tunnel en tu caso se está abriendo solo en localhost:80 y no en “la red” con esa IP.
Cómo solucionarlo: Asigna rollingdemo.local a 127.0.0.1 en /etc/hosts. De esa forma, curl http://rollingdemo.local o el navegador funcionará, siempre y cuando tengas minikube tunnel corriendo.
Por qué 404 si vas a http://localhost/: El Ingress hace host-based routing. Necesita el Host: rollingdemo.local; “localhost” no coincide, y NGINX responde con 404.
Así ya podrás ver tu página “VERSION 1” sin problemas. ¡Éxitos!