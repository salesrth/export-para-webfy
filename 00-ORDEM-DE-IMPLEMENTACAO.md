# Ordem de implementação — o que construir primeiro

> Este pacote cresceu pra ~30 arquivos, do pipeline central até LGPD,
> acessibilidade, escala e billing. Um dev abrindo isso pela primeira vez
> não precisa implementar tudo antes de ter algo demonstrável. Este arquivo
> categoriza em 3 níveis: o que faz o produto EXISTIR (P0), o que importa
> antes de escalar de verdade mas não trava um piloto pequeno (P1), e o que
> é hardening pra depois do lançamento inicial (P2).
>
> **Honestidade sobre a categorização:** nem tudo cabe limpo numa caixa só.
> Onde uma peça tem parte P0 e parte P2 dentro do MESMO documento (é o caso
> da segurança de prompt injection), isso está dito explicitamente abaixo
> em vez de forçado numa categoria única.

## P0 — sem isso não existe produto

O mínimo pra um piloto pequeno rodar de ponta a ponta e gerar um manual
real, navegável, pra um vendedor mostrar a um dono de negócio.

- **`02-agentes/agent-01-ingestor-sinais.md` até `agent-10-explicador-
  didatico.md`** — os 10 agentes do pipeline central (fases 1 a 4 de
  `00-arquitetura/ARQUITETURA-AGENTES.md §1`). `agent-11-regenerador-
  modulo.md` fica de fora daqui de propósito — é regeração pontual pós-
  entrega, não faz parte do caminho de geração inicial (ver P1).
- **Os 3 schemas de dados** — `01-contratos-de-dados/input-sinais-negocio
  .schema.json`, `output-manual-marca.schema.json`, `estado-geracao
  .schema.json`. Sem contrato de dados fechado, os 10 agentes não têm como
  se falar de forma confiável.
- **A lógica de logo completa** — `agent-06-logo-guardiao.md` +
  `agent-07-logo-alternativas.md`, incluindo o teto de custo de
  `04-integracao-webfy/SERVICO-GERACAO-DE-IMAGEM.md §6` (no máximo 1
  geração automática completa por `lead_id`). Isso não é opcional nem pra
  MVP: sem o teto, o primeiro piloto já sai com risco de custo de API sem
  controle.
- **O gate de qualidade básico** — os 5 eixos de `agent-09-revisor-
  qualidade.md` (especificidade, fidelidade factual, isolamento entre
  leads, racional causal completo, e a VERSÃO BÁSICA do 5º eixo — ausência
  de claim óbvio/desproporcional tipo "o melhor da cidade" sem fonte
  rastreável) e o ciclo de correção/escalonamento de `ARQUITETURA-AGENTES
  .md §1`. Nenhum manual sai sem gate — isso é P0 mesmo num piloto de 1
  lead. O agente é um prompt único com os 5 eixos embutidos: não há como
  implementar "só 4" sem editar o prompt, então na prática o 5º eixo
  entra junto desde o primeiro dia — só o REFINAMENTO dele (detecção
  sofisticada de padrão sutil de injeção) é que fica em P2, não a
  checagem básica (ver `SEGURANCA-PROMPT-INJECTION.md` abaixo, nota de
  categorização mista).
- **Semântica básica de erro** — `ARQUITETURA-AGENTES.md §6`: retry com
  backoff pra falha transitória/estrutural (teto de 3) e o fato de que
  falha de qualidade usa um orçamento de tentativa separado do gate. Sem
  isso, o primeiro rate limit do provedor de LLM já derruba o piloto.
- **Os estados 2.1 a 2.4 da aba UI** — `04-integracao-webfy/
  ESPECIFICACAO-ABA-UI.md §2.1-2.4` (antes de gerar, gerando/progresso,
  pronto, erro/lacunas pendentes). O estado 2.0 (upsell de entitlement)
  fica em P1 — um piloto interno pode simplesmente assumir que o add-on já
  está ativo pra quem está testando.
- **O fluxo de PIN de acesso** — `ESPECIFICACAO-ABA-UI.md §5` completo
  (geração, tela de acesso, teto de 5 tentativas, reset) e os campos
  correspondentes de `estado-geracao.schema.json` (`pin_acesso`,
  `pin_tentativas_falhas`, `pin_bloqueado`). O link público sem PIN expõe o
  manual de qualquer negócio pra qualquer pessoa com a URL — não é algo pra
  adiar nem no piloto menor.
- **Mitigação básica de prompt injection** — `00-arquitetura/
  SEGURANCA-PROMPT-INJECTION.md §2` (delimitação dado-vs-instrução no
  prompt) e `§3` (validação estrutural de saída rejeitando output fora de
  schema). Ver nota de categorização mista abaixo — só a base é P0, não o
  documento inteiro.

## P1 — importante antes de escalar de verdade, não trava um piloto pequeno

Coisas que um piloto com poucos vendedores/leads pode viver sem por um
tempo, mas que viram bloqueador assim que o produto sai do teste fechado.

- **Entitlement/paywall** — `04-integracao-webfy/INTEGRACAO-CRM.md §0` (gate
  de add-on antes do Passo 2 do pipeline) e o estado 2.0 da UI
  (`ESPECIFICACAO-ABA-UI.md §2.0`). Num piloto fechado com contas
  conhecidas, dá pra assumir entitlement ativo manualmente; em produção
  real, cada disparo sem esse gate é custo de processamento vazando pra
  conta sem direito de uso.
- **White-label completo** — `ESPECIFICACAO-ABA-UI.md §6` (`nome_exibicao_
  vendedor` em tela de PIN, manual web, PDF, email automático). Um piloto
  interno pode tolerar branding neutro genérico por um tempo; venda real
  pra agências/vendedores que revendem o produto não tolera.
- **Versionamento** — `00-arquitetura/VERSIONAMENTO.md` + `agent-11-
  regenerador-modulo.md`. Regeração pontual pós-entrega (troca de logo,
  nova micro-conversa preenchendo lacuna) importa quando o produto já tem
  manuais entregues circulando, não no primeiro lote gerado.
- **Observabilidade/métricas** — `00-arquitetura/OBSERVABILIDADE.md`
  (taxa de aprovação do gate, agregados por categoria de negócio). Sem
  volume, não há sinal agregado útil pra olhar — vira relevante quando o
  pipeline já processa leads o suficiente pra enxergar tendência.
- **LGPD** — `00-arquitetura/LGPD-E-DADOS-PESSOAIS.md` (tratamento de
  `reviews_amostra` como dado pessoal de terceiro). Categorizado P1 e não
  P0 apenas pela lente de "o que impede um piloto fechado de rodar" — isso
  **não** é uma recomendação de adiar a decisão jurídica de verdade; é
  sobre ordem de implementação técnica, não sobre risco legal, que corre em
  paralelo independente desta lista.
- **Escala/concorrência** — `00-arquitetura/ESCALA-E-CONCORRENCIA.md` (fila
  de processamento, lock de 1 pipeline por `lead_id`, limitador de
  concorrência de geração de imagem). Um piloto de poucos leads simultâneos
  não sente a ausência disso; volume real sente rápido.

## P2 — hardening, pode vir depois do lançamento inicial

- **Estratégia de testes formal (evals)** — `00-arquitetura/
  ESTRATEGIA-DE-TESTES.md` completa (bateria de casos por agente,
  regressão bloqueando deploy). O piloto inicial roda sem isso; assim que
  o produto começa a iterar prompt em produção com volume real, a ausência
  vira risco de regressão silenciosa.
- **Segurança de prompt injection completa** — `00-arquitetura/
  SEGURANCA-PROMPT-INJECTION.md`, especificamente o eixo completo de
  detecção sofisticada no gate de qualidade (reconhecer padrão sutil de
  linguagem injetada, não só claim óbvio) e a bateria formal de casos de
  teste de ataque (`ESTRATEGIA-DE-TESTES.md`, lacuna já registrada lá — não
  existe hoje nenhum dos 5 casos obrigatórios mínimos dedicado a isso).
  **Nota de categorização mista:** a base desta proteção (delimitação
  dado-vs-instrução + validação estrutural, §2-3 do documento) é P0 — sem
  isso o pipeline aceita instrução de estranho como comando desde o
  primeiro lead processado. O que é P2 é o refinamento: detecção mais
  sofisticada de padrão sutil e a cobertura formal de teste contra técnicas
  de ataque variadas.
- **Acessibilidade AA completa** — `03-template-manual-final/
  ACESSIBILIDADE.md` (contraste 4.5:1 em toda combinação exibida, tela de
  PIN incluída). Um piloto pode sair no ar com um nível de rigor menor;
  produção real vendida como manual profissional não deveria.
- **Aplicações práticas detalhadas** — `03-template-manual-final/
  ESTRUTURA-MANUAL.md §6` (specs completas de cartão de visita, post,
  fachada, uniforme, por categoria de negócio). O manual pode nascer só com
  paleta + logo + tipografia (`ESTRUTURA-MANUAL.md §3-5`) e ainda ser um
  produto entregável — a seção de aplicações é o que separa "manual básico"
  de "manual completo", mas não é o que separa "existe" de "não existe".

## Onde isso se conecta

- `README.md` — árvore de arquivos completa do pacote.
- `CHRONOLOGIA.md` — o passo a passo cronológico que os itens P0 do
  pipeline central implementam concretamente.
- `00-arquitetura/ARQUITETURA-AGENTES.md` — ordem/paralelismo dos 10
  agentes P0 e a semântica de erro (§6) que também é P0.
