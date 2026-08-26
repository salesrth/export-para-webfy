# Export pra webfy — geração automática de manual de marca

> **Isto é blueprint + schemas + prompts de agente + spec de UI. Não é
> código pronto pra rodar.** Um dev (ou uma sessão Claude Code dentro do
> repo do webfy) usa este pacote pra implementar a feature no stack real da
> plataforma — este pacote não conhece esse stack e não deveria conhecer.
>
> **Próximo passo de leitura recomendado depois deste README:**
> `00-ORDEM-DE-IMPLEMENTACAO.md` — o que implementar primeiro (P0) pra ter
> um MVP funcional, o que espera escala de verdade (P1), e o que é
> hardening pra depois do lançamento (P2).

## O que é isto

A BNP desenvolveu, pra uso interno, um método de criar manual de marca
(entrevista estruturada em 8 módulos + design system com racional causal
por escolha visual) e um jeito de orquestrar agentes de IA especializados
pra produzir e auditar esse conteúdo. Este pacote adapta esse método pra
rodar **sem entrevista humana**, em escala, dentro de outra plataforma
("webfy"): uma plataforma que já varre a web achando negócios locais sem
site, gera site automático pra eles, e vende esse site através de um
vendedor. A adição aqui é uma aba nova que gera automaticamente um manual
de marca completo pra cada negócio encontrado, como argumento extra de
venda.

## Pra quem serve

Dev do webfy que vai implementar a feature, ou uma sessão de IA de
desenvolvimento trabalhando dentro do repo do webfy com acesso ao stack
real da plataforma.

## Estrutura do pacote

```
export-para-webfy/
├── README.md
├── 00-ORDEM-DE-IMPLEMENTACAO.md
├── CHRONOLOGIA.md
├── 00-arquitetura/
│   ├── ARQUITETURA-AGENTES.md
│   ├── META-ARQUIVOS.md
│   ├── VERSIONAMENTO.md
│   ├── OBSERVABILIDADE.md
│   ├── ESTRATEGIA-DE-TESTES.md
│   ├── LGPD-E-DADOS-PESSOAIS.md
│   ├── ESCALA-E-CONCORRENCIA.md
│   └── SEGURANCA-PROMPT-INJECTION.md
├── 01-contratos-de-dados/
│   ├── input-sinais-negocio.schema.json
│   ├── output-manual-marca.schema.json
│   └── estado-geracao.schema.json
├── 02-agentes/
│   ├── agent-01-ingestor-sinais.md
│   ├── agent-02-descoberta-automatica.md
│   ├── agent-03-posicionamento.md
│   ├── agent-04-sistema-cor.md
│   ├── agent-05-tipografia-e-composicao.md
│   ├── agent-06-logo-guardiao.md
│   ├── agent-07-logo-alternativas.md
│   ├── agent-08-compositor-manual.md
│   ├── agent-09-revisor-qualidade.md
│   ├── agent-10-explicador-didatico.md
│   └── agent-11-regenerador-modulo.md
├── 03-template-manual-final/
│   ├── ESTRUTURA-MANUAL.md
│   ├── EXEMPLO-PREENCHIDO.md
│   └── ACESSIBILIDADE.md
└── 04-integracao-webfy/
    ├── ESPECIFICACAO-ABA-UI.md
    ├── INTEGRACAO-CRM.md
    └── SERVICO-GERACAO-DE-IMAGEM.md
```

## Como navegar o pacote

- **`00-ORDEM-DE-IMPLEMENTACAO.md`** — antes de implementar qualquer coisa,
  leia isto. Categoriza todo o pacote em P0 (sem isso não existe produto),
  P1 (importante antes de escalar, não trava um piloto pequeno) e P2
  (hardening pra depois do lançamento).
- **`CHRONOLOGIA.md`** — o documento central. Passo a passo cronológico
  completo, do lead sem site até o manual entregue, com agente/tempo/
  bifurcação lógica-de-logo em cada passo, e um diagrama do fluxo inteiro.
  Comece por aqui pra entender o todo antes de mergulhar nas peças.
- **`00-arquitetura/`** — como os 10 agentes se orquestram entre si (ordem,
  paralelismo, gates de qualidade, e a semântica de erro/retry por tipo de
  falha em `ARQUITETURA-AGENTES.md §6`) e como funciona o meta-arquivo de
  estado por lead (equivalente ao `.agent/state.json` da BNP, mas por
  negócio em vez de por task de projeto). Também cobre versionamento
  pós-entrega (`VERSIONAMENTO.md`), métricas do pipeline e o loop de
  feedback do vendedor (`OBSERVABILIDADE.md`), estratégia de testes de
  agente (`ESTRATEGIA-DE-TESTES.md`), o tratamento de dado pessoal em
  reviews de cliente (`LGPD-E-DADOS-PESSOAIS.md`), o comportamento do
  pipeline sob volume alto de leads — fila de processamento, corrida entre
  vendedores no mesmo lead, e throughput de geração de imagem
  (`ESCALA-E-CONCORRENCIA.md`) — e a defesa em camadas contra texto
  adversarial vindo dos sinais coletados na web aberta
  (`SEGURANCA-PROMPT-INJECTION.md`).
- **`01-contratos-de-dados/`** — os 3 JSON Schemas que definem exatamente o
  que entra (sinais coletados do negócio), o que sai (manual de marca
  gerado) e como o progresso é rastreado (estado de geração por lead,
  incluindo o PIN de acesso ao link público e o histórico de versões).
- **`02-agentes/`** — os 11 prompts de agente, um arquivo por agente, no
  formato exato do padrão `.claude/agents/*.md` da BNP (frontmatter +
  tags `<role>`/`<execution>`/`<output_format>`/`<constraints>`). Cobrem
  desde a ingestão de sinais até o gate final de qualidade e a regeração
  pontual de módulo (agent-11) — com atenção especial aos agentes 06 e 07,
  que resolvem a lógica central de "o negócio já tem logo ou não".
- **`03-template-manual-final/`** — a estrutura de seções do documento
  final entregue ao dono do negócio, um exemplo completo e fictício
  (uma padaria com logo simples já existente) que serve de referência de
  tom e qualidade, e os requisitos de acessibilidade (`ACESSIBILIDADE.md`)
  que valem pra página web e pro PDF exportado.
- **`04-integracao-webfy/`** — as 3 pontas que conectam o pipeline à
  plataforma real: a especificação de UX da aba nova (incluindo o fluxo de
  PIN de acesso ao link público), quando/como o CRM registra a geração, e a
  interface pluggable pro serviço externo de geração de imagem (sem cravar
  qual API usar, mas já com o teto de custo por lead e a pendência jurídica
  de direitos autorais).

## Decisões que este pacote NÃO re-decide

Já vieram cravadas no pedido original e são tratadas aqui como dado de
entrada, não como escolha em aberto: sem entrevista humana ao vivo (troca
por inferência automática com score de confiança por módulo); lógica de
"logo existente vira âncora, logo novo é sempre gerado em 3 opções, muda só
o enquadramento"; formato de saída (página web navegável + PDF + link
público); a existência de um gate de qualidade equivalente ao
`bnp-logic-auditor`/`bnp-brand-reviewer` da BNP, adaptado ao domínio; a
feature ser add-on pago/premium do webfy, com gate de entitlement antes do
disparo (`04-integracao-webfy/INTEGRACAO-CRM.md §0`); e white-label total
pro dono do negócio final — nenhuma marca "webfy" visível em PIN/manual/PDF/
mensagem automática, substituível por `nome_exibicao_vendedor`
(`04-integracao-webfy/ESPECIFICACAO-ABA-UI.md §6`).

**Idioma — decisão consciente, não esquecimento:** este blueprint assume
PT-BR fixo em todo o pipeline (sinais, prompts de agente, manual gerado,
UI). Nenhum schema tem campo de idioma nesta versão. Se o webfy expandir
pra mercado não lusófono no futuro, isso exige revisão explícita deste
pacote inteiro (prompts de agente, textos de UI, formato de data/moeda) —
não é um campo isolado a adicionar depois.

## O que fundamenta o racional causal de cada elemento gráfico

Psicologia de design genérica e bem estabelecida — psicologia da cor,
hierarquia tipográfica, Gestalt, teoria de affordance, custo
preditivo/previsibilidade de padrão, contraste e legibilidade. Nunca o
conteúdo do livro proprietário "Design Lógico" da BNP, que não pertence ao
produto webfy e não foi consultado pra escrever este pacote.
