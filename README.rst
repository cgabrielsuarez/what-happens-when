Introducción...
====================

El texto de este repositorio es un intento de traducción al
español (de España) del texto original en inglés del repositorio 
https://github.com/alex/what-happens-when

Consideré la información muy didáctica e importante para repasar 
conceptos básicos y entender, para el que no tuviera cierta idea ya, 
la "magia" que ocurre ante una acción tan cotidiana. Por lo tanto, 
he realizado esta traducción (Que aún tiene muchas cosas por corregir) 
creyendo que podría serle útil a alguien de habla hispana.

Comencemos:

Qué ocurre cuando...
====================

Este repositorio es un intendo de responder la vieja pregunta "¿Qué ocurre
cuando escribes google.com en la barra de direcciones de tu navegador web
y presionas Enter?"

En vez de usar la típica historia, vamos a intentar responder esta pregunta
de la manera más detallada posible. Sin omitir nada.

Esto es un proceso colaborativo, así que promuévelo e intenta ayudar.
Hay toneladas de detalles perdidos esperando por ti para ser añadidos. 
Así que envíanos un pull requets, ¡por favor!.

Todo esto esta licenciado bajo los términos de la `Creative Commons Zero`_

Tabla de contenidos
====================

.. contents::
   :backlinks: none
   :local:

La tecla "g" es pulsada
----------------------
La siguiente sección explica todo acerca del teclado físico y las 
interrupciones del sistema operativo. Pero, un montón de cosas
que no son explicadas pasan después de eso.
Cuando tú pulsas la "g" el navegador revice el evento
y toda la maquinaria de autocompletado se pone en marcha.
Dependiendo del algoritmo de tu navegador y de si estás 
en modo privado/incógnito o no hay sugerencias para ser
presentadas en el dropdown bajo la barra de direcciones.
Muchos de esos algoritmos priorizan los resultados basados
en el historial de búsqueda y marcadores.
Tú estás escribiendo "google.com" así que nada de eso importa,
pero un montón de código va a correr antes de que obtengas eso
y las sugerencias serán refinadas con cada pulsación de tecla.
Esto incluso puede hacer que te sea sugerido "google.com" antes 
de que lo escribas.


La tecla "Enter" toca fondo.
---------------------------

Para escoger un punto de partida, vamos a elegir la tecla Enter del teclado
siendo presionada hasta el fondo. En este punto, un circuito eléctrico 
específico a la tecla Enter es cerrado (ya sea directamente o capacitativamente). 
Esto permite que una pequeña cantidad de corriente fluya hacia la 
circuitería lógica del teclado, la cual escanea el estado del interruptor de 
cada tecla, desactiva el ruido eléctrico del rápido e intermitente 
cierre del interruptor, y lo convierte a una clave entera, en este caso "13".
La controladora del teclado entonces codifica la clave para ser transportada 
a la computadora. Esto sucede ahora de manera casi universal sobre Universal Serial Bus 
(USB) o mediante conexiones Bluetooth, pero historicamente ha sido sobre
PS/2 o conexiones ADB.

*En el caso de teclados USB:*

- La circuitería del teclado usb es alimentada por un suministro de 5V
  recibido por el pin 1 desde la controladora USB del host.

- La clave generada es almacenada en una memoria interna de la circuitería
  en un registro llamado "endpoint".

- La controladora USB del host sondea ese endpoint cada ~10ms (el valor
  mínimo declarado por el teclado), obtiene entonces el valor de la clave
  almacenado en el registro.

- Este valor va al USB SIE (Serial Interface Engine) para ser convertido
  en un o más packetes USB que sigue el protocolo USB de bajo nivel.

- Esos paquetes son enviados por una señal electrica diferencial sobre
  los pines D+ y D- (la segunda mitad) a una velocidad máxima de 1.5 Mb/s, 
  como dispositivo HID (Human Interface Device), siempre es declarado
  como un "dispositivo de baja velocidad" (USB 2.0 compliance) 

- Esta señal en serie es entonces decodificada por la controladora USB
  del host e interpretada por el driver del teclado universal Human Interface Device 
  (HID). El valor de esta tecla es entonces pasada a la capa de abtracción de
  hardware del sistema operativo.


*En el caso de teclados virtuales (como en pantallas de dispositivos táctiles):*

- Cuando el usuario pone sus dedos en una pantalla capacitiva táctil, una
  pequeña cantidad de corriente se transfiere al dedo. Esto completa el 
  circuito a través del campo electrostático de la capa conductora
  y crea una caída del voltaje en ese punto de la pantalla. Entonces, el 
  ``screen controller`` lanza una interrupción informando de la coordenada
  de la presión de la tecla.

- Entonces, el sistema operativo móvil notifica la aplicación actualmente
  enfocada del evento de presión en uno de los elementos de su GUI
  (los cuales ahora son los botones de la aplicación de teclado virtual)

- El teclado virtual puede ahora lanzar una interrupción de software
  enviando un mensaje de tecla presionada de vuelta al OS.

- Esta interrupción notifica a la aplicación enfocada actual de un 
  evento de 'key pressed' (tecla presionada). 


Disparos de interrupción [NO en teclados USB]
---------------------------------------

El teclado envia señales en su línea de petición de interrupción, la cual
es mapeada a un ``vector de interrupción`` (entero) por el controlador de
interrupciones. La CPU usa la ``Tabla de descripción de interrupciones`` (IDT)
para mapear los vectores de interrupción a funciones (``Manejadoras de interrupción``)
que son proporcionadas por el Kernel.
Cuando una interrupción llega, la CPU indexa la IDT con el vector de interrupción
y ejecuta el manejador apropiado. 


(En Windows) Un mensaje ``WM_KEYDOWN`` es enviado a la app.
--------------------------------------------------------

El HID pasa el evento de tecla pulsada al driver ``KBDHID.sys`` el cual
convierte el HID en un scancode. En este caso, el scancode es 
``VK_RETURN`` (``0x0D``). El driver ``KBDHID.sys`` con ``KBDCLASS.sys``
(keyboad class driver). Este driver es responsable de manejar
todas las entradas de teclado y keypad de una manera segura.
Entonces llama a ``Win32K.sys`` (después de potencialmente pasa el
mensaje a través de filtros de teclados de terceros que están instalados).
Todo esto ocurre en el modo kernel.

``Win32K.sys`` intuye qué ventana es la ventana activa a través de
``GetForegroundWindows()`` API. Esta API provee el manejo de ventana
de la barra de direcciones del navegador. La ventana principal "message pump"
entonces llama a ``SendMessage(hWnd, WM_KEYDOWN, VK_RETURN, lParam)``. 
``lParam`` es una máscara de bits que indica más información
sobre la pulsación de la tecla: repetición (0 en este caso),
el scancode actual (puede ser "OEM dependent", pero generalmente
no sería para ``VK_RETURN``), si fueron pulsadas también 
teclas extendidas (e.g alt, shift, ctrl) (No fueron), y 
algún otro estado.

La api de Windows ``SendMessage`` es una sencilla funcion que añade 
el mensaje a una cola para un manejo de ventana particular.
Más tarde, la principal funcion de procesamiento de mensaje
(llamada ``WindowProc``) asignada al ``hWnd`` es llamada 
para procesar cada mensaje de la cola.

La ventana (``hWnd``) que está activa es actualmente un 

The window (``hWnd``) that is active is actually an edit control and the
``WindowProc`` in this case has a message handler for ``WM_KEYDOWN`` messages.
This code looks within the 3rd parameter that was passed to ``SendMessage``
(``wParam``) and, because it is ``VK_RETURN`` knows the user has hit the ENTER
key.

(En OS X) Un evento ``KeyDown`` NSEvent es enviado a la app
--------------------------------------------------

La señal de interupcion dispara un evento de interrupcion en el driver
'I/O Kit Kext Keyboard'. El driver traduce la señal en un keycode el cual
es pasado al proceso ``WindowServer`` de OS X. Como resultado, el ``WindowServer``
despacha un evento a la aplicación (por ejemplo activo o a la escucha) apropiada
a través de su puerto Mach donde es situado en un evento de cola.
Los Eventos pueden ser leídos de esta cola por theads con los 
suficientes privilegios llamando a la función ``mach_ipc_dispatch``. 
Esto mayormente ocurre a través de, y es manejado, por un blucle del evento principal
``NSApplication``, vía ``NSEvent`` de ``NSEventType`` ``KeyDown``. 

(En GNU/Linux) El servidor Xorg a la escucha de keycodes
---------------------------------------------------

Cuando un servidor gráfico ``X server`` está en uso, ``X`` usará 
el driver de eventos genéricos ``evdev`` para obtener la pulsación de la tecla.
Un remapeo de keycodes a scancodes es realizado con keymaps especificos y reglas
de ``X server``. 
Cuando el mapeo de scancode de la tecla pulsada está completo, el ``X server``
envía el carácter al ``window manager`` (Gestor de ventana: DWM, metacity, 
i3, etc) entonces el ``window manager`` a cambio envía el caracter a la 
ventana enfocada. La API de la ventana que recive el caracter imprime
el simbolo apropiado en campo apropiado que tiene el foco.

Parse URL
---------
* El navegador ahora tiene la siguiente información contenida en la URL 
  (Uniform Resource Locator):

    - ``Protocol``  "http"
        Usa 'Hyper Text Transfer Protocol'

    - ``Resource``  "/"
        Trae la página principal (index)


¿Es esto una URL o un término de búsqueda?
-----------------------------
Cuando no se le da un protocolo o un nombre de dominio válido al navegador,
éste procede a pasar el texto introducido en la barra de direcciones 
a el motor de búsqueda configurado por defecto.
En muchos casos la URL tiene una parte especial de texto añadida 
que le dice al motor de búsqueda que viene de la barra de direcciones
de un navegador en particular.

Convertir caracteres Unicode no-ASCII en un nombre de host
------------------------------------------------
* El navegador comprueba el nombre de host por caracteres que no sean ``a-z``,
  ``A-Z``, ``0-9``, ``-``, or ``.``.

* Como el nombre de host es ``google.com`` no habrá ninguno, pero si
  lo hubiera, el navegador aplicaría codificación `Punycode`_ a la porción
  del nombre de host de la URL.

Comprueba lista HSTS
---------------
* El navegador comprueba su lista precargada 'HSTS (HTTP Strict 
  Transport Security)'. Esto es una lista de sitios web que han 
  solicitado ser contactados solamente por HTTPS.

* Si el sitio web está en la lista, el navegador envía su solicitud
  vía HTTPS en lugar de HTTP. De otra forma, la solicitud inicial
  es enviada por HTTP.
  (Ten en cuenta que un sitio web HTTP puede seguir usando la 
  política HSTS *sin* estar en la lista HSTS. La primera solicitud
  HTTP hacia el sitio web hecha por un usuario recibirá una respuesta
  solicitando que el usuario sólo envió solicitudes HTTPS. Sin embargo,
  esta única solicitud HTTP podría potencialmente dejar al usuario
  vulnerable ante un `downgrade attack`_, este es el porque de que 
  la lista HSTS sea incluida en los navegadores web modernos.

Resolución DNS 
----------

* El navegador comprueba si el dominio está en su caché (para ver la 
  caché DNS en Chrome, ve a `chrome://net-internals/#dns <chrome://net-internals/#dns>`_).
* Si no es encontrado, el navegador llama a la función de biblioteca 
  ``gethostbyname`` (varía dependiendo del OS) para hacer la resolución dns.
* ``gethostbyname`` comprueba si el nombre de host puede ser resuelto por
  referencia en el fichero local ``hosts`` (Cuyo lugar donde se encuentra 
  varía en cada OS) antes de intentar resolver el nombre de host a través de DNS.
* Si ``gethostbyname`` no lo tiene cacheado o no puede encontrarlo
  en el archivo ``hosts``, entonces realiza una solicitud al servidor DNS
  condigurado en el stack de red. Esto se realiza típicamente contra el 
  router local o contra el servidor de cacheo DNS del ISP.
* Si el servidor DNS está en la misma subred, la biblioteca de red 
  sigue el ``ARP process`` por debajo del servidor DNS.
* Si el servidor DNS está en una subred diferente, la biblioteca de red
  sigue el ``ARP process`` por debajo de el gateway por defecto.

El proceso ARP
-----------

Con el fin de enviar un mensaje broadcast ARP (Protocolo de Resolución de 
direcciones), el stack de red necesita saber la dirección IP objetivo.
También es necesario saber la dirección MAC de la interfaz que será usada
para enviar el mensaje broadcast ARP.

La caché ARP es checkeada en primer lugar buscando la entrada de nuestra
ip objetivo. Si está en la caché, la función de biblioteca retorna el 
resultado: Ip Objetivo = MAC.

Si la entrada no está en la caché ARP:
* La tabla de rutas es observada para ver si la Ip Objetivo está en 
  alguna de las subredes de la tabla de rutas local. Si lo está, 
  la biblioteca usa la interfaz asociada a esa subnet. Si no lo está,
  la biblioteca usa la interfaz que tiene la subred de nuestro
  default gateway.

* La dirección MAC de la interfaz de red seleccionada es encontrada.

* La biblioteca de red envía una request ARP de capa 2 (Capa de Enlace
  según el modelo `OSI`_) 


``ARP Request``::

    Sender MAC: interface:mac:address:here
    Sender IP: interface.ip.goes.here
    Target MAC: FF:FF:FF:FF:FF:FF (Broadcast)
    Target IP: target.ip.goes.here

Dependiendo de qué tipo de hardware está entre la computadora y el router:

Directamente conectada:

* Si la computadora está directamente conectada al router, el router responde
  con una ``ARP reply`` (mira más abajo)

Hub:

* Si la computadora está conectada a un hub, el hub envíara por broadcast
  la ARP request por todos sus puertos. Si el router está conectado en
  el mismo "cable", responderá con un ``ARP Reply`` (mira más abajo)

Switch:

* Si la computadora está conectada a un switch, el switch comprobará
  su tabla CAM/MAC local para ver en que puerto está la dirección MAC
  que estamos buscando. Si el switch no tiene una entrada para esa 
  dirección MAC, reenviará una ARP request a todos los demás puertos.

* Si el switch tiene una entrada en la tabla MAC/CAM enviará el
  ARP request al puerto que tiene la dirección MAC que estamos buscando.

* Si el router está en el mismo "cable", responderá con un ``ARP Reply``
  (mira más abajo)

``ARP Reply``::

    Sender MAC: target:mac:address:here
    Sender IP: target.ip.goes.here
    Target MAC: interface:mac:address:here
    Target IP: interface.ip.goes.here

Ahora la biblioteca de red tiene la dirección IP de nuesto servidor DNS
or de nuestro gateway por defecto. Esto resume el proceso DNS:
* El puerto 53 es abierto para enviar una request UDP al servidor
  DNS. (Si el tamaño de la respuesta es muy largo, se usará TCP)
* Si el servidor DNS local/ISP no tiene el registro, entonces es 
  solicitada una búsqueda recursiva que fluye por la lista de 
  servidores DNS hasta que encuentra el SOA y si encuentra la respuesta,
  la devuelve.

Abriendo un socket
-------------------
Una vez que el navegador recive la dirección IP del servidor de destino,
toma eso y el número de puerto dado en la URL (el procolo HTTP puesto 80
por defecto, y HTTPS al puerto 443), y hace una llamada a una función
de la biblioteca del sistema llamada ``socket`` y solicita un stream socket 
TCP - ``AF_INET/AF_INET6`` y ``SOCK_STREAM``.

* Esta request es pasara en primer lugar a la capa de Transporte donde 
  es elaborado el segmento tcp. El puerto de destino es agregado al header
  y el puerto de origen es elegido por el rango de puertos dinámicos del
  Kernel. (ip_local_port_range en Linux)
* Este segmento es enviado a la capa de red la cual agrega un header IP
  adicional. La dirección ip de destino del server así como la de la máquina
  actual son insertadas al formar el paquete.
* El paquete llega a la capa de Enlace. Un frame header es agregado el cual
  incluye la dirección MAC de la tarjeta de red así como la dirección MAC del
  gateway (route local). Como antes, si el kernel no conoce la dirección MAC
  address del gateway, deberá lanzar una consulta ARP en broadcast para
  encontrarla.

En este punto, el paquete esta listo para ser transmitido a través de lo siguiente:

* `Ethernet`_
* `WiFi`_
* `Cellular data network`_

Para la mayoría de conexiones de internet de hogar y pequeñas empresas, los
paquetes pasaran desde tu computadora, posiblemente a través de una red local
y entonces a través del modem (Modulador/Desmodulador) el cual convierte
la señal digital compuesta por unos y ceros a una señal analógica que posibilita
la transmisión a través de cable telefónico o de una conexión telefónica 
inalámbrica. En el otro lado de la conexión, hay otro modem que convierte la
señal analógica de nuevo a señales digitales que será procesada en el siguiente
`nodo de red`_ donde, entre otras cosas, las direcciones de origen y destino 
serán analizadas.

Negocios más grandes y algunas nuevas conexiones residenciales tienen
fibra o conexiones Ethernet directas en cuyo caso la información 
se mantiene en digital y pasa directamente al siguiente `nodo de red`_ 
para ser procesada.

Eventualmente, el paquete alcanzará el router que gestiona la subred local.
Desde ahí, continuará el viaje hacia el sistema de routers de borde autónomo (AS),
otros ASes, y finalmente el servidor de destino. Cada router del camino extrae
la dirección de destino de la cabezera IP y lo enruta debidamente hacia el 
siguiente salto. El campo de tiempo de vida (TTL) en el heade IP es 
decrementado por cada uno de los routers por lo que pasa. El paquete 
será descartado si el campo TTL alcanza zero o si el actual router no 
tiene espacio en su cola. (Debido a causas de congestión de la red).

Este envío y recibo ocurre multiples veces siguiendo el flujo de 
una conexión TCP:

* El cliente elige un número de secuencia inicial (ISN) y envía el paquete
  al server con el bit SYN seteado indicando que está seteado el ISN.
* El server recive el SYN y si está de humor:
   * El servidor elige su número de secuencia inicial.
   * El server setea el SYN para indicar que está eligiendo su ISN.
   * El servidor copia el (número ISN del cliente + 1) a su campo ACK
     y agrega el flag ACK para indicar el acuerdo de recibo del primer
     paquete.
* El cliente negocia la conexión enviando un paquete:
   * Incrementa su propio número de secuencia. 
   * Incrementa el número del negociación del receptor.
   * Setea el campo ACK
* Los datos son transmitidos de la siguiente manera:
   * Desde un lado se envía N bytes de datos, esto incrementa 
     el SEQ del N.
   * Cuando el otro lado acusa el recibo de ese paquete (o una
     cadena de paquetes), el envía un paquete ACK con el valor ACK
     igual al último de la secuencia recibida desde el otro.
* Para cerrar la conexión:
   * El que cierra envía un paquete FIN
   * El otro lado devuelve un ACK en respuesta al paquete FIN
     y envía su propio FIN.
   * El que inició el cierre acusa el recibo enviando un ACK como
     respuesta al ACK del paso anterior.

TLS handshake
-------------
* La computadora cliente envía un mensaje ``ClientHello`` al servidor
  con la versión de su Capa de Transporte Seguro (TLS), lista de 
  algoritmos de cifrado y métodos de compresión disponibles.
* El servidor responde con un mensaje ``ServerHello`` hacia el cliente
  con su versión de TLS, cifrado seleccionado, el método de compresion
  seleccionado y certificado público firmado por un CA (Certificate Authority).
  El certificado contiene una clave pública que será usada por el cliente para
  cifrar el resto de la negicoación hasta que una clave simétrica
  pueda ser agregada posteriormente.

* El ciente verifica el certificado digital contra una lista de CAs confiables.
  Si se establece una relación de confianza basadose en el CA, el cliente
  genera una cadena de bytes pseudo aleatorios y encripta esto con la
  clave pública del servidor. Esos bytes aleatorios pueden ser usados
  para determinar la clave simétrica.

* El servidor descifra los bytes aleatorios usando su clave privada
  y usa esos bytes para generar su propia copia de la clave simétrica maestra.

* El cliente envia un mensaje ``Finished`` al servidor, cifrando el hash de 
  la transmisión, en este punto, con la clave simétrica.

* El servidor genera su propio hash, y entonces descifra el hash enviado
  por el cliente para verificar si concuerda, si lo hace, envía su propio
  mensaje ``Finished`` al cliente, también cifrado con la clave simétrica.

* A partir de ahora, en la sesión TLS la transmisión de los datos de la
  aplicación (HTTP) será cifrada con la correspondiente clave simétrica. 

Procolo HTTP
-------------

Si el navegador web usado fue escrito por Google, en lugar de enviar una request
HTTP para solicitar la página, el envíara una request para intentar negociar con el
servidor un "upgrade" de HTTP al protocolo SPDY.

Si el cliente está usando el procolo HTTP y no soporta SPDY, el envía una request
al servidor con esta forma:

    GET / HTTP/1.1
    Host: google.com
    Connection: close
    [other headers]

donde ``[other headers]`` hace referencia a una serie de pares clave-valor
separados por coma según el formato de la especificación HTTP y separados
por una sola línea nueva. (Estamos asumiendo que el navegador web que
está siendo usado no tiene ningún bug que viola la especificación HTTP.
Asumimos también que el navegador web está usando ``HTTP/1.1``, de otro modo
podría no incluid el header ``Host`` en la request y la versión especificada
en el ``GET`` request sería ``HTTP/1.0`` o ``HTTP/0.9``.)

HTTP/1.1 define la opción de cierre de la conexión desde el que envia la 
señal que la conexión será cerrada antes de completar la respuesta. Por ejemplo::

    Connection: close

aplicaciones HTTP/1.1 que no soporten conexiónes persistentes DEBEN incluir
el la opción de cierre de la conexión en cada mensaje.

Después de enviar la requests y los headers, el navegador web envía una
única línea nueva vacía al servidor indicando que el contenido de la 
requests está completo.

El server responde con una código de respuesta dando a entender el estatus
de la request y responde de la siguiente manera::

    200 OK
    [response headers]

Seguido por una única línea nueva, y entonces envía el payload del contenido
HTML de ``www.google.com``. EL servidor entonces cierra la conexión, o si 
los headers enviados por el cliente lo solicitan, mantiene la conexión
abierta para ser rehusada para las request adicionales.

Si los headers HTTP enviados por el navegador web incluyen suficiente
información para que el servidor web pueda determinar si la versión 
del archivo cacheado por el navegador no ha sido modificada desde la última
visita (por ejemplo si el navegador web inclute un header ``ÈTag``),
el servidor web podría responder con una request con el siguiente aspecto::

    304 Not Modified
    [response headers]

Y no retornar ningún payload y entonces el navegador web mostraría
el HTML de su caché.

Después de parsear el HTML, el navegador web (y el servidor) repite 
este proceso para cada recurso (imágenes, CSS, favicon.ico, etc) haciendo
referencia a la página HTML, exceptuando ``GET / HTTP/1.1`` cuya request será
``GET /$(URL relative to www.google.com) HTTP/1.1``.

Si el HTML referencia a un recurso de un dominio diferente que ``www.google.com``,
el navegador web vuelve a repetir los pasos que siguió para resolver el 
dominio ``www.google.com`` pero en este caso para ese dominio.
El header ``Host`` en la request será seteado con el nombre apropiado en lugar
de ``google.com``


Manejo de request del servidor HTTP
--------------------------
El servidor HTTPD (Http Daemon) es el que maneja las request/respondes 
en el lado del servidor. Los servidores HTTPD más comunes son Apache y
Nginx para Linux e IIS para Windows.

* El HTTPD (HTTP Daemon) recibe la request.
* El servidor descompone la request a los siguientes parámetros:
  * HTTP Request Method (ya sea ``GET``, ``HEAD``, ``POST``, ``PUT``,
     ``DELETE``, ``CONNECT``, ``OPTIONS``, or ``TRACE``). En el caso de una
    url introducida directamente en la barra de direcciones, será ``GET``.
   * Dominio, en este caso - google.com.
   * Ruta/página solicitada, en este caso - / (al no especificar página o ruta
     ,/ es la ruta por defecto).
* El servidor verifica que hay un Host Virtual configurado en el servidor
  que corresponde con google.com.
* El servidor verifica que goog.eocm puede aceptar requests GET.
* El servidor verifica que el cliente está permitido a usar ese método
  (by IP, autenticación, etc).
* Si el servidor tiene un modulo rewrite instalado (como mod_rewrite en Apache
  o URL Rewrite para IIS), este intenta emparejar la request contra 
  alguna de las reglas configuradas. Si se encuentra una regla que coincida,
  el servidor usa esa regla para reescribir la request.
* El servidor va a devolver el contenido que corresponda con la request,
  en nuestro caso, traerá el archivo index, así como "/" es el fichero principal.
  (en algunos casos esto puede ser modificado, pero es el método más común)
* El servidor parsea el archivo de acuerdo con el manejador. Si Google
  está usando PHP, el server usa PHP para interpretar el fichero index y 
  devuelve la salida al cliente.

Detrás de los escenarios del Browser
----------------------------------

Una vez que el servidor suministra los rescursos (HTML, CSS, JS, images, etc.)
al navegador, lo que sigue es el siguiente proceso.

* Parseo - HTML, CSS, JS
* Renderizado - Construcción del árbol DOM → Árbol Render → Diseño del 
  árbol Render → Pintando el árbol 

Navegador
-------

La funcionalidad de un navegador es presentar el recurso web que elijas,
solicitándolo al servidor web y mostrándolo en la ventana del navegador.
Este recurso es usualmente un documento HTML, pero puede ser también
un PDF, imagen o algún otro tipo de contenido. La localización de ese
recurso es especificado por el usuario usando una URI (Uniform Resource
Identifier).

La forma en la que el navegador interpreta archivos HTML es especificada
en las especificaciones HTML y CSS. Estas especificaciones son mantenidas 
por la organización W3C (World Wide Web Consortium), que es una de las 
organizaiciones estándarizadoras para la web.

Las interfaces de los navegadores tienen mucho en común unas con otras.
Los elementos más comunes en la interfaz de usuario son:

* Una barra de direcciones para insertar una URI.
* Botones de atrás y adelante.
* Opciones de Marcadores
* Botones de refresco y stop para refrescar o detener el contenido
  de la carga de los documentos actuales.
* Botón Home que te devuelce a la home page.

**Estructura de alto nivel de los navegadores**

Los componentes de los navegadores son:

* **Interfaz de usuario:** La interfaz de usuario incluye una barra
  de direcciones, botones atrás/adelante, menú de marcadores, etc.
  Mostrado en cada parte del navegador a excepción de la
  parte de la ventana donde ves la página solicitada.
* **Motor del browser:** El motor del navegador ordena las actiones entre
  UI y el motor de renderizado.
* **Motor de renderizado:** El motor de renderizado es responsable de 
  mostrar el contenido solicidado. Por ejemplo, si el contenido solicitado
  es HTML, el motor de renderizado parsea el HTML y el CSS y muestra
  el contenido parseado en la pantalla.
* **Networking:** Networking maneja las llamadas de red com requests HTTP,
  usando diferentes implementaciones para diferentes plataformas detrás
  de una interfaz que no depende de la plataforma.
* **UI backend:** El backend UI es usado para dibujar widgets básicos
  como cajas de selección y ventanas. Este backend expone una interfaz
  genérica que no es específica a la plataforma.
  En lugar de eso, usa métodos de interfaz de usuario del propio
  sistema operativo.
* **Motor de Javascript:** El motor de javascript e usado para parsear
  y ejecutar código JavaScript.
* **Almacenamiento de datos:** El almacenamiento de datos es una capa
  de persistencia. El navegador necesita almacenar datos localmente, 
  como cookies. Los navegadores también soportan mecanismo de almacenamiento
  como localStorage, IndexedDB, WebSQL and FileSystem.

Parseo HTML 
------------

El motor de renderizado comienza obteniendo el contenido del documento 
solicitado desde la capa de red. Esto será realizado en trozos de 8kB.

El trabajo principal del parseador HTML es parsear las marcas HTML a un
árbol.

El árbol de salida (el "parse tree") es un árbol de elementos DOM y nodos
de atributos. DOM es un acronimo de Document Object Model. Es un la 
representacion objeto de un documento HTML y la interfaz de elementos
HTML al mundo exterior como JavaScript. La raíz del ñarbol es el
objeto "Document". Antes de cualquier manipulación vía scripting, el
DOM tiene una relación uno-a-uno con el marcado.


**El algoritmo de parseo**

El HTML no puede ser parseado usando los típicos parseadores 
que van de arriba-abajo o fondo-superficie.

Las razones son:
* La propia naturaleza del lenguaje.
* El hecho de que los navegadores tienen tolerancia de error 
  para soportar casos conocidos de HTML inválido.
* El proceso de parseo es reentrante. Para otros lenguajes, la fuente
  no cambia durante el parseo, pero en HTML, código dinámico (como 
  elementos de un script que contiene llamadas a 'document.write()') pueden
  añadir tokens extra, entonces el proceso de parseo actualmente 
  modifica la entrada.

Ya qu enoes posible usar técnicas de parseo convencionales, el navegador 
utiliza un parseo personalizado para parsear HTML. El algorimo de parseo
es descrito en detalle en la especificación de HTML5.

El algoritmo consiste en dos escenarios: tokenización y construcción del árbol.

**Acciones cuando el parseo ha acabado**

El navegador comienza a traer los recursos externos enlazados a la página
(CSS, images, Javascript files, etc.).

En este momento, el navegador marca el documento como interactivo y 
comienza scripts de parseo que están en modo "diferido": aquellos
que deberían ser ejecutados después de que el documento sea parseado.
El estado del documento es seteado a "completo" y el evento "load" es
disparado.

Nótese que no hay nunca un error de "Sintaxis inválida" en una página HTML.
Los navegadores corrigen cualquier contenigo inválido y continúan.


Interpretación CSS
------------------

* Parsea archivos CSS, contenido de etiquetas ``<style>`` , y valores de 
  atributos ``style`` usando `"gramática, lexico y sintáxis CSS"`_
* Cada archivo CSS es parseado en un ``StyleSheet object``, donde cada objeto
  contiene reglas CSS con selectores y objetos correspondientes  a 
  la gramática CSS. 
* Un parseador CSS puede ir de arriba a abajo o de atrás para adelante
  cuando un generador de parseo especifico es usado.

Renderizado de la página.
--------------
* Crea un 'Frame Tree' o 'Render Tree' atravesando los nodos del DOM y
  calculando los valores del estilo CSS de cada nodo.
* Calcula el ancho de cada nodo en el 'Frame Tree' de arriba a abajo
  sumando el ancho principal de cada nodo hijo y de los márgenes
  horizontales del nodo, bordes y padding(relleno) 
* Calculate the actual width of each node top-down by allocating each node's
  available width to its children.
* Calcula el alto de cada nodo de abajo a arriba aplicando ajuste de texto
  y sumando las alturas de el nodo hijo y los márgenes, bordes y relleno.
* Calcula las coordenadas de cada nodo usando la información calculada
  arriba.
* Se realizan pasos más complicados cuando hay elementos que son ``floated``,
  o posicionados ``absolutely`` or ``relatively`` u otras características
  más complejas son usadas. Mira
  http://dev.w3.org/csswg/css2/ and http://www.w3.org/Style/CSS/current-work
  para más detalles.
* Crea capas que describen que partes de la página pueden ser animadas
  como un grupo sin volver a ser rasterizadas. Cada objeto frame/render 
  es asignado a una capa.
* Las texturas son situadas en cada capa de la página.
* Los objetos frame/render para cada capa son atravesadas y comandos 
  de dibujado son ejecutadas para cada una de sus respectivas capas. Esto
  puede ser realizado por la CPU o directamente por la GPU usando D2D/SkiaGL.
* Todos los pasos de arriba pueden reutilizar los valores calculados
  desde la última vez que la página fue renderizada, de esta forma,
  esos cambios incrementales requieren menos trabajo.
* Las capas de la página son enviadas al proceso de composición donde
  son combinadas con capas de otro contenido visible como el navegador (chrome?)
  iframes y paneles añadidos.
* Las posiciones de las capas finales son procesadas y comandos de 
  composición son emitidos a via Direct3D/OpenGL. Los búferes de comandos de 
  la GPU se descargan en la GPU para la representación asíncrona y la trama 
  se envía al servidor de ventanas.

Renderizado de GPU
-------------

* Durante el proceso de renderizado las capas de computación gráfica
  pueden usar ``CPU`` de propósito general o el procesador gráfico
  ``GPU``.

* Cuando usa ``GPU`` para el renderizado gráfico las capas de gráficas de
  software dividen la tarea en varios pedazos, de esta forma
  se puede aprovechar el procesamiento en paralelo masivo para operaciones
  de coma flotante requeridos por el proceso de renderizado.

Servidor de ventanas.
-------------

Post renderizado y ejecución inducida por el usuario
-----------------------------------------

Después de que el renderizado haya sido completado, el navegador ejecuta
el código JavaScript como resultado de algún mecanismo de timing 
(Como las animaciones Google Doogle) o interacción del usuario 
(Escribiendo una consulta en la caja de búsqueda y reciviendo sugerencias).
Plugins como Flash o Java pueden ser ejecutados también, aunque no es el caso
en la página principal de Google. Los Scripts pueden realizar requests adicionales,
que pueden modificar la página o su composición, causando así otra vez
un renderizado y dibujado de la página.

.. _`Creative Commons Zero`: https://creativecommons.org/publicdomain/zero/1.0/
.. _`"CSS lexical and syntax grammar"`: http://www.w3.org/TR/CSS2/grammar.html
.. _`Punycode`: https://en.wikipedia.org/wiki/Punycode
.. _`Ethernet`: http://en.wikipedia.org/wiki/IEEE_802.3
.. _`WiFi`: https://en.wikipedia.org/wiki/IEEE_802.11
.. _`Cellular data network`: https://en.wikipedia.org/wiki/Cellular_data_communication_protocol
.. _`analog-to-digital converter`: https://en.wikipedia.org/wiki/Analog-to-digital_converter
.. _`network node`: https://en.wikipedia.org/wiki/Computer_network#Network_nodes
.. _`varies by OS` : https://en.wikipedia.org/wiki/Hosts_%28file%29#Location_in_the_file_system
.. _`简体中文`: https://github.com/skyline75489/what-happens-when-zh_CN
.. _`downgrade attack`: http://en.wikipedia.org/wiki/SSL_stripping
.. _`OSI Model`: https://en.wikipedia.org/wiki/OSI_model
