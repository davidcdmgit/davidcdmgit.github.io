
# Introducción


El presente documento describe una posible implementación de la gestión de usuarios de una aplicación web.

Algunos ATLs y ACLs utilizados en este documento

- ATL: Acrónimo de Tres Letras
- ACL: Acrónimo de Cuatro Letras (y sí, ACL es un ATL ;)
- CRUD
- MFA
- 2FA
- OTP

# Aspectos de seguridad

La seguridad del sistema es un aspecto complejo, que hay que tener en cuenta en diversos momentos:

- Comunicaciones entre cliente y el servidor
  - Siempre utilizando un protocolo seguro (HTTPS ~ HTTP+SSL)
  - Pasar únicamente la información necesaria y siempre asumir que puede venir modificada por terceros
  - No enviar un identificador de usuario (excepto en el login). Este dato debería estar almacenado en sesión
  - Pasar la información con el método adecuado: GET para operaciones idempotentes, POST para el resto
- Almacenamiento en el servidor
  - Permisos adecuados en archivos y carpetas
  - Nunca almacenar credenciales o datos sensibles en archivos de tipos legibles (TXT,CSV,XML) en carpetas accesibles públicamente
- Tratamiento de los datos
  - Solo se debería asumir que los datos correctos son los que están en sesión. El resto de los datos (GET,POST,FILES,COOKIES, etc.) hay que validarlos. Siempre.
  - La validación será lo más restrictiva posible siempre que se pueda. Se podrán usar métodos existentes en el lenguaje, o expresiones regulares (sobre todo si el tipo de dato no está previsto en el lenguaje por ser algo específico del programa, país, etc, p.e. la matrícula de un vehículo)
  - El código debe ser el adecuado para no permitir mecanismos que permitan ataques: usar librerías actualizadas, no usar funciones peligrosas sin control previo (p.e. extract)
  - Las operaciones con las bases de datos deben realizarse con SQL parametrizado para evitar ataques de inyección SQL. De esta manera el gestor sabe qué operación tiene que hacer y nunca permite que los datos las alteren.
- Comunicaciones en el interfaz de usuario
  - Nunca mostrar más información de la necesaria (por ejemplo, para evitar que se sepa si un usuario está dado de alta en un sistema)
  - Uso del canal de comunicación privado: correo, SMS, …  (EJEMPLO QUE NO SEA EL ANTERIOR???) 

# Gestión de credenciales

Denominamos credenciales a aquellos elementos que acreditan la autorización para acceder a ciertos datos o realizar ciertos procesos. Su gestión es una parte crítica de la aplicación, pues cualquier error en su manejo conduciría a situaciones críticas como permitir el CRUD de información privada.

La gestión de acceso de los usuarios al sistema se puede programar en cada aplicación, o se pueden utilizar APIs externos conocidos como pueden ser Google o el DNI electrónico. 

Las credenciales mínimas para entrar en un sistema tradicionalmente han sido el usuario y la contraseña. Pero debido a las vulnerabilidades de los sistemas, cada vez más se recomienda el uso de autentificación en varios en varios (al menos dos) pasos.

## Autentificación en varios factores (MFA)


Se basas en combinar varios factores para autentificarnos en un sistema:

- factor de conocimiento: algo que sabes (contraseña, respuesta a una pregunta personal, …)
- factor de posesión: algo que tienes (tarjeta, móvil, un certificado en un pendrive, …)
- factor de herencia: algo que eres (huella dactilar, voz, reconocimiento facial, …)
- factor de hubicación: tu posición en el momento de la autentificación.

Cuando son dos los factores involucrados, se suele denominar 2FA.

Por ejemplo, en el cajero del banco usamos a la vez una tarjeta que tenemos y un pin que sabemos, y en el móvil es muy común utilizar además de un patrón que sabemos, la huella o el reconocimiento facial para tener acceso. Existen apps de gasolineras que, además de un pin, tenemos que tener activada la ubicación y estar en la propia gasolinera para activar el surtidor.

Tres factores los usaríamos cuando al meter una contraseña en un sistema, se nos envía una OTP a nuestro móvil, y para acceder ponemos nuestra huella.

## Almacenamiento de las credenciales

La tabla usuarios de la base de datos almacenará un registro por cada usuario que tenga acceso al sistema. Aunque los datos mínimos requeridos serán el nombre de usuario, la contraseña y el correo, iremos viendo a lo largo de este documento que será necesario almacenar mucha más información que nos ayudará a gestionar adecuadamente la seguridad del sistema.

Para empezar el diseño de la tabla habrá que tomar una serie de decisiones que serán fundamentales para el futuro desarrollo de la aplicación. Por ejemplo:

- la clave primaria de la tabla de usuarios podría ser usuario (texto) o codUsuario (int). De la segunda manera (que implicaría hacer un índice UNIQUE en usuario), los joins entre tablas serían más rápidos al tratarse de un número y no una cadena. También nos facilitaría un futuro cambio de nombre de un usuario del sistema (si es que ese proceso se permite).
- el usuario y el correo pueden ser un único campo. Esto implicaría que el correo sería el dato que nos identifica en el sistema. Recuerde que el correo es un campo requerido puesto que, generalmente, es la única vía de comunicación automática con el usuario (en algún sistema también podría valer el número de teléfono).

## La contraseña: almacenamiento

El almacenamiento de la contraseña en claro (sin cifrar) en el servidor no es una opción válida, aún en el caso de que esté perfectamente configurado y actualizado, y sea completamente seguro. El acceso a la tabla siempre es posible por usuarios (y aplicaciones) con los permisos adecuados sobre esa base de datos. Y alguien podría pensar que, efectivamente, si esos usuarios/aplicaciones ya tienen acceso al resto de nuestros datos, para qué les interesa la contraseña?. Pero es que conocer el valor de nuestra contraseña en ese sistema podría revelar un patrón para deducir otras contraseñas en otros sistemas. Incluso la contraseña almacenada en claro podría ser, en algún caso, la misma contraseña que el correo electrónico de contacto, lo cual sería como dejar la puerta abierta a acceder a otras aplicaciones donde ese correo está indicado como contacto (si no implementa mecanismos tipo 2FA, OTP, etc.)

Una vez aclarado que la contraseña no puede guardarse en claro, habrá que decidir cómo la almacenamos. Dado que cifrar es el proceso de cambiar unos datos originales a una forma irreconocible, habrá que guardarla cifrada. Pero un cifrado suele tener un mecanismo de vuelta atrás (para devolver el cifrado a su texto original utilizando una o varias claves), y esto no nos interesa en el caso de las contraseñas.

Por ello utilizaremos una **función hash**, que podría considerarse un tipo de cifrado sin vuelta atrás, teniendo en cuenta que:

- La entrada puede ser de cualquier tamaño, pero la salida siempre tiene una longitud fija.
- Un pequeño cambio en la entrada genera un hash muy distinto.
- Realmente no tiene vuelta atrás porque los datos originales se han perdido, pero la misma entrada siempre generará el mismo hash, por lo que es muy adecuado para autentificación (comparar el hash de una contraseña introducida por el usuario con el hash almacenado)
- Se denomina colisión cuando dos textos distintos generan el mismo hash. Por eso dos contraseñas distintas podrían dar acceso en un login a un mismo usuario. Solo en el caso de que el rango de entradas esté limitado, un algoritmo de hash podría llegar a evitarlas.
      
Algoritmos de hash son MD5, SHA1, SHA256, SHA512, BCrypt, etc. Aunque todos ellos son funciones hash útiles, algunos de ellos no tienen la complejidad necesaria como para ser utilizados con contraseñas (p.e. MD5). Su uso es más habitual como hash para comprobar si un archivo es la versión original (o ha sido modificado por un virus o hacker), o para generar tokens puntuales de validación.


## La contraseña: lo complicamos un poco

Nuestra aplicación debería tener en cuenta ciertos aspectos importantes para complicar posibles ataques al sistema. Entre ellos podríamos indicar los siguientes:

### Complejidad.
Solo validaremos como correcta una contraseña que cumpla unos mínimos requisitos como por ejemplo: longitud mínima o una cierta variedad de tipos de caracteres (entre numéricos, alfabéticos y símbolos especiales)

### Cambios periódicos.
Para obligar a un cambio periódico solo necesitaremos tener un campo en la base de datos donde almacenemos el momento futuro donde será necesario avisar/cambiar: compararlo en el login y actualizarlo a futuro en cada cambio de la contraseña.

### Detección de duplicidades/similitudes:
Que no se pueda cambiar la clave por la misma es sencillo de implementar con un simple if. Que no se pueda cambiar a una clave usada anteriormente nos obligaría a llevar un historial de hashes utilizados por usuario. Que no se pueda cambiar a una clave similar a cualquiera anterior implicaría guardar un historial de hashes creado en el momento de hashear una contraseña con todas sus posibles alternativas en claro (? uf!)

### Uso de sales personalizadas. 
Una sal criptográfica es una cadena que, combinada de alguna manera con la password del usuario, hace que el hash generado sea más difícil de atacar utilizando tablas de hashes precalculados.

Por ejemplo, si un usuario decide (mala idea) utilizar de contraseña de su sistema abc123. el SHA512 sería el siguiente: 

4e5fb359b9fd8a1c1ae85e431936d3fc80cc4c917620162d0fd1930c47bb0cd8d7424c99e6c88b6e09d406f8be2c51b17eecf460a2ffb2cfc8c6994c9b4d90fb

(un verdadero hacker la reconocería sin usar un ordenador !), pero si, por ejemplo, utilizamos la sal “h4ñ6´;2!xd” concatenada por ambos extremos, el hash es

a0912643bc1d5fca16942973697cdae26e827bcacc7286f930c09532cb42667ca57e8fc713b5a25fb266dfb4de72f2dd32acf9a6f79b261245586219801fc3cf

que posiblemente no esté en muchas tablas rainbow.

Obviamente el uso de sales hace que haya que utilizarlas en el login cambiando la password proporcionada por el usuario de la misma manera que se utilizó al almacenarla.

Las sal de un sistema puede ser la misma para todos los usuarios (se guardaría en un único sitio, pero implicaría que cambiarla no sería fácil, pues habría que distinguir los usuarios con la sal original y las nuevas). Mejor idea es que la sal se genere aleatoriamente y se guarde para cada usuario.

### Algoritmos actualizados

El hecho de que los algoritmos se vuelvan vulnerables con el tiempo, por ejemplo, por mejoras en las potencias de cálculo y almacenamiento de los nuevos ordenadores, hace que el algoritmo que nos vale hoy pueda no valer para mañana. Para solucionar esto, lo ideal sería proporcionar a nuestro sistema la capacidad de incorporar los nuevos algoritmos de la manera más automática posible. 

Para ello habría que obligar al usuario a cambiar la contraseña para almacenar el hash según el nuevo algoritmo. La imposibilidad de hacer esto en masa con todos a la vez  (por muchos avisos que les hagamos, a un usuario solo se le puede obligar a cambiar la password en su próxima autentificación), nos fuerza a tener un campo en la base de datos que nos indique con qué algoritmo está generado su hash. De esta manera, la aplicación solo tendrá que autentificarlo en función de dicho algoritmo y,  cuando el usuario decida cambiar su contraseña (porque quiere o porque a pasado un cierto tiempo y le obligamos), cifrar el nuevo hash de la nueva contraseña con el algoritmo más adecuado en ese momento.

Las nuevas librerías de la mayoría de los lenguajes ya incorporan estas mejoras (generación aleatoria de sales y actualización de algoritmos) en sus métodos, haciendo todo muy fácil para los programadores. 

Por ejemplo, en PHP, existen las siguientes funciones:

**password_hash**(string $password, integer $algo, array $options = ?): string      ($algo= PASSWORD_DEFAULT )

**password_verify**(string $password, string $hash): bool

La función password_hash nos devuelve el hash que almacenaremos en la tabla de usuarios de la base de datos. Le tendremos que proporciona los siguientes datos:

- la password en claro
- el algoritmo que queremos. Con PASSWORD_DEFAULT utiliza el más adecuado, que podría cambiar a medida que tengamos la librería actualizada)
- un array con opciones generales, por ejemplo, el coste: digamos que es un número proporcional al número de iteraciones que va a realizar el algoritmo para complicar el cálculo del hash. Cuanto más alto, más se dificulta un ataque por fuerza bruta, pero también se tarda más en calcular el hash en cada login.

Realmente, esta cadena devuelta no solo tiene el hash, puesto que almacena cuatro informaciones necesarias para posteriormente realizar la validación:

- algoritmo utilizado
- opciones (p.e. coste)
- sal
- el propio hash

![Formato password hash](img/passwordhash.jpg)

Finalmente, en la autentificación utilizaremos password_verify pasándole la contraseña introducida por el usuario y el hash almacenado en la base de datos, devolviendo un booleano que nos indicará si esa password se corresponde con ese hash (si la autentificación ha sido correcta).

## Autentificación de usuario (login)

La idea para realizar un login sería entonces almacenar lo que devuelve la función hash en el campo password, De esta manera, se compara el hash de la password que ha introducido el usuario en el login con el hash almacenado en la base de datos en el registro correspondiente al usuario introducido. Si los hashes coinciden las credenciales serían correctas.

Aunque este razonamiento es cierto, existen varios problemas de seguridad, y entre ellos es que si utilizamos claves sencillas, conocidas, o con patrones comunes (abc123, querty, etc.), el uso  de tablas rainbow (una lista de pares de hashes precalculados y sus entradas originales) hace que sea muy sencillo encontrar una clave que nos permita autentificarnos sin problema.

Como no hay vuelta atrás, típicos ataques podrían ser usando fuerza bruta, diccionarios o tablas rainbow.

Pero no está recomendado utilizar directamente el valor que devuelven puesto que alguno de estos algortimos está “roto”, y el uso de tablas rainbow hace que pueda llegar a no ser especialmente complicado atacar el sistema.





Detección de ataques por fuerza bruta




cambio de password implica →
- actualizar campo de timestampProximoCambioPassword
- si almacenamos “a mano” el algoritmo, actualizar algoritmoHash al mejor hasta el momento
- si almacenamos “a mano” la sal, actualizarla


campos de la tabla usuarios para hacer un login completo: (ver tabla más abajo)
- campo BOOL de usuario desactivado (no está en la tabla)
- última fecha de autentificación correcta (y penúltima… ?, histórico → en log?)
- n.º intentos seguidos de error en autentificación → si pasa un MAX desactivar usuario




# Aspectos de seguridad: Logs.


Incluso realizando una buena programación para evitar todo tipo de ataques externos, siempre existirán vulnerabilidades de uno u otro tipo por lo que, en estos casos, es importante utilizar dos herramientas que pueden paliar (solo en parte) las consecuencias de algún problema: backups y logs. Tanto la copia de seguridad de los datos, como el seguimiento de los logs del sistema  es labor del administrador del servidor, existiendo multitud de herramientas existentes en el mercado para su automatización.

Pero cuando hablamos de logs no solo nos referimos al sistema operativo o al servidor web (que disponen de sus propios logs), sino de un log propio de nuestra aplicación donde se registran todas las operaciones realizadas así como los posibles ataques detectados. Y esto sí que es responsabilidad del programador. Además de registrar dichas operaciones habrá que prever su visualización (a través de los interfaces adecuados, integradados en el propio interfaz de la aplicación, o no), y solo para aquellos usuarios que dispongan de permisos.


![CREATE TABLE log](img/create-table-log.jpg)

![CREATE TABLE log](img/create-table-usuarios.jpg)
