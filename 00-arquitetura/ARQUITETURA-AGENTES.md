# Arquitetura de agentes — pipeline de manual de marca automático

> Espelha o padrão de orquestração de `.claude/agents/*.md` da BNP (ver
> `.claude/agents/bnp-logic-auditor.md` e `.claude/agents/bnp-brand-reviewer.md`
> como referência de formato de agente e `.claude/skills/descoberta-marca/SKILL.md`
> como referência do método de 8 módulos). Adaptação: sem entrevista ao vivo,
> tudo roda automático a partir de sinais coletados — ver `CHRONOLOGIA.md` §0
> pra decisão completa.

## 1. Visão geral da orquestração

10 agentes, 4 fases. Cada fase tem um gate de qualidade implícito (o agente
seguinte não consome output malformado) e a fase final tem um gate explícito
(agent-09). Nenhum agente roda git, nenhum agente escreve fora do registro do
próprio lead (isolamento por `lead_id` — ver §4).

```
FASE 1 — INGESTÃO (sequencial)
  agent-01-ingestor-sinais
       │
       ▼
FASE 2 — DESCOBERTA E SISTEMA VISUAL (paralelo depois do agent-02)
  agent-02-descoberta-automatica
       │
       ├──▶ agent-03-posicionamento
       ├──▶ agent-04-sistema-cor  ──┐
       ├──▶ agent-05-tipografia-e-composicao
       └──▶ [agent-06-logo-guardiao OU agent-07-logo-alternativas]  (ver §3)
                                     │
FASE 3 — COMPOSIÇÃO (sequencial, depende de tudo acima)
  agent-08-compositor-manual
       │
       ▼
  agent-10-explicador-didatico  (enriquece cada elemento com racional)
       │
       ▼
FASE 4 — GATE (sequencial, bloqueia entrega)
  agent-09-revisor-qualidade
       │
       ├─ aprovado / aprovado_com_ressalvas → estágio "pronto"
       └─ reprovado → volta pro agente responsável pelo achado, no máximo
                       2 ciclos automáticos; 3º reprovado escala pra fila
                       humana de suporte/operações do próprio webfy — nunca
                       pro vendedor, nunca pra BNP (ver agent-09-revisor-
                       qualidade.md)
```

## 2. Por que essa ordem

- **agent-01 é sempre primeiro e sozinho.** Ele normaliza o input bruto da
  varredura (`01-contratos-de-dados/input-sinais-negocio.schema.json`) e
  decide se há sinal suficiente pra sequer iniciar. Sem isso, todo agente
  posterior herdaria lixo.
- **agent-02 bloqueia a fase 2 inteira.** Os 8 módulos de descoberta (adaptados
  de `descoberta-marca/SKILL.md`) alimentam posicionamento, cor e tom — rodar
  qualquer um desses antes seria arbitrário.
- **agent-03/04/05/06-ou-07 são paralelos entre si** depois do agent-02: não
  há dependência de dado entre posicionamento, paleta, tipografia e logo (cada
  um lê o mesmo output do agent-02, não um do outro). Only exception: se
  agent-06 detectar logo existente, a paleta do agent-04 deve ler as cores
  extraídas por ele — nesse caso agent-06 roda primeiro e agent-04 é
  disparado depois com esse insumo extra (ver §3).
- **agent-08 é o único que consolida tudo.** Ele não inventa conteúdo, só
  monta o schema de saída (`output-manual-marca.schema.json`) a partir dos
  outputs dos agentes 2-7.
- **agent-10 roda depois do agent-08**, não em paralelo, porque ele anexa o
  racional causal (mecanismo → percepção → aplicação → recomendação) em cima
  de elementos já decididos — ele não decide paleta/logo, só explica.
- **agent-09 é sempre o último gate**, equivalente ao papel que
  `bnp-logic-auditor` e `bnp-brand-reviewer` cumprem na BNP: nada sai sem
  passar por ele. Read-only, nunca corrige sozinho — aponta e devolve.

## 3. Bifurcação logo existente vs. logo novo

```
agent-01 encontra logo_url (ou foto marcada e_provavel_logo=true)?
  │
  ├── SIM ──▶ agent-06-logo-guardiao roda PRIMEIRO na fase 2
  │             extrai cores/estilo/qualidade do logo existente
  │             → esse output alimenta agent-04-sistema-cor
  │             → agent-07-logo-alternativas roda em seguida (não em vez de),
  │               gerando as 3 opções como upsell opcional
  │
  └── NÃO ──▶ agent-06 não roda (pula, marca "sem logo existente" no estado)
                agent-04-sistema-cor deriva a paleta do posicionamento/
                personalidade (agent-03), sem insumo de logo
                agent-07-logo-alternativas roda sozinho, gerando as 3 opções
                como escolha inicial obrigatória
```

Detalhe completo da lógica de decisão de cada lado: `02-agentes/agent-06-logo-guardiao.md`
e `02-agentes/agent-07-logo-alternativas.md`.

## 4. Isolamento entre leads (equivalente ao §0.5 do CLAUDE.md da BNP)

Cada execução do pipeline roda com um único `lead_id` em escopo. Nenhum
agente pode ler sinal, output parcial ou estado de outro `lead_id` na mesma
chamada. Isso é o equivalente direto de "nunca cruzar dado entre clientes" —
aqui, "cliente" é o negócio/lead capturado pela varredura. O agent-09 audita
isso explicitamente (ver `02-agentes/agent-09-revisor-qualidade.md`).

## 5. Tempo e custo (estimativa, pra dimensionar fila de processamento)

| Fase | Agentes | Tempo estimado | Paralelizável |
|---|---|---|---|
| 1 | agent-01 | 5-15s | não |
| 2 | agent-02 | 15-40s | não (bloqueia fase) |
| 2 | agent-03, 04, 05, 06/07 | 20-60s cada | sim, entre si |
| 2 | geração de imagem (3 logos) | 20-90s, depende do provedor | sim (as 3 em paralelo) |
| 3 | agent-08, agent-10 | 10-30s | não |
| 4 | agent-09 | 15-30s | não |

Total end-to-end razoável: **2 a 5 minutos** do disparo até "pronto", sem
contar fila de processamento se houver muitos leads simultâneos. Ver
`CHRONOLOGIA.md` pro detalhamento passo a passo com esses tempos aplicados
ao fluxo completo.

## 6. Semântica de erro e retry por agente

Nem toda falha é do mesmo tipo, e tratar todas como se fossem custa caro
(retry cego gastando o orçamento do gate de qualidade em algo que não tem
nada a ver com qualidade de conteúdo) ou devagar demais (o gate de
qualidade "tentando corrigir" o que na verdade foi um timeout de rede).
Três categorias, cada uma com orçamento de tentativa próprio:

### 6.1 Falha transitória (infra/provedor, não é falha do agente)

- **O que é:** timeout de chamada ao modelo, rate limit do provedor de LLM
  ou do serviço de geração de imagem, erro de rede, erro 5xx do provedor.
- **Comportamento:** retry automático com backoff (ex: exponencial), teto
  pequeno de tentativas — recomendação: **3**. Não conta como reprovação do
  gate de qualidade — é reprocessamento da MESMA chamada, não uma rodada
  pelo ciclo do agent-09.
- **Depois do teto:** se as 3 tentativas falharem, isso vira uma falha
  registrada em `estado_geracao.erro` (`etapa`, `mensagem`, `ocorrido_em`,
  ver `01-contratos-de-dados/estado-geracao.schema.json`) e o lead vai pro
  estágio `falhou` — não fica girando indefinidamente.

### 6.2 Falha de qualidade (o agente respondeu, mas o conteúdo não presta)

- **O que é:** output bem formado, mas reprovado no critério de conteúdo —
  o ciclo já documentado em `02-agentes/agent-09-revisor-qualidade.md`
  (genérico, fato inventado, cruzamento de lead, racional ausente, cor fora
  da paleta).
- **Comportamento:** NÃO é retry cego da mesma chamada. É o ciclo do
  agent-09 — reprovado volta pro agente responsável pelo achado
  especificamente, no máximo 2 ciclos automáticos (3ª reprovação escala pra
  fila humana de suporte/operações do webfy, ver §1 acima e
  `agent-09-revisor-qualidade.md`).
- **Diferença chave com 6.1:** falha transitória é "o modelo não respondeu
  direito" (causa técnica); falha de qualidade é "o modelo respondeu, mas o
  julgamento de conteúdo reprovou" (causa de critério). São orçamentos de
  tentativa separados — um nunca consome o outro.

### 6.3 Falha estrutural (schema inválido, campo obrigatório ausente)

- **O que é:** o agente respondeu, mas o output não bate com o
  schema/formato esperado (`01-contratos-de-dados/*.schema.json` pros
  campos finais, ou o `<output_format>` de cada `agent-XX.md` pros
  intermediários) — campo obrigatório ausente, tipo errado, enum inválido,
  JSON malformado. Esta é também a segunda camada de defesa contra prompt
  injection (`00-arquitetura/SEGURANCA-PROMPT-INJECTION.md §3`): output
  fora de schema é rejeitado antes de propagar pro próximo agente, seja a
  causa um bug de formatação comum ou uma tentativa de manipulação via
  texto de origem.
- **Comportamento:** trate como falha transitória **PARA FINS DE RETRY**
  (pode ser erro momentâneo do modelo produzindo saída malformada, sem
  relação com a qualidade do conteúdo em si) — mesmo teto de 3 tentativas
  com backoff de §6.1.
- **Diferença crítica:** se persistir após os retries, escala **DIRETO**
  pra fila humana — não entra no ciclo de 2-3 tentativas do gate de
  qualidade (agent-09). São categorias de problema diferentes que não
  deveriam competir pelo mesmo orçamento de tentativas: um agente que não
  consegue produzir JSON válido depois de 3 tentativas não vai
  magicamente acertar rodando mais 2-3 vezes pelo ciclo do agent-09 — esse
  ciclo foi desenhado pra corrigir JULGAMENTO de conteúdo, não formato.

### 6.4 Resumo — três orçamentos de tentativa independentes

| Categoria | Gatilho | Retry | Teto | Depois do teto |
|---|---|---|---|---|
| Transitória | timeout / rate limit / erro de rede | automático, backoff | 3 | `estado_geracao.erro` + estágio `falhou` |
| Qualidade | agent-09 reprova conteúdo | volta pro agente responsável pelo achado | 2 ciclos (3ª reprovação escala) | fila humana de suporte/operações webfy |
| Estrutural | schema inválido / campo obrigatório ausente | automático, backoff (mesmo tratamento de 6.1) | 3 | fila humana DIRETO — não passa pelo ciclo de qualidade |

Regra de ouro: falha transitória e falha estrutural competem pelo MESMO
orçamento de retry técnico (3 tentativas, backoff), porque as duas são "o
agente não produziu uma resposta utilizável" por motivo técnico. Falha de
qualidade tem orçamento PRÓPRIO (2-3 ciclos do agent-09), porque é um
problema de julgamento de conteúdo, não técnico — as duas contagens nunca
se somam nem se substituem uma pela outra.
