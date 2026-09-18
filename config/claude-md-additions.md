# Adiciones a `~/.claude/CLAUDE.md`

Copia y pega el bloque de abajo al final de tu `CLAUDE.md` global (el que está en `~/.claude/CLAUDE.md`, no el de proyecto). Esto le enseña a Claude **los quirks críticos** de NotebookLM (vía `notebooklm-py`) y de Exa que, si no se conocen, cuestan horas de debugging.

> [!NOTE]
> **¿Venías de la versión anterior?** Si tu `CLAUDE.md` tiene la sección "NotebookLM MCP: verificar auth ANTES de cualquier fase autónoma" (la del flujo `nlm login` + `refresh_auth` de `notebooklm-mcp-cli`), reemplázala por el bloque de abajo. Si todavía tienes otras skills o agentes que usan `mcp__notebooklm-mcp__*`, deja la sección vieja solo para ellos y aclara que `deep-research` ya no la usa.

---

````markdown
## NotebookLM (notebooklm-py): verificar auth ANTES de cualquier investigación autónoma

La sesión de NotebookLM no es perdurable: expira sin aviso. Antes de cualquier flujo autónomo que use NotebookLM para investigar (skill `deep-research`, fases de research), **confirmar primero que el auth sigue válido**.

**Health check correcto (hace una llamada real a Google):**

```bash
notebooklm auth check --test --json
```

Exigir **las dos cosas**: `"status": "ok"` y `"checks": {"token_fetch": true}`.

**NO sirven como health check:** `notebooklm doctor` ni `notebooklm status`. Son locales: `doctor` puede decir "All checks passed" con la sesión vencida.

**Si falla (flujo end-to-end que Claude ejecuta, NO el usuario):**

1. Probar primero `notebooklm auth refresh` (refresco del lado del servidor) y volver a correr el check.
2. Si sigue fallando, **Claude lanza `notebooklm login` vía Bash** con `run_in_background=true`. Abre un navegador y espera hasta 5 minutos; el único paso humano es completar el login de Google en esa ventana. El comando termina solo cuando detecta el login.
3. Volver a correr `notebooklm auth check --test --json`. No hace falta ningún paso de "recarga": el CLI lee la sesión guardada en cada llamada.
4. **NO proceder** con la investigación hasta que el check devuelva `status: ok`.

**Otros quirks de notebooklm-py:**

- **Encoding en Windows:** toda llamada con `--json` debe ir con `PYTHONUTF8=1`. Sin eso, los acentos salen como `�` y el reporte queda corrupto sin avisar.
- **Deep research tarda más de 5 minutos a veces:** usar `notebooklm research wait --timeout 1800`. El default (300 s) se rinde antes de importar las fuentes.
- **Choque de nombres:** `notebooklm-py[mcp]` y el paquete viejo `notebooklm-mcp-cli` instalan ambos un ejecutable llamado `notebooklm-mcp`. Si configuras el MCP de notebooklm-py, lánzalo con `uvx --from "notebooklm-py[mcp]" notebooklm-mcp` o con la ruta absoluta.

**Aplicable a notebooklm-py 0.8.x (validado 2026-09-18).** Si el upstream cambia el comportamiento, confirmar contra los release notes del repo (`teng-lin/notebooklm-py`) antes de aplicar ciegamente.

## Exa: el research vigente es `agent_run`

- Las herramientas viejas `deep_researcher_start` / `deep_researcher_check` ya no existen. La investigación de Exa es el **Exa Agent**: herramienta MCP `agent_run` (API REST `POST /agent/runs`).
- Para seguir una corrida, llamar `agent_run` solo con `runId`. Reenviar `query` arranca otra corrida pagada.
- Si solo aparece la herramienta `authenticate` del servidor de Exa, falta el OAuth: pasar el link al usuario de inmediato (el callback a `localhost` dura poco). Si el navegador termina en error de conexión, pedirle la URL completa de la barra de direcciones y usar `complete_authentication`.
````

---

## Opcional: preferencia de MCPs sobre WebSearch nativo

Si quieres que Claude priorice tus herramientas de research sobre el `WebSearch` builtin en tus proyectos, añade esto a tu CLAUDE.md **de proyecto** (no al global — es por-proyecto según el tech stack):

```markdown
## Investigación

Preferir herramientas de research sobre WebSearch puro: `mcp__exa__*`, `mcp__firecrawl__*`, `mcp__context7__*` y el CLI `notebooklm` (notebooklm-py). Para queries grandes existe la skill `deep-research`.
```
