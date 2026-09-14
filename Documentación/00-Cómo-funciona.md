# Cómo funciona

## 1. Qué es y qué no es

Es una aplicación de escritorio de **un solo proyecto** que lee archivos, los guarda en PostgreSQL y
escribe archivos. No tiene API, ni servicio, ni red.

Eso la separa de JudoAdministración a propósito. Aquélla la usan cinco puestos a la vez en un pabellón
y tiene que aguantar que dos personas anoten resultados al mismo tiempo: por eso además de PostgreSQL
tiene una API y un servicio de servidor. Ésta la abre **una persona, en su equipo**, para montar el
ranking de la temporada y sacar el PDF: le basta con hablar con PostgreSQL directamente.

Lo que sí comparte es el **motor de base de datos**, y a propósito. Quien usa esta aplicación ya tiene
PostgreSQL montado, porque es lo que necesita JudoAdministración. Aprovecharlo evita tener dos
tecnologías de base de datos en la misma federación para las mismas competiciones: una copia de
seguridad, un procedimiento de recuperación y un sitio donde mirar cuando algo va mal. La base es
SUYA —con su propio rol y su propia base, sin tocar la de JudoAdministración— pero vive en el mismo
clúster.

Lo que sí se comparte con JudoAdministración es todo lo que se ve y todo lo que se publica: la misma
paleta (`Styles/Tema.axaml`), los mismos estilos de botón, la misma cabecera en los PDF y el mismo
workflow de instaladores. Las dos acaban en el mismo tablón y tienen que parecer de la misma casa.

## 2. Las piezas

```
Models/          Los dos archivos JSON con los que habla: resultados y participantes.
Services/
  Datos/         PostgreSQL: la conexión, el esquema, las temporadas y los ajustes.
  Sistema/       Ejecutar órdenes con permisos de administrador (portado de JudoAdministración).
  Actualizacion/ Comprobar, descargar y sustituir la aplicación.
  Importacion/   Meter un resultados.json en la base de datos.
  Clasificacion/ El cálculo del ranking. Es el corazón.
  Ranking/       Poner el Ranking en un participantes.json.
  Pdf/           El informe, con QuestPDF.
ViewModels/      Una por pantalla. CommunityToolkit.Mvvm.
Views/           Su XAML. Las empareja ViewLocator por el nombre.
```

## 3. La base de datos

PostgreSQL, en el equipo. El esquema entero está en `Services/Datos/BaseDatos.cs`, con un comentario
por tabla explicando para qué está.

Tres piezas que conviene no confundir:

- **`ConfiguracionConexion`** — a qué base apunta ESTE equipo. Un JSON en la carpeta de datos del
  usuario, no junto al ejecutable: en Windows la aplicación se instala en Archivos de programa, donde
  un usuario normal no puede escribir, y además una actualización reemplaza esa carpeta entera.
- **`InstalacionBaseDatos`** — crear el rol y la base entrando como superusuario. Se hace una vez.
- **`SesionDatos`** — la conexión con la que trabaja la aplicación ahora mismo, **si la hay**. No es
  un singleton fijo a propósito: en un primer arranque no existe, y en uno cualquiera puede haber
  configuración pero no conexión (PostgreSQL parado, contraseña cambiada). Registrarla como servicio
  fijo impediría abrir la aplicación justo cuando hay que entrar a arreglarlo.

Lo importante del esquema:

- **`temporadas`** → **`circuitos`** → **`competiciones`** → **`resultados`**. Una competición es un
  archivo importado, o sea un evento de JudoAdministración, o sea **una categoría**.
- **`circuitos`** es el grupo dentro del que se clasifica: el oficial de la federación, el
  extraoficial de iniciación. Una temporada no produce un ranking sino varios, y sumarlos mezclaría
  dos que no se parecen ni en el nivel ni en quién compite. Todo el cálculo pasa **dentro de uno**:
  las jornadas se numeran por circuito (`competiciones.orden`), los puntos suman por circuito y el
  informe sale por circuito con su propio título (`circuitos.titulo`). Ver el apartado 5.
- **`personas`** va por **`id_licencia`**. Es la diferencia grande con el ranking que había antes:
  aquél leía PDF y tenía que reconocer al mismo judoka por el nombre, con su lista de alias para los
  acentos rotos y los apellidos a medias. Aquí la identidad la da la federación y no cambia.
- **`resultados.peso_clave`** es el peso ya normalizado con el que se agrupa el ranking. Se calcula
  **al importar** y se guarda, en vez de al vuelo, para que una fila corregida a mano se quede donde
  la han puesto.
- **`resultados.activo`** permite anular un resultado sin borrarlo, con su motivo: es lo que hace
  falta para justificar por qué la clasificación publicada no coincide con el acta.
- **`baremos` + `baremo_reglas` + `competiciones.id_baremo`** es lo que hace que cambiar el baremo no
  reescriba el histórico. Ver el apartado 4.

El esquema se crea al arrancar y solo da de alta lo que falta (`CREATE TABLE IF NOT EXISTS`), igual
que en JudoAdministración: los cambios sobre una base que ya existe se aplican a mano, con la
excepción acotada del bloque `Migraciones` de `BaseDatos.cs`. Ver 03-La-base-de-datos, «Cambios de
esquema».

## 4. El baremo se versiona

Es la decisión de diseño menos evidente de la aplicación, así que conviene tenerla clara antes de
tocar `BaremoTabla`.

El baremo **no es una tabla que se edita**: es una sucesión de versiones. Cada competición guarda con
cuál entró (`competiciones.id_baremo`) y sigue puntuando con ésa para siempre. Guardar desde la
pantalla de configuración **crea una fila nueva** en `baremos` y la deja apuntada en el ajuste
`baremo_actual`, que es el que usarán las competiciones que se carguen después.

El motivo: la federación cambia el baremo de un año para otro, y a veces a mitad de temporada. Si las
reglas se editaran en sitio, ese cambio reescribiría hacia atrás la clasificación que ya se publicó en
noviembre —quien iba tercero pasaría a ir quinto sin que nadie hubiera tocado un resultado—. Eso no es
configurar, es falsear el histórico.

Consecuencia para el cálculo: **los puntos se resuelven fila a fila**, con el baremo de la competición
de cada resultado, y no de una vez con una sola tabla. En una temporada a mitad de cambio conviven
dos, así que `Clasificador` se los trae todos juntos (`BaremoTabla.ReglasDe`) en vez de consultar uno
por jornada.

Para el caso en que sí hay que recalcular hacia atrás —un baremo que se metió MAL desde el principio—
está `BaremoTabla.Reasignar`, que la pantalla de competiciones ofrece fila a fila con un aviso claro.
Nunca automáticamente.

## 5. El cálculo

Todo en `Services/Clasificacion/Clasificador.cs`, y solo ahí: la tabla de la pantalla y la del PDF
salen las dos de él, así que no pueden decir cosas distintas.

Los cuatro pasos están escritos en el comentario de cabecera de esa clase. Lo que conviene tener
presente al tocarlo:

- **Todo pasa dentro de un circuito.** Las cuatro funciones públicas (`Categorias`, `Jornadas`,
  `Individual`, `PorClub`) reciben un `idCircuito`, no una temporada: el circuito ya sabe de qué
  temporada es, y pasar las dos cosas solo abriría la puerta a que no casaran. Quien llama resuelve
  el circuito activo con `Circuitos.Activo(conn, idTemporada)`.
- **El grupo es (categoría, sexo, peso).** El sexo no está dentro de la clave del peso porque no es
  parte del peso: es otra tabla. Un −44 masculino y un −44 femenino se llaman igual y no tienen nada
  que ver.
- **Las N mejores jornadas** es lo que evita que quien no puede ir a todas las copas quede
  descolgado. Las que no cuentan salen igualmente en el informe, en gris y en cursiva: son resultados
  reales y la gente los busca.
- **Los empates comparten posición** (1, 2, 3, 3, 5…). El nombre solo ordena; no deshace el empate.

## 6. Actualizar

`Services/Actualizacion` está portado de JudoAdministración y recortado a lo que aquí hay: **un solo
paquete**, el de la aplicación (allí hay dos, porque el servicio de servidor se actualiza aparte y se
puede quedar en otra versión).

Lo demás es igual, empezando por lo que no es evidente: un programa no puede sobrescribirse a sí mismo
mientras se ejecuta, así que `ActualizadorAplicacion` escribe un guion, lo deja lanzado y cierra la
aplicación; el guion espera a que el proceso muera, sustituye y vuelve a abrirla. El permiso de
administrador se pide **antes** de cerrar, no después: un diálogo de contraseña que aparece cuando la
aplicación ya ha desaparecido de la pantalla no se entiende, y si nadie lo contesta el equipo se queda
sin la versión vieja y sin la nueva.

Aquí la actualización **no toca la base de datos**, a diferencia de la de JudoAdministración, que
antes de nada tiene que volcar la del servidor. Las tablas que falten las crea la propia aplicación al
arrancar.

## 7. Lo que no está hecho

- **Control de licencias.** La licencia de esta aplicación (LICENSE) concede el uso gratuito e
  ilimitado, como la de JudoCombates, así que no hay nada que comprobar. Si algún día se decide
  cobrarla, lo que hay que portar es `Services/Licencia` de JudoAdministración entero
  (`ArchivoLicencia`, `ClaveLicencias`, `HuellaEquipo`, `ServicioLicencia`) con **su propio par de
  claves**, poner su pantalla delante de la ventana principal y cambiar la cláusula 2 del LICENSE.
  Importante: el orden que describe `ClaveLicencias` —emitir primero, publicar la clave después— no
  es opcional.
- **Editar un resultado desde la pantalla.** La tabla `resultados` tiene ya `activo` y `nota` para
  anular uno concreto, pero no hay interfaz. Mientras tanto se corrige en JudoAdministración y se
  vuelve a exportar, que además es lo correcto: el ranking no debería poder decir algo que el acta no
  dice.
