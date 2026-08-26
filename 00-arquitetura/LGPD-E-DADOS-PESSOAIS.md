# LGPD e dados pessoais — o uso de reviews de clientes reais como insumo de IA

> Foco específico: o campo `google_business_profile.reviews_amostra` de
> `01-contratos-de-dados/input-sinais-negocio.schema.json`. É o único ponto
> do pipeline onde texto escrito por uma pessoa física identificável (o
> cliente que deixou a review) vira insumo direto de IA — os outros campos
> são dados do NEGÓCIO (nome, categoria, endereço), não de pessoa física.

## 1. Por que isso é dado pessoal potencialmente sensível

O titular do dado aqui **não é o dono do negócio/lead** — é o cliente que
escreveu a review no Google. Esse é um ponto de confusão fácil: o pipeline
inteiro gira em torno do negócio, mas o texto de `reviews_amostra` foi
escrito por um terceiro que nunca deu consentimento explícito pro webfy
processar o texto dele com IA (deu consentimento, no máximo, pro Google
publicar a review).

Texto livre de review pode conter, mesmo sem querer:
- Nome do reviewer aparecendo dentro do próprio texto ("aqui é a Maria,
  adorei o atendimento").
- Menção a terceiros (filhos, familiares, outras pessoas presentes no
  atendimento).
- Situação sensível (queixa de saúde, num negócio de categoria clínica;
  situação financeira; reclamação que envolve conflito nomeado).

Ou seja: mesmo com o campo `autor` estruturalmente ausente do schema (ver
§2), o TEXTO em si ainda pode carregar dado pessoal identificável embutido.
Tratar como "anônimo só porque não tem campo de nome" seria falso senso de
segurança.

## 2. Regra de anonimização — antes de qualquer agente processar o texto

`input-sinais-negocio.schema.json` já reflete essa regra estruturalmente:
`reviews_amostra[].texto` e `.nota` são os únicos campos aceitos — **não
existe campo `autor` no schema**, de propósito. Isso significa:

- `agent-01-ingestor-sinais`, na normalização do dump bruto (Passo 2), deve
  **descartar qualquer nome/identificador de autor** presente no dump bruto
  da varredura antes de gravar `reviews_amostra` — nunca carregar esse
  campo adiante só porque a fonte original tinha.
- Nenhum agente posterior (`agent-02` em diante) recebe nome de reviewer em
  hipótese alguma — o contrato de schema já impede isso estruturalmente,
  mas a regra vale como princípio mesmo se o schema for estendido no
  futuro: nome de autor de review nunca entra no pipeline.
- Texto embutido com nome próprio de terceiro dentro do corpo da review
  (não no campo de autor, no texto mesmo) é harder de filtrar
  automaticamente — recomendação: `agent-01` ou uma etapa de pré-
  processamento aplica um filtro best-effort de nomes próprios antes de
  normalizar, mas isso não é garantia de anonimização perfeita, é mitigação
  razoável. Não travar o pipeline por causa disso; travar seria
  desproporcional a um texto público que o próprio reviewer publicou.

## 3. Prazo de retenção

| Dado | Retenção recomendada | Por quê |
|---|---|---|
| Dump bruto da varredura (inclui texto de review, possivelmente com nome de autor antes da normalização) | Curto — alinhado ao ciclo de vida do lead no pipeline (ex: 30-90 dias), configurável pelo webfy | É o dado mais sensível (mais completo, potencialmente com identificador). Não tem razão de negócio pra reter além do necessário pra reprocessar o lead em caso de erro. |
| `input-sinais-negocio` normalizado (já sem campo de autor) | Igual ao ciclo de vida do lead — mantido enquanto o lead estiver ativo no CRM, removido/anonimizado quando o lead for descartado (não convertido, marcado como perdido) | É o insumo rastreável do manual — precisa existir enquanto o manual puder ser regenerado, mas não precisa sobreviver ao lead. |
| `output-manual-marca` (o manual final entregue) | Mais longo — é o produto/deliverable, tem razão de negócio pra persistir mesmo depois do lead esfriar (histórico de venda, ver `00-arquitetura/VERSIONAMENTO.md`) | Não contém texto de review verbatim — `identidade.modulos[].conteudo` é síntese/paráfrase, não cópia direta do texto da review (ver constraint do agent-02 e exemplo em `03-template-manual-final/EXEMPLO-PREENCHIDO.md`, que parafraseia "fresquinho", "de vizinho" em vez de citar review completa). O risco de dado pessoal aqui é bem mais baixo, mas não zero — se um agente cometer erro e citar review verbatim, isso é achado de gate (`agent-09`, eixo fidelidade/especificidade cruzado com este documento). |

## 4. Direito ao esquecimento — se o dono do negócio pedir exclusão

Dois cenários possíveis, com resposta diferente:

**a) O dono do negócio pede que sinais/reviews usados sejam removidos do
processamento** (ex: "não quero que vocês usem os comentários dos meus
clientes pra gerar isso"): atender é razoável mesmo sem ser
tecnicamente o titular do dado pessoal da review — é decisão de produto/
relacionamento comercial, não obrigação estrita de LGPD nesse caso
específico, mas negar sem motivo forte prejudica a confiança na feature.
Ação: remover `reviews_amostra` do `input-sinais-negocio` daquele
`lead_id`, marcar `fonte_coleta` do dado removido, e disparar regeração
(completa ou pontual conforme `00-arquitetura/VERSIONAMENTO.md`, já que a
mudança de sinal pode invalidar múltiplos módulos que dependiam de
review).

**b) Um reviewer/pessoa física pede a exclusão do próprio texto** (cenário
menos provável de chegar até o webfy diretamente, mas juridicamente é o
titular de dado correto pra fazer esse pedido): é pedido de exclusão de
dado pessoal de fato, sob LGPD. Ação: remover o texto específico daquele
reviewer de `reviews_amostra` em qualquer `lead_id` onde apareça, do dump
bruto e do input normalizado; não é necessário regenerar o manual
retroativamente se o conteúdo já sintetizado (paráfrase, sem citação
verbatim) não identifica a pessoa — mas se o gate (`agent-09`) encontrar
alguma citação verbatim remanescente associável a essa pessoa, isso conta
como achado de fidelidade/isolamento a corrigir com prioridade.

Em ambos os casos: a exclusão do dado de origem não precisa,
necessariamente, apagar o manual JÁ ENTREGUE ao dono do negócio (é produto
comercial dele) — o que precisa é impedir que aquele texto específico seja
reutilizado numa regeração futura ou noutro processamento.

## 5. Onde isso se conecta

- `01-contratos-de-dados/input-sinais-negocio.schema.json` —
  `reviews_amostra` é o campo regido por este documento.
- `02-agentes/agent-01-ingestor-sinais.md` — ponto de normalização onde a
  anonimização precisa acontecer.
- `02-agentes/agent-02-descoberta-automatica.md` — consumidor do texto,
  regido pela regra de paráfrase (nunca citação verbatim identificável).
- `00-arquitetura/ESTRATEGIA-DE-TESTES.md` §3.2 — caso de teste de reviews
  ofensivas/spam é adjacente, mas cobre filtro de conteúdo, não anonimização
  de identidade; os dois são checagens diferentes sobre o mesmo campo.
