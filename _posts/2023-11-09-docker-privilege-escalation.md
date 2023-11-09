---
layout: post
section-type: post
title: Root por accidente, escalando privilegios con Docker
category: tech
tags: [ 'security' ]
---

Es casi imposible negar que los Containers se han convertido en uno de los estándares mas aceptados al momento de desplegar aplicaciones en entornos Cloud, con lo cual, no resulta extraño en estos días encontrarnos con un host o server con Docker instalado.

Ahora bien, de la misma manera que Docker nos facilita muchísimas tareas, es interesante considerar que puertas nos abre, desde el punto de vista de la seguridad y las posibilidades de utilizarlo para escalar privilegios.

Pero primero tenemos que entender las bases de que son los Docker Containers y como funcionan.

### Como funcionan los containers?

Explicar en detalle del funcionamiento de los containers da para un buen rato adentrándonos en el funcionamiento del Kernel de Linux, pero para lo que nos importa aquí, lo mas critico a entender es que Docker utiliza un conjunto de herramientas del Kernel para poder aislar uno o varios procesos que corren en el OS host (es decir, que estos procesos no puedan ver otros corriendo en el mismo sistema), aislar el filesystem (es decir, limitar a que partes del filesystem del host los procesos que corren en el container van a poder acceder), y los usuarios (los usuarios, a nivel OS, dentro del container son “independientes” de los usuarios del host); entre otras cosas, como aislar y virtualizar también el espacio de networking, el uso de CGroups para limitar recursos, etc.

Entonces, en una definición muy básica, podríamos decir que un container “es un conjunto de procesos, aislados del resto de los procesos del sistema donde corren”.

Que es lo que nos importa puntualmente en esto? Que dentro del container nosotros podemos definir o usar usuarios diferentes a nuestro usuario en el host, y además podemos montar arbitrariamente directorios del host dentro del filesystem del container.

Es importante aclarar que existen diferencias importantes en como funciona Docker entre Linux, Windows y MacOS, ya que los containers funcionan de forma nativa solo en Linux (en Windows, el funcionamiento nativo es solo si activamos Docker para trabajar con Windows Containers, pero el funcionamiento es bastante diferente); en Windows corren sobre WSL2 o Hyper-V, y en MacOS, dentro de una VM que corre Linux de forma transparente al usuario. Si bien esto no implica que no podamos aprovechar el acceso al daemon de Docker como vector de ataque, el proceso descripto puede variar dependiendo el OS, y algunas cosas pueden no ser posibles.

*En este post pondremos el foco en Docker corriendo en un entorno Linux-based.*

### Que puedo hacer con acceso al Daemon

Simplemente con tener acceso al Daemon de docker, es decir, permisos para correr el comando docker en el host, podemos:

- Ver containers corriendo: esto ya nos abre la puerta a obtener un montón de información sobre lo que tenemos corriendo en ese host.
- Ver log de containers en ejecución o detenidos: poder ver los logs, tanto de containers corriendo, como de containers detenidos pero que no han sido eliminados, también nos puede dar información super interesante acerca de configuraciones, errores, etc.
- Acceder a la config del container: dando un paso mas, con docker inspect, podemos visualizar la configuración de como fue lanzado el container (variables de entorno, volúmenes montados, etc.) de donde podemos obtener desde passwords e info sensible, hasta la ubicación exacta de donde están los datos que utiliza la aplicación/servicio.
- Correr comandos dentro de un container: al tener acceso al daemon, no solo podemos ver, también, en la mayoría de los casos, podemos acceder a una shell dentro del container, lo que nos permite no solo visualizar el contenido del filesystem, sino ejecutar comandos dentro del mismo.
- Lanzar un container: por ultimo, no solo podemos acceder a los containers que están corriendo, también, al tener acceso al daemon, es muy probable que podamos descargar imágenes y/o lanzar un container nuevo, con el setup que queramos.

Ahora bien, pensemos en un escenario de ejemplo, supongamos que somos desarrolladores, que tenemos acceso a un server de prod, y en ese server nuestras apps corren containerizadas, pensemos en que hay una DB y una api, y para poder ver los logs de esa app, nos dieron acceso al Daemon de docker. Quien administraba ese servidor estaba “un poco preocupado” por la seguridad, asi que no nos dio acceso root, ni tenemos acceso a las carpetas donde se encuentran nuestra app, ni los datos de la DB, supongamos que todo esto esta dentro de `/opt/insecure-company/`

<u>Caso 1: acceder a archivos “privados” dentro de un container corriendo</u>

![Untitled](/img/posts/docker-privilege-escalation/caso1-1.png)

Como podemos ver, nuestro usuario no tiene acceso al directorio “privado”, pero si podemos utilizar docker. Con una simple inspección de los containers corriendo, podemos ver que algunos directorios privados están montados dentro del container, con lo cual, puedo entonces acceder, via el container, a información almacenada dentro de esos directorios.

![Untitled](/img/posts/docker-privilege-escalation/caso1-2.png)

![Untitled](/img/posts/docker-privilege-escalation/caso1-3.png)

Como podemos ver, dentro del container somos root, y siguiendo los paths que encontramos con el inspect, podemos ver que estaríamos accediendo a data existente dentro del directorio “privado” (puntualmente a data de `/opt/insecure-company/example-voting-app/db-data/_data` y `/opt/insecure-company/example-voting-app/healthchecks`) aun sin tener permiso para acceder a ellos con nuestro usuario a nivel host.

<u>Caso 2: lanzar un container montando un volumen con un directorio al que no tengo acceso</u>

Tener acceso al daemon no solo nos permite acceder a containers que ya están corriendo, también podemos lanzar un container nuevo, aprovechando esto podríamos acceder a información extra, ademas de la que ya tendríamos acceso via los containers que están corriendo actualmente.

![Untitled](/img/posts/docker-privilege-escalation/caso2-1.png)

![Untitled](/img/posts/docker-privilege-escalation/caso2-2.png)

Como se puede ver, simplemente lanzando un nuevo container (ya sabemos que la imagen `postgres:15-alpine` tiene las herramientas básicas para recorrer un filesystem, pero al tener acceso al daemon podríamos incluso descargar cualquier otra imagen), podemos acceder a cualquier información que este dentro del host, aprovechándonos del hecho de que los permisos del filesystem validan el user id, pero nosotros dentro del container podemos crear y utilizar usuarios a nuestro criterio (y ser root), no estamos limitados a los usuarios “reales” que tenemos acceso en el host. En las imágenes de ejemplo podemos ver que accedimos al `/etc/shadow` lo que nos permitiría intentar crackear las pass de los usuarios, y ademas pudimos obtener la priv key del usuario `ubuntu` que podría, por ejemplo, darnos acceso a otros hosts.

<u>Caso 3: usar el volume mount para agregarme a sudoers</u>

Los volume mounts no solo nos permiten leer, también podemos escribir archivos.

![Untitled](/img/posts/docker-privilege-escalation/caso3-1.png)

Teniendo un poco de conocimiento de como funcionan los controles de un sistema Linux, podemos aprovechar el hecho de poder montar cualquier volumen para superar dichos controles, tal como vemos en la imagen, con un par de simples comandos, pudimos obtener acceso root al host, siendo que nuestros permisos originales eran bastante limitados.

### Conclusiones

Como podemos ver, el tener acceso al daemon de docker, nos abre las puertas a utilizar como vector de ataque. Aun consiguiendo acceso a una cuenta con “pocos privilegios”, el hecho de tener permisos para acceder y lanzar containers, sumado a un poco de conocimiento sobre como funciona una plataforma Linux-based y un poco de ingenio, nos puede permitir desde acceder a información a la que no se supone que deberíamos tener acceso, hasta quedarnos con el control completo de dicho sistema.

<u>Que consideraciones podríamos tomar para evitar esto</u>

En primer lugar, idealmente solo los admins deberían tener acceso al daemon de docker en entornos críticos. Aunque de todas maneras, ya sabemos que no es un escenario ideal el administrar entornos productivos utilizando solo docker directo sobre los servers.

Si bien existen controles extras que se pueden agregar, y Docker soporta, a modo de plugin, la implementación de authorization ([Docker Authorization Plugin](https://docs.docker.com/engine/extend/plugins_authorization/)), esto no viene implementado por default, con lo cual es algo que si no configuramos, no esta disponible, y cualquier usuario con acceso al daemon tiene control total sobre docker en ese host.

Es importante recalcar que siempre el escenario recomendado para correr workloads containerizados en entornos productivos/críticos es utilizar un orquestador como Kubernetes. Dichas soluciones, no solo nos abstraen de la gestión del host y el daemon que nos permite containerizar procesos, sino que también nos dan sistemas de RBACs y gestión de permisos para definir que puede y que no hacer cada usuario. Por ejemplo, la política por default de K8s es que cualquier usuario nuevo “no tiene acceso a nada, salvo que se indique lo contrario”, es decir, si no le configuro permisos, el usuario ni siquiera va a poder ver que cosas están corriendo dentro del entorno.

Tratar de entender a fondo las herramientas que utilizamos para trabajar en nuestro dia a dia nos permite no solo explotar todas sus ventajas, sino también considerar los posibles riesgos que implica su uso.

La primera vez que vi docker en funcionamiento, allá por el 2015, me genero un gran impacto, creo que fue porque entendí que podia implicar un cambio de paradigma importante. Desde ese momento me empece a interiorizar un poco mas en su funcionamiento, y si bien hoy en dia el entendimiento de Docker como superficie de ataque es de publico conocimiento ([Docker Engine Security](https://docs.docker.com/engine/security/)), haber seguido un camino de razonamiento lógico para poder “descubrir” estas cosas en etapas tempranas de aceptación de estas tecnologías en base a entender el funcionamiento de cada uno de los componentes, fue uno de esos grandes “momentos eureka”.

Si llegaste hasta aca, espero que esta info te haya resultado útil!
