---
layout: post
section-type: post
title: Subdomain Takeover by experience
category: tech
tags: [ 'security' ]
---

![Hoodie](/img/posts/subdomain-takeover-by-experience/hoodie.png)

Un día me pasaron como tarea hacer limpieza de algunos recursos que ya no estaban en uso en la cuenta de Azure de una empresa en la que trabajaba, era un proyecto que había iniciado hacia unos meses atrás, y por cuestiones de política de la empresa había quedado parado un tiempo, con lo cual algunos recursos se apagaron y dieron de baja. Luego el proyecto se retomó con otro foco, con lo cual varias de las cosas que se habían creado en la nube estaban totalmente obsoletas. Dentro de las tareas, una de ellas era analizar que subdominios del dominio principal de la empresa, que se habían usado para el proyecto, ya no estaban más en uso.

Luego de revisar algunos documentos y el código de algunas de las apps que se habían desplegado y ya no se iban a utilizar, arme una lista de dominios a validar, y empecé a chequear uno por uno a donde estaban dirigidos, algunos eran registros A, otros eran alias CNAME a otros registros A del mismo dominio, y otros alias a dominios que crea Azure al crear algunos tipos de recursos.

La forma más fácil se chequear si había algo en uso era hacer un request (todos los dominios eran usados en WebApps o APIs vía https) y verificar las respuestas: si no había respuesta, el dominio no estaba en uso, si había respuesta, tocaba validar de que se trataba con el equipo para chequear si seguía siendo necesario ese servicio. Todo iba “normal” hasta que llegue a un dominio que me dio una respuesta bastante alejada de lo que esperaba, un sitio estático, en un idioma distinto a los que soportan todos los sitios de la empresa (el sitio estaba en idioma Indonesio), revisé el sitio más en detalle e inmediatamente pensé:

![You have been hacked](/img/posts/subdomain-takeover-by-experience/you-have-been-hacked.png)

Empecé a rastrear los servers que estaban originalmente detrás de ese dominio, y para mí sorpresa: *estaban apagados*. Como podía entonces uno de los sitios haber sufrido un ataque, si el server ni siquiera estaba encendido? Había algo que no estábamos teniendo en cuenta.

Vuelvo a revisar el sitio a ver si encontraba algo más que me de indicios de que estaba pasando, en principio todo el funcionamiento del sitio era "normal", había un redirect de http a https, tal como están todos los sitios que manejábamos usualmente configurados, el certificado del sitio era válido, generado por Let's Encrypt (servicio que utilizábamos) para ese dominio, todo tenía sentido... Reviso entonces el dominio: era un CNAME a un dominio autogenerado por Azure a uno de nuestros recursos…

Y de repente la cosa tomó sentido, el CNAME apuntaba a un dominio autogenerado por Azure (del tipo `*.eastasia.cloudapp.azure.com`), que toma el nombre del recurso, que… habíamos eliminado!

Alguien había encontrado que teníamos un dominio con un alias configurado a un dominio que ya no existía, pero que era gestionado por Azure y generado en base al nombre del recurso, lo único que tuvieron que hacer fue desde otra cuenta de Azure, generar un recurso con el mismo nombre que tenía el que nosotros habíamos eliminado para que el dominio original volviera a existir y bum! ***Domain Pwned!*** A partir de ese momento tenían control de todo lo que estuviera bajo ese subdominio, incluyendo la posibilidad, por ejemplo, de generar un certificado válido para el mismo usando un challenge http para validar el control sobre el dominio. Este tipo de ataques se denomina “Subdomain Takeover”.

La *solución* al problema fue bastante simple una vez que encontramos ésto, eliminamos el subdominio de la zona DNS del dominio de la empresa, y como ya el registro no existía, no podían tener más control del contenido de ese subdominio.

Simplemente por curiosidad empecé a buscar algunas palabras clave del sitio que habíamos visto alojado en nuestro dominio, y, ya a esa altura no fue una gran sorpresa habiendo descubierto como habían tomado el control del dominio, encontré múltiples réplicas del mismo sitio en otros dominios, que por lo que pude verificar habían sido “secuestrados” usando la misma metodología.

Lo más importante a entender de todo esto es, creo, la posibilidad de sufrir un ataque de tal implicancia, de una forma tan, en principio, sencilla.

Obviamente que requiere la ocurrencia de varios factores, el principal es que la empresa había “descuidado” un subdominio, dejándolo activo con un alias apuntando a “nada” pero donde esa “nada” pudo ser controlada de forma bastante trivial por el atacante. Y por otro lado requiere, del lado del atacante, estar escaneando constantemente miles de dominios en internet hasta dar con alguno mal configurado y vulnerable a este tipo de ataque. Pero hago hincapié nuevamente en las implicancias, ya que de forma bastante sencilla el atacante puede tomar control de un dominio válido de la empresa, y usarlo a su criterio.

En este caso el sitio que fue hosteado no tenía nada que ver con el negocio, pero no hubiera sido muy difícil, una vez adquirido el control del dominio, hostear por ejemplo una réplica de un form de login de la empresa y usarlo en una campaña de phishing para robar credenciales de usuarios, que a simple vista, estarían cargando sus credenciales en un subdominio de la empresa, **lo cual casi no levantaría sospechas!** si es un subdominio de la empresa.

Básicamente los vectores de ataque que este tipo de situaciones abre, está sujeto casi exclusivamente a la creatividad del atacante, y entre otras cosas podría aprovechar el control de un subdominio valido para: robar cookies, cross-site request forgery (CSRF), abusar CORS, y saltar content security policies (CSP), sin contar, por supuesto, las posibles consecuencias en la "confianza" a la empresa que sufre este tipo de ataques si el dominio es utilizado para mostrar algún mensaje en contra de la empresa o incluso simplemente contenido que no tenga que ver con el negocio.

### Posibles mitigaciones:

- **Eliminar subdominios no utilizados:** Es crucial revisar y eliminar subdominios que ya no están en uso. Si un servicio no se está utilizando, el subdominio asociado debería ser desvinculado, y esto ya reduciría muchísimo la superficie de vulnerabilidad, ya que como mencionamos anteriormente, es clave encontrar un subdominio con una configuracion vulnerable para poder explotarlo.
- **Vigilancia constante:** Implementar soluciones que monitoreen activamente la infraestructura digital en busca de cambios no autorizados o cambios en el contenido de los sitios accesibles públicamente. También podemos validar si todos los registros de nuestro dominio contienen configuraciones validas, en caso de encontrar alguno que no, alertar, porque podría ser potencialmente vulnerable a este tipo de ataques.

Es clave entender que la Ciberseguridad de las empresas no está sujetas solo a un simple escaneo de vulnerabilidades y mantener al día los antivirus y actualizaciones. Los vectores que los atacantes pueden utilizar son múltiples, y están, cada vez más, con el foco puesto en los eslabones mas débiles de la cadena: explotar el factor humano y/o las fallas en la configuración.
