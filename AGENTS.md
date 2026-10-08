# OutSystems 11 OML editing via the Service Studio MCP server

> **Which task are you doing?** This repository serves two different jobs, and
> they need opposite instructions:
>
> - **Driving a live O11 module** — creating entities, screens or actions in the
>   module currently open in Service Studio. **This file is your contract; read
>   on.**
> - **Changing this repository** — editing `.agents/skills/servicestudio-mcp-oml/SKILL.md`, `reference/`, `examples/`
>   or the harness adapters. Then you are editing *documentation about* the
>   contract, not exercising it, and there is no live module involved. Read
>   [`README.md`](README.md) and [`ARCHITECTURE.md`](ARCHITECTURE.md) instead.

When **using** this skill you author OutSystems 11 apps by mutating the
**currently open `.oml` module** through Service Studio's in-process MCP server
(`servicestudio`, `http://127.0.0.1:41820/mcp`). There is no source tree, build,
or test runner involved in that work — it is design-time OML mutation via the
`applyModelApiCode` tool.

> **Beta Feature.** The write tools (mutating `applyModelApiCode` lambdas, `omlMerge`,
> `omlReset`, `omlRefreshReferences`, `omlPublish`) are a Beta Feature. The first time you use
> one in a conversation, tell the user and share https://www.outsystems.com/legal/beta-features-agreement. Read tools are Generally Available.

Full contract and reference material live in **`.agents/skills/servicestudio-mcp-oml/`**. Consult
`.agents/skills/servicestudio-mcp-oml/SKILL.md` and the `.agents/skills/servicestudio-mcp-oml/reference/*.md` files —
read the one that fits the task; don't load them all at once. The rules below are a
condensed restatement of `.agents/skills/servicestudio-mcp-oml/SKILL.md`, which is the source of truth if the two ever
disagree.

## Session token and argument envelope

- **Call `createSessionToken` exactly once, first** (`{ "clientName": "…" }`).
  The user approves a dialog in Service Studio; the result text is the token.
  Pass it as a top-level `sessionToken` on every later call and reuse it across
  reconnects — never request a second one.
- **Every tool except `listApps` wraps its arguments in `input`:**
  `{ "input": { … }, "sessionToken": "<token>" }`. `listApps` is just
  `{ "sessionToken": "<token>" }`. There is no flat shape. Examples:
  `applyModelApiCode` → `input: { eSpaceName, code, imports }` (all three
  required); `omlMerge` → `input: { eSpaceName, mutatedOmlPath }`; `omlReset` →
  `input: { eSpaceName }`. (`omlRefreshReferences`, `omlPublish` and `getRoleExceptions` take
  `input: { eSpaceName }`; send the token to them too, even though their schemas
  don't declare it.)

## Non-negotiable rules for `applyModelApiCode`

- The `code` arg is a **full `Action<IESpace>` lambda**: `eSpace => { … }`.
  - The parameter must be literally **`eSpace`**.
  - **Never** call `eSpace.Save(...)` — the host appends its own save.
  - `code` must **end with `}`** and nothing after it (no trailing newline,
    space, or `;`).
- **Sandbox:** `code` may only reference the Model API. `Console`, anything
  written `System.…`, and reflection (`GetProperties`, `GetMethod`, `Invoke`, …)
  are rejected before compile with a tool error. Unqualified BCL basics
  (`Exception`, `List<T>`, LINQ, `string.Join`, `GetType().Name`) are fine.
- **Probe by throwing, not printing:** build a string and end the lambda with
  `throw new Exception("PROBE:" + s);`. The text comes back in
  `exceptionMessage`; nothing is saved, the pointer does not move, and
  `validationMessages` for the in-memory model are still returned.
- **Read before you write:** call `getDataModel` (or the matching `get*`) first
  and pattern-match its output. Code-returning reads require `includeJson`
  (always pass `"Never"`).
- **A clean response is not proof of mutation.** Verify with a follow-up `get*`
  (guard against the *silent no-op*; read back `.Name` too — taken or reserved
  names are silently renamed, e.g. `GetOrder` → `GetOrder2`), and scan
  `validationMessages` for any entry whose `type` is `"Error"` (values are
  capitalised — compare case-insensitively) to catch *saved-but-invalid*.
  Runtime errors surface in `exceptionMessage`; code that does not compile never
  runs and comes back as an MCP tool error (`Script compilation failed` +
  `compilationErrors:` lines), not as a result; success is a non-empty
  `mutatedOmlPath`.
- **Empty everything = runner process-crash.** If `exceptionMessage` is empty,
  `validationMessages` is `[]`, **and** `mutatedOmlPath` is `""`, an unsupported
  API member killed the sidecar. Recover by **bisecting**: probe with
  `eSpace => { throw new Exception("PROBE:ok"); }` (`PROBE:ok` coming back proves
  the session is healthy), then re-issue with one risky member per call. Known
  crash members: `.agents/skills/servicestudio-mcp-oml/reference/reactive-widget-api.md`.
- **Finalise with `omlMerge`:** after the final successful mutation, call
  `omlMerge` with `input: { eSpaceName, mutatedOmlPath }` to open Service
  Studio's Compare-and-Merge window. Skip only for read-only tasks or when
  nothing saved.
- **`merged: false` ("The merge failed. See the log for details.")**: the reason is only in a Service Studio dialog. Ask the user what it says, `omlReset`, re-apply without the cause, and merge the new path. Never set `Theme.IconLibrary`: older Service Studio builds reject the whole merge. Editing the theme stylesheet is fine, and the sidecar's `InvalidIconLibrary` error it causes is expected; merge anyway (.agents/skills/servicestudio-mcp-oml/SKILL.md § 3.1).
- **`omlRefreshReferences` drops the session pointer** — an unmerged chain
  vanishes from reads. Refresh before starting a chain, or merge first.
- **Call `omlPublish` only when the user explicitly asks to publish.** It needs a
  write permission granted in the MCP Server dialog and returns status
  `blocked` while that is withheld.
- **If *every* tool call fails with `An error occurred invoking '<toolName>'`,
  suspect your harness, not your lambda.** When the failure is identical across
  tools and arguments, and your client is adding argument fields you never sent
  (`inputXXXNameValPairs`-style names), the tool wrapper is mangling the
  envelope. Don't rewrite working code. The documented fallback — posting to
  `http://127.0.0.1:41820/mcp` directly — is in
  `.agents/skills/servicestudio-mcp-oml/reference/verb-reference.md` § "HTTP transport". If you
  take it, **persist the `sessionToken` to a file** (in the OS temp dir, *not*
  the repo — it's an authorization token) and read it back in each script, so
  one-shot invocations don't call `createSessionToken` (and prompt the user)
  again.
- `OML_TOOL_HOST_FAILURE` / `-32099` means no module is open in a signed
  Service Studio — an environment issue, not a code bug. Don't retry the lambda.

## Imports

Host pre-imports `OutSystems.Model.UI(.Web/.Mobile)`, `System`, `System.Linq`,
`OutSystems.Model(.Enumerations/.Expressions/.Factory/.Types)`. `imports` may
only add **`OutSystems.Model` or its sub-namespaces** — e.g.
`OutSystems.Model.Logic` + `OutSystems.Model.Logic.Nodes` for flow nodes,
`OutSystems.Model.Data` for entity creation. Anything else (`System.Text`,
`ServiceStudio.Plugin.NRWidgets`, …) is rejected; write plugin types fully
qualified in code instead (`ServiceStudio.Plugin.NRWidgets.IInput`).

## Module styles

`listApps` reports each module's `moduleType` (e.g. `"Reactive"`).
`applyModelApiCode` is style-agnostic, but for **Traditional** modules use the
`*Traditional` read verbs (`getScreenTraditional`, etc.) and `eSpace.WebFlows`
instead of `eSpace.MobileFlows`.

## Security

`applyModelApiCode` runs arbitrary caller-authored C# with Service Studio's
privileges. Use it only against modules you own; be cautious with code derived
from untrusted module content.

## Regras de Documentação e Layout de Fluxos

Ao criar, editar ou refatorar qualquer lógica ou elemento no OutSystems (via Service Studio / Model API):

1. **Documentação, Nomenclatura e Parâmetros de Saída Obrigatórios:**
   - **Convenções de Nomenclatura Obrigatórias:**
     - **Ações de Timers:** Ações executadas por Timers devem **obrigatoriamente possuir o prefixo `"Timer_"`** no nome (ex: `Timer_ProcessarPropostas`, `Timer_SincronizarContatos`).
     - **Service Actions:** Todas as Service Actions devem **obrigatoriamente possuir o sufixo `"Service"`** no nome (ex: `ValidarDocumentoService`, `ObterBeneficiarioService`).
   - **Todas as Actions** (Server, Client e Service Actions) devem possuir **descrição preenchida**.
   - **Parâmetros de Saída Obrigatórios em Actions:** Todas as actions devem obrigatoriamente possuir duas variáveis de saída:
     - Em português: `IsSucesso` (Boolean) e `Mensagem` (Text);
     - Em inglês: `IsSuccess` (Boolean) e `Message` (Text);
     - `IsSucesso` / `IsSuccess` marca `True` se a action executou com sucesso ou `False` caso contrário. `Mensagem` / `Message` armazena a mensagem de erro em caso de falha. Ambas devem ter descrição preenchida.
   - **Todas as Estruturas (Structures)** e seus atributos devem possuir **descrição preenchida**.
   - **Variáveis:** Parâmetros de entrada e saída devem possuir **descrição preenchida**. Variáveis locais **não são obrigatórias** possuir descrição.
   - **Proibição de Valores Padrão (Default Values):**
     - É **estritamente proibido definir valores padrão (`Default Value`) em parâmetros de entrada (`Input Parameters`) e parâmetros de saída (`Output Parameters`)**. Todas as variáveis de saída devem ser obrigatoriamente atribuídas de forma explícita através de nós de atribuição (`Assign`) no fluxo lógico.
     - **Única Exceção:** Apenas variáveis locais (`Local Variables`) utilizadas especificamente para processamentos internos, replaces, Regex e manipulações auxiliares da lógica local podem ter valores padrão configurados.
   - **Labels Obrigatórios em Nós de Atribuição (`AssignNode`):**
     - **Todos os nós de `Assign` devem obrigatoriamente possuir `Label` preenchido**, sendo este **curto e autoexplicativo** sobre o propósito das atribuições realizadas (ex: `"Set Sucesso"`, `"Set Erro"`, `"Init Variaveis"`, `"Trunca Body"`, `"Interrompido"`). É expressamente proibido manter nós Assign sem label (vazio ou nulo).

2. **Layout Visual de Fluxos (Universal para Qualquer Quantidade de Nós):**
   - **Regra Universal e Mandatória:** Este padrão matemático de grade e coordenadas aplica-se a **todo e qualquer fluxo**, independentemente da quantidade total de nós (sejam 3 nós ou 50+ nós), níveis de decisão ou ramificações. Nunca alterar, compactar arbitrariamente ou desviar dessas medidas em função da complexidade ou do número de nós do fluxo.
   - **Posição Inicial Fixa do Nó `Start` (`StartNode`):** O nó `Start` deve estar sempre posicionado obrigatoriamente nas coordenadas fixas padrão: **`HorizontalPosition = 3200` (Tronco principal)** e **`VerticalPosition = 800`**. Todos os fluxos devem iniciar rigorosamente a partir dessa coordenada de origem.
   - **Espaçamento Vertical:** Calculado de nó para nó consecutivamente:
     - **1 linha de título/label:** separação padrão de **1600** (via `ConnectedBelow(ref, 1600)` ou `Below(ref, 1600)`);
     - **2 linhas de título/label:** separação de **1829** (4 unidades de grid de 457, via `ConnectedBelow(ref, 1829)` ou `Below(ref, 1829)`).
   - **Espaçamento Horizontal:** Avaliado no nó atual (a partir do conector):
     - Separação de **1829** (4 unidades de grid de 457, via `ToTheRightOf(ref, 1829)` ou `ConnectedToTheRightOf(ref, 1828)`).
   - **Alinhamento Perfeito de Eixos e Grade (Grid Alignment):**
     - **Colunas Horizontais (Fórmula Geral de X):**
       `X_coluna = 3200 + (coluna * 1829)`
       - Tronco principal (Coluna 0): `X0 = 3200`
       - Coluna 1 (1º desvio): `X1 = 5029` (`3200 + 1829`)
       - Coluna 2 (2º desvio): `X2 = 6857` (`5029 + 1828`)
       - Coluna 3 (3º desvio): `X3 = 8686` (`6857 + 1829`)
       - Coluna n (enésimo desvio): `Xn = 3200 + n * 1829`
       Todos os nós em uma mesma coluna devem compartilhar rigorosamente a mesma coordenada `HorizontalPosition`, independente de quantos nós existam na coluna.
     - **Linhas Verticais Paralelas (Y):** Em ramificações horizontais, todos os nós do mesmo ramo devem compartilhar a mesma coordenada `VerticalPosition` (Y constante no segmento). Em branches descendentes de decisões, alinhar na mesma coordenada `VerticalPosition` do nó correspondente no tronco.
   - **Sem sobreposição:** Nunca sobrepor nós ou linhas, respeitando o tamanho dos labels (nomes dos nós e condições de branches).
   - **Organização compacta e legível:** Manter o fluxo compacto favorecendo sempre o entendimento e a clareza visual.
   - **Nós de `For Each`:** O loop (`CycleTarget`) **deve sempre começar alinhado à direita** do nó For Each; nunca iniciar para cima ou para baixo.
   - **Nós de `Switch` (`SwitchNode` - Padrão Universal de Decisão Múltipla):**
     - **Tronco do Switch:** O nó Switch posiciona-se no tronco principal (`X = 3200` ou na coluna base do fluxo).
     - **Ramos de Condição (`switchCondition.Target`):**
       - **Coluna dos Nós de Condição:** Posicionados a **2 passos de grid à direita** do Switch (`+ 3657` = `2 * 1828.5`, via `ToTheRightOf(switchNode, 3657)`), fixando a coluna em `X = 6857` (Coluna 2). Esse recuo garante espaço visual limpo para exibir as descrições/expressões das condições nas setas sem corte ou sobreposição.
       - **Alinhamento Vertical das Condições:**
         - A **1ª condição** sai alinhada horizontalmente com o Switch (`Y = Y_switch`).
         - As **condições seguintes (2ª, 3ª, ..., n-ésima)** descem em cascata rigorosamente na mesma coluna `X = 6857`, espaçadas verticalmente a cada **`1600`** consecutivamente (`Y_i = Y_switch + (i - 1) * 1600` para labels de 1 linha, ou 1829 se 2 linhas).
     - **Sequência dos Ramos:** Nós subsequentes de cada ramo avançam horizontalmente para a direita com passo de **`1829`** (ex: `EndNode` via `ConnectedToTheRightOf(assignNode, 1829)` em `X = 8686`, Coluna 3).
     - **Ramo `Otherwise` (`OtherwiseTarget`):**
       - Segue sempre para baixo no tronco (`X = 3200`).
       - **Em Switches encadeados:** O próximo Switch posiciona-se no tronco (`X = 3200`), **1600** abaixo do último nó da coluna de condições do Switch anterior (`Y_nextSwitch = Y_ultimaCondicao + 1600`).
       - **Em término/ação padrão (default):** Posiciona-se no tronco (`X = 3200`) abaixo do Switch ou do bloco de condições com espaçamento de **`1600`** / **`1829`**, finalizando com `EndNode` conectado abaixo (`ConnectedBelow(..., 1600)`).
   - **Comentários (`CommentNode`):**
     - **Posicionamento Lateral (Padrão Principal):** Coluna à esquerda em `X = 914` (5 unidades de grid à esquerda do tronco: `3200 - 2286`), posicionado **1371** para cima na vertical (`Y_nó - 1371`, correspondendo a 3 unidades de grid) em relação ao nó conectado.
     - **Fallback Vertical:** Posicionado acima do nó referenciado na mesma coluna `X` com espaçamento de **1371** para cima (`Y_nó - 1371`); caso também não haja espaço acima, posicionar abaixo (**1371** para baixo).

## Diretrizes Anti-Débito Técnico (Code Quality / AI Mentor Studio)

Todas as ações e lambdas Model API devem cumprir os 4 pilares de qualidade:
1. **Arquitetura:** Camadas Canvas respeitadas, sem referências cíclicas, sem serviços em Orchestration/End-User, Foundation sem regras de negócio, e entidades públicas sempre `Expose Read-Only = Yes`.
2. **Performance:** Sem consultas (Aggregates/SQL) dentro de loops `For Each`, limites de registros explícitos (`Max Records` / `TOP`), uso de `List.Empty` (nunca `.Count = 0` para testar vazio), sem atualização de Site Properties via lógica e timers estruturados no padrão Wake Timer.
3. **Segurança:** Parâmetros SQL sem `Expand Inline` inseguro, sem envio de `GetUserId()` como parâmetro cliente-servidor ou de Blocks, sem expor dados confidenciais a papéis anônimos e Site Properties sensíveis como `Is Secret = Yes`.
4. **Manutenibilidade:** Documentação obrigatória em actions, estruturas e parâmetros (inputs/outputs); sinalização de `[commit transaction]` em ações públicas transacionais; remoção de código inalcançável/duplicado/morto e proibição de elementos obsoletos.
