# BankLite — pipelines

[![CI](https://github.com/Luisortiz2406/banklite-pipelines/actions/workflows/ci.yml/badge.svg)](https://github.com/Luisortiz2406/banklite-pipelines/actions/workflows/ci.yml)

Workflow reutilizable de GitHub Actions (`workflow_call`) para los 15 repositorios
de BankLite (`banklite-<servicio>`). Cada repositorio lo llama con un `deploy.yml`
corto; así la lógica de pruebas, imagen y despliegue vive en un solo lugar.

## Qué hace

`.github/workflows/servicio.yml` tiene tres jobs encadenados con `needs`:

```
test ──► docker ──► deploy
```

| Job | Cuándo corre | Qué hace |
|---|---|---|
| `test` (Pruebas) | Pull requests y push a `main` | `npm ci`, ESLint, Prettier (`format:check`), `npm run build` y `npm test` (si existen; con Jest, con informe de cobertura). Con `postgres: true` levanta un Postgres 16 y exporta `TEST_DATABASE_URL`. Al final corre la `prueba-extra` del servicio, si tiene. |
| `docker` (Imagen Docker) | Solo push a `main` (o ejecución manual en `main`), si `test` pasó | Construye la imagen con el `Dockerfile` del repo y la sube a GHCR (`ghcr.io/luisortiz2406/banklite-<servicio>`) con dos etiquetas: el SHA del commit y `latest`. Borra las versiones viejas y deja las últimas 5. |
| `deploy` (Despliegue) | Solo si `docker` pasó | `railway up --ci` sube el código del commit al servicio de Railway (proyecto `lively-warmth`, entorno `production`), Railway lo construye con el `Dockerfile` y el `railway.json` del repo, y el job espera a que el despliegue quede en `SUCCESS` (falla si queda en `FAILED` o `CRASHED`). |

En un pull request solo corre `test`; `docker` y `deploy` aparecen como omitidos
(*skipped*). Si `test` falla en `main`, no se construye la imagen ni se despliega.

### Por qué Railway construye otra vez la imagen (opción B)

La imagen de GHCR es privada y Railway solo puede leer registros privados en el plan
Pro. Por eso el despliegue usa `railway up` (Railway construye desde el código del
mismo commit) y la imagen de GHCR queda como **registro** de cada versión, etiquetada
con su SHA.

**Mejora futura (opción A):** que Railway despliegue exactamente la imagen
`ghcr.io/…:<sha>` probada en Actions (servicio con fuente *Docker image* y
credenciales del registro). Eso evita construir dos veces y el rollback sería volver a
correr el workflow con un SHA anterior. Requiere el plan Pro de Railway o imágenes
públicas.

## Cómo se usa

En cada repositorio de servicio, `.github/workflows/deploy.yml`:

```yaml
name: Despliegue

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  workflow_dispatch:

permissions:
  contents: read
  packages: write

jobs:
  pipeline:
    uses: Luisortiz2406/banklite-pipelines/.github/workflows/servicio.yml@v1
    with:
      servicio: savings-service
      postgres: true
    secrets: inherit
```

`permissions` lo pone el repositorio que llama: el workflow reutilizable no puede
pedir más permisos que esos (`packages: write` es para subir y borrar imágenes en GHCR).

### Entradas

| Entrada | Tipo | Por defecto | Para qué |
|---|---|---|---|
| `servicio` | texto | — (obligatoria) | Nombre del servicio en Railway (`savings-service`, `api-gateway`, `frontend`, …) |
| `postgres` | sí/no | `false` | Postgres de prueba y `TEST_DATABASE_URL` para las pruebas con base |
| `prueba-extra` | texto | `''` | Comando que se corre al final de `test` |
| `node-version` | texto | `20` | Versión de Node.js |
| `umbral-lineas` | número | `0` | Cobertura mínima de líneas (%) de Jest; con `0` solo se muestra |
| `umbral-ramas` | número | `0` | Cobertura mínima de ramas (%) de Jest; con `0` solo se muestra |

Configuración de cada repositorio (lo mismo que probaba su `ci.yml`):

| Repositorio | `postgres` | `prueba-extra` |
|---|---|---|
| account, compliance, rewards | `true` | `npm run verificar:esquema` (esquema SQL igual a las entidades; instala `psql` si falta) |
| auth, card, deposit, fraud, ledger, payment, savings | `true` | — |
| api-gateway, kyc, notification, transfer | `false` | — |
| frontend | `false` | `sh scripts/probar-imagen.sh` (construye la imagen de nginx y prueba rutas, archivos que no deben quedar publicados y la CSP) |

### Cobertura de pruebas

Si el servicio tiene Jest configurado en su `package.json`, el paso de pruebas corre
`npm test -- --coverage` sobre **todo `src/`** (sin `*.spec.ts` ni `main.ts`), de modo
que un archivo sin pruebas cuenta como 0 % en vez de quedar fuera del cálculo. Después:

- **Resumen de la corrida:** en la pestaña de la corrida de Actions aparece la tabla de
  líneas, sentencias, funciones y ramas, y los 5 archivos con menos cobertura.
- **Artefacto `cobertura`:** el informe `lcov` y `coverage-summary.json` (14 días).
- **Umbral opcional:** con `umbral-lineas` y `umbral-ramas` el job falla si la cobertura
  queda por debajo. Con el valor por defecto (`0`) solo se muestra el reporte y no se
  bloquea nada, así que los repositorios que ya usan `@v1` no cambian de comportamiento.

Para exigir un mínimo, el repositorio lo pide en su `deploy.yml`:

```yaml
    with:
      servicio: ledger-service
      postgres: true
      umbral-lineas: 45
      umbral-ramas: 23
```

Los servicios sin Jest (frontend, que usa `node --test`) siguen con `npm test` y el paso
de cobertura avisa que no hay reporte. Un servicio sin pruebas aparece sin reporte.

### Secrets

| Secret | Dónde | Para qué |
|---|---|---|
| `RAILWAY_TOKEN` | Secret de cada repositorio de servicio (no en este) | Token de **proyecto** de Railway (`lively-warmth`, entorno `production`). Lo usa `deploy`. |
| `GITHUB_TOKEN` | Lo da Actions | Subir y borrar imágenes en GHCR |

`secrets: inherit` le pasa al workflow reutilizable los secrets del repositorio que
llama. Si falta `RAILWAY_TOKEN`, `deploy` falla con un mensaje claro (las pruebas de
los pull requests no lo necesitan).

### Versiones de este workflow

Los servicios llaman a la etiqueta `v1`. Un cambio compatible se mergea en `main` y
se mueve `v1`; un cambio que rompe la interfaz (entradas, secrets) sale como `v2` y
cada servicio se actualiza con su propio PR.

## Requisitos

- **Acceso de los otros repositorios:** este repositorio es privado, así que en
  *Settings → Actions → General → Access* debe estar «Accessible from repositories
  owned by the user 'Luisortiz2406'». Sin eso, los `deploy.yml` fallan al resolver
  `uses:`.
- **Autodeploy de Railway apagado** en cada servicio que use este pipeline (su repo
  desconectado en Railway); si no, cada push a `main` desplegaría dos veces.

## Rollback

- **Una versión no arrancó:** si el healthcheck no pasa, Railway mantiene la versión
  anterior. Para volver a una versión anterior que sí quedó activa: panel de Railway
  → servicio → *Deployments* → la versión anterior → *Rollback*. Después se corrige
  con un PR nuevo (o `git revert`) para que `main` quede igual a lo desplegado.
- **El pipeline está mal y hay que volver a como era antes:** se vuelve a conectar el
  repositorio en Railway (servicio → *Settings → Source → Connect Repo*, rama `main`,
  o `railway service source connect --repo Luisortiz2406/banklite-<servicio> --branch main --service <servicio>`)
  y Railway vuelve a desplegar en cada push.

## Consumo de Actions

Corredores Linux (multiplicador 1x); cada job se redondea al minuto.

| Corrida | Minutos aproximados |
|---|---|
| Pull request (solo `test`) | 1–2 |
| Push a `main` (`test` + `docker` + `deploy`, que espera el build de Railway) | 5–8 |

Con unos 40 PR y 40 merges al mes en todos los repositorios son ~350 min/mes, dentro
de los 2.000 min/mes del plan Free para repositorios privados. El espacio en GHCR se
controla dejando 5 versiones por imagen.

## CI de este repositorio

`.github/workflows/ci.yml` revisa los workflows con
[actionlint](https://github.com/rhysd/actionlint) en cada PR.
