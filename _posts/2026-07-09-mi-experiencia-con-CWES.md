---
layout: post
section-type: post
title: Mi experiencia con CWES
category: cybersec
tags: [ 'HTB', 'CWES', 'Cybersecurity' ]
seo:
  image: '/img/posts/mi-experiencia-con-cwes/cwes.jpeg'
  description: 'Después de 7 días de examen (sí, leíste bien, 7 días de examen), te cuento un poco cómo fue mi experiencia rindiendo el Certified Web Exploitation Specialist de HackTheBox'
---
Hace unos días recibí el resultado del examen de HackTheBox para Certified Web Exploitation Specialist (CWES) y después de varios meses de preparación y días intensos de examen, ¡quedó aprobado! Quería compartirles en este post un poco de lo que fue la experiencia tanto de la preparación como del examen.

![HTB CWES Cert](/img/posts/mi-experiencia-con-cwes/cwes.jpeg)

## Preparación

Mi idea original era rendir Certified Penetration Testing Specialist (CPTS), del cual ya completé todos los módulos de la Academy de HTB, pero como al completar CPTS cubre aproximadamente un 60% de CWES, y el examen de CWES es más corto, decidí pasar primero por esa experiencia, principalmente para enfrentarme al armado del reporte.

La HTB Academy en sí me pareció super bien estructurada, cada módulo teórico tiene ejercicios específicamente diseñados para poner en práctica lo visto en la teoría y eso me pareció genial. En cuanto a nivel, la realidad es que con el background en Dev e Infra previo, no se me hizo difícil progresar en la academy, pero sí puedo decir que no es algo 100% para principiantes, es necesario contar con un conocimiento previo de Networking, HTTP, y algo de programación para poder llevarlo de forma constante y no tener que ir a ver material complementario.

En total, creo que terminaron siendo algunos meses de preparación, considerando que le dedicaba unas 2 o 3 horas por semana.

Una vez terminado el path completo en la academy, lo que hice fue practicar con boxes easy y medium, priorizando los que eran Linux y con Web Apps, y resolví el fortress Akerva que muchos recomiendan como práctica para el examen. Tanto con los boxes como el fortress, fui formando también el hábito de escribir potenciales vulnerabilidades a reportar en conjunto con registrar el paso a paso de la resolución y también cómo reproducir la explotación, para poder usar eso como input luego para armar el reporte.

Para el reporting usé SysReptor con los templates oficiales de HTB. Como nunca había usado esta solución, aproveché la práctica con boxes y el fortress para simular escribir un reporte completo y estar un poco más cómodo también con este proceso antes del examen.

## Examen

Sobre el examen en sí, se simulaba un engagement donde había que testear 5 webapps, para las cuales tenías una indicación aproximada de dónde encontrar los flags (e.g. ganando acceso a un admin panel, o explotando un RCE, etc.). Como parte del scope me dieron un dominio y una IP, que pertenecían al "website público" de la empresa que estábamos testeando en el engagement, sin ningún usuario y sin ningún dato extra. Quedaba como parte de las tareas de la auditoría descubrir las 4 apps restantes y conseguir el acceso inicial. El objetivo era entregar un Final Report detallando las vulnerabilidades encontradas, simulando el entregable que tendría un Pentest real, pero poder presentar el reporte necesitaba tener 80 puntos de 100 que era el total disponible con los 10 flags.

El primer día fue un poco frustrante, logré identificar los 4 sitios restantes y me organicé para enfocarme de a uno a la vez. Completé la enumeración del website público, pude conseguir algunas credenciales pero que no me daban acceso al flag. Después de varias horas y casi llegando al final del primer día, pude dar con el primer flag, y habiéndole dedicado ya unas 5hs el primer día, lo dejé ahí.

Al día siguiente avancé con un poco más de ritmo, logré conseguir en poco más de una hora el 2do flag del website y pasé a la siguiente aplicación. El acceso inicial en la 2da app lo conseguí más rápido, con info que había obtenido de la enumeración del website público, así que unos minutos después ya tenía el 3er flag (y ya estaba un poco más animado también). Antes de terminar el 2do día logré acceso al 4to y 5to flag, después de dedicarle algo más de 7 hs.

El tercer día fue de poco más de 5 hs de full foco en el examen, y terminé el día con 8 flags. Como cada flag entregaba un puntaje diferente, al final del tercer día tenía 70 puntos, con lo cual necesitaba al menos un flag más para poder cubrir el requisito mínimo de la parte práctica.

Arranqué el 4to día con full foco en la última web app que me quedaba por explotar y que tenía los 2 flags restantes, y antes de llegar al mediodía, después de poco más de 2 horas encontré el primero, con lo que tenía la práctica cubierta. El plan era cortar para almorzar y retomar ya directamente empezando con el reporte, pero durante el almuerzo me quedé pensando algunos posibles testeos y a la vuelta le dediqué unas casi 3 horas más y logré acceso al último flag. A este punto ya tenía 100/100 de la práctica cubierta, así que empecé a enfocarme en el reporte.

Con el mismo mecanismo con el que había estado practicando con los boxes, a medida que avanzaba con las pruebas fui tomando notas bastante detalladas de lo que intentaba, qué funcionaba y qué no, capturas de pantalla y output de los comandos que corría en la terminal, en conjunto con "potenciales vulns". Al terminar con la práctica tenía un listado de 21 vulns potenciales a reportar, pero antes de ponerme a escribir en detalle todas, le di una mirada más fina al scope (que te lo dan como parte del "enunciado" del examen, simulando con esto también un pentest real), y descarté 2 vulns que quedarían fuera del alcance solicitado del engagement y puse el foco en escribir las 19 restantes.

Si bien al momento de entregarte el material para el examen te brindan un .doc con el formato del reporte, en mi caso había de antemano configurado la instancia con SysReptor y los templates, así que fui completando usando la tool todos los campos pendientes del cuerpo del reporte más la especificación de cada vuln. En ningún momento desde el lado de HTB te ponen una limitación en cuanto al uso de IA, en mi caso lo que hice fue de antemano armarme un skill con guidelines para corregir las vulnerabilidades, teniendo en cuenta algunos patrones que tomé del módulo de Documentation & Reporting propio de la HTB Academy, más algunas cosas extra que fui viendo en foros o comentarios de otras personas que habían rendido ya el examen.

El flujo para armar el reporte fue entonces: escribía un primer draft de forma 100% manual, con el fin de que las vulns tengan "mi voz" y no sea un texto automático producido por una IA, y una vez que estaba la vuln completa con todos los campos y el step-by-step para reproducirla, hacía una primera revisión manual yo mismo. Seguí este proceso con las 19 vulns, lo que me llevó, adjuntando los screenshots y volviendo a repetir en algunos casos el attack chain para poder sacar alguna captura extra o especificar con mayor detalle los pasos de la explotación, aproximadamente entre 1 a 2 horas por cada vuln. Regla de 3 simple, aprox unas 20 hs para el primer draft del reporte, que se dividieron más o menos en 2 horas al final del 4to día, algo más de 8 horas el 5to día, y unas 9 horas del 6to día.

El 7mo día fue el más intenso, le dediqué unas 12 horas en total, desde la mañana hasta las 2:30 AM que entregué el Final Report. Las 12 horas fueron de corregir vuln por vuln usando Claude con la skill, ajustando los detalles que sugería en cada una. Un punto clave que vale la pena mencionar es que la skill no reescribía ningún campo de la vulnerabilidad, sino que sugería mejoras y el motivo de por qué, nuevamente con el fin de mantener mi propia voz en la escritura, con lo cual no era que "las pasaba por la IA y listo", sino que revisé el output de cada corrección, en base a eso apliqué a criterio propio lo que realmente me parecía relevante mejorar, y volvía a iterar con la corrección de la nueva versión. En solo un caso hice más de una iteración del proceso.

Una vez que terminé con la corrección de todas las vulns, me tomé poco más de una hora para escribir y corregir el Executive Summary, donde detallé a alto nivel los findings y agregué una recomendación de qué sería mejor priorizar para mitigar primero. Sumé también algunos detalles extras sobre el pentest en sí y buenas prácticas generales para mejorar el proceso de software development de la empresa ficticia, ya que la mayoría de las vulnerabilidades estaban relacionadas a fallas en la implementación.

## Resultado

Una vez hecho el submit del final report, directamente te bloquean el acceso al environment y al enunciado, lo único que queda es un mensaje de que está en proceso de corrección y que pueden tardar hasta 20 días hábiles en dar una respuesta. Por suerte para mi ansiedad, en 4 días tuve la corrección, donde me dieron un feedback bastante detallado de la corrección del final report e incluso algunas sugerencias de cosas que se podrían mejorar en la escritura de vulns, confirmando que es verdadero lo que comentan varios, que son bastante exigentes con el Final Report, y que efectivamente se toman el tiempo de leerlo completo, ya que las sugerencias eran bastante específicas a mi reporte en sí.

Cabe aclarar que los primeros 4 días fueron durante semana y tuve la oportunidad de estar sin tareas durante esos días, con lo que me pude dedicar prácticamente con full foco al examen en horario laboral. Los 3 días siguientes fueron fin de semana largo por un feriado, así que también me dediqué con full foco a esto.

Si llegaste hasta acá y estás buscando algún consejo sobre el examen, creo que el principal que puedo dar es que no subestimes el tiempo que demanda la escritura de las vulns y el armado del reporte. La etapa de explotación fue bastante entretenida, por momentos algo frustrante, pero como todo pentest real, con sus altibajos hasta que encontrás una vuln, y sentís esa inyección de adrenalina que te lleva a seguir buscando las que siguen ;). Por otro lado, son 7 días de examen, y en mi caso, usé los 7 días a un ritmo de, podría decir, casi trabajo normal, un promedio de 8 horas diarias, que luego en la práctica fueron días un poco más tranquilos y otros bastante más intensos, pero creo que eso también es importante para destacar, poder encontrar un balance para no quemarte los primeros días, porque el escenario está preparado para ocupar el tiempo que te dan.
