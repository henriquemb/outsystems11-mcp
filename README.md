# OutSystems 11 MCP & AI Engineering Guidelines

> **Fork do Repositório Oficial:** Este projeto é um *fork* customizado e aprimorado a partir do repositório oficial [OutSystems/outsystems11-mcp](https://github.com/OutSystems/outsystems11-mcp).  
> Ele adiciona diretrizes estritas de governança, padrões de código limpo (Anti-Débito Técnico), regras mandatórias de documentação e uma especificação matemática universal de layout visual de fluxos para desenvolvimento assistido por agentes de IA no OutSystems 11 via Service Studio Model API.

---

## 📌 Visão Geral

O **OutSystems 11 MCP** permite que agentes de IA (Google Antigravity, Claude Code, GitHub Copilot, Cursor, etc.) inspecionem, refatorem e gerem lógica e estruturas diretamente nos módulos `.oml` abertos no **Service Studio** através de um servidor MCP local embutido (`http://127.0.0.1:41820/mcp`).

Todas as alterações em tempo de design são regidas pela **Model API** do OutSystems e passam pelo ciclo de revisão e validação antes da aceitação final (Compare-and-Merge).

---

## 🚀 Como Iniciar o Servidor MCP no Service Studio

1. Abra o módulo desejado no **Service Studio** (versão 11.55.91 ou superior).
2. No menu superior, vá em **Edit > MCP Configuration...**.
3. Confirme a porta padrão (`41820`) e clique em **Start MCP Server**.
4. No primeiro comando emitido pelo agente, aprove a solicitação de autorização no Service Studio para gerar o handshake de confiança.

---

## 🏛️ Regras e Padrões Mandatórios de Desenvolvimento

Todos os agentes de IA e desenvolvedores que operam neste repositório devem seguir obrigatoriamente as diretrizes abaixo:

### 1. Nomenclatura Obrigatória
* **Ações de Timers:** Ações executadas por Timers devem **obrigatoriamente possuir o prefixo `"Timer_"`** no nome (ex: `Timer_ProcessarPropostas`, `Timer_SincronizarContatos`).
* **Service Actions:** Todas as Service Actions devem **obrigatoriamente possuir o sufixo `"Service"`** no nome (ex: `ValidarDocumentoService`, `ObterBeneficiarioService`).

---

### 2. Documentação e Parâmetros Padronizados
* **Descrição Obrigatória:**
  * Todas as Actions (Server, Client e Service Actions) devem possuir **descrição preenchida** detalhando seu propósito.
  * Todas as Estruturas (Structures) e seus atributos devem possuir **descrição preenchida**.
  * Parâmetros de entrada (`Input Parameters`) e saída (`Output Parameters`) devem possuir **descrição preenchida**.
* **Parâmetros de Saída Obrigatórios em Actions:**
  * Toda Action deve possuir obrigatoriamente dois parâmetros de saída:
    * Em português: **`IsSucesso`** (`Boolean`) e **`Mensagem`** (`Text`);
    * Em inglês: **`IsSuccess`** (`Boolean`) e **`Message`** (`Text`).
  * `IsSucesso` marca `True` se a ação foi executada com sucesso ou `False` em caso de erro. `Mensagem` armazena a mensagem de erro (ou informativa).
* **Proibição de Valores Padrão (`Default Value`):**
  * É **estritamente proibido configurar valores padrão em parâmetros de entrada e de saída**. Todas as saídas devem ser atribuídas explicitamente no fluxo lógico através de nós de atribuição (`Assign`).
  * **Exceção única permitida:** Apenas variáveis locais (`Local Variables`) utilizadas para manipulação interna (contadores, buffers, replaces, regex) podem possuir valor default.
* **Labels Obrigatórios em Nós de Atribuição (`AssignNode`):**
  * **Todo nó `Assign` deve obrigatoriamente possuir `Label` preenchido**, sendo este **curto e autoexplicativo** sobre o propósito das atribuições (ex: `"Set Sucesso"`, `"Set Erro"`, `"Init Variaveis"`, `"Update DataAnterior"`, `"Trunca BodyFinal"`). É proibido manter nós Assign sem label (vazio ou nulo).

---

### 3. Padrão Universal de Layout Visual de Fluxos (Model API)

Esta especificação geométrica aplica-se a **todo e qualquer fluxo**, independentemente do tamanho (sejam 3 nós ou 50+ nós):

#### A. Origem e Eixo Base
* **Posição Inicial Fixa do Nó `Start`:** O nó `Start` deve estar sempre posicionado em **`HorizontalPosition = 3200`** (Tronco principal) e **`VerticalPosition = 800`**. Todos os fluxos iniciam rigorosamente a partir dessa coordenada.

#### B. Espaçamento Vertical Consecutivo ($\Delta Y$)
* **1 linha de título/label:** separação exata de **`1600`** (via `ConnectedBelow(ref, 1600)` ou `Below(ref, 1600)`).
* **2 linhas de título/label:** separação exata de **`1829`** (4 unidades de grid de 457, via `ConnectedBelow(ref, 1829)` ou `Below(ref, 1829)`).

#### C. Espaçamento Horizontal e Fórmula de Colunas ($X$)
Qualquer desvio ou coluna horizontal segue a fórmula matemática:
$$\mathbf{X_{coluna} = 3200 + (coluna \times 1829)}$$

* **Coluna 0 (Tronco Principal):** `X0 = 3200`
* **Coluna 1 (1º Desvio à Direita):** `X1 = 5029` (`3200 + 1829`)
* **Coluna 2 (2º Desvio à Direita):** `X2 = 6857` (`5029 + 1828`)
* **Coluna 3 (3º Desvio à Direita):** `X3 = 8686` (`6857 + 1829`)
* **Coluna $n$ ($n$-ésimo Desvio):** `Xn = 3200 + n * 1829`

> Todos os nós em uma mesma coluna devem compartilhar rigorosamente a mesma coordenada `HorizontalPosition`.

#### D. Paralelismo e Branches Horizontais ($Y$)
* Em ramificações horizontais, todos os nós do mesmo segmento mantêm a mesma coordenada `VerticalPosition` ($Y$ constante).
* Em branches descendentes de decisões, alinhar na mesma coordenada $Y$ do nó correspondente no tronco para manter o paralelismo visual.

#### E. Loops (`For Each`)
* O ciclo iterativo (`CycleTarget`) **deve sempre iniciar alinhado à direita** do nó `For Each` (nunca para cima, para baixo ou para a esquerda).

#### F. Padrão Universal para Nós de Decisão Múltipla (`SwitchNode`)
* **Tronco do Switch:** Fica posicionado no tronco principal (`X = 3200`).
* **Coluna das Condições:** Deslocada **2 passos de grid à direita** em **`X = 6857`** (Coluna 2, via `ToTheRightOf(switchNode, 3657)`), garantindo espaço visual limpo para ler os labels das condições nas setas sem sobreposição.
* **Alinhamento em Cascata das Condições:**
  * 1ª condição sai alinhada horizontalmente com o Switch ($Y = Y_{switch}$).
  * Condições seguintes (2ª até $n$-ésima) descem na mesma coluna `X = 6857` com passo vertical consecutivo de **`1600`** ($Y_i = Y_{switch} + (i - 1) \times 1600$).
* **Ramo `Otherwise`:** Segue sempre para baixo no tronco (`X = 3200`). Em switches encadeados, o próximo Switch posiciona-se no tronco **`1600`** abaixo do último nó da coluna de condições do switch anterior.

#### G. Comentários (`CommentNode`)
* **Posicionamento Lateral (Padrão):** Coluna à esquerda em **`X = 914`** com deslocamento de **`1371`** para cima em relação ao nó conectado ($Y_{nó} - 1371$).
* **Fallback Vertical:** Diretamente acima do nó na mesma coluna com deslocamento de **`1371`** para cima ($Y_{nó} - 1371$).

---

### 4. Diretrizes Anti-Débito Técnico (AI Mentor Studio / Code Quality)
1. **Arquitetura Canvas:** Hierarquia respeitada (`Orchestration -> End-User -> Core -> Foundation`). Módulos Foundation agnósticos de negócio. Entidades públicas sempre com `Expose Read-Only = Yes`. Zero referências cíclicas.
2. **Performance:** Proibido executar Aggregates ou consultas SQL dentro de loops (`For Each`). Limites explícitos de registros (`Max Records` / `TOP`). Verificação de listas vazias exclusivamente via `.List.Empty` (nunca `.Count = 0`). Proibido alterar Site Properties em tempo de execução via lógica.
3. **Segurança:** Parâmetros SQL sem `Expand Inline` desnecessário (quando exigido, usar `BuildSafe_InClauseIntegerList()` ou `EncodeSql()`). Proibido passar `GetUserId()` do cliente para o servidor. Site Properties com segredos/senhas configuradas como `Is Secret = Yes`.
4. **Manutenibilidade:** Código limpo, sem lógica inalcançável (dead code) ou elementos obsoletos (deprecated). Sinalização explícita de `[commit transaction]` em ações transacionais públicas.

---

## 📂 Estrutura do Repositório

```
├── .agents/
│   ├── mcp_config.json               # Configuração do MCP Server local
│   ├── rules/
│   │   ├── outsystems-code-quality.md   # Diretrizes Anti-Débito Técnico (AI Mentor Studio)
│   │   └── outsystems-code-standards.md # Padrões de documentação e layout visual
│   └── skills/
│       └── servicestudio-mcp-oml/       # Skill Model API (SKILL.md, referências e exemplos)
├── AGENTS.md                         # Contrato principal e regras executáveis dos agentes
├── ARCHITECTURE.md                   # Tenets e arquitetura da Skill Model API
├── CONTRIBUTING.md                   # Diretrizes para contribuição com o projeto
├── LICENSE                           # Termos de licença do repositório
├── MCP-DESIGN-CHOICES.md             # Decisões de design e comportamento do MCP Server
└── README.md                         # Documentação central do projeto
```

---

## 📄 Licença e Termos

* As ferramentas de escrita e mutação no OML via Model API são um recurso beta da OutSystems. Consulte o [OutSystems Beta Features Agreement](https://www.outsystems.com/legal/beta-features-agreement).
* Projeto baseado em [OutSystems/outsystems11-mcp](https://github.com/OutSystems/outsystems11-mcp).
