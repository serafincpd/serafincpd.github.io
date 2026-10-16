---
layout: post
section-type: post
title: Resolviendo Microcorruption
category: reversing
tags: [ 'Reversing', 'Cybersecurity' ]
seo:
  image: '/img/posts/microcorruption/tutorial-ui.png'
  description: 'Hace unos días decidí desempolvar un poco los skills de Reversing, y encontré un challenge estilo CTF que me pareció interesante y a la vez entretenido, así que en base a las notas que fui tomando de la resolución de los primeros niveles, armé este post para compartir un poco de qué va.'
---
Hace unos días decidí desempolvar un poco los skills de Reversing, y encontré un challenge estilo CTF que me pareció interesante y a la vez entretenido, así que en base a las notas que fui tomando de la resolución de los primeros niveles, armé este post para compartir un poco de qué va.

### ¿Qué es Microcorruption?

Microcorruption es un CTF gratuito de reverse engineering que corre 100% en el navegador. La idea: sos un hardware hacker conectado vía JTAG a una serie de cerraduras digitales "Lockitall". El objetivo es bypasear el chequeo de password de la cerradura leyendo el firmware, entendiendo la lógica, y encontrando la condición de desbloqueo.

No es necesario que instales nada, simplemente con acceder a la web te dan una UI de debugger con disassembly, estado de registros, memory dump y consola de debug.

A diferencia de la mayoría de los CTFs de reversing que apuntan a x86 o ARM, Microcorruption se basa en el MSP430 de Texas Instruments, un microcontrolador de 16 bits que suele usarse en dispositivos IoT, redes de sensores y sistemas embebidos.

### ¿Cómo es el escenario del CTF?

La empresa ficticia "Lockitall" fabrica la serie LockIT Pro de cerraduras electrónicas. Cada nivel presenta una nueva revisión de software de la cerradura, con el chequeo de password volviéndose un poco más sofisticado. La cerradura también tiene un periférico "HSM-1" (Hardware Security Module) con el que podés interactuar, dependiendo el nivel.

Cada nivel incluye un bloque de contexto describiendo la versión de la cerradura, qué cambió desde la revisión anterior, y (a veces) una pista sobre qué intentaba arreglar el desarrollador.

El workflow es bastante realista:

1. Abrir el nivel, leer el contexto
2. Mirar el código de disassembly de `check_password` (o la función equivalente)
3. Setear un breakpoint al entry de la función: `b check_password`
4. Correr con un password de prueba
5. Inspeccionar el estado de los registros en el breakpoint
6. Step-through de la función, observando registros y memoria
7. Razonar sobre la lógica: qué hace cada instrucción, cuál es la condición de éxito
8. Encontrar la vulnerabilidad en la lógica y explotarla

Una vez que explotamos la vulnerabilidad y obtenemos la forma de abrir la cerradura, pasamos al siguiente nivel.

### Resolviendo los niveles

Te dejo acá las notas que fui tomando y algunos aprendizajes interesantes durante el proceso, no voy a dejar la solución final de cada nivel, pero en la mayoría de los casos siguiendo los pasos también los vas a poder resolver!

#### Tutorial

El tutorial en sí contiene un step-by-step que explica desde cómo utilizar la plataforma hasta llegar a la solución de este nivel. Al acceder a cada nivel nos vamos a encontrar con una interfaz similar a la siguiente:

![Tutorial UI](/img/posts/microcorruption/tutorial-ui.png)

Acá podemos ver en la parte superior izquierda la ventana de "Disassembly", que nos muestra el código cargado en el dispositivo, por debajo tenemos el "Live Memory Dump" que muestra el estado actual de la memoria, y finalmente del lado derecho tenemos los registros de la CPU y la "Debugger Console" que es donde vamos a interactuar con los comandos.

El primer paso para poder resolver este primer desafío, es poner el foco en la función que hace el check del password:

```nasm
4484 <check_password>
4484:  6e4f           mov.b	@r15, r14
4486:  1f53           inc	r15
4488:  1c53           inc	r12
448a:  0e93           tst	r14
448c:  fb23           jnz	$-0x8 <check_password+0x0>
448e:  3c90 0900      cmp	#0x9, r12
4492:  0224           jz	$+0x6 <check_password+0x14>
4494:  0f43           clr	r15
4496:  3041           ret
4498:  1f43           mov	#0x1, r15
449a:  3041           ret
```

Seteamos un breakpoint en esta función con `b check_password` en la debugger console, y ejecutamos con `s` y luego `c` para continuar hasta el breakpoint o la primera interrupción. Vemos ahí que nos da un input para poner el password, así que ingresamos un password de prueba, por ejemplo `test` y le damos a "Send". Instantáneamente se va a interrumpir la ejecución cada vez que hagamos un input, pero como no tenemos interés por ahora en debuggear el cómo maneja el input, le damos `c` para continuar hasta el breakpoint, ya que lo que nos interesa es chequear cómo valida la password. Ingresamos entonces a la función que nos interesa, donde vemos que la primera instrucción es `mov.b @r15, r14`, lo que lee el byte de memoria en la dirección correspondiente a lo que tiene almacenado el `r15` y lo guarda en `r14`, vamos a chequear el estado de los registros entonces:

![Tutorial Reg State 1](/img/posts/microcorruption/tutorial-reg-state-1.png)

Tenemos entonces que `r15 = 0x439c`, entonces nos interesa lo que hay en esa posición de memoria, vamos a chequear en el Live Memory Dump:

![Tutorial Mem Dump 1](/img/posts/microcorruption/tutorial-mem-dump-1.png)

Vemos entonces que `r15` "apunta" a la primera posición del lugar donde se almacenó en memoria el password que ingresamos. Entonces lo esperado sería ver en `r14 = 0074` luego de ejecutar esta instrucción. Le damos `s` para avanzar un step y vemos el estado de los registros en la siguiente instrucción:

![Tutorial Reg State 2](/img/posts/microcorruption/tutorial-reg-state-2.png)

Tenemos entonces el valor del primer carácter que ingresamos cargado en `r14` tal como era esperado, observando las siguientes operaciones vemos que:

```nasm
4484 <check_password>
4484:  6e4f           mov.b	@r15, r14
4486:  1f53           inc	r15                             <---- Estamos acá!
4488:  1c53           inc	r12
448a:  0e93           tst	r14
448c:  fb23           jnz	$-0x8 <check_password+0x0>
448e:  3c90 0900      cmp	#0x9, r12
4492:  0224           jz	$+0x6 <check_password+0x14>
4494:  0f43           clr	r15
4496:  3041           ret
4498:  1f43           mov	#0x1, r15
449a:  3041           ret
```

El próximo paso es incrementar `r15`, que en la instrucción anterior se usó como índice para leer el primer carácter ingresado, con lo cual estaríamos preparando el índice para leer el siguiente. Luego se incrementa `r12`, que estaba en `0x0000` por lo que pasaría a `0x0001`, y finalmente se evalúa si `r14` (que es donde guardamos el byte leído) es `0x0000`, o sea, si ya pasamos el último carácter. `jnz` va a saltar hacia atrás a `<check_password+0x0>`, es decir, al inicio de la función nuevamente, hasta que hayamos leído `0x0000`, lo que implica que va a iterar con la misma lógica hasta leer el último carácter de la "clave" ingresada. Siguiendo esta lógica, el valor de los registros va a ser: `r15` va a estar apuntando 1 posición más del último carácter (el primer null byte encontrado), `r12` va a tener el recuento de caracteres leídos + 1 (porque vamos a estar posicionados en el null byte), y `r14` va a estar en `0x0000`. Cuando lleguemos a este estado, `jnz` no va a realizar el jump, de modo que pasaremos a la instrucción siguiente, es decir `cmp	#0x9, r12`.

En este punto vamos a estar evaluando entonces si `r12` (nuestro contador de caracteres) es igual a 9. Si es igual, el flag Z se va a setear en 1, por lo que el siguiente `jz` se va a ejecutar para pasar a `<check_password+0x14>`, o sea a `mov	#0x1, r15` y terminar la ejecución de la función de forma satisfactoria, de lo contrario se va a setear `r15` en 0 antes de terminar. Vamos a ver más detalles sobre las calling convention más adelante, pero por ahora lo importante a entender es que el `r15` se utiliza para el valor de retorno de la función, en este caso devolviendo `0x1` (equivalente a True) si la validación es positiva, o `0x0` (equivalente a False) en caso contrario.

Teniendo en cuenta toda esta lógica, podríamos decir que la forma de que podamos abrir esta cerradura es lograr que el `r12` termine con el valor 9 luego de leer la clave, por lo tanto cualquier clave de 8 caracteres debería ser aceptada. Probamos nuevamente con `12345678` por ejemplo y logramos abrir la cerradura.

Para completar el nivel escribimos `solve` en la debug console y nos va a dejar ingresar el password contra la "puerta real", con lo que abrimos la cerradura y pasamos el nivel de tutorial!

#### Level 1 - New Orleans

Contexto:
```tasm
DETAILS

The LockIT Pro a.01  is the first of a new series  of locks. It is
controlled by a  MSP430 microcontroller, and is  the most advanced
MCU-controlled lock available on the  market. The MSP430 is a very
low-power device which allows the LockIT  Pro to run in almost any
environment.

The  LockIT  Pro   contains  a  Bluetooth  chip   allowing  it  to
communiciate with the  LockIT Pro App, allowing the  LockIT Pro to
be inaccessable from the exterior of the building.

There is  no default password  on the LockIT  Pro---upon receiving
the LockIT Pro, a new password must be set by connecting it to the
LockIT Pro  App and  entering a password  when prompted,  and then
restarting the LockIT Pro using the red button on the back.

This is Hardware  Version A.  It contains  the Bluetooth connector
built in, and one available port  to which the LockIT Pro Deadbolt
should be connected.

This is Software Revision 01.
```

Empezamos como en el tutorial por inspeccionar el código en el Disassembly view, y vemos que tenemos nuevamente una función de `check_password`, pero esta vez con una lógica diferente:

```nasm
44bc <check_password>
44bc:  0e43           clr	r14
44be:  0d4f           mov	r15, r13
44c0:  0d5e           add	r14, r13
44c2:  ee9d 0024      cmp.b	@r13, 0x2400(r14)
44c6:  0520           jnz	$+0xc <check_password+0x16>
44c8:  1e53           inc	r14
44ca:  3e92           cmp	#0x8, r14
44cc:  f823           jnz	$-0xe <check_password+0x2>
44ce:  1f43           mov	#0x1, r15
44d0:  3041           ret
44d2:  0f43           clr	r15
44d4:  3041           ret
```

Antes de analizar la lógica, corremos el código para poder chequear qué valores toman en runtime cada uno de los registros. Creamos un breakpoint en la función de chequeo con `b check_password`, y luego hacemos `c` para dar con el prompt de contraseña. Ingresamos una contraseña cualquiera, por ejemplo, `password`, ya que nuestro objetivo por ahora es ver cómo se ejecuta la función, y una vez que le damos "Send", hacemos `c` nuevamente en la Debug Console. Llegamos a la primera instrucción de la función con el siguiente estado en los registros:

![New Orleans Reg State 1](/img/posts/microcorruption/new-orleans-reg-state-1.png)

De forma similar a cómo vimos en el caso anterior, tenemos en `r15` la posición de memoria donde está el valor que ingresamos nosotros como contraseña. Si analizamos ahora la lógica de la función de `check_password`, tenemos que el primer paso es poner en `0x0000` a `r14`, luego copiar el valor de `r15` en `r13` y luego realizar `r14+r13` y guardarlo en `r13`. La instrucción siguiente, compara el byte almacenado en la dirección a la que apunta `r13`, con la dirección que resulta de `0x2400+r14`, por lo tanto, esto sería, comparar el primer carácter de lo que ingresamos, con el valor almacenado en `0x2400`.

![New Orleans Mem Dump 1](/img/posts/microcorruption/new-orleans-mem-dump-1.png)

Vemos en el memory dump que los valores no coinciden, por lo tanto en esta primera ejecución la comparación va a fallar. La siguiente instrucción es un `jnz`, al ser distintos los valores comparados, esta instrucción va a ejecutar el jump porque el flag Z va a ser 0, por lo tanto nos iríamos a `0x44d2`, y terminaría la ejecución de la función. Ya tenemos un indicio entonces de que el primer valor del password debería ser una `d` según lo que vimos hasta ahora.

Ejecutamos nuevamente con `dassword` como contraseña de prueba, para que la primera comparación sea satisfactoria. Ejecutamos step by step hasta que llegamos nuevamente al `cmp.b` y al `jnz` y vemos que al pasar la comparación, la siguiente instrucción es `inc r14` lo que incrementaría `r14` en 1 (considerando el estado actual pasaríamos a tener `r14 = 0x0001`) y luego evalúa si `r14` es igual a `0x8`, como no es igual, el `jnz $-0xe <check_password+0x2>` se ejecuta, y volvemos a la posición `44be`. Si seguimos la lógica, comparamos ahora el 2do byte ingresado, con el valor almacenado en `0x2401`. Por lo tanto, ya tenemos la lógica completa, se va a realizar esta comparación hasta que `r14` tenga el valor `0x8`, es decir, vamos a estar comparando los primeros 7 bytes de lo que ingresamos como contraseña, con los valores almacenados en `0x2400-0x2407`, por lo tanto chequeando el memory dump, los 7 bytes de contraseña deberían ser los 7 bytes almacenados en esas posiciones. Hacemos `reset` en la Console, y probamos ingresar esta contraseña, validando que es la correcta para completar el nivel.

Para terminarlo, damos `solve` en la Debug Console, e ingresamos la contraseña que acabamos de descubrir.

#### Level 2 - Sydney

Contexto:
```tasm
DETAILS

The LockIT Pro a.02  is the first of a new series  of locks. It is
controlled by a  MSP430 microcontroller, and is  the most advanced
MCU-controlled lock available on the  market. The MSP430 is a very
low-power device which allows the LockIT  Pro to run in almost any
environment.

The  LockIT  Pro   contains  a  Bluetooth  chip   allowing  it  to
communiciate with the  LockIT Pro App, allowing the  LockIT Pro to
be inaccessable from the exterior of the building.

There is  no default password  on the LockIT  Pro---upon receiving
the LockIT Pro, a new password must be set by connecting it to the
LockIT Pro  App and  entering a password  when prompted,  and then
restarting the LockIT Pro using the red button on the back.

This is Hardware  Version A.  It contains  the Bluetooth connector
built in, and one available port  to which the LockIT Pro Deadbolt
should be connected.

This is  Software Revision 02.  We have received reports  that the
prior  version of  the  lock was  bypassable  without knowing  the
password. We have fixed this and removed the password from memory.
```

Siguiendo la misma lógica que venimos usando hasta ahora, inspeccionamos el código de `check_password` y vemos que otra vez el código es diferente:

```nasm
448a <check_password>
448a:  bf90 2f47 0000 cmp	#0x472f, 0x0(r15)
4490:  0d20           jnz	$+0x1c <check_password+0x22>
4492:  bf90 637a 0200 cmp	#0x7a63, 0x2(r15)
4498:  0920           jnz	$+0x14 <check_password+0x22>
449a:  bf90 5441 0400 cmp	#0x4154, 0x4(r15)
44a0:  0520           jnz	$+0xc <check_password+0x22>
44a2:  1e43           mov	#0x1, r14
44a4:  bf90 3a34 0600 cmp	#0x343a, 0x6(r15)
44aa:  0124           jz	$+0x4 <check_password+0x24>
44ac:  0e43           clr	r14
44ae:  0f4e           mov	r14, r15
44b0:  3041           ret
```

Con el mismo proceso, seteamos un breakpoint en la función con `b check_password` y al ejecutar ingresando `password` nuevamente como contraseña de prueba, llegamos otra vez a un estado similar, `r15` con un puntero a la dirección de memoria donde está la contraseña que ingresamos.

Chequeando la lógica de la función, vemos ahora que se ejecuta directamente un `cmp` (a diferencia de los anteriores que eran `cmp.b` ahora estamos trabajando con 16 bits en lugar de con 8) de `0x472f` con el valor almacenado en `0x0(r15)`, es decir, los primeros 2 bytes de lo que ingresamos. Es decir, el valor esperado es `0x472f` para los primeros 2 bytes. Dado que el input de contraseña nos permite definir directamente los valores en hexa, probamos cargar como contraseña `472f` y ejecutamos nuevamente. Sin embargo, vemos que la comparación falla! Si investigamos un poco, vamos a encontrar que el microcontrolador MSP430 utiliza una arquitectura little-endian exclusivamente, con lo cual el word `0x472f` en memoria se representaría como `2f 47`, por lo tanto, dado que los caracteres que ingresamos se guardan en memoria, debemos ingresarlos en este orden. Probamos nuevamente entonces ingresar como hex input de la contraseña `2f47` y pasamos el primer `jnz`.

Siguiendo la lógica del programa, la comparación se realiza tomando los primeros 8 bytes consecutivos, por lo tanto para poder llegar a obtener la contraseña en hex, revisamos las comparaciones que siguen, las ordenamos de a pares de bytes siguiendo el orden little-endian y vamos a poder pasar este nivel.

#### Level 3 - Hanoi

Contexto:
```tasm
DETAILS

The LockIT Pro b.01  is the first of a new series  of locks. It is
controlled by a  MSP430 microcontroller, and is  the most advanced
MCU-controlled lock available on the  market. The MSP430 is a very
low-power device which allows the LockIT  Pro to run in almost any
environment.

The  LockIT  Pro   contains  a  Bluetooth  chip   allowing  it  to
communiciate with the  LockIT Pro App, allowing the  LockIT Pro to
be inaccessable from the exterior of the building.

There  is no  default  password  on the  LockIT  Pro HSM-1.   Upon
receiving the  LockIT Pro,  a new  password must  be set  by first
connecting the LockitPRO HSM to  output port two, connecting it to
the LockIT Pro App, and entering a new password when prompted, and
then restarting the LockIT Pro using the red button on the back.

LockIT Pro Hardware  Security Module 1 stores  the login password,
ensuring users  can not access  the password through  other means.
The LockIT Pro  can send the LockIT Pro HSM-1  a password, and the
HSM will  return if the password  is correct by setting  a flag in
memory.

This is Hardware  Version B.  It contains  the Bluetooth connector
built in, and two available  ports: the LockIT Pro Deadbolt should
be  connected to  port  1,  and the  LockIT  Pro  HSM-1 should  be
connected to port 2.

This is Software Revision 01,  allowing it to communicate with the
LockIT Pro HSM-1
```

En este nivel ya el enunciado nos aclara que tenemos un dispositivo extra conectado en un puerto, que va a ser el encargado de validar si la password ingresada es correcta o no, con lo cual de antemano debemos considerar que la lógica puede diferir bastante con lo que veníamos viendo hasta ahora.

En principio, ya el nombre de la función es diferente, así que seteamos un breakpoint en la nueva función con `b test_password_valid`. El mensaje para ingresar el password nos dice que debe tener entre 8 y 16 caracteres, así que ingresamos como prueba `1234567890123456` de 16 caracteres para evaluar cómo se comporta la función.

Al entrar a `test_password_valid`, vemos que `r15` contiene `0x2400`, que inspeccionando el Live Memory Dump, vemos que se corresponde al primer byte de la contraseña que ingresamos, que ocupa los 16 bytes siguientes. La lógica de la función es:

```nasm
4454 <test_password_valid>
4454:  0412           push	r4
4456:  0441           mov	sp, r4
4458:  2453           incd	r4
445a:  2183           decd	sp
445c:  c443 fcff      mov.b	#0x0, -0x4(r4)
4460:  3e40 fcff      mov	#0xfffc, r14
4464:  0e54           add	r4, r14
4466:  0e12           push	r14
4468:  0f12           push	r15
446a:  3012 7d00      push	#0x7d
446e:  b012 7a45      call	#0x457a <INT>
4472:  5f44 fcff      mov.b	-0x4(r4), r15
4476:  8f11           sxt	r15
4478:  3152           add	#0x8, sp
447a:  3441           pop	r4
447c:  3041           ret
```

La lógica de la función en este caso, cambia bastante a lo que veíamos anteriormente, e introduce varios conceptos extras: manejo de stack, calling conventions, reservar espacio en el stack para variables. Para entender bien la lógica, vamos a ir evaluando cada porción de código en base a su función:

```nasm
4454:  0412           push	r4
4456:  0441           mov	sp, r4
4458:  2453           incd	r4
445a:  2183           decd	sp
```

Esta primera parte de la función se encarga de configurar la pila: primero, guarda en el stack el valor actual de `r4`, luego copia el valor de `sp` (stack pointer - es el puntero a la dirección de memoria donde está el último valor del stack) a `r4`, para usarlo como puntero, luego incrementa en 2 el valor de `r4` y decrementa en 2 el de `sp`, con esto crea un "Stack Frame" para almacenar valores locales.

El paso siguiente `mov.b	#0x0, -0x4(r4)` inicializa en 0x00 el espacio de memoria para la variable que acaba de reservar.

El segmento siguiente se encarga de preparar el llamado a la función, pasando los parámetros via el stack como calling convention:

```nasm
4460:  3e40 fcff      mov	#0xfffc, r14
4464:  0e54           add	r4, r14
4466:  0e12           push	r14
4468:  0f12           push	r15
446a:  3012 7d00      push	#0x7d
446e:  b012 7a45      call	#0x457a <INT>
```

El primer set de instrucciones, guarda `0xfffc` (que es el equivalente a -4) en `r14` y luego con el `add r4, r14` guarda en `r14` el puntero a la variable que acabamos de inicializar, luego carga en el stack el valor de `r14` (que es el puntero a la variable), el de `r15` (que es el puntero a la contraseña que ingresamos en memoria) y el valor `0x7d` pasando estos 3 parámetros a la función `<INT>` que emula el llamado al dispositivo externo que valida la contraseña. Y luego tenemos el siguiente segmento:

```nasm
4472:  5f44 fcff      mov.b	-0x4(r4), r15
4476:  8f11           sxt	r15
4478:  3152           add	#0x8, sp
447a:  3441           pop	r4
447c:  3041           ret
```

Finalmente, cuando vuelve de la función, guarda el valor de la "variable" (referenciada con `-0x4(r4)`) en `r15` para retornarlo como resultado de la función y luego hace `sxt r15`, que toma el bit más significativo del byte y lo extiende a los 16 bits del registro (es decir, si el bit 7 es 1, los bits de 8 a 15 se ponen en 1, si es 0, los bits 8 a 15 se setean en 0), esto se utiliza para convertir un número negativo de 8 bits en uno de 16 bits. Luego, con `add #0x8, sp` estamos liberando del stack el espacio reservado al principio, y finalmente hace `pop r4` para recuperar el valor de `r4` que había mandado al stack al inicio de la función.

Ahora bien, de lo visto hasta ahora no tenemos nada que podamos manipular, así que toca seguir observando la lógica luego de completar `test_password_valid`. Para eso vamos a poner el foco en la sección de código que llama a la función, y ver qué operaciones realiza una vez obtenida la respuesta de la llamada.

```nasm
4520 <login>
4520:  c243 1024      mov.b	#0x0, &0x2410
4524:  3f40 7e44      mov	#0x447e "Enter the password to continue.", r15
4528:  b012 de45      call	#0x45de <puts>
452c:  3f40 9e44      mov	#0x449e "Remember: passwords are between 8 and 16 characters.", r15
4530:  b012 de45      call	#0x45de <puts>
4534:  3e40 1c00      mov	#0x1c, r14
4538:  3f40 0024      mov	#0x2400, r15
453c:  b012 ce45      call	#0x45ce <getsn>
4540:  3f40 0024      mov	#0x2400, r15
4544:  b012 5444      call	#0x4454 <test_password_valid>
4548:  0f93           tst	r15
454a:  0324           jz	$+0x8 <login+0x32>
454c:  f240 6200 1024 mov.b	#0x62, &0x2410
4552:  3f40 d344      mov	#0x44d3 "Testing if password is valid.", r15
4556:  b012 de45      call	#0x45de <puts>
455a:  f290 b700 1024 cmp.b	#0xb7, &0x2410
4560:  0720           jnz	$+0x10 <login+0x50>
4562:  3f40 f144      mov	#0x44f1 "Access granted.", r15
4566:  b012 de45      call	#0x45de <puts>
456a:  b012 4844      call	#0x4448 <unlock_door>
456e:  3041           ret
4570:  3f40 0145      mov	#0x4501 "That password is not correct.", r15
4574:  b012 de45      call	#0x45de <puts>
4578:  3041           ret
```

Como vemos en esta sección, la llamada a `test_password_valid` ocurre en `0x4544`, con lo cual analizamos qué ocurre después de esto, y nos interesa en particular el siguiente bloque:

```nasm
4544:  b012 5444      call	#0x4454 <test_password_valid>
4548:  0f93           tst	r15
454a:  0324           jz	$+0x8 <login+0x32>
454c:  f240 6200 1024 mov.b	#0x62, &0x2410
4552:  3f40 d344      mov	#0x44d3 "Testing if password is valid.", r15
4556:  b012 de45      call	#0x45de <puts>
455a:  f290 b700 1024 cmp.b	#0xb7, &0x2410
4560:  0720           jnz	$+0x10 <login+0x50>
```

Al salir de la función, recordemos que por la calling convention y la lógica que vimos implementada en `test_password_valid`, el valor de retorno se deja en `r15`, por lo tanto en `0x4548` lo que está haciendo es chequear si `r15` es `0x0` y en base a setear un valor en la posición de memoria `0x2410` que luego usa para comparar en `0x455a` y en base a eso llamar hacer un `jnz` dependiendo si el password es correcto o no. En base a esta lógica, nos damos cuenta que para pasar la validación la clave está en lograr que `0x2410` sea igual a `0xb7`, ahora, volviendo al análisis de la función `test_password_valid`, sabemos que la clave que ingresamos se guarda en memoria a partir de `0x2400`, es decir que si la longitud no se valida correctamente y logramos un "overflow" ingresando más de 16 caracteres (que según el prompt del password era el máximo permitido), podemos pisar la posición de memoria `0x2410` con un valor arbitrario y lograr de este modo bypassear el control.

Con una prueba rápida validamos que podemos ingresar sin problemas más de 16 caracteres, cargando como contraseña, por ejemplo, `1234567890123456789` y viendo el live memory dump:

![Hanoi Mem Dump 1](/img/posts/microcorruption/hanoi-mem-dump-1.png)

Como podemos ver, no hay validación de que la contraseña ingresada tenga más de 16 caracteres, lo que nos permite aprovechar el overflow para pisar el valor de memoria que necesitamos que contenga el valor deseado para abrir la cerradura.

Haciendo uso del hex input ingresamos un password que contenga `0xb7` en la posición deseada (el carácter número 17) y logramos dejar el valor almacenado en la posición de memoria `0x2410`, completando el nivel.

#### Level 4 - Cusco

Contexto:
```tasm
DETAILS

The LockIT Pro b.02  is the first of a new series  of locks. It is
controlled by a  MSP430 microcontroller, and is  the most advanced
MCU-controlled lock available on the  market. The MSP430 is a very
low-power device which allows the LockIT  Pro to run in almost any
environment.

The  LockIT  Pro   contains  a  Bluetooth  chip   allowing  it  to
communiciate with the  LockIT Pro App, allowing the  LockIT Pro to
be inaccessable from the exterior of the building.

There  is no  default  password  on the  LockIT  Pro HSM-1.   Upon
receiving the  LockIT Pro,  a new  password must  be set  by first
connecting the LockitPRO HSM to  output port two, connecting it to
the LockIT Pro App, and entering a new password when prompted, and
then restarting the LockIT Pro using the red button on the back.

LockIT Pro Hardware  Security Module 1 stores  the login password,
ensuring users  can not access  the password through  other means.
The LockIT Pro  can send the LockIT Pro HSM-1  a password, and the
HSM will  return if the password  is correct by setting  a flag in
memory.

This is Hardware  Version B.  It contains  the Bluetooth connector
built in, and two available  ports: the LockIT Pro Deadbolt should
be  connected to  port  1,  and the  LockIT  Pro  HSM-1 should  be
connected to port 2.

This is Software Revision 02. We have improved the security of the
lock by  removing a conditional  flag that could  accidentally get
set by passwords that were too long.
```

El escenario al que nos enfrentamos en este caso, pareciera contener el fix para el write de memoria que habíamos logrado explotar en el nivel anterior para escribir en memoria a partir de un overflow del input.

Analizando el código, la lógica de la función `test_password_valid` es exactamente igual a la anterior, con la diferencia que ahora el password que ingresamos, según el valor que la función recibe en `r15`, está alojado en la stack, y si prestamos atención a la lógica de la función login, en particular a las últimas 2 instrucciones, nos vamos a encontrar con algo interesante:

```nasm
4500 <login>
4500:  3150 f0ff      add	#0xfff0, sp
4504:  3f40 7c44      mov	#0x447c "Enter the password to continue.", r15
4508:  b012 a645      call	#0x45a6 <puts>
450c:  3f40 9c44      mov	#0x449c "Remember: passwords are between 8 and 16 characters.", r15
4510:  b012 a645      call	#0x45a6 <puts>
4514:  3e40 3000      mov	#0x30, r14
4518:  0f41           mov	sp, r15
451a:  b012 9645      call	#0x4596 <getsn>
451e:  0f41           mov	sp, r15
4520:  b012 5244      call	#0x4452 <test_password_valid>
4524:  0f93           tst	r15
4526:  0524           jz	$+0xc <login+0x32>
4528:  b012 4644      call	#0x4446 <unlock_door>
452c:  3f40 d144      mov	#0x44d1 "Access granted.", r15
4530:  023c           jmp	$+0x6 <login+0x36>
4532:  3f40 e144      mov	#0x44e1 "That password is not correct.", r15
4536:  b012 a645      call	#0x45a6 <puts>
453a:  3150 1000      add	#0x10, sp
453e:  3041           ret
```

En `0x453a`, vemos que hace un `add  #0x10, sp`, lo que mueve el puntero del stack 16 posiciones de memoria más adelante, dado que el tamaño máximo de la contraseña es 16, pero lo interesante es que nuevamente, si no se valida que este máximo se respete al hacer el input, podemos lograr un overflow para pisar parte del stack por fuera de lo esperado, y como la siguiente instrucción es un `ret`, podríamos tomar control de la dirección de retorno.

Antes de seguir, tengamos en cuenta cómo funciona `ret`: al ejecutar la instrucción, el procesador toma los primeros 2 bytes del stack, los carga en el registro `pc` (que actúa como puntero a la próxima instrucción), y mueve el stack pointer 2 posiciones.

Esto implica que si podemos pisar el stack, podemos poner una dirección arbitraria y hacer que al terminar la función `login`, en lugar de volver al punto esperado, la ejecución se dirija a `0x4446` — que como vemos en el segmento anterior es la dirección de `unlock_door`. Podemos entonces ingresar más de 16 caracteres y que los caracteres 17 y 18 sean `0x46` y `0x44` (en ese orden, por ser little-endian).

Probamos entonces ingresar como contraseña 16 caracteres arbitrarios seguidos de `FD` (el byte equivalente a `F` es `0x46` y a `D` es `0x44`), y ponemos un breakpoint en `0x453a` con `b 453a`, ingresamos la contraseña y continuamos hasta el breakpoint. En ese punto vemos que:

![Cusco Mem Dump 1](/img/posts/microcorruption/cusco-mem-dump-1.png)

Y además `sp = 0x43ee`, por lo tanto, al ejecutar `add  #0x10, sp`, obtenemos `sp = 0x440e`, que es donde está ubicada la dirección de retorno que pudimos inyectar con el overflow pisando el stack. Al continuar la ejecución, el `ret` va a pasar a ejecutar la función `unlock_door` dejando el nivel resuelto.

### Conclusiones

Si bien el entorno es completamente ficticio, tanto la mecánica para resolver los niveles, como la emulación del entorno de trabajo me parecieron bastante realistas, y el incremento en el nivel de dificultad está bien pensado para poder ir sumando nuevos conceptos de a poco. El tutorial arranca con una comparación básica, luego empezamos a sumar acceso a memoria, calling conventions, manejo del stack, al menos hasta ahora, me pareció una muy buena alternativa para ir subiendo de a poco la curva de dificultad.

Por otro lado, aunque la emulación es sobre un microcontrolador específico y es tal vez un poco diferente a lo que nos podemos encontrar al hacer reversing de un binario de x86 o ARM, creo que sigue siendo una buena opción para entender cómo empezar a leer ASM, interpretar las distintas operaciones y flujos de control, y familiarizarnos un poco con los procesos de debugging y reversing.

Para terminar, te dejo por acá un pequeño cheat-sheet con varias de las funciones que fuimos viendo en el código a modo de referencia rápida. Si llegaste hasta acá, espero que te haya servido de algo! Gracias por leerme!

### MSP430 cheatsheet

| Instruction      | What it does                                                           | Example                                                                        |
| ---------------- | ---------------------------------------------------------------------- | ------------------------------------------------------------------------------ |
| `mov.b src, dst` | Move a **byte** from src to dst                                        | `mov.b @r15, r14` — read the byte at the address stored in r15, store in r14   |
| `mov src, dst`   | Move a **16-bit word**                                                 | `mov r15, r13` — copy r15 into r13                                             |
| `cmp.b src, dst` | Compare byte: subtract src from dst, set Z flag if equal               | `cmp.b @r13, 0x2400(r14)`                                                      |
| `cmp src, dst`   | Compare 16-bit word                                                    | `cmp #0x472f, 0x0(r15)`                                                        |
| `tst dst`        | Test operand: sets Z flag based on whether dst is zero                 | `tst r14` — sets Z if r14 is 0x0000                                            |
| `jnz offset`     | Jump if not zero (Z flag clear)                                        | `jnz $-0x8` — jump back 8 bytes if last comparison/tst was unequal             |
| `jz offset`      | Jump if zero (Z flag set)                                              | `jz $+0x6` — jump forward 6 bytes if last comparison/tst was equal             |
| `inc dst`        | Increment by 1                                                         | `inc r12` — r12 = r12 + 1                                                      |
| `incd dst`       | Increment by 2                                                         | `incd r4` — r4 = r4 + 2 (used for word-sized stack adjustments)                |
| `decd dst`       | Decrement by 2                                                         | `decd sp` — sp = sp - 2                                                        |
| `clr dst`        | Clear (set to 0)                                                       | `clr r14`                                                                      |
| `add src, dst`   | dst = dst + src                                                        | `add r4, r14`                                                                  |
| `push src`       | Push src onto stack (`sp -= 2`, then `*sp = src`)                      | `push r15`                                                                     |
| `pop dst`        | Pop top of stack into dst (`dst = *sp`, then `sp += 2`)                | `pop r4`                                                                       |
| `call addr`      | Push return address onto stack, jump to addr                           | `call #0x457a <INT>`                                                           |
| `ret`            | Pop top of stack into PC (jump there)                                  | `ret`                                                                          |
| `sxt dst`        | **Sign-extend** a byte in dst's low byte to fill the whole 16-bit word | `sxt r15` — if r15's high byte was the sign extension, copy bit 7 to bits 8-15 |
