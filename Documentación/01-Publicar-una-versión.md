# Publicar una versión

Igual que en JudoAdministración y en JudoCombates: se empuja un tag y GitHub Actions genera los tres
instaladores, cada uno en su propio runner.

## 1. Antes de nada, la versión

La versión manda desde `Directory.Build.props`, **no** desde el nombre del tag:

```xml
<Version>1.0.0.1</Version>
```

El workflow lo comprueba antes de gastar los tres runners: si el tag es `v1.0.0.2` y ahí pone
`1.0.0.1`, se para. Publicar un `v1.0.0.2` con binarios sellados `1.0.0.1` es exactamente el lío que
esa comprobación evita.

## 2. Empujar el tag

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

Desde la pestaña **Actions**, «Instaladores» ▸ *Run workflow*. Sin marcar la casilla «publicar»
compila los tres y los deja como artefactos descargables, sin tocar ninguna release.

## 4. Lo que hay que dejar montado UNA vez

El workflow publica la release en un repositorio **público aparte** —`JuSaFe/JudoClasificacion-Releases`,
la variable `REPO_PUBLICO`— y no en éste. Es el mismo reparto que JudoAdministración: el código en un
repositorio y las descargas en otro que solo tiene portada, documentación y releases.

Para que funcione hacen falta dos cosas, y todavía **no están hechas**:

1. **Crear el repositorio** `JuSaFe/JudoClasificacion-Releases`.
2. **Crear el secreto `TOKEN_RELEASES`** en este repositorio (Settings ▸ Secrets and variables ▸
   Actions): un PAT de alcance fino con permiso *Contents: Read and write* **sobre el repositorio
   público**. El `GITHUB_TOKEN` del runner no vale: solo alcanza a este repositorio.

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
