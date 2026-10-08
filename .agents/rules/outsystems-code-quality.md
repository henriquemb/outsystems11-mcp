---
description: Padrões de Qualidade de Código e Prevenção de Débito Técnico OutSystems (Code Quality / AI Mentor Studio)
trigger: always_on
---

# Diretrizes Anti-Débito Técnico OutSystems (Code Quality / AI Mentor Studio)

Todas as implementações, manutenções e códigos gerados via Model API ou manipulados no Service Studio devem obedecer estritamente às regras abaixo para mitigar e impedir débitos técnicos:

---

## 1. Arquitetura (Architecture)
* **Hierarquia das Camadas:** Respeitar o Architecture Canvas: `Orchestration -> End-User -> Core -> Foundation`.
* **Sem serviços em Orchestration ou End-User:** Módulos de telas ou orquestração nunca devem prover serviços ou lógicas reutilizáveis para camadas inferiores ou horizontais.
* **Isolamento da Camada Foundation:** Módulos de fundação nunca podem consumir módulos Core (a fundação deve ser estritamente agnóstica de regras de negócio).
* **Zero Referências Cíclicas:** Proibidas dependências cíclicas entre módulos ou aplicações.
* **Entidades Públicas Read-Only:** Todas as entidades públicas **devem obrigatoriamente** estar configuradas como `Expose Read-Only = Yes`. Operações de CUD (Create/Update/Delete) devem ser encapsuladas em Server Actions públicas de negócio com as devidas validações.
* **Sem Telas em Core/Foundation:** Telas de negócio só podem residir em End-User ou Orchestration (exceção para popups reutilizáveis nomeados com sufixo `popup`).
* **Modularização vs. Monólitos:** Evitar módulos com excesso de conceitos misturados ou elementos públicos desnecessários.

---

## 2. Desempenho (Performance)
* **Queries fora de Loops:** É **terminantemente proibido** executar Aggregates ou SQL queries dentro de loops (`For Each`). O carregamento deve ser feito antes do loop com joins ou filtros adequados.
* **Limite Explícito de Registros:** 
  * Aggregates devem ter a propriedade `Max Records` definida.
  * Consultas SQL avançadas devem limitar o número de linhas com `TOP` (SQL Server) ou `ROWNUM` (Oracle).
* **Consolidação de Aggregates:** Evitar chamadas sequenciais de Aggregates encadeados que poderiam ser resolvidos em uma única consulta com Join.
* **Verificação de Lista Vazia:** Nunca usar `.List.Count = 0` ou `.List.Count > 0` para testar existência de dados; usar sempre a propriedade nativa `.List.Empty`.
* **Site Properties Estáticas:** Nunca atualizar Site Properties dinamicamente via lógica de aplicação em tempo de execução (isso invalida os caches do módulo).
* **Eventos de Tela Leves (Reactive/Mobile):** Não chamar Server Actions pesadas dentro dos eventos de ciclo de vida `On Initialize`, `On Ready` ou `On Render`.
* **Sessão e ViewState Otimizados:**
  * Não colocar estruturas complexas ou coleções volumosas em Session Variables.
  * (Traditional) Evitar carregar dados da Preparation no ViewState para consumo em Screen Actions.
* **Padrão Wake Timer:** Timers com previsão de duração maior que alguns minutos devem implementar controle de timeout explícito, commits parciais e reagendamento automático (`Wake<Timer>`).

---

## 3. Segurança (Security)
* **Prevenção de SQL Injection:**
  * Desativar `Expand Inline` por padrão em parâmetros de SQL.
  * Quando indispensável (ex: filtros dinâmicos de lista `IN (...)`), usar obrigatoriamente `BuildSafe_InClauseIntegerList()` ou `EncodeSql()`. Nunca fazer concatenação manual de strings.
* **Segurança de Identidade (`GetUserId`):**
  * Em Reactive/Mobile, **nunca** passar o identificador do usuário (`GetUserId()`) como parâmetro do cliente para o servidor nem como input de Block. O backend deve obter `GetUserId()` diretamente no contexto do servidor.
* **Acesso às Telas e Ações:** Não expor telas com dados sensíveis para as roles `Anonymous` ou `Registered` genéricas sem checagem de perfil/role específica.
* **Endpoints REST Seguros:** Exigir conexões protegidas (HTTPS/SSL) e autenticação explícita para serviços REST expostos.
* **Proteção de Segredos:** Site Properties contendo credenciais, chaves ou senhas devem ter a propriedade `Is Secret = Yes`.
* **Botões com Restrição de Acesso:** Botões desabilitados por motivo de segurança ou permissão devem ter `Visible = False` (não apenas `Enabled = False`).

---

## 4. Manutenibilidade (Maintainability)
* **Convenções de Nomenclatura Obrigatórias:**
  * **Ações de Timers:** Ações executadas por Timers devem **obrigatoriamente possuir o prefixo `"Timer_"`** no nome (ex: `Timer_ProcessarPropostas`, `Timer_SincronizarContatos`).
  * **Service Actions:** Todas as Service Actions devem **obrigatoriamente possuir o sufixo `"Service"`** no nome (ex: `ValidarDocumentoService`, `ObterBeneficiarioService`).
* **Documentação Obrigatória e Parâmetros Padronizados:**
  * Todas as Actions (Server, Client e Service Actions) **devem possuir descrição preenchida**.
  * **Parâmetros de Saída Obrigatórios em Actions:** Todas as actions devem possuir obrigatoriamente duas variáveis de saída:
    * `IsSucesso` (Boolean) e `Mensagem` (Text) caso implementado em português;
    * `IsSuccess` (Boolean) e `Message` (Text) caso implementado em inglês.
    * `IsSucesso` / `IsSuccess` deve ser setado como `True` em caso de sucesso da execução e `False` em caso de falha. `Mensagem` / `Message` armazena a mensagem de erro em caso de falha.
  * Todas as Estruturas (Structures) e seus atributos **devem possuir descrição preenchida**.
  * Parâmetros de Entrada e Saída (Input, Output) **devem possuir descrição preenchida**; variáveis locais (Local Variables) **não são obrigatórias**.
  * **Proibição de Valores Padrão (Default Values):**
    * É **estritamente proibido** configurar valores padrão (`Default Value`) em parâmetros de entrada (`Input Parameters`) e parâmetros de saída (`Output Parameters`). Os valores de saída devem ser obrigatoriamente atribuídos através de nós `Assign` no fluxo lógico.
    * **Exceção única:** Apenas variáveis locais (`Local Variables`) destinadas a processamentos internos (contadores, buffers, replaces, regex) podem possuir valores padrão.
  * **Labels Obrigatórios em Nós de Atribuição (`AssignNode`):**
    * Todos os nós de `Assign` devem possuir obrigatoriamente `Label` preenchido, de forma curta e autoexplicativa (ex: `"Set Sucesso"`, `"Set Erro"`, `"Init Variaveis"`, `"Trunca Body"`). Proibido manter nós Assign sem label (nulo/vazio).
  * Se uma ação pública gerencia transações com `CommitTransaction` ou `AbortTransaction`, a descrição deve conter explicitamente a sinalização (ex: `[commit transaction]`).
* **Código Limpo e Sem Duplicação:**
  * Reutilizar lógica através de Actions compartilhadas em vez de duplicar fluxos.
  * Proibido manter código desabilitado morto ou lógica inacessível (condições fixas `True` / `False`).
  * Remover Actions ou Aggregates declarados que não são utilizados.
  * Não manter comentários com flag "Is Reminder" em código final.
  * Não consumir elementos obsoletos (deprecated).

---

## 5. Layout Visual de Fluxos (Model API) - Universal para Qualquer Quantidade de Nós
* **Regra Universal e Mandatória:** Este padrão matemático de grade e coordenadas aplica-se a **todo e qualquer fluxo**, independentemente da quantidade total de nós (sejam 3 nós ou 50+ nós), níveis de decisão ou ramificações. Nunca alterar, compactar arbitrariamente ou desviar dessas medidas em função da complexidade ou do número de nós do fluxo.
* **Posição Inicial Fixa do Nó `Start` (`StartNode`):** O nó `Start` deve estar sempre posicionado obrigatoriamente nas coordenadas fixas padrão: **`HorizontalPosition = 3200` (Tronco principal)** e **`VerticalPosition = 800`**. Todos os fluxos devem iniciar rigorosamente a partir dessa coordenada de origem.
* **Espaçamento Vertical:** Calculado de nó para nó consecutivamente:
  * **1 linha de título/label:** separação padrão de **1600** (via `ConnectedBelow(ref, 1600)` ou `Below(ref, 1600)`);
  * **2 linhas de título/label:** separação de **1829** (4 unidades de grid de 457, via `ConnectedBelow(ref, 1829)` ou `Below(ref, 1829)`).
* **Espaçamento Horizontal:** Avaliado no nó atual (a partir do conector):
  * Separação de **1829** (4 unidades de grid de 457, via `ToTheRightOf(ref, 1829)` ou `ConnectedToTheRightOf(ref, 1828)`).
* **Alinhamento Perfeito de Eixos e Grade (Grid Alignment):**
  * **Colunas Horizontais (Fórmula Geral de X):**
    `X_coluna = 3200 + (coluna * 1829)`
    * Tronco principal (Coluna 0): `X0 = 3200`
    * Coluna 1 (1º desvio): `X1 = 5029` (`3200 + 1829`)
    * Coluna 2 (2º desvio): `X2 = 6857` (`5029 + 1828`)
    * Coluna 3 (3º desvio): `X3 = 8686` (`6857 + 1829`)
    * Coluna n (enésimo desvio): `Xn = 3200 + n * 1829`
    Todos os nós em uma mesma coluna devem compartilhar rigorosamente a mesma coordenada `HorizontalPosition`, independente de quantos nós existam na coluna.
  * **Linhas Verticais Paralelas (Y):** Em ramificações horizontais, todos os nós do mesmo ramo devem compartilhar a mesma coordenada `VerticalPosition` (Y constante no segmento). Em branches descendentes de decisões, alinhar na mesma coordenada `VerticalPosition` do nó correspondente no tronco.
* **Sem sobreposições:** Nunca sobrepor nós ou linhas, respeitando o tamanho dos labels (nomes dos nós e condições de branches).
* **Organização compacta e legível:** Manter o fluxo compacto favorecendo sempre o entendimento e a clareza visual.
* **Loops (`For Each`):** A ramificação de ciclo (`CycleTarget`) **deve sempre iniciar alinhada à direita** do nó `For Each`; nunca iniciar para cima ou para baixo.
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
* **Comentários (`CommentNode`):**
  * **Posicionamento Lateral (Padrão Principal):** Coluna à esquerda em `X = 914` (5 unidades de grid à esquerda do tronco: `3200 - 2286`), posicionado **1371** para cima na vertical (`Y_nó - 1371`, correspondendo a 3 unidades de grid) em relação ao nó conectado.
  * **Fallback Vertical:** Posicionado acima do nó referenciado na mesma coluna `X` com espaçamento de **1371** para cima (`Y_nó - 1371`); caso também não haja espaço acima, posicionar abaixo (**1371** para baixo).
