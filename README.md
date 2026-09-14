<!--
  PORTADA DEL REPOSITORIO PÚBLICO DE DESCARGAS.

  No se edita allí: el trabajo `publicar` de .github/workflows/instaladores.yml copia este archivo
  como README.md del repositorio público en cada release, junto con la licencia, el aviso de
  terceros y la documentación. Cualquier cambio hecho a mano en el otro repositorio se pierde en la
  siguiente publicación; se edita aquí.
-->

<div align="center">

<img src="Assets/Icons/judo-256.png" alt="Judo Clasificación" width="128">

# Judo Clasificación

**El ranking de la temporada, sacado de los resultados de las competiciones.**

[![Descargar](https://img.shields.io/badge/Descargar-última%20versión-brightgreen?logo=github)](../../releases/latest)
[![Licencia](https://img.shields.io/badge/Licencia-Uso%20libre%20·%20sin%20derivados-green.svg)](LICENSE)
[![Windows · macOS · Linux](https://img.shields.io/badge/Windows%20·%20macOS%20·%20Linux-multiplataforma-informational)](#instalación)

</div>

---

## Qué es

La aplicación que lleva la **clasificación del circuito**: se cargan los archivos de resultados que
exporta [**JudoAdministración**](https://github.com/JuSaFe/JudoAdministracion-Releases), y con ellos
sale el ranking de cada categoría, cada sexo y cada peso, más el ranking de clubes y el informe en
PDF listo para publicar.

Cierra el círculo por el otro lado: con la clasificación que hay hasta ese momento numera a los
inscritos de la **siguiente** competición, y ese archivo se importa de vuelta en JudoAdministración
para que el sorteo separe a los cabezas de serie.

Una temporada puede llevar **varios circuitos a la vez** —el oficial de la federación y el
extraoficial de iniciación—, cada uno con sus jornadas, sus puntos y su propio informe.

No es un cliente de nadie: **funciona sola**. Habla directamente con PostgreSQL, sin servidor de por
medio.

## Instalación

Descarga el instalador de tu sistema desde la [**última versión**](../../releases/latest):

| Sistema | Archivo |
|---|---|
| Windows 10/11 (x64) | `.exe` |
| macOS (Apple Silicon) | `.dmg` |
| Linux (x64) | `.AppImage` |

Es **autocontenido**: no hace falta instalar .NET. Sí hace falta **PostgreSQL** en marcha en el
equipo —el mismo que ya necesita JudoAdministración—: la aplicación crea allí su propia base de
datos y su propio usuario la primera vez que se abre, sin tocar los de nadie más.

Cómo montarla, la contraseña que entrega y las copias de seguridad, en la
[**documentación**](Documentación/).

## Soporte

¿Un fallo, una duda o una propuesta? Abre una
[**incidencia**](../../issues/new) describiendo qué ocurre, en qué sistema y con qué versión.

El desarrollo se lleva en un repositorio privado; este de aquí es el punto de descarga y el canal
de incidencias.

## Licencia

Software **de código propietario y uso gratuito**. Texto completo en **[LICENSE](LICENSE)**.

**Se permite**, gratis y sin límite de equipos, usuarios ni tiempo:

- Descargar, instalar y ejecutar el programa, incluso con fines profesionales o comerciales.
- Redistribuirlo **íntegro y sin modificar**, a título gratuito y conservando todos los avisos.

**No se permite** modificarlo, crear obras derivadas, distribuir versiones modificadas, republicarlo
en otro repositorio ni reutilizar partes de él en otros proyectos.

La **propiedad intelectual del programa es de Juan Cotolí San Félix**.

> **Marcas y logotipos.** Los escudos federativos que incluye la aplicación son propiedad de sus
> respectivas entidades y no están cubiertos por esta licencia.

Las bibliotecas de terceros que incorpora se rigen por sus propias licencias, detalladas en
[TERCEROS.md](TERCEROS.md).

---

<div align="center">
<sub>Hecho para el tatami. 🥋</sub>
</div>
