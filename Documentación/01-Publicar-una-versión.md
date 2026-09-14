# Publicar una versión

Igual que en JudoAdministración y en JudoCombates: se empuja un tag y GitHub Actions genera los tres
instaladores, cada uno en su propio runner.

## 0. Las ramas

Dos, como en el resto de los proyectos:

| Rama | Para qué |
|---|---|
| **`develop`** | Donde se trabaja. Aquí no se publica nada. |
| **`master`** | De donde salen las versiones. Lo que hay aquí es lo que se ha publicado o está a punto. |

Se trabaja en `develop`, se fusiona en `master` cuando la versión está lista, y **el tag se pone
sobre `master`**. El workflow se dispara con el tag, no con la fusión: empujar a `master` sin tag no
publica nada.

```bash
git switch master && git merge develop && git push
```

## 1. Antes de nada, la versión

La versión manda desde `Directory.Build.props`, **no** desde el nombre del tag:

```xml
<Version>1.0.0.1</Version>
```

El workflow lo comprueba antes de gastar los tres runners: si el tag es `v1.0.0.2` y ahí pone
`1.0.0.1`, se para. Publicar un `v1.0.0.2` con binarios sellados `1.0.0.1` es exactamente el lío que
esa comprobación evita.

## 2. Empujar el tag

Con `master` ya al día y estando en ella:

```bash
git tag v1.0.0.1 && git push origin v1.0.0.1
```

Salen:

| Sistema | Archivo | Herramienta |
|---------|---------|-------------|
| Windows | `JudoClasificacion-<versión>-win-x64.exe` | Inno Setup |
| macOS | `JudoClasificacion-<versión>-osx-arm64.dmg` | hdiutil |
| Linux | `JudoClasificacion-<versión>-x86_64.AppImage` | appimagetool |

Cada sistema compila el suyo porque las herramientas de empaquetado solo existen en su sistema. El
`dotnet publish` sí es multiplataforma, pero por sí solo no produce nada instalable.

## 3. Probar sin publicar

Desde la pestaña **Actions**, «Instaladores» ▸ *Run workflow*. Lanzado así compila los tres y los
deja como artefactos descargables **sin publicar ninguna release**: el trabajo `publicar` solo corre
cuando lo que se ha empujado es un tag.

## 4. Lo que hay que dejar montado UNA vez

El workflow publica la release en un repositorio **público aparte** —`JuSaFe/JudoClasificacion-Releases`,
la variable `REPO_PUBLICO`— y no en éste. Es el mismo reparto que JudoAdministración: el código en un
repositorio y las descargas en otro que solo tiene portada, documentación y releases.

Para que funcione hacen falta dos cosas:

1. ~~**Crear el repositorio** `JuSaFe/JudoClasificacion-Releases`~~ — **hecho**. Está creado,
   público, y con su rama `master` inicializada: portada, licencia, aviso de terceros, el icono y
   esta documentación. Lo que hay allí lo reescribe el workflow en cada publicación, así que **no se
   edita a mano**: la portada se edita en `Empaquetado/publico/README.md` de este repositorio.
2. **Crear el secreto `TOKEN_RELEASES`** en este repositorio (Settings ▸ Secrets and variables ▸
   Actions): un PAT de alcance fino con permiso *Contents: Read and write* **sobre el repositorio
   público**. El `GITHUB_TOKEN` del runner no vale: solo alcanza a este repositorio. **Esto sigue
   pendiente**, y sin ello la publicación se para en el primer trabajo.

El workflow comprueba que el secreto existe **antes** de compilar nada, así que si falta lo dice en
el primer trabajo y no después de gastar tres runners.

Si se prefiere publicar las releases aquí mismo —como hace JudoLicencias—, hay que cambiar el trabajo
`publicar` del workflow para usar `GITHUB_TOKEN` contra `github.repository`.

## 5. A mano, sin GitHub

```bash
bash Empaquetado/generar-instaladores.sh          # el del sistema en el que se ejecute
bash Empaquetado/generar-instaladores.sh mac
bash Empaquetado/generar-instaladores.sh linux
.\Empaquetado\generar-instaladores.ps1            # Windows, con Inno Setup instalado
```

Lo intermedio queda en `dist/` (ignorado por git) y el archivo final en `Instaladores/`.
