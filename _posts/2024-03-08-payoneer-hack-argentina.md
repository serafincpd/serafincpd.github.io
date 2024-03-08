---
layout: post
section-type: post
title: Analizando algunos aspectos del hack a cuentas de Payoneer en Argentina
category: tech
tags: [ 'security' ]
---

Como ya es de público conocimiento, durante Enero se vaciaron varias cuentas de Payoneer de usuarios residentes en Argentina. Si bien existen varias notas y análisis al respecto de cómo fue la cadena de ataque, me gustaría compartirles por este medio lo que pude analizar en el momento, a partir de algunos de los SMS que recibí durante esos días.

Entre los días 10 de Enero y 12 de Enero del 2024, me llegaron algunos mensajes como estos:

![SMS 1](/img/posts/payoneer-hack-argentina/sms-1.png)

![SMS 2](/img/posts/payoneer-hack-argentina/sms-2.png)

Desde un primer momento los dominios me resultaron sospechosos, pero como estaba haciendo otras cosas, no le di demasiada importancia, asumí de entrada que podría ser un caso de smishing.

Unas horas después, con la curiosidad ganándome la pulseada interna, me puse a intentar investigar un poco que tenían esos dominios. Un poco me esperaba ya de antemano que encontrar:

![Fake Login](/img/posts/payoneer-hack-argentina/fake-login.png)

Para el momento en el que me puse a mirar, el dominio `aceptarpago.com` ya no estaba mas accesible, pero el dominio `alertaspayoneer.com` tenia una copia del login form real de Payoneer. Obviamente, se trataba de un form falso para intentar, asumía yo hasta ese momento, quedarse con las credenciales de los usuarios que cayeran en la trampa… Después vamos a ver que no era necesario siquiera cargar la contraseña real de la cuenta para terminar siendo victima del ataque, que iba a terminar siendo bastante mas complejo de lo que en un principio parecía.

Habiendo visto que se trataba de un sitio falso, intenté enseguida ver que información podía recolectar del dominio, las IPs en uso, y el código fuente de la pagina.

### Analizando el Código

Si miramos el código fuente de esta página, nos vamos a encontrar con algunas cosas bastante particulares, que nos pueden llamar la atención.

En primer lugar, miremos este fragmento:

![CSS Code](/img/posts/payoneer-hack-argentina/code-css.png)

Es poco común encontrar en un sitio como el de Payoneer que los estilos están embebidos en el HTML, ya que es una práctica bastante estandarizada separar las hojas de estilos para poder reutilizar el código en múltiples páginas independientes.

Por otro lado, si analizamos el código Javascript, que a su vez también estaba directamente embebido en la página, nos encontramos con fragmentos de código que, al menos, llama la atención:

![JS Code](/img/posts/payoneer-hack-argentina/code-js.png)

Si miramos el código, vemos un par de referencias a elementos con nombres de clase `ojito` y `ver-clave`, y por otro lado el ID del input `exampleInputPassword1` que son detalles que son poco probable que este presente en el código de una empresa como Payoneer.

Ya estos puntos nos dan algunos indicios de que la página pudo haber sido clonada, y para simplificar el proceso se embebió todo el código en el mismo archivo y se hicieron modificaciones mínimas.

### Analizando el dominio y direcciones IP

Al encontrarse aún activo uno de los dominios, haciendo uso de algunas técnicas básicas de OSINT pude obtener algo más de información:

```bash
$ dig a alertaspayoneer.com @8.8.8.8

; <<>> DiG 9.18.18-0ubuntu0.22.04.1-Ubuntu <<>> a alertaspayoneer.com @8.8.8.8
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 2250
;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; OPT PSEUDOSECTION:
; EDNS: version: 0, flags:; udp: 512
;; QUESTION SECTION:
;alertaspayoneer.com.           IN      A

;; ANSWER SECTION:
alertaspayoneer.com.    21600   IN      A       188.68.217.14

;; Query time: 28 msec
;; SERVER: 8.8.8.8#53(8.8.8.8) (UDP)
;; WHEN: Fri Jan 12 17:12:43 -03 2024
;; MSG SIZE  rcvd: 64
```

El dominio apunta a una única IP `188.68.217.14`, de la cual podemos a su vez obtener un poco mas de información:

![ARIN Whois](/img/posts/payoneer-hack-argentina/arin-whois.png)

![RIPE Whois](/img/posts/payoneer-hack-argentina/ripe-whois.png)

Cómo se puede ver, haciendo uso de un servicio como whois podríamos intentar obtener algo mas de información acerca de las entidades que tienen control sobre los rangos de IPs a los que pertenece esa dirección, determinando que se trata de una IP asignada a Rusia, puntualmente a un proveedor de Hosting llamado Selectel. De nuevo, extraño para Payoneer.

Analizando el dominio, quien es la entidad registrante y a nombre de quien esta registrado, encontré la siguiente información:

![DNS 1](/img/posts/payoneer-hack-argentina/dns-1.png)

![DNS 2](/img/posts/payoneer-hack-argentina/dns-2.png)

Cosas que nos podrían llamar la atención? Bueno, lo principal creo, es que el dominio fue registrado el 11 de enero, que coincide justamente con la fecha que me empezaron a llegar los SMS. Por otro lado, NameSilo utiliza PrivacyGuardian, que oculta la información de quien registro el dominio, lo cual en este caso resulta ideal para quien esta detrás de este setup, ya que sus datos no son publicados en los servicios de whois.

Si juntamos esto, sumado al análisis del código fuente de la página, mas la ubicación del hosting donde se encontraba alojada, parecen evidencias bastante solidas para concluir que claramente se trataba de un intento para obtener credenciales reales de usuarios a partir del engaño.

### Implicancias del ataque

Si bien lo que analizamos hasta acá, parece un simple smishing para intentar robar credenciales validas de usuarios, el ataque, que logró tener éxito y vaciar varias cuentas de usuarios, pareciera haber sido bastante mas complejo.

Dado que ya hay algo de información sobre eso, no me voy a detener en analizar en detalle la cadena completa de ataque, pero en resumen, pareciera ser que los atacantes se hicieron con el control de un Gateway SMS, que puntualmente era el que usaba Payoneer para enviar los SMS con los tokens del 2FA, lo les permitía ver pasar todos los códigos que se les enviaban a los usuarios, y, a su vez, enviar mensajes falsos, lo que les permitió ejecutar la campaña de smishing.

Ahora, que rol jugaría el falso form de login en esta cadena, y por qué ni siquiera fue necesario que los usuarios pongan las credenciales reales? Bueno, según mi punto de vista, el hecho de intentar capturar las credenciales fue principalmente un intento de “acelerar” el proceso, ya que aún teniendo solo el usuario de la cuenta, y explotando el hecho de “poder ver pasar” los tokens 2FA, habrían podido hacer el reset de cualquier cuenta que tuviera el 2FA via SMS, y que los mensajes pasara por el gateway que tenían bajo control. Entiendo que la razón detrás de haber montado sitios falsos, fue principalmente para intentar obtener las credenciales y ahorrarse un paso. El punto es que los usuarios afectados no necesariamente necesitaban caer en la trampa de dar sus credenciales en el sitio falso, ya que, si los atacantes obtuvieron nombres de usuarios válidos por cualquier otro medio, podrían haberlos utilizados igualmente en esta cadena de ataque.

Según la información que pude encontrar publicada durante esos días, aparentemente los únicos usuarios que estuvieron realmente a salvo fueron aquellos que tenían el 2FA habilitado con Notificación desde la App instalada en un dispositivo, pero aparentemente esta opción no estaba disponible para todos los usuarios. Lo cual deja una doble enseñanza: a los proveedores de servicios, dejar de ofrecer SMS como 2FA, y a los usuarios, no usar SMS si hay otra opción disponible!

### Qué podemos hacer ante un caso así?

Qué podemos hacer como usuarios ante el caso de recibir mensajes de smishing/phishing como estos?

- Primero y principal: nunca hacer click en un link que nos genere cualquier tipo de duda o sospecha.
- Verificar accediendo directamente desde la aplicación/sitio real si lo que estamos recibiendo es real y se ve reflejado en la app/plataforma.
- Ante cualquier sospecha de que nuestras credenciales hayan sido comprometidas, cambiar las contraseñas.
- No usar SMS como 2FA en todas las plataformas que permitan seleccionar otra opción.
- Reportar a la plataforma que estamos recibiendo este tipo de mensajes o intentos de robo de credenciales.

### Links a mas información sobre el caso

[Vacían cuentas de Payoneer y no dan explicaciones a los clientes](https://blog.segu-info.com.ar/2024/01/vacian-cuentas-de-payoneer-y-no-dan.html)

[Payoneer accounts in Argentina hacked in 2FA bypass attacks](https://www.bleepingcomputer.com/news/security/payoneer-accounts-in-argentina-hacked-in-2fa-bypass-attacks/)

[Hackearon una billetera digital y vaciaron cuentas en dólares de usuarios argentinos](https://www.infobae.com/economia/2024/01/19/hackearon-una-billetera-digital-y-vaciaron-cuentas-en-dolares-de-usuarios-argentinos/)

[Hackeo a Payoneer en primera persona: el relato de un joven argentino al que le robaron sus ahorros de dos años en dólares](https://www.infobae.com/economia/2024/01/19/hackeo-a-payoneer-en-primera-persona-el-relato-de-un-joven-argentino-al-que-le-robaron-sus-ahorros-de-dos-anos-en-dolares/)
