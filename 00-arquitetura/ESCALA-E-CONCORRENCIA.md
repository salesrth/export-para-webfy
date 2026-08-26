# Escala e concorrência — o pipeline sob volume alto de leads

> A webfy varre a web em volume — potencialmente muitos negócios
> capturados ao mesmo tempo, cada um podendo disparar o pipeline (ver
> `04-integracao-webfy/INTEGRACAO-CRM.md §1`, Opção A automática). Este
> documento cobre como o pipeline se comporta quando isso acontece de
> verdade, não no caso de 1 lead isolado que os outros documentos
> descrevem. Três frentes: fila de processamento, corrida entre vendedores
> no mesmo lead, e limite de geração simultânea de imagem.

## 1. Fila de processamento — assíncrono, não tempo real garantido

`ARQUITETURA-AGENTES.md §5` estima 2 a 5 minutos end-to-end por lead — essa
estimativa vale pra um lead processando sozinho, sem fila. Sob volume alto
(muitos leads capturados na mesma janela de tempo), o pipeline processa em
fila, não em paralelo ilimitado:

- **O disparo entra numa fila, não roda instantaneamente.** `estagio_atual`
  pode ficar em `aguardando_sinais`/`sinais_coletados` por mais tempo que o
  normal simplesmente porque há fila à frente, não porque algo falhou — a
  UI (`04-integracao-webfy/ESPECIFICACAO-ABA-UI.md §2.2`) já é desenhada
  pra isso: não bloqueia navegação, mostra progresso lendo `estagio_atual`,
  e o texto "normalmente pronto em 2 a 5 minutos" deve virar "pode levar
  mais tempo em horários de pico" quando o backend souber que há fila
  grande — não é uma promessa de tempo fixo independente de carga.
  Implementação exata do texto dinâmico é decisão do webfy; o contrato
  aqui é só que a UI não deve mentir sobre tempo fixo sob fila real.
- **Priorização não é decidida por este blueprint.** Se o webfy quiser
  priorizar (ex: lead que o vendedor já abriu na aba processa antes de
  lead que ainda não foi visto por ninguém), isso é política de fila do
  lado da implementação — este documento não crava ordem de prioridade,
  só estabelece que fila existe e a UI precisa refletir isso honestamente.
- **Falha de fila não é falha de pipeline.** Um lead "travado" há muito
  tempo em `aguardando_sinais` pode ser fila normal sob carga alta ou pode
  ser erro real — a diferença não é visível só olhando `estagio_atual`.
  Recomendação: expor também uma métrica de profundidade de fila (quantos
  leads aguardando) na observabilidade operacional do webfy (fora do
  escopo de `OBSERVABILIDADE.md`, que cobre métrica de PRODUTO/qualidade,
  não capacidade de infraestrutura) — sem isso, "lento" e "quebrado" ficam
  indistinguíveis pra quem opera.

## 2. Dois vendedores disparando o mesmo lead ao mesmo tempo

Times de vendas compartilham carteira de leads às vezes — dois vendedores
da mesma equipe podem abrir o mesmo lead e ambos clicarem "Gerar manual de
marca" quase ao mesmo tempo (Opção B), ou um clicar enquanto o disparo
automático (Opção A) já está em curso pro mesmo `lead_id`.

- **Regra dura: 1 pipeline em execução por `lead_id` por vez.** Antes de
  iniciar uma nova execução completa (agent-01 até agent-09), o
  orquestrador confere `estagio_atual` do estado de geração
  (`01-contratos-de-dados/estado-geracao.schema.json`). Se já está em
  qualquer estágio "em curso" (`sinais_coletados` até `em_revisao_
  qualidade`), o novo disparo **não inicia uma segunda execução** — ele
  só reflete/observa o estado que já está rodando. É o mesmo princípio de
  lock que evita corrida de escrita, aplicado aqui pra evitar corrida de
  processamento.
- **O que os dois vendedores veem:** ambos veem a mesma tela de progresso
  (§2.2 da UI) apontando pro mesmo `estagio_atual` — não duas barras de
  progresso divergentes, não dois manuais gerados em paralelo pro mesmo
  negócio. Quando o pipeline termina, os dois veem o mesmo resultado
  (`pronto`), sem duplicação.
- **Exceção que NÃO é corrida, é ação legítima:** um vendedor clicando
  "Regenerar logo" (`ESPECIFICACAO-ABA-UI.md §2.3`) enquanto outro só está
  olhando o manual pronto não é corrida — é ação humana explícita sobre um
  manual já `pronto`, coberta pela regra de teto de custo
  (`SERVICO-GERACAO-DE-IMAGEM.md §6`), não pela regra de lock deste
  documento (que é sobre 2 pipelines completos concorrentes, não sobre 2
  cliques em botões diferentes).
- **Implementação do lock:** este blueprint não prescreve o mecanismo
  exato (lock otimista por versão, lock pessimista por linha, fila com
  chave única em `lead_id`) — é decisão de stack do webfy. O contrato é
  comportamental: nunca 2 execuções completas do pipeline rodando ao mesmo
  tempo pro mesmo `lead_id`, ponto.

## 3. Limite de geração simultânea de imagem (throughput, não custo)

`SERVICO-GERACAO-DE-IMAGEM.md §6` já define o teto de custo — no máximo 1
geração automática completa (3 opções) por `lead_id`. Isso limita QUANTO
se gasta por lead, mas não resolve um problema diferente: QUANTAS chamadas
de geração de imagem o provedor externo aguenta receber **ao mesmo tempo**
quando muitos leads batem no Passo 7 (`CHRONOLOGIA.md`) na mesma janela.

- **Throughput é capacidade, não valor gasto.** Mesmo respeitando o teto
  de custo por lead à risca, 500 leads capturados na mesma hora geram até
  1500 chamadas de imagem (3 por lead) tentando disparar próximas do mesmo
  momento — a maioria dos provedores de API de geração de imagem tem
  limite de requisições simultâneas (rate limit), e estourar isso não é
  "gastar mais", é começar a receber erro/timeout em massa.
- **Concorrência controlada, não disparo livre.** A chamada ao serviço de
  geração de imagem (`SERVICO-GERACAO-DE-IMAGEM.md §2-4`) deve passar por
  um limitador de concorrência do lado do orquestrador do webfy — um teto
  configurável de quantas chamadas de imagem podem estar em voo ao mesmo
  tempo, com o excedente entrando em fila (não descartado, não retry
  imediato agressivo). As 3 opções de um mesmo lead continuam podendo
  rodar em paralelo entre si (`SERVICO-GERACAO-DE-IMAGEM.md §4`); o
  limite aqui é sobre quantos LEADS DIFERENTES têm chamadas de imagem em
  voo ao mesmo tempo, não sobre paralelismo dentro de 1 lead.
- **Timeout já configurado ajuda, mas não resolve throughput sozinho.** O
  timeout de 60-90s por imagem (`SERVICO-GERACAO-DE-IMAGEM.md §4`) evita
  que 1 chamada lenta trave o pipeline daquele lead — mas não evita que
  1000 chamadas simultâneas estourem o rate limit do provedor todas ao
  mesmo tempo. São mecanismos complementares: timeout protege 1 lead,
  limitador de concorrência protege o provedor (e por extensão, todos os
  leads na fila).
- **Sinal de estouro deveria aparecer na observabilidade.** Se o
  limitador de concorrência estiver constantemente no teto (fila de
  imagem sempre cheia), isso é sinal de que o volume de leads superou a
  capacidade contratada com o provedor — informação operacional pro
  webfy decidir se aumenta o teto de concorrência (custo maior) ou aceita
  fila mais longa pra geração de imagem especificamente (ver §1 deste
  documento, mesma lógica de fila honesta aplicada aqui).

## 4. Onde isso se conecta

- `00-arquitetura/ARQUITETURA-AGENTES.md §5` — estimativa de tempo por
  lead isolado, base pra entender o que a fila (§1 aqui) adiciona em cima.
- `01-contratos-de-dados/estado-geracao.schema.json` — `estagio_atual` é o
  campo que o lock de §2 confere antes de permitir novo disparo.
- `04-integracao-webfy/SERVICO-GERACAO-DE-IMAGEM.md §4 e §6` — requisitos
  não-funcionais (timeout, paralelismo das 3 opções) e teto de custo por
  lead; este documento adiciona a camada de throughput entre leads
  diferentes que aqueles parágrafos não cobrem.
- `04-integracao-webfy/ESPECIFICACAO-ABA-UI.md §2.2` — tela de progresso
  que precisa refletir fila real, não tempo fixo, quando há carga alta.
