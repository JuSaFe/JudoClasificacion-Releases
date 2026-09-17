# La base de datos

## Qué se monta y dónde

Una base de datos **PostgreSQL** llamada `judoclasificacion`, propiedad de un rol del mismo nombre, en
el servidor local del equipo.

**La aplicación no instala PostgreSQL.** Se da por hecho que está, porque el equipo ya lo necesita
para JudoAdministración. Si no está, la pantalla lo dice y no deja seguir: montar un servidor de base
de datos no es cosa de esta aplicación, y JudoAdministración ya tiene su pantalla para eso.

## La primera vez

La pantalla **Base de datos** —que sale sola al arrancar sin conexión, y a la que después se llega
por la rueda dentada de la barra, **⚙ ▸ Base de datos**— ofrece dos caminos, que son dos situaciones
distintas:

### Crear la base de datos

1. Se sondea el equipo: ¿hay algo escuchando en el 5432?, ¿hace falta contraseña de superusuario?

   Esto último se **prueba**, no se deduce del sistema operativo: en macOS con Homebrew el
   superusuario del clúster es el propio usuario que ha iniciado sesión, y en Linux con autenticación
   *peer* pasa algo parecido. En esos casos no se pide ninguna contraseña, porque no existe.

2. Se entra como superusuario y se ejecuta lo que haría a mano alguien que supiera:

   ```sql
   CREATE ROLE "judoclasificacion" WITH LOGIN PASSWORD '…';
   CREATE DATABASE "judoclasificacion" OWNER "judoclasificacion";
   ```

3. Se comprueba **de verdad** que se puede entrar con el rol nuevo antes de dar nada por bueno: crear
   el rol y la base puede salir bien y aun así no poder conectar, si el `pg_hba.conf` del equipo no
   admite contraseña para conexiones locales. Mejor decirlo ahí que en la primera pantalla.

4. Se guarda la configuración de este equipo y **se enseña la contraseña generada**.

### Ya tengo una

Para el equipo reinstalado, el segundo equipo o la copia restaurada. Se piden servidor, base, usuario
y la contraseña que se entregó al crearla, y no se toca nada más: los datos están donde estaban.

### Cuando ya hay conexión

La pantalla **no vuelve a pedir nada**. Dice a qué base de datos está conectada —nombre, servidor y
usuario— y deja seguir. El formulario de los dos caminos solo reaparece si se pulsa **Cambiar de base
de datos**, que es algo que se hace una vez cada varios años.

Esto no es cosmética: la pantalla se abre también desde ⚙ ▸ Base de datos con la aplicación
funcionando, y enseñar allí un formulario pidiendo la contraseña del superusuario se lee como que la
conexión no está hecha.

Y si se llega aquí con la conexión **caída** teniendo configuración guardada —PostgreSQL parado, por
ejemplo—, la contraseña viene ya puesta: este equipo la tiene en su archivo de configuración, así que
basta con arrancar el servidor y pulsar «Continuar».

## La contraseña

La genera la aplicación, no la elige el usuario: 20 caracteres de un alfabeto **sin parecidos** (ni
`I`, ni `l`, ni `O`, ni `0`) y sin símbolos. Esa contraseña se lee de una pantalla y se teclea a mano
meses después; confundir una ele con un uno no puede ser posible, y los símbolos complican copiarla en
una terminal sin entrecomillar.

**Se enseña una sola vez y hay que guardarla.** Este equipo ya la tiene, pero es la llave de los
datos: el día que se reinstale el sistema, se cambie de equipo o se restaure una copia, es lo único
que deja volver a entrar en la base que ya está.

### Por qué se guarda en claro en el equipo

El archivo de configuración (`conexion.json`, en la carpeta de datos del usuario) lleva la contraseña
sin cifrar, y conviene saber por qué antes de darle vueltas: es la contraseña de un rol de PostgreSQL
que solo escucha en este equipo y que solo puede tocar su propia base. Quien pueda leer ese archivo es
quien ha iniciado sesión en el equipo, y ése ya puede abrir la aplicación y ver los mismos datos.
Cifrarla con una clave que tendría que estar también en el equipo no añadiría nada, solo lo parecería.

### Si se pierde

No se puede recuperar: PostgreSQL no la guarda, guarda su hash. Lo que sí se puede es **reponerla**,
volviendo a ejecutar «Crear la base de datos» con el mismo nombre: el rol ya existe y se le pone una
contraseña nueva (`ALTER ROLE`), la base ya existe y se conserva con todos sus datos, y se entrega la
contraseña nueva. Hace falta, otra vez, la del superusuario.

## Por qué la configuración no va junto al ejecutable

Dos motivos que van juntos:

- En Windows la aplicación se instala en Archivos de programa, donde un usuario normal **no puede
  escribir**.
- Una actualización **reemplaza esa carpeta entera**, y la configuración tiene que sobrevivir a la
  actualización, que es precisamente cuando más falta hace.

Por eso vive en la carpeta de datos del usuario:

| Sistema | Ruta |
|---------|------|
| Windows | `%APPDATA%\JudoClasificacion\conexion.json` |
| macOS | `~/Library/Application Support/JudoClasificacion/conexion.json` |
| Linux | `~/.config/JudoClasificacion/conexion.json` |

Al ser **del usuario**, la aplicación tiene que arrancar siempre con la cuenta de quien está delante.
Por eso el instalador de Windows lanza la aplicación al terminar con la bandera `runasoriginaluser`
(ver `Empaquetado/windows/JudoClasificacion.iss`): el instalador corre elevado, y sin esa bandera la
aplicación heredaría la cuenta del administrador que aceptó el aviso de Windows. La configuración se
guardaría entonces en el perfil de ese administrador y la aplicación volvería a pedir la contraseña
en cada arranque normal.

## Copias de seguridad

Con las herramientas de PostgreSQL, igual que las de JudoAdministración:

```bash
pg_dump -U judoclasificacion -h localhost judoclasificacion > copia.sql
```

Para restaurar en otro equipo: crear allí la base con **⚙ ▸ Base de datos**, restaurar el
volcado encima, y después conectarse con «Ya tengo una».

## Cambios de esquema

Las tablas se crean al arrancar con `CREATE TABLE IF NOT EXISTS`, así que una versión nueva de la
aplicación **da de alta sola lo que falte**. Lo que no hace es migrar lo que ya existe: cambiar el
tipo de una columna, partir una tabla o cualquier cosa que pueda perder datos se aplica a mano.

Es la misma decisión que en JudoAdministración y por el mismo motivo: una migración automática en el
arranque es una migración que se ejecuta sin que nadie la esté mirando.

### La excepción: el bloque `Migraciones`

`BaseDatos.cs` tiene un bloque `Migraciones` que **sí** se ejecuta en cada arranque, con dos
condiciones: cada orden es **idempotente** (`ADD COLUMN IF NOT EXISTS`) y **no puede perder datos**.
Hoy tiene una sola:

```sql
ALTER TABLE competiciones
    ADD COLUMN IF NOT EXISTS id_circuito INT REFERENCES circuitos(id) ON DELETE CASCADE;
```

Va ahí y no en la lista de «a mano» porque sin esa columna la aplicación no arranca: dejarlo para
después significaría que cualquiera que actualice se encuentra la aplicación rota sin saber por qué.

Detrás corre `AdoptarCompeticionesSueltas`, que mete en el circuito por defecto de su temporada
—creándolo si no está— las competiciones que se importaron antes de que existieran los circuitos. El
histórico de quien ya usaba la aplicación es el circuito **oficial**, porque es lo que era.

Añadir algo a este bloque solo está justificado si cumple las dos condiciones. Si no, va a mano y se
documenta aquí.
