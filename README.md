<div align="center">

# `deep-research`

**El botón de Research de claude.ai, replicado dentro de Claude Code.**

Dispara el Exa Agent + NotebookLM Deep Research en paralelo, cross-valida hallazgos entre ambos motores, y persiste todo en tu proyecto — con un solo comando.

[![Built for Claude Code](https://img.shields.io/badge/Built%20for-Claude%20Code-D97757?style=flat-square)](https://docs.claude.com/en/docs/claude-code/overview)
[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg?style=flat-square)](./LICENSE)
[![Español](https://img.shields.io/badge/Idioma-Espa%C3%B1ol-red?style=flat-square)](#)

</div>

---

## ¿Qué es esto?

Un set de **skills + rules + config** para Claude Code que replica la experiencia del botón "Research" de claude.ai (el que visita 50+ fuentes y sintetiza) — pero **dentro de tu terminal**, integrado a tu proyecto, y con cross-validation entre dos motores de research independientes.

```
Tú:  /deep-research investiga Bifrost vs LiteLLM para producción
     │
     ▼
  Claude dispara en paralelo ─────┬──▶ Exa Agent (agent_run) ──┐
                                  │                             ├──▶ SYNTHESIS.md
                                  └──▶ NotebookLM deep ─────────┘    (cross-validated)
     │
     ▼
  Persiste 4 archivos en .planning/deep-research/<fecha>-<slug>/
  y te da un resumen de 5 líneas con el veredicto.
```

**Por qué dos motores en paralelo:** cada uno tiene carácter distinto — Exa es bueno con GitHub issues, threads de comunidad, páginas de precios y citas directas; NotebookLM es bueno con síntesis narrativa y papers. Correr ambos y sintetizar las convergencias atrapa alucinaciones y expande cobertura. Una corrida de Exa con `effort=high` cuesta ~$0.50 USD (medido); NotebookLM es gratis.

**Portable por diseño:** la skill detecta qué tiene configurado cada usuario (nombre del servidor MCP de Exa, si hay `EXA_API_KEY`, si está instalado el CLI de NotebookLM) y trabaja con los motores que encuentre. Si falta uno, corre con el otro y lo deja anotado en el reporte.

---

## ¿Qué incluye este repo?

```
deep-research/
├── skills/
│   ├── deep-research/SKILL.md     ← orquestador principal (dispara motores + sintetiza)
│   └── find-docs/SKILL.md          ← wrapper de Context7 para docs de libraries
├── rules/
│   └── context7.md                 ← rule global: siempre leer docs con ctx7
├── config/
│   ├── mcp-servers.json            ← template de los 4 MCPs (placeholders para keys)
│   └── claude-md-additions.md      ← snippet a pegar en tu CLAUDE.md global
└── README.md                       ← este archivo
```

---

## Tabla de contenidos

- [Quickstart (para tu Claude)](#quickstart-para-tu-claude)
- [Instalación manual paso a paso](#instalación-manual-paso-a-paso)
- [Migrar desde la versión anterior](#migrar-desde-la-versión-anterior)
- [Arquitectura](#arquitectura)
- [Cómo usarlo](#cómo-usarlo)
- [Troubleshooting](#troubleshooting)
- [FAQ](#faq)
- [Créditos](#créditos)

---

## Quickstart (para tu Claude)

> [!TIP]
> **¿Tu Claude Code está instalando esto por ti?** Si le dijiste "instala este repo", léele este README completo y ejecuta los pasos en orden. Pide al usuario sus API keys **solo cuando llegues al paso 4**. No improvises — los quirks de NotebookLM y Exa (pasos 3 y 6) están documentados porque son trampas reales.

**Pre-requisitos que tu sistema necesita:**

| Requisito | Verificar con | Si falta |
|---|---|---|
| Claude Code instalado | `claude --version` | [docs.claude.com](https://docs.claude.com/en/docs/claude-code/overview) |
| Python 3.10+ | `python --version` | [python.org](https://python.org) |
| `uv` (recomendado) o `pipx` | `uv --version` | [docs.astral.sh/uv](https://docs.astral.sh/uv/) |
| Node.js 18+ | `node --version` | [nodejs.org](https://nodejs.org) |
| Git | `git --version` | [git-scm.com](https://git-scm.com) |

**Cuentas que necesitarás (tener a mano antes de empezar):**

| Servicio | Para qué | Obligatorio | Gratis |
|---|---|---|---|
| [Exa](https://exa.ai) | Deep research motor #1 (Exa Agent) | Recomendado | Créditos de prueba, después pay-per-use (~$0.50 por corrida con `effort=high`) |
| Cuenta Google | NotebookLM (motor #2) | Recomendado | 100% gratis |
| [Firecrawl](https://firecrawl.dev) | Scraping/crawling de URLs | No (opcional) | Free tier generoso |
| [Context7](https://context7.com) | Docs de libraries | No (funciona sin cuenta) | Free tier sin auth |
| [Papersflow](https://papersflow.ai) | Papers académicos | No (opcional) | Guest mode funciona |

Con uno solo de los dos motores la skill funciona igual, pero pierdes la cross-validation.

---

## Instalación manual paso a paso

### 1. Clonar el repo

```bash
git clone https://github.com/josuebustosn/deep-research.git
cd deep-research
```

### 2. Instalar NotebookLM — el CLI `notebooklm` de [notebooklm-py](https://github.com/teng-lin/notebooklm-py)

> [!NOTE]
> **Es lo único que se instala localmente.** Los MCPs (Exa, Context7, Papersflow, Firecrawl) son **HTTP hosted**: no hay nada que instalar, solo se configuran como URLs (paso 4). NotebookLM se usa a través de un CLI, no de un MCP.

Elige una opción:

**Opción A — con `uv` (recomendado, entorno aislado):**
```bash
uv tool install notebooklm-py
```

**Opción B — con `pipx`:**
```bash
pipx install notebooklm-py
```

**Opción C — con `pip`:**
```bash
pip install notebooklm-py
```

Verificar:
```bash
notebooklm --version       # debe imprimir: NotebookLM CLI, version 0.8.x
```

### 3. Autenticar NotebookLM (una sola vez)

```bash
notebooklm login
notebooklm auth check --test --json
```

`login` abre un navegador con un perfil persistente → inicias sesión con tu cuenta Google → guarda la sesión en `~/.notebooklm/profiles/default/`. El `auth check --test` debe devolver `"status": "ok"` y `"token_fetch": true`.

> [!WARNING]
> La sesión **expira con el tiempo sin aviso**. Cuando pase, tu Claude lo detecta con `auth check --test`, prueba `notebooklm auth refresh` y, si no alcanza, relanza `notebooklm login` para que completes el login en el navegador. El flujo está documentado en `config/claude-md-additions.md`. Ojo: `notebooklm doctor` **no** sirve para saber si la sesión sigue viva (es un chequeo local).

### 4. Configurar los MCPs en `~/.claude.json`

La forma más segura es con el propio CLI de Claude Code (hace merge, no sobrescribe):

```bash
claude mcp add --transport http exa "https://mcp.exa.ai/mcp?tools=web_search_exa,web_fetch_exa,web_search_advanced_exa,get_code_context_exa,agent_run" -s user
claude mcp add --transport http context7 https://mcp.context7.com/mcp -s user
claude mcp add --transport http papersflow https://doxa.papersflow.ai/mcp -s user
claude mcp add --transport http firecrawl "https://mcp.firecrawl.dev/<TU_FIRECRAWL_API_KEY>/v2/mcp" -s user   # opcional
```

El template equivalente en JSON está en `config/mcp-servers.json`.

**Campos a reemplazar:**

- `<TU_FIRECRAWL_API_KEY>` → tu API key de Firecrawl (la consigues en [firecrawl.dev/app/api-keys](https://firecrawl.dev/app/api-keys))

**Exa autentica por OAuth:** la primera vez que Claude intente usarlo, solo verá una herramienta `authenticate`. Claude te pasa un link, autorizas en el navegador y aparecen las herramientas reales (`agent_run`, etc.). Si prefieres API key, créala en [dashboard.exa.ai/api-keys](https://dashboard.exa.ai/api-keys) y agrégala a la URL como `&exaApiKey=<key>`. Ojo: si usas `?tools=...`, **`agent_run` tiene que estar en la lista** — activar herramientas opcionales reemplaza el set por defecto.

Context7 y Papersflow funcionan sin auth.

> [!TIP]
> **¿Tu Claude está haciendo esto por ti?** Haz merge seguro, no sobrescritura:
> 1. Prefiere `claude mcp add ... -s user` (hace merge solo). Si tienes que editar el JSON a mano: `Read` `config/mcp-servers.json` de este repo y el `~/.claude.json` del usuario.
> 2. Si el usuario ya tiene una clave `mcpServers`, **preserva** todos los MCPs previos y **agrega** los de este repo dentro del mismo objeto. No borres nada que ya esté.
> 3. Si el usuario ya tiene un servidor de Exa con otro nombre, no lo dupliques: solo asegúrate de que `agent_run` esté habilitado.
> 4. **Pídele la API key de Firecrawl al usuario** ANTES de escribir el archivo (no dejes `<TU_FIRECRAWL_API_KEY>` en el JSON real). Si no la tiene, ofrécele omitirlo — la skill no depende de Firecrawl.
> 5. Dile al usuario que reinicie Claude Code para que cargue los MCPs nuevos.

**Opcional — MCP de NotebookLM.** La skill usa el CLI y no lo necesita, pero si quieres usar NotebookLM como herramientas MCP en otras tareas:

```bash
claude mcp add-json notebooklm '{"type":"stdio","command":"uvx","args":["--from","notebooklm-py[mcp]","notebooklm-mcp"],"env":{"PYTHONUTF8":"1"}}' -s user
```

Se lanza con `uvx --from ...` a propósito: el paquete viejo `notebooklm-mcp-cli` instala un ejecutable con el mismo nombre (`notebooklm-mcp`) y, si tienes los dos, no sabes cuál arranca.

### 5. Copiar skills y rules a tu `~/.claude/`

**Linux/macOS/Git Bash:**
```bash
mkdir -p ~/.claude/skills ~/.claude/rules
cp -r skills/deep-research ~/.claude/skills/
cp -r skills/find-docs ~/.claude/skills/
cp rules/context7.md ~/.claude/rules/
```

**PowerShell (Windows):**
```powershell
New-Item -ItemType Directory -Force -Path "$HOME\.claude\skills", "$HOME\.claude\rules" | Out-Null
Copy-Item -Recurse -Force skills\deep-research "$HOME\.claude\skills\"
Copy-Item -Recurse -Force skills\find-docs "$HOME\.claude\skills\"
Copy-Item -Force rules\context7.md "$HOME\.claude\rules\"
```

### 6. Pegar el snippet en tu CLAUDE.md global

Abre `~/.claude/CLAUDE.md` (créalo si no existe) y **pega al final** el contenido del bloque de `config/claude-md-additions.md` de este repo.

Este snippet le enseña a Claude los quirks que, sin ellos, te costarían horas debuggeando:
1. El health check real de NotebookLM es `notebooklm auth check --test --json`; `doctor` y `status` son locales y mienten con la sesión vencida.
2. El flujo de re-auth end-to-end (`auth refresh` → `login` en background) que Claude ejecuta sin intervención humana más que el login en el navegador.
3. `PYTHONUTF8=1` en Windows para que los acentos no se corrompan en el JSON.
4. Exa: el research es `agent_run`, y para seguir una corrida se llama con `runId` (reenviar `query` cobra otra corrida).

### 7. Verificar que todo corre

Reinicia Claude Code (cierra y abre de nuevo, para que cargue los MCPs). Después, en cualquier proyecto:

```
> /deep-research investiga cuál es la mejor forma de integrar NotebookLM con Claude Code en 2026
```

Si todo está bien, Claude debería:
- Detectar la skill `deep-research`
- Detectar los motores disponibles y hacer el preflight de auth de NotebookLM
- Disparar Exa y NotebookLM en paralelo
- Persistir 4 archivos (`exa-report.md`, `nlm-report.md`, `SYNTHESIS.md`, `sources.md`) en `.planning/deep-research/<fecha>-<slug>/`
- Darte un resumen con el veredicto

---

## Migrar desde la versión anterior

La primera versión de este repo usaba el MCP de [`notebooklm-mcp-cli`](https://github.com/jacob-bd/notebooklm-mcp-cli) (`nlm login` + `refresh_auth`) y las herramientas `deep_researcher_start` / `deep_researcher_check` de Exa, que ya no existen. Para migrar:

1. Instala `notebooklm-py` y autentica (pasos 2 y 3).
2. En tu servidor MCP de Exa, cambia `?tools=` para incluir `agent_run` y quitar `deep_researcher_start`, `deep_researcher_check`, `company_research_exa`, `linkedin_search_exa` y `crawling_exa`. Upstream reemplazó `company_research_exa` y `linkedin_search_exa` por `web_search_advanced_exa`, y `crawling_exa` ya no aparece entre las herramientas documentadas: para leer páginas está `web_fetch_exa`.
3. Copia de nuevo `skills/deep-research` a `~/.claude/skills/`.
4. En tu `CLAUDE.md` global, reemplaza la sección vieja de NotebookLM por el snippet nuevo (paso 6).
5. **No desinstales `notebooklm-mcp-cli` a ciegas:** si otras skills o agentes tuyos usan `mcp__notebooklm-mcp__*`, siguen dependiendo de él. En ese caso, fija su servidor MCP con la ruta absoluta del ejecutable viejo para que no choque con el de notebooklm-py.

---

## Arquitectura

```
┌─────────────────────────────────────────────────────────────────┐
│                         CLAUDE.md (global)                      │
│  Quirks de NotebookLM (auth check real, re-auth, UTF-8) y Exa   │
└────────────────────────────────┬────────────────────────────────┘
                                 │ informa
                                 ▼
┌─────────────────────────────────────────────────────────────────┐
│                       Skills + Rules                            │
│  ┌──────────────┐  ┌──────────┐  ┌─────────────────┐            │
│  │ deep-research│  │ find-docs│  │ rules/context7  │            │
│  │ (orquestador)│  │ (docs)   │  │ (rule global)   │            │
│  └──────┬───────┘  └─────┬────┘  └─────────────────┘            │
└─────────┼────────────────┼──────────────────────────────────────┘
          │ dispara        │ consulta
          ▼                ▼
┌─────────────────────────────────────────────────────────────────┐
│               MCP Servers (HTTP)          +   CLI local         │
│  ┌─────┐  ┌──────────┐  ┌──────────┐  ┌────┐   ┌─────────────┐  │
│  │ exa │  │ context7 │  │papersflow│  │ fc │   │ notebooklm  │  │
│  │HTTP │  │  HTTP    │  │   HTTP   │  │HTTP│   │(notebooklm- │  │
│  └──┬──┘  └─────┬────┘  └─────┬────┘  └─┬──┘   │     py)     │  │
│     │           │             │         │      └──────┬──────┘  │
└─────┼───────────┼─────────────┼─────────┼─────────────┼─────────┘
      ▼           ▼             ▼         ▼             ▼
   exa.ai    context7.com   papersflow  firecrawl  Google NotebookLM
  (OAuth o                  (guest OK)  (API key)  (sesión de Google)
   API key)
```

**Flujo de una query de research:**

1. Usuario escribe `/deep-research <pregunta>` (o la skill se dispara proactivamente por keywords como "investiga a fondo").
2. Claude detecta los motores disponibles: busca la herramienta `agent_run` de Exa y el CLI `notebooklm`.
3. Preflight de NotebookLM: `notebooklm auth check --test --json`. Si falla → `auth refresh` → `login` en background → re-verifica.
4. Claude dispara **Exa + NotebookLM en paralelo** (ambos async): `agent_run` con la query estructurada, y `notebooklm source add-research --mode deep --no-wait` + `research wait` en background.
5. Polling: Exa se consulta con `runId` hasta `outputReady`; NotebookLM avisa solo cuando termina.
6. Claude baja el reporte completo de NotebookLM con `research status --json` (con `PYTHONUTF8=1`).
7. Claude sintetiza convergencias y divergencias entre ambos motores.
8. Persiste 4 archivos + scan del proyecto por referencias al topic (por si hay decisions que revisar).
9. Resumen tight al usuario con veredicto de 1 línea + paths + cost.

---

## Cómo usarlo

### Activación explícita

```
/deep-research <cualquier pregunta de investigación>
```

### Activación por keywords

La skill se dispara automáticamente con cualquiera de estos:

- `"deep research ..."`
- `"investigación profunda sobre ..."`
- `"investiga a fondo ..."`
- `"comprehensive research on ..."`
- `"lanza investigación de ..."`

### Flags útiles

| Flag | Qué hace |
|---|---|
| `--source=exa` | Solo Exa (si NotebookLM auth está jodido) |
| `--source=nlm` | Solo NotebookLM (si quieres costo $0) |
| `--source=papers` | Agrega papersflow (académico) |
| `--source=all` | Exa + NLM + papers + context7 |
| `--mode=fast` | NLM fast mode (~30s, ~10 fuentes) |
| `--effort=<nivel>` | Esfuerzo del Exa Agent: `minimal`, `low`, `medium`, `high` (default), `xhigh` |
| `--no-persist` | No escribe archivos, solo resumen inline |
| `--slug=<nombre>` | Override del slug auto-generado |

### Cuándo NO usarlo

- Preguntas factuales rápidas → usa `WebSearch` builtin
- Dudas de API de libraries → usa `/find-docs` (context7)
- Preguntas sobre tu propio proyecto → `Read` directo
- Debug de errores específicos → wrong tool

---

## Troubleshooting

<details>
<summary><b>NotebookLM devuelve "Authentication expired or invalid"</b></summary>

1. `notebooklm auth refresh` (rápido, sin navegador).
2. Si sigue fallando: `notebooklm login` y completa el login en la ventana que se abre.
3. Verifica con `notebooklm auth check --test --json` (debe dar `"status": "ok"` y `"token_fetch": true`).

No te fíes de `notebooklm doctor`: es un chequeo local y puede decir "All checks passed" con la sesión vencida.

</details>

<details>
<summary><b>El reporte de NotebookLM sale con caracteres raros (`�`) en vez de acentos</b></summary>

Es Windows escribiendo el JSON con la página de códigos de la consola. Corre las llamadas con `PYTHONUTF8=1` delante, por ejemplo:

```bash
PYTHONUTF8=1 notebooklm research status -n <NOTEBOOK_ID> --json > nlm-raw.json
```

La skill ya lo hace; si lo ves corrupto, tu copia de la skill está desactualizada.

</details>

<details>
<summary><b>La investigación de NotebookLM "termina" a los 5 minutos sin fuentes importadas</b></summary>

`notebooklm research wait` tiene un timeout por defecto de 300 s. El modo deep a veces tarda más, y si el CLI se rinde antes, no importa nada y la web queda con un modal de "Add sources?". Usa `--timeout 1800`.

</details>

<details>
<summary><b>Solo veo una herramienta `authenticate` de Exa (no aparece `agent_run`)</b></summary>

Falta el OAuth del MCP hosted. Claude llama `authenticate` y te pasa un link: ábrelo **de inmediato** (la escucha del callback en `localhost` dura poco; un link viejo falla sin avisar). Si al autorizar el navegador termina en error de conexión, copia la URL completa de la barra de direcciones (`http://localhost:<puerto>/callback?code=...`) y pásasela a Claude para que llame `complete_authentication`.

Si ya autenticaste y aun así no aparece `agent_run`, revisa que esté en el `?tools=` de tu URL de Exa.

</details>

<details>
<summary><b>Exa retorna "quota exceeded", 402 o 429</b></summary>

- Sin créditos: agrega un método de pago en el [dashboard de Exa](https://dashboard.exa.ai) (pay-per-use; una corrida `effort=high` ronda $0.50).
- 429 en la Agent API = límite de corridas concurrentes. Espera a que termine la anterior.
- Mientras tanto, usa `--source=nlm` para correr solo NotebookLM (gratis).

</details>

<details>
<summary><b>Tengo dos `notebooklm-mcp` y no sé cuál arranca</b></summary>

`notebooklm-py[mcp]` y el paquete viejo `notebooklm-mcp-cli` instalan un ejecutable con el mismo nombre. La skill no se ve afectada (usa el CLI `notebooklm`), pero para tus servidores MCP:
- notebooklm-py: lánzalo con `uvx --from "notebooklm-py[mcp]" notebooklm-mcp`.
- Legado: si lo sigues usando, fija la ruta absoluta del ejecutable viejo en su `command`.

Para ver cuál tienes: `which -a notebooklm-mcp` (o `where notebooklm-mcp` en Windows) y `notebooklm-mcp --help` de cada uno.

</details>

<details>
<summary><b>Claude no detecta la skill `deep-research`</b></summary>

- Reinicia Claude Code (cerrar y abrir — así re-escanea `~/.claude/skills/`).
- Verifica que el archivo está en `~/.claude/skills/deep-research/SKILL.md` (no en un subdir raro).
- Verifica que el frontmatter YAML del SKILL.md no está roto (líneas `---` al inicio y fin del bloque).

</details>

<details>
<summary><b>Los MCPs no aparecen en la lista de tools disponibles</b></summary>

- Corre `claude mcp list` para ver el estado de cada MCP y `claude mcp get <nombre>` para el detalle.
- Verifica que tu `~/.claude.json` es JSON válido.
- Reinicia Claude Code.
- Para los HTTP: prueba `curl <url>` — si responde, el endpoint está vivo.

</details>

---

## FAQ

**¿Cuánto cuesta usar esto?**

- NotebookLM: **$0** (cuota generosa de Google, gratis).
- Context7: **$0** en tier sin auth.
- Papersflow: **$0** en guest mode.
- Firecrawl: **$0** en free tier (suficiente para uso personal).
- Exa: créditos de prueba, después pay-per-use. Una corrida del Exa Agent con `effort=high` costó **$0.50** en la prueba de referencia (60 búsquedas). Si usas `--source=nlm` queda todo gratis, pero pierdes el cross-validation.

**¿Por qué NotebookLM por CLI y no por MCP?**

El CLI de notebooklm-py funciona en la misma sesión sin reiniciar Claude Code, no suma decenas de herramientas al contexto, y no choca con el ejecutable del paquete viejo. El MCP de notebooklm-py es una capa delgada sobre la misma lógica, así que no pierdes capacidad: si lo quieres para otras tareas, está documentado en el paso 4.

**¿Por qué no usar solo el `WebSearch` nativo de Claude Code?**

Porque el `WebSearch` es un hit y listo — no explora múltiples fuentes, no sintetiza, no cross-valida. Para preguntas chicas (2-3 sources) sí sirve. Para decisiones de arquitectura donde necesitas cruzar evidencia de 20-40 fuentes distintas, no alcanza.

**¿Por qué dos motores en paralelo en vez de solo uno?**

Cross-validation atrapa alucinaciones. Si Exa y NotebookLM dicen lo mismo con fuentes independientes, tu confianza sube mucho. Si divergen, eso es una señal importante de que el topic está contestado. Un solo motor te da la falsa sensación de certeza.

**¿Funciona en Windows, macOS, Linux?**

Sí, los tres. `notebooklm-py` es Python y funciona cross-platform; en Windows la skill usa `PYTHONUTF8=1` para evitar problemas de encoding. Los MCPs HTTP obviamente son agnósticos al OS. El paso manual de copiar skills a `~/.claude/` tiene comandos para bash y PowerShell en [Instalación](#instalación-manual-paso-a-paso).

**¿Puedo customizar las skills?**

Sí. Después de copiarlas a `~/.claude/skills/`, son tuyas. Editarlas cambia cómo Claude se comporta. Si mejoras algo, mándame un PR.

**¿Esto reemplaza las skills de GSD (`get-shit-done`)?**

No. Es complementario. GSD es un framework de planeación de proyectos. `deep-research` es una skill de investigación que *puede* ser usada dentro de un flujo GSD (p.ej. antes de decidir stack tech en `/gsd-discuss-phase`), pero también funciona standalone en cualquier proyecto.

---

## Créditos

- **Inspirado en** el botón "Research" de [claude.ai](https://claude.ai) — este setup lo replica dentro de Claude Code.
- **NotebookLM:** [teng-lin/notebooklm-py](https://github.com/teng-lin/notebooklm-py) — CLI `notebooklm` (y MCP opcional). La primera versión de este repo usaba [jacob-bd/notebooklm-mcp-cli](https://github.com/jacob-bd/notebooklm-mcp-cli).
- **Exa Agent:** [exa.ai](https://exa.ai) — [Agent API](https://exa.ai/docs/reference/agent-api/overview) / MCP [`exa-labs/exa-mcp-server`](https://github.com/exa-labs/exa-mcp-server)
- **Context7:** [context7.com](https://context7.com)
- **Firecrawl:** [firecrawl.dev](https://firecrawl.dev)
- **Papersflow:** [papersflow.ai](https://papersflow.ai)

## License

[MIT](./LICENSE) — úsalo, modifícalo, compártelo.

---

<div align="center">

**¿Problemas? ¿Sugerencias?** Abre un [issue](https://github.com/josuebustosn/deep-research/issues) o manda un PR.

</div>
