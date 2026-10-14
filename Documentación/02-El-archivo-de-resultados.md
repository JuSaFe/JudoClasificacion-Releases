# El archivo de resultados

Es el contrato entre JudoAdministración y esta aplicación. Lo escribe
`ViewModels/Events/ExportImportViewModel.ExportarResultados` allí y lo lee
`Services/Importacion/ImportadorResultados` aquí; los dos modelos —`ResultadosEventoExport` y
`ResultadosEventoArchivo`— tienen que decir lo mismo.

## Forma

```json
{
  "Evento": "Copa FVJudo Castellón",
  "Categoria": "Alevín",
  "FechaInicio": "2025-11-02",
  "FechaFin": "2025-11-02",
  "Ciudad": "Castellón de la Plana",
  "Comunidad": "Comunidad Valenciana",
  "Pais": "España",
  "Ambito": "Autonómico (clubes)",
  "Elite": false,
  "Resultados": [
    {
      "Dni": "12345678Z",
      "IdLicencia": 71633,
      "Nombre": "Anahí",
      "Apellidos": "Valladares Mishell",
      "FechaNacimiento": "2014-05-02",
      "Sexo": "W",
      "ClubCodNombre": "NOV",
      "ClubNombre": "JUDO CLUB NOVELDA",
      "Comunidad": "Comunidad Valenciana",
      "Pais": "España",
      "Peso": 52,
      "SinLimite": true,
      "Elite": false,
      "Puesto": 1,
      "Descalificado": false
    }
  ],
  "PorEquipos": false,
  "Equipos": []
}
```

### Un evento por equipos

```json
{
  "Evento": "Liga de Clubes · 1ª jornada",
  "Categoria": "Senior",
  "FechaInicio": "2026-10-11",
  "…": "la misma cabecera",
  "Ambito": "Autonómico (clubes)",
  "Resultados": [],
  "PorEquipos": true,
  "Equipos": [
    {
      "Division": "Masculino",
      "Sexo": "M",
      "Equipo": "JUDO CLUB ALICANTE A",
      "CodAgrupacion": "JCA",
      "NombreAgrupacion": "JUDO CLUB ALICANTE",
      "Puesto": 1,
      "EquiposDivision": 8,
      "Miembros": [
        { "Dni": "12345678Z", "IdLicencia": 71633, "Nombre": "Carlos", "Apellidos": "García", "Sexo": "M" }
      ]
    }
  ]
}
```

## Lo que hay que saber

- **Un archivo es un evento, y un evento es una categoría.** La jornada de una copa en la que se
  disputan las cinco categorías son cinco archivos.
- **`IdLicencia` es la identidad.** Es la que da la federación y no cambia de una competición a otra:
  es lo que permite seguir al mismo judoka por toda la temporada sin listas de alias. `Dni` va también
  porque es la clave con la que se vuelve a escribir en JudoAdministración al asignar el ranking.
- **Se exportan TODOS los competidores**, con puesto o sin él. Cuántos había en el peso es un dato del
  que dependen algunos baremos, y perderlo obligaría a exportar dos archivos.
- **`Puesto` puede ser nulo.** Es la mayoría en un cuadro grande, y también lo es mientras el combate
  que decide ese puesto sigue sin disputarse.
- **El peso son tres columnas**: los kilos, `SinLimite` (el peso abierto, el `+`) y `Elite` (la
  variante de los eventos a dos niveles). Aquí se normalizan a una clave, `-46` o `+52`, y de fábrica
  los élite cuentan dentro de su peso normal (ver `Services/Clasificacion/PesoClave`).

## Los eventos por equipos

`PorEquipos` y `Equipos` son **opcionales**: un archivo anterior no los trae, se lee como `false` y
vacío, y entra exactamente igual que antes.

- **`PorEquipos: true` cambia qué se lee.** En un evento por equipos los puestos son de los equipos,
  así que `Resultados` viene **vacío a propósito**: ningún judoka tiene puesto individual, y contarlo
  como participante de un peso falsearía el ranking individual. Si aun así trajera algo, se ignora con
  un aviso; lo mismo con `Equipos` en un archivo que no está marcado como por equipos.
- **Cada equipo** trae su `Division` —el «peso» de los equipos: cada una es un cuadro y aquí una tabla
  aparte—, el `Sexo` de la división (`M`, `W`, o `X` si es mixta), su nombre, la **agrupación** a la
  que representa (`CodAgrupacion` / `NombreAgrupacion`: club, comunidad o país según `Ambito`; vacías
  en un combinado), el `Puesto` (nulo si no consiguió ninguno), cuántos equipos había en la división
  (`EquiposDivision`) y sus `Miembros`.
- **Se exportan todos los equipos**, con puesto o sin él, por lo mismo que los competidores de un peso.
- **La identidad es la agrupación**, no el nombre del equipo: «JC Alicante A» en una jornada y «JC
  Alicante» en la siguiente son el mismo club. Sin agrupación, el nombre del equipo.
- **Los miembros se guardan pero no cuentan**: el puesto es del equipo. No se dan de alta en
  `personas`, porque pueden venir sin licencia y el archivo no trae la fecha de nacimiento.
- Entran en su propia tabla (`resultados_equipos` + `miembros_equipo`) y en su propia clasificación,
  la de equipos (ver `Clasificador.PorEquipos`); ni el ranking individual ni el de clubes los ven.
  Reimportar el archivo reemplaza sus equipos igual que reemplaza los resultados individuales.

## Añadir un campo

Se añade en los dos modelos, **opcional** aquí. Los archivos antiguos siguen entrando: lo que
`System.Text.Json` no encuentra se queda en su valor por defecto, y lo que le sobra lo ignora. Lo
único que no se puede hacer es **renombrar** algo que ya existe.

## De dónde sale el puesto

De los combates. JudoAdministración vuelca el medallero de cada peso a la inscripción de sus
competidores (`registros.puesto`) en cuanto se anota un resultado, y lo vuelve a escribir si ese
resultado se corrige; ver `Services/Competicion/PuestosEvento` allí. La exportación además recalcula
el evento entero antes de escribir el archivo, porque lo que diga se arrastra a la clasificación de
toda la temporada.

Los pesos **sin combates** son la excepción: ahí no hay cuadro del que deducir nada y `registros.puesto`
se queda como esté, que es lo que permite anotar a mano el resultado de una competición que no se
disputó con la aplicación.
