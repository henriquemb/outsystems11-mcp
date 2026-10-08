---
description: Diretrizes de documentação e layout visual de fluxos de lógica OutSystems
trigger: always_on
---

# Regras de Boas Práticas: Documentação e Layout de Fluxos

Ao criar, editar ou refatorar qualquer lógica ou elemento no OutSystems (via Service Studio / Model API), siga rigorosamente as seguintes diretrizes:

## 1. Documentação, Nomenclatura e Parâmetros Obrigatórios
* **Convenções de Nomenclatura Obrigatórias:**
  * **Ações de Timers:** Ações executadas por Timers devem **obrigatoriamente possuir o prefixo `"Timer_"`** no nome (ex: `Timer_ProcessarPropostas`, `Timer_SincronizarContatos`).
  * **Service Actions:** Todas as Service Actions devem **obrigatoriamente possuir o sufixo `"Service"`** no nome (ex: `ValidarDocumentoService`, `ObterBeneficiarioService`).
* **Todas as Actions** (Server, Client e Service Actions) **devem possuir descrição preenchida** clara e objetiva sobre o propósito da ação.
* **Parâmetros de Saída Padronizados (Obrigatório em Todas as Actions):**
  * Todas as Actions devem obrigatoriamente possuir duas variáveis/parâmetros de saída (`OutputParameters`):
    * **Em português:** `IsSucesso` (Boolean) e `Mensagem` (Text).
    * **Em inglês:** `IsSuccess` (Boolean) e `Message` (Text).
  * **Comportamento e Semântica:**
    * `IsSucesso` / `IsSuccess`: Deve ser atribuído como `True` se a ação foi executada com sucesso, ou `False` caso ocorra falha/erro.
    * `Mensagem` / `Message`: Deve conter a mensagem descritiva do erro em caso de falha (ou vazia/informativa em caso de sucesso).
    * Ambas as variáveis de saída **devem possuir descrição preenchida**.
* **Todas as Estruturas (Structures)** e seus respectivos atributos **devem possuir descrição preenchida**.
* **Documentação de Variáveis:** Parâmetros de entrada (`InputParameters`) e parâmetros de saída (`OutputParameters`) **devem possuir descrição preenchida**. Variáveis locais (`LocalVariables`) **não são obrigatórias** possuir descrição.
* **Proibição de Valores Padrão (Default Values):**
  * É **estritamente proibido** definir valores padrão (`Default Value`) em parâmetros de entrada (`Input Parameters`) e parâmetros de saída (`Output Parameters`). Os valores das saídas devem ser explicitamente atribuídos via nós de atribuição (`Assign`) no fluxo lógico da ação.
  * **Exceção permitida:** Somente variáveis locais (`Local Variables`) utilizadas especificamente para manipulação ou processamento interno local (ex: variáveis de substituição/replace, Regex, acumuladores) podem possuir valores padrão definidos.
* **Labels Obrigatórios em Nós de Atribuição (`AssignNode`):**
  * **Todos os nós de `Assign` devem possuir obrigatoriamente `Label` preenchido**, sendo este **curto e autoexplicativo** sobre o propósito das atribuições realizadas (ex: `"Set Sucesso"`, `"Set Erro"`, `"Init Variaveis"`, `"Trunca Body"`). É proibido manter nós Assign sem label (nulo/vazio).

## 2. Layout Visual de Fluxos (Universal para Qualquer Quantidade de Nós)
Ao posicionar ou criar nós (`Nodes`) no fluxo lógico de uma Action:

* **Regra Universal e Mandatória:** Este padrão matemático de grade e coordenadas aplica-se a **todo e qualquer fluxo**, independentemente da quantidade total de nós (sejam 3 nós ou 50+ nós), níveis de decisão ou ramificações. Nunca alterar, compactar arbitrariamente ou desviar dessas medidas em função da complexidade ou do número de nós do fluxo.
* **Posição Inicial Fixa do Nó `Start` (`StartNode`):** O nó `Start` deve estar sempre posicionado obrigatoriamente nas coordenadas fixas padrão: **`HorizontalPosition = 3200` (Tronco principal)** e **`VerticalPosition = 800`**. Todos os fluxos devem iniciar rigorosamente a partir dessa coordenada de origem.
* **Espaçamento entre Nós:**
  * **Vertical:** Calculado de nó para nó consecutivamente:
    * **1 linha de título/label:** separação padrão de **1600** (via `ConnectedBelow(ref, 1600)` ou `Below(ref, 1600)`);
    * **2 linhas de título/label:** separação de **1829** (4 unidades de grid de 457, via `ConnectedBelow(ref, 1829)` ou `Below(ref, 1829)`).
  * **Horizontal:** Avaliado no nó atual (a partir do conector):
    * Separação de **1829** (4 unidades de grid de 457, via `ToTheRightOf(ref, 1829)` ou `ConnectedToTheRightOf(ref, 1828)`).
  * **Evitar Sobreposição:** Em nenhuma hipótese deve haver sobreposição de nós, conectores ou linhas de fluxo.
  * **Tamanho de Labels:** Respeitar integralmente o comprimento do texto dos labels (nomes dos nós, condições de `If`, `Switch`, etc.), garantindo que os nós adjacentes não fiquem colados ou cortem a visibilidade do texto.
* **Organização do Fluxo e Alinhamento Perfeito de Eixos e Grade (Grid Alignment):**
  * O fluxo deve ser mantido de forma **compacta**, priorizando sempre a legibilidade, clareza e entendimento visual imediato do fluxo de negócio.
  * **Colunas Horizontais (Fórmula Geral de X):**
    `X_coluna = 3200 + (coluna * 1829)`
    * Tronco principal (Coluna 0): `X0 = 3200`
    * Coluna 1 (1º desvio): `X1 = 5029` (`3200 + 1829`)
    * Coluna 2 (2º desvio): `X2 = 6857` (`5029 + 1828`)
    * Coluna 3 (3º desvio): `X3 = 8686` (`6857 + 1829`)
    * Coluna n (enésimo desvio): `Xn = 3200 + n * 1829`
    Todos os nós em uma mesma coluna devem compartilhar rigorosamente a mesma coordenada `HorizontalPosition`, independente de quantos nós existam na coluna.
  * **Linhas Verticais Paralelas (Y):** Em ramificações horizontais, todos os nós do mesmo ramo devem compartilhar a mesma coordenada `VerticalPosition` (Y constante no segmento). Em branches descendentes de decisões, alinhar na mesma coordenada `VerticalPosition` do nó correspondente no tronco.
* **Alinhamento de `For Each`:**
  * Nós de **For Each** devem **sempre iniciar o loop (`CycleTarget`) alinhado à direita** do nó For Each.
  * É expressamente proibido direcionar o início do loop para cima ou para baixo.
  * A saída do For Each (`Target` após conclusão da iteração) segue o fluxo para baixo ou para a esquerda conforme o design, mantendo o padrão limpo e sem cruzar linhas.
* **Nós de `Switch` (`SwitchNode` - Padrão Universal de Decisão Múltipla):**
  * **Tronco do Switch:** O nó Switch posiciona-se no tronco principal (`X = 3200` ou na coluna base do fluxo).
  * **Ramos de Condição (`switchCondition.Target`):**
    * **Coluna dos Nós de Condição:** Posicionados a **2 passos de grid à direita** do Switch (`+ 3657` = `2 * 1828.5`, via `ToTheRightOf(switchNode, 3657)`), fixando a coluna em `X = 6857` (Coluna 2). Esse recuo garante espaço visual limpo para exibir as descrições/expressões das condições nas setas sem corte ou sobreposição.
    * **Alinhamento Vertical das Condições:**
      * A **1ª condição** sai alinhada horizontalmente com o Switch (`Y = Y_switch`).
      * As **condições seguintes (2ª, 3ª, ..., n-ésima)** descem em cascata rigorosamente na mesma coluna `X = 6857`, espaçadas verticalmente a cada **`1600`** consecutivamente (`Y_i = Y_switch + (i - 1) * 1600` para labels de 1 linha, ou 1829 se 2 linhas).
  * **Sequência dos Ramos:** Nós subsequentes de cada ramo avançam horizontalmente para a direita com passo de **`1829`** (ex: `EndNode` via `ConnectedToTheRightOf(assignNode, 1829)` em `X = 8686`, Coluna 3).
  * **Ramo `Otherwise` (`OtherwiseTarget`):**
    * Segue sempre para baixo no tronco (`X = 3200`).
    * **Em Switches encadeados:** O próximo Switch posiciona-se no tronco (`X = 3200`), **1600** abaixo do último nó da coluna de condições do Switch anterior (`Y_nextSwitch = Y_ultimaCondicao + 1600`).
    * **Em término/ação padrão (default):** Posiciona-se no tronco (`X = 3200`) abaixo do Switch ou do bloco de condições com espaçamento de **`1600`** / **`1829`**, finalizando com `EndNode` conectado abaixo (`ConnectedBelow(..., 1600)`).
* **Alinhamento e Espaçamento de Comentários (`CommentNode`):**
  * **Posicionamento Lateral (Padrão Principal):** Coluna à esquerda em `X = 914` (5 unidades de grid à esquerda do tronco: `3200 - 2286`), posicionado **1371** para cima na vertical (`Y_nó - 1371`, correspondendo a 3 unidades de grid) em relação ao nó conectado.
  * **Fallback Vertical:** Posicionado acima do nó referenciado na mesma coluna `X` com espaçamento de **1371** para cima (`Y_nó - 1371`); caso também não haja espaço acima, posicionar abaixo (**1371** para baixo).
