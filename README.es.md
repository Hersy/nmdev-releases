# nmdev

[English](README.md) · Español

Una herramienta de línea de comandos para un equipo mobile de dos apps (iOS y Android) sobre
Bitbucket y Jira: lleva un ticket hasta un PR publicado, responde reviews, revisa PRs ajenos,
mantiene los epics al día con `develop`, compila y distribuye builds QA/Prod por Firebase App
Distribution y, en segundo plano, porta cambios entre las dos apps. Las etapas de agente corren
sobre el CLI de claude, codex o gemini.

Solo macOS. Todos los verbos aceptan `--json` (un único objeto JSON de resultado, siempre en
inglés) y `-C <DIR>` (correr en ese repo en vez del directorio actual).

- [Instalación](#instalación)
- [Setup](#setup)
- [Estructura del pack](#estructura-del-pack)
- [Verbos](#verbos)
- [Referencia de configuración](#referencia-de-configuración)
- [Entorno](#entorno)
- [Problemas comunes](#problemas-comunes)

## Instalación

```sh
brew install hersy/tap/nmdev
nmdev setup
```

La fórmula instala `nmdev`, los completions de zsh/bash/fish, `man nmdev` y `man nmdev-<verbo>`
(las páginas en español van en `man/es`) y el `Gemfile` de fastlane que usa `ship`. Para actualizar:
`brew upgrade hersy/tap/nmdev`.

Herramientas que nmdev busca en el `PATH` de login (setup imprime el comando de instalación de cada
una que falte): `git` (`xcode-select --install`), `graphify` (`uv tool install graphifyy`), el CLI
de `firebase` (`mise use -g npm:firebase-tools`) y el CLI del backend de agente que elijas
(`claude`, `codex` o `gemini`). Para compilar las apps además necesitás las herramientas de cada
plataforma (`xcodebuild`, `pod`, Gradle).

## Setup

`nmdev setup` escribe `~/.config/nmdev/config.toml`. Cualquier otro verbo sin configuración falla
con `no hay configuración: corré nmdev setup`; setup nunca arranca solo.

Secciones, en orden (`nmdev setup <sección>` pregunta solo una):

| Sección | Qué hace |
|---|---|
| `language` | `en` o `es`, preguntado en los dos idiomas; todo lo que sigue sale en ese idioma |
| `pack` | el pack del equipo: un directorio (se copia) o una URL git (se clona) en `~/.config/nmdev/pack` |
| `tools` | lista las herramientas que faltan con su comando de instalación; setup sigue igual |
| `repos` | busca el clon de cada repo en las raíces habituales, o te pide la ruta; `-` lo saltea |
| `forge` | cuenta y token de Bitbucket (oculto), validado en vivo, guardado en el Keychain |
| `tracker` | email y token de Jira (oculto), validado en vivo, guardado en el Keychain |
| `firebase` | verifica el login del CLI de Firebase; si falla imprime `firebase login` |
| `backend` | backend de agente, su modelo por defecto y fuerte, effort; verifica el login del CLI |
| `sync` | sync en segundo plano sí o no; si es sí corre `install` (hooks + LaunchAgent) |

No se escribe nada hasta que todas las secciones estén respondidas y la configuración combinada
valide. Después setup muestra las claves que cambió, corre `install` si `sync.enabled` es true y
termina con un `doctor` completo y el próximo comando a correr. Volver a correrlo edita la
configuración existente: los valores actuales son los default y los ítems del Keychain que ya
existen se ofrecen como "conservar". Los comentarios del archivo se pierden al reescribirlo; el
archivo anterior queda como `config.toml.bak`.

Modos:

```sh
nmdev setup                                  # interactivo, todas las secciones
nmdev setup backend                          # una sección
nmdev setup --pack git@git.example.com:team/nmdev-pack.git
nmdev setup --pack ~/team-pack               # un directorio se copia, no se linkea
nmdev setup --answers answers.toml           # mergea una config parcial, pregunta solo lo que falta
nmdev setup --pack ~/team-pack --answers answers.toml --yes   # desatendido
```

- `--pack <DIR|URL>`: los valores de equipo de `preset.toml` se aplican sin preguntar. Un pack
  clonado se actualiza con fast-forward en cada `setup` posterior; si no puede (tiene cambios
  locales), setup frena y no escribe nada.
- `--answers <FILE>`: una config parcial en TOML más un `language` opcional en la raíz, mergeada
  sobre la actual. Los tokens se rechazan: cualquier clave que contenga `token`, `password`,
  `secret` o `api_key` hace fallar la corrida. Los tokens solo se tipean, ocultos.
- `--yes`: no pregunta nada; conserva los valores actuales, completa los default y toma `language`
  de `NMDEV_LANG` o del locale del sistema. Si todavía faltan valores obligatorios, falla con la
  lista. Sin terminal, setup nunca pregunta: falla igual si faltan datos.
- Sin pack: setup abre una plantilla comentada de las secciones de equipo en `$VISUAL`/`$EDITOR` y
  la reabre con los errores hasta que valide. Guardarla sin cambios aborta. Sin pack, `commit`,
  `review` y sync no están disponibles hasta que existan los archivos del pack.

Ejemplo de `answers.toml` (valores neutros):

```toml
language = "es"

[forge]
account = "dev@example.com"

[repos.app-ios]
path = "~/src/app-ios"

[repos.app-android]
path = "~/src/app-android"

[agent]
backend = "claude"

[sync]
enabled = false
```

## Estructura del pack

Un pack es un directorio o repo git que el equipo mantiene en privado. nmdev define solo su
estructura:

```
team-pack/
├── preset.toml          config parcial: los valores del equipo
├── prompts/
│   ├── triage.md        prompt de triage de sync
│   ├── port.md          prompt de port de sync
│   └── rosetta.md       notas de equivalencias entre plataformas para portar
├── skills/
│   ├── commit-message/SKILL.md
│   ├── pull-request/SKILL.md
│   ├── pr-review/SKILL.md
│   └── pr-rebuttal/SKILL.md
└── golden.toml          el eval set de sync
```

Cada skill es `skills/<nombre>/SKILL.md` más `references/*.md` opcionales. Setup apunta
`skills_dir` a `pack/skills` y `sync.eval_set` a `pack/golden.toml`; los prompts se leen desde
`pack/prompts/` en su lugar. `doctor` verifica que estén todos los archivos de arriba y nunca
contacta el remoto del pack.

`preset.toml` admite cualquier clave de la config. Las claves personales solo precargan la pregunta
(gana lo que responda el usuario): `language`, `forge.account`, `tracker.email`, todo lo de
`agent`, `repos.<slug>.path`, `review.repos.<slug>.path` y `backends.<name>.model` /
`strong_model`. Todo lo demás es valor de equipo y pisa la config. Ejemplo mínimo:

```toml
[repos.app-ios]
platform = "ios"
name = "iOS app"
counterpart = "app-android"
slug = "app-ios"
compile = ["xcodebuild", "build", "-quiet", "-workspace", "App.xcworkspace", "-scheme", "Dev"]

[forge]
workspace = "my-workspace"
hotfix_reviewers = ["Reviewer One"]

[tracker]
site = "https://tracker.example.com"
# ...
```

## Verbos

Flags comunes a casi todos: `--json`, `-y/--yes` (saltea las aprobaciones), `--build qa|prod|both`
(compila en segundo plano después de publicar). PR1 es un PR de feature contra su epic; PAP es el
PR del epic contra `develop`.

### Flujo

- **`nmdev task <TICKET>`** — lleva un ticket de Jira (clave o URL) hasta un PR1 publicado: brief,
  implementación, compilación, commit, PR. Con `tracker.repo` configurado, `<TICKET>` también puede
  ser un id del repo de tickets (`app-ios/074`, `_shared/012`), y una clave de Jira sin archivo ahí
  recibe un stub espejo en la carpeta del repo ruteado. El archivo del ticket queda en
  `status: in-progress` en cada corrida, y `pr` lo pasa a `in-review`. Jira se llama solo con un
  epic activo: sin epic, el ticket sale de su archivo y en Jira no se lee, mueve ni asigna nada. Un
  ticket cuya categoría de estado en Jira es done se rechaza. `--repo` (default: según los tags de plataforma del
  ticket), `--epic`, `--from-file <FILE>` (texto del ticket en vez de Jira), `--answers <FILE>`
  (respuestas a las preguntas de un brief que frenó), `--fresh`, `--build`, `-y`.
- **`nmdev address`** — responde el review del PR de esta rama: rebate o contesta lo que no
  requiere cambio y arregla el resto en un commit. `--pr`, `--dry-run`, `--fresh`, `--build`, `-y`,
  `--learn-style` (arma el skill `pr-rebuttal` a partir de tus respuestas; gasta tokens).
- **`nmdev review [URL | REPO N]`** — revisa un PR abierto ajeno con tu voz; publica si decís que
  sí. Sin argumentos lista los PRs que te esperan. `--verdict approve|request-changes`,
  `--dry-run`, `--fresh`, `-y`.
- **`nmdev commit`** — commitea lo staged con un mensaje del skill `commit-message`; nunca pushea.
  `--epic`, `--message-file`, `-y`.
- **`nmdev align`** — mergea `develop` en el epic y después el epic en esta rama. `--epic`, `--all`
  (todos los epics del squad a los que les falta `develop`), `-y`.
- **`nmdev pr`** — publica o actualiza el PR1 de esta rama tal como está commiteada (alinea
  primero). `--epic`, `--allow-large` (más de 30 archivos sin preguntar), `--build`, `-y`.
- **`nmdev push`** — commitea lo staged, alinea y publica o actualiza el PR1, con una sola
  aprobación. `--epic`, `--message-file`, `--allow-large`, `--build`, `-y`.
- **`nmdev pap`** — publica o actualiza el PAP (epic → `develop`) con sus PR1, issues, reviewers y un
  build QA. `--prod` (también Prod), `--qa-ticket <KEY>` (comenta ahí los links de los builds), `-y`.
- **`nmdev hotfix`** — publica o actualiza un PR de hotfix contra la rama de release abierta, si no
  contra `master`; compila Prod y después QA. `--base`, `--qa-only`, `--backport` (después del
  merge: abre el PR `[SYNC] master -> develop`), `-y`.
- **`nmdev ship`** — compila este checkout tal como está, con cambios sin commitear, y lo distribuye.
  `--qa`, `--prod`, `--both`, `--dev <N>` (también DEV-N, repetible; `--devN` también anda),
  `-m <MESSAGE>` (encabezado del release), `--no-attach` (no toca el PR), `--keep` (conserva los
  binarios locales).
- **`nmdev status`** — dónde está esta rama respecto de `develop` y su epic, su PR, builds, ports y
  qué correr después. `--all`: todos los repos, desde cualquier lado.

### Sync en segundo plano

Los hooks de commit encolan cada commit; un LaunchAgent corre `nmdev sync`, que lo triagea y porta
lo que le falta a la otra app, según `sync.modes`.

- **`nmdev sync`** — una pasada de sync. La corre el LaunchAgent, nunca a mano.
- **`nmdev sync stats`** — métricas de ajuste desde el ledger. `--since 7d|Nd|all`.
- **`nmdev sync replay <REPO> <SHA>`** — reproduce un commit sobre un ledger descartable.
  `--dry-run`, `--live`.
- **`nmdev sync eval`** — puntúa triage y paridad sobre el eval set. `--golden <FILE>`.
- **`nmdev sync bootstrap verify`** — verifica que el kit de bootstrap todavía aplica y compila
  sobre `origin/develop` de la app Android.
- **`nmdev sync enqueue <REPO> <SHA> [BRANCH]`** — encola un commit como lo hacen los hooks.
- **`nmdev ports [JOB] [REASON]`** — ports esperando revisión: listarlos, ver uno,
  `--accept [ITEM...]` (sin ítems: todos; `--dry-run` commitea solo local) o `--dismiss` con
  `--wrong` / `--wontfix` (`--golden <FILE>` suma la decisión a un eval set).
- **`nmdev retry <JOB>`** — vuelve a correr un job fallido o con build fallido desde su último estado.
- **`nmdev pause`** / **`nmdev resume`** — frena / reanuda el procesamiento de jobs; los hooks siguen
  encolando.
- **`nmdev install`** — instala los hooks de commit en cada repo configurado y el LaunchAgent
  `dev.hersy.nmdev.sync`.

### Epics del tracker

- **`nmdev config`** — el epic activo, sus claves y el estado del brief. **`nmdev config get <KEY>`**
  muestra un valor (`epic` o una clave del epic). **`nmdev config set epic <SLUG>`** cambia de epic;
  `--jira <KEY> [--space <KEY>]` lo crea antes, con `jira_board` del primer board del proyecto y
  `home_page` del espacio (si una búsqueda falla, avisa y deja la clave vacía). **`nmdev config set epic none`** apaga el seguimiento.
  **`nmdev config set <KEY> <VALUE>`** edita una clave del epic activo. Los epics viven en
  `epics/<slug>.toml` del repo de tickets (claves: `jira_project` obligatoria; `jira_board`,
  `confluence_space`, `home_page`, `brief_page`, `lang`, `status_started`, `status_review`, `branch`), el slug
  activo en `active-epic` al lado; cada cambio se commitea ahí.
- **`nmdev tickets sync [--dry-run]`** — trae el proyecto de Jira del epic activo al repo de tickets
  con una búsqueda JQL paginada. Un archivo cuyo `jira` coincide con un issue solo reescribe `status`,
  `jira_status`, `assignee` y `sprint`, nunca el cuerpo: `status_started` de Jira pasa a `in-progress`,
  `status_review` a `in-review`, cualquier otro status de la categoría done a `done`; cualquier otro
  status deja `status` como está y se escribe tal cual en `jira_status`, que se borra cuando el status
  vuelve a mapear. Un campo que Jira deja vacío se borra. El sprint es el sprint activo del issue, si no
  el último, leído del campo custom de sprint del sitio. Un issue sin archivo se vuelve un stub en la
  carpeta a la que la etiqueta `task_tags` de su resumen lo manda, si no `_shared`. Muestra cada campo
  cambiado y cada archivo creado y nunca commitea (`--json` sugiere el commit); `--dry-run` no escribe
  nada. Sin epic activo no llama a Jira y lo dice.
- **`nmdev brief check`** — si la página de Brief del epic activo (`brief_page`) está al día, para el
  hook que exige el brief. Está desactualizada cuando `brief_page` está vacío o la página no existe,
  cuando otra página de `confluence_space` se editó después de la versión actual del Brief, o cuando
  un issue de `jira_project` se creó, o cambió su summary o description (leído de su changelog),
  después de esa versión; los cambios de status, assignee y sprint no cuentan. El motivo nombra el
  disparador más nuevo; `--json` da `fresh`, `reason` y `trigger` (`kind` `page` o `issue`, su id o
  clave, `change`, `at`). Códigos de salida: `0` al día o sin epic activo, `1` desactualizado
  (siguiente: correr la skill brief), `2` no se puede verificar (Confluence o Jira no responden o
  rechazan el token; la causa va en el motivo y `fresh` es null). `nmdev config` muestra el mismo
  estado como `fresh`, `stale` o `unknown`.

### Mantenimiento

- **`nmdev doctor`** — verifica herramientas, backends, hooks, LaunchAgent, tokens del Keychain,
  acceso a Jira y Confluence, login de Firebase, build lock, pack y config. `--fix` instala antes el bundle de gems de fastlane.
- **`nmdev setup [SECTION]`** — escribe la config; ver [Setup](#setup).
- `nmdev --license` imprime la licencia.

## Referencia de configuración

`~/.config/nmdev/config.toml`. Las rutas relativas se resuelven contra el directorio de la config;
`~` se expande. Las claves desconocidas son error. "—" significa obligatorio, sin default.

### Raíz

| Clave | Default | Significado |
|---|---|---|
| `version` | — | formato de la config; tiene que ser `1` |
| `language` | `NMDEV_LANG`, si no el locale | `en` o `es` |
| `skills_dir` | — | directorio de los skills (setup: `pack/skills`) |

### `[agent]`

| Clave | Default | Significado |
|---|---|---|
| `agent.nmdev_bin` | `nmdev` | binario que corren el LaunchAgent y los jobs en segundo plano |
| `agent.backend` | `claude` | `claude`, `codex` o `gemini`, para todas las etapas |
| `agent.model` | sin definir | un modelo para todas las etapas; deprecado a favor de `backends.<name>.model` (setup lo saca) |
| `agent.effort` | sin definir | `low`, `medium` o `high` para todas las etapas |

### `[backends.*]`

Resolución del modelo: `NMDEV_AGENT_MODEL` > `stages.<stage>.model` > `agent.model` > el `model`
del backend activo (o `strong_model` en las etapas fuertes) > el de fábrica.

| Clave | Default | Significado |
|---|---|---|
| `backends.claude.bin` | `claude` | binario del CLI de claude |
| `backends.claude.setting_sources` | `""` | se pasa a `--setting-sources` de claude |
| `backends.claude.model` | `sonnet` | modelo por defecto |
| `backends.claude.strong_model` | `opus` | modelo de las etapas fuertes (`resolve`, `task_implement`, `sync_resolve`) |
| `backends.codex.bin` | `codex` | binario del CLI de codex |
| `backends.codex.model` | de fábrica | modelo por defecto |
| `backends.codex.strong_model` | de fábrica | modelo fuerte |
| `backends.gemini.bin` | `gemini` | binario del CLI de gemini |
| `backends.gemini.keychain_item` | `nmdev-gemini` | ítem del Keychain con la API key de Gemini |
| `backends.gemini.model` | `flash` | modelo por defecto |
| `backends.gemini.strong_model` | `pro` | modelo fuerte |

### `[repos.<slug>]`

Una tabla por repo de app, con el slug de Bitbucket como clave.

| Clave | Default | Significado |
|---|---|---|
| `repos.<slug>.platform` | — | `ios`, `android`, `flutter` o `docs` |
| `repos.<slug>.name` | — | nombre visible, también se le pasa a los agentes |
| `repos.<slug>.counterpart` | — | slug de la otra app (sync y respuestas espejadas) |
| `repos.<slug>.path` | vacío | clon local; vacío significa salteado (`doctor` lo reporta) |
| `repos.<slug>.slug` | — | slug del repo en Bitbucket |
| `repos.<slug>.compile` | — | argv que compila la app (gate del agente y de sync) |
| `repos.<slug>.prepare` | `[]` | argv que corre antes de compilar en un worktree nuevo (ej. `pod install`) |
| `repos.<slug>.prepare_when_changed` | `""` | vuelve a correr `prepare` solo si cambió este archivo |
| `repos.<slug>.symlinks` | `[]` | archivos no versionados que se linkean en los worktrees (ej. `local.properties`) |
| `repos.<slug>.ship.workspace` | sin definir | iOS: `.xcworkspace` a compilar |
| `repos.<slug>.ship.output_dir` | sin definir | dónde quedan los binarios |
| `repos.<slug>.ship.firebase_app` | sin definir | ID de app de Firebase para todos los targets |
| `repos.<slug>.ship.targets.<target>.tag` | — | tag del build en los releases (ej. `QA`) |
| `repos.<slug>.ship.targets.<target>.scheme` | sin definir | scheme de iOS |
| `repos.<slug>.ship.targets.<target>.task` | sin definir | task de Gradle de Android |
| `repos.<slug>.ship.targets.<target>.variant_dir` | sin definir | directorio de salida de la variante Android |
| `repos.<slug>.ship.targets.<target>.firebase_app` | `ship.firebase_app` | ID de app de Firebase por target |

### `[stages.<stage>]`

Overrides por etapa. Etapas: `message`, `assess`, `resolve`, `task_brief`, `task_implement`,
`address_triage`, `address_fix`, `address_style`, `pr`, `review`, `review_style`, `triage`,
`parity`, `port`, `sync_message`, `sync_resolve`, `sync_pr`.

| Clave | Default | Significado |
|---|---|---|
| `stages.<stage>.backend` | `agent.backend` | backend de esta etapa |
| `stages.<stage>.model` | resuelto como arriba | modelo de esta etapa |
| `stages.<stage>.effort` | `agent.effort` | `low`, `medium`, `high` |
| `stages.<stage>.max_turns` | por etapa | tope de turnos |
| `stages.<stage>.read` | por etapa | puede leer archivos |
| `stages.<stage>.edit` | por etapa | puede editar archivos |
| `stages.<stage>.shell` | por etapa | prefijos de comandos de shell permitidos |

### `[forge]` (Bitbucket)

| Clave | Default | Significado |
|---|---|---|
| `forge.workspace` | — | workspace de Bitbucket |
| `forge.account` | — | tu cuenta de Bitbucket (personal) |
| `forge.reviewers` | `work` | reviewers del PR1: `default` (los del repo), `work` / `pap` / `sync` (copia los de tu último PR de ese tipo) o nombres separados por coma |
| `forge.hotfix_reviewers` | — | reviewers de hotfix, por nombre |

### `[tracker]` (Jira)

| Clave | Default | Significado |
|---|---|---|
| `tracker.site` | — | ej. `https://tracker.example.com` |
| `tracker.cloud_id` | — | cloud ID de Jira |
| `tracker.epic_key` | — | clave de proyecto de los epics (`NMDEV_EPIC_KEY` la pisa); `*` acepta cualquier clave y deja correr `pap`, `pr`, `align` y `task --epic` desde y hacia cualquier rama que no sea base; `task` sin rama del epic padre sale de la rama actual, pusheada y no base |
| `tracker.task_tags` | — | tag de plataforma del ticket → slug del repo, ej. `{ IOS = "app-ios" }`; una entrada `"*"` (cualquier repo) manda los tickets sin tag que coincida a ese repo, o al actual si está vacía, y levanta los chequeos de tipo, estado final y tags rechazados (`NMDEV_ANY_REPO=1` equivale a `"*" = ""`) |
| `tracker.refused_tags` | `[]` | tags que `task` rechaza (plataforma de otro equipo) |
| `tracker.repo` | sin definir | repo de tickets: archivos de epic y tickets en markdown (`config`) |

Los valores de cada tablero viven solo en el archivo del epic activo, `<tracker.repo>/epics/<slug>.toml`.
Una config que todavía define `tracker.project_keys`, `started`, `review`, `order` o `task_types` se
rechaza. Los tipos de issue van en la tabla opcional `[task_types]` del epic (tipo de issue → prefijo de
branch, por defecto `{ Bug = "fix/", Task = "feat/" }`).

### `[distributor]` (Firebase App Distribution)

| Clave | Default | Significado |
|---|---|---|
| `distributor.firebase_project` | — | ID del proyecto de Firebase |
| `distributor.tester_groups` | — | grupos de testers, separados por coma |
| `distributor.team_id` | sin definir | team ID de Apple para firmar iOS |
| `distributor.dev_envs` | — | números de entorno DEV que acepta `ship --dev` |

### `[filters]`

| Clave | Default | Significado |
|---|---|---|
| `filters.message_patterns` | — | regex: los commits cuyo mensaje coincide no se sincronizan |
| `filters.plumbing_globs` | — | archivos de plomería (no se portan) |
| `filters.forbidden_globs` | — | archivos que agentes y ports nunca tocan |

### `[sync]`

| Clave | Default | Significado |
|---|---|---|
| `sync.enabled` | `true` | sync en segundo plano activado (los hooks encolan, el LaunchAgent corre) o no |
| `sync.modes` | — | `"origen->destino" = "shadow" \| "local" \| "live"`: solo registra, commitea local, o pushea y abre el PR |
| `sync.bootstrap_kit` | — | directorio del kit de bootstrap |
| `sync.eval_set` | `golden.toml` | eval set (setup: `pack/golden.toml`) |
| `sync.max_ports_per_day` | `10` | tope diario de ports |
| `sync.max_jobs_per_day` | `30` | tope diario de jobs |
| `sync.release_prefixes` | `["release/", "releases/"]` | prefijos de ramas de release |
| `sync.assess_threshold` | `0.6` | el upkeep rechaza un merge evaluado con este riesgo o más |
| `sync.epic_upkeep` | `false` | mantiene los epics al día con `develop` en segundo plano |
| `sync.pap_upkeep` | `false` | mantiene los PAPs actualizados en segundo plano |

### `[review]`

| Clave | Default | Significado |
|---|---|---|
| `review.legacy_runs` | sin definir | directorio de reviews anteriores para reutilizar |
| `review.repos.<slug>.platform` | — | plataforma de un repo que revisás |
| `review.repos.<slug>.path` | el clon de `repos` | clon donde revisar; obligatorio si el slug no está en `repos` |

### `[address]`

| Clave | Default | Significado |
|---|---|---|
| `address.rebuttal_repos` | — | repos cuyos PRs alimentan `address --learn-style` |
| `address.bot_names` | — | autores de comentarios que se ignoran como bots |
| `address.rebuttal_thin` | `15` | con menos respuestas que esto, el estilo aprendido se marca como flaco |

## Entorno

| Variable | Efecto |
|---|---|
| `NMDEV_LANG` | `en` o `es`; pisa `language` |
| `NMDEV_AGENT_BACKEND` / `NMDEV_AGENT_MODEL` / `NMDEV_AGENT_EFFORT` | pisan backend, modelo y effort en todas las etapas |
| `NMDEV_EPIC_KEY` | pisa `tracker.epic_key` |
| `NMDEV_ANY_REPO` | `1` equivale a una entrada `"*" = ""` en `tracker.task_tags` si no tiene una |
| `NMDEV_FLOW_STATE` | directorio de estado del flujo (default `~/.local/state/nmdev/flow`) |
| `NMDEV_BUILD_LOCK=0` | desactiva el build lock; `NMDEV_BUILD_LOCK_DIR` lo mueve |
| `NMDEV_BUILD_TIMEOUT` | timeout de build en segundos |
| `NMDEV_SHIP_KEEP=1` | igual que `ship --keep` |
| `NMDEV_GEMFILE` | `Gemfile` de fastlane para builds de desarrollo |
| `CODEX_API_KEY` / `CODEX_HOME` | autenticación de codex |

## Problemas comunes

**Empezá por `nmdev doctor`.** Una línea por chequeo: `✓` pasó, `✗` falló, `·` informativo.

```
✓ git en el PATH: /usr/bin/git
✓ repo app-ios: /Users/me/src/app-ios
✗ repo app-android:
· el LaunchAgent corre `nmdev sync`: no habilitado
✓ token de Bitbucket en el Keychain: dev.hersy.nmdev.bitbucket
✗ token de Jira en el Keychain (task, pr, pap --qa-ticket): dev.hersy.nmdev.jira
✓ pack del equipo (commit, review, sync): /Users/me/.config/nmdev/pack
✓ worker no pausado: corriendo
```

Arreglá lo que falla y volvé a correrlo. Un error de config nombra el archivo y la clave culpable.

**Ítems del Keychain** (keychain de login, los escribe setup, nunca pasan por la línea de comandos):

| Ítem (service) | Cuenta | Contiene |
|---|---|---|
| `dev.hersy.nmdev.bitbucket` | `forge.account` | token de Bitbucket |
| `dev.hersy.nmdev.jira` | tu email de Jira | token de Jira, también para leer Confluence (scopes `read:page:confluence`, `read:space:confluence`) |
| `nmdev-gemini` (`backends.gemini.keychain_item`) | — | API key de Gemini, solo con el backend gemini |

Para rotar un token: `nmdev setup forge` / `nmdev setup tracker` y elegí "reemplazar". Para
inspeccionarlo: `security find-generic-password -s dev.hersy.nmdev.jira`.

**Logins de los CLIs**: son de cada CLI; nmdev los verifica, nunca los guarda. Si fallan, setup
imprime el comando y ofrece reintentar o saltear; logueate desde otra terminal:

| Backend / herramienta | Login |
|---|---|
| claude | `claude auth login` |
| codex | `codex login` (o definí `CODEX_API_KEY`) |
| gemini | API key en el ítem del Keychain de arriba (`nmdev setup backend`) |
| Firebase | `firebase login` |

**Sync en segundo plano sí/no.** `nmdev setup sync` cambia `sync.enabled`. Sí: setup corre
`install` (hooks en cada repo, LaunchAgent `dev.hersy.nmdev.sync`). No: los hooks dejan de encolar y
`doctor` muestra hooks y LaunchAgent como `· no habilitado`. Para pausar sin tocar la config:
`nmdev pause` / `nmdev resume`. Log del worker: `~/.local/state/nmdev/sync/worker.log`.

**Idioma.** Orden: `NMDEV_LANG` > `language` en la config > locale del sistema (`LC_ALL`, `LANG`;
`es*` → español) > inglés. `NMDEV_LANG=en nmdev doctor` para una vez. La salida `--json` es siempre
en inglés. Para cambiarlo de forma permanente: `nmdev setup language`.

**Falta un repo.** Clonalo y corré `nmdev setup repos`. Setup nunca clona repos.

**El pack no se actualiza.** Setup frena si `~/.config/nmdev/pack` no puede hacer fast-forward;
resolvé los cambios locales ahí y volvé a correr `nmdev setup`.
