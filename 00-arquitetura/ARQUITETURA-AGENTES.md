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
