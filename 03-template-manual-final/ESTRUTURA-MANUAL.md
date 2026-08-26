# Estrutura do manual de marca — documento final entregue ao dono do negócio

> Isto é o TEMPLATE de seções, não o conteúdo em si (isso está em
> `EXEMPLO-PREENCHIDO.md`). Vale pros 3 formatos de saída: página web
> interativa na aba do webfy, PDF exportado, e link público — os 3 renderizam
> o mesmo conteúdo, só muda a casca.

## Tom esperado em TODAS as seções

Linguagem acessível pro **dono do negócio local**, não pro designer. Isso
significa: zero jargão sem explicação ao lado ("hairline" vira "linha fina";
"hex" vira "código da cor"), frases curtas, exemplos concretos do dia a dia
do negócio (fachada, cardápio, uniforme), tom prestativo e nunca condescendente.
Cada seção que recomenda algo também diz por quê, em 1-2 frases simples —
nunca "use esta cor" sem contexto.

## 1. Capa

- Nome do negócio + uma versão do logo (existente ou a alternativa
  escolhida) em destaque.
- Subtítulo: "Manual de marca — como usar sua identidade visual do jeito
  certo".
- Data de geração.

## 2. Sobre a marca

- Síntese em linguagem simples do posicionamento e personalidade inferidos
  (agent-03), reescrita para o dono do negócio entender de imediato "isso
  sou eu" — nunca um relatório técnico de inferência.
- Se módulos ficaram em `[LACUNA]` (ex: narrativa, tensão do fundador),
  seção curta e honesta: "essas partes a gente ainda não sabe sobre o seu
  negócio — conversa rápida com o vendedor completa isso."

## 3. Paleta de cores

- Cada cor com sua amostra visual, nome do papel (principal, destaque,
  neutro), código hex, e o racional em linguagem simples: por que essa cor
  faz sentido pra ESSE negócio (mecanismo → percepção → aplicação →
  recomendação do agent-10, traduzido pro dono).
- Se a paleta veio do logo existente, uma linha dizendo isso: "essas cores
  vieram direto da sua logo atual — só organizamos elas pra usar sempre do
  mesmo jeito."

## 4. Tipografia

- As 2 famílias (título/corpo) com exemplo visual do nome do negócio escrito
  nelas.
- Racional simples de por que essa combinação combina com o negócio.
- Regra prática: "sempre use estas duas fontes, nunca uma terceira" —
  explicado com exemplo do que acontece quando se mistura fontes demais
  (visual bagunçado, perde reconhecimento).

## 5. Logo e regras de uso

**Se tem logo existente:**
- Logo atual em destaque, com regras de uso (espaço ao redor, tamanho
  mínimo, fundos onde funciona, o que não fazer) em linguagem simples.
- Seção separada, claramente opcional: "3 opções novas, caso queira trocar
  no futuro" — cada uma com o conceito explicado, sem pressão de troca.

**Se não tem logo existente:**
- As 3 opções apresentadas como escolha inicial, cada uma com o conceito em
  1-2 frases e por que combina com o negócio.
- Espaço claro pra indicar "qual dessas você prefere" (campo de decisão que
  a aba UI captura — ver `04-integracao-webfy/ESPECIFICACAO-ABA-UI.md`).

**Em ambos os casos, nota obrigatória junto das 3 opções geradas:**
**[PENDENTE-JURIDICO]** a titularidade dos direitos autorais sobre um logo
gerado por IA ainda está em análise jurídica — o manual não deve afirmar
que o direito autoral pertence automaticamente ao dono do negócio até essa
definição estar fechada (ver `04-integracao-webfy/SERVICO-GERACAO-DE-
IMAGEM.md §7`). Linguagem sugerida pro dono do negócio, sem alarmar:
"esse logo foi gerado por inteligência artificial — a parte de direito
autoral sobre esse tipo de imagem ainda está em definição no mercado, e
vamos te avisar assim que tivermos uma posição clara sobre isso."

**Segunda nota obrigatória, junto da anterior:** aviso de risco de colisão
com marca registrada de terceiro (`02-agentes/agent-07-logo-alternativas.md`,
constraint correspondente) — não é bloqueio, é alerta. Linguagem sugerida:
"essas opções não passaram por uma busca de anterioridade de marca —
recomendamos checar isso antes de registrar ou usar comercialmente de
forma definitiva, principalmente se você pretende entrar com pedido de
franquia ou tiver receio de disputa de marca."

## 6. Aplicações práticas

Uma subseção por contexto relevante à categoria do negócio, cada uma com
uma prévia visual e 2-3 frases de orientação prática. Specs mínimas por
peça (o compositor aplica a que for relevante à categoria do negócio —
uma clínica não recebe subseção de embalagem, por exemplo):

- **Cartão de visita** — proporção padrão 9cm x 5cm (paisagem, padrão de
  gráfica no Brasil). Logo no canto superior esquerdo ou centralizado no
  topo (nunca esticada pra preencher espaço — ver regra de uso do logo,
  §5); nome do negócio em destaque com a fonte de título; paleta principal
  como cor de fundo ou faixa, nunca as 4 cores da paleta competindo ao
  mesmo tempo no mesmo cartão.
- **Post de rede social** — duas proporções cobertas: quadrada (1:1, feed)
  e vertical (9:16, story/reel). Área segura pra texto: manter margem
  interna de pelo menos 8% da largura/altura em todos os lados livre de
  texto ou elementos importantes da logo, porque o Instagram e outras
  plataformas cortam a prévia e sobrepõem UI própria (ícones, avatar,
  legenda) nas bordas — texto colado na borda vira ilegível ou cortado.
- **Fachada/placa** (se negócio físico) — aqui o requisito não é o
  contraste AA 4.5:1 de tela (`ACESSIBILIDADE.md`, que vale pra web/PDF);
  é legibilidade física a distância. Orientação prática: contraste forte
  entre fundo e texto da placa (cor escura da paleta contra cor clara, ou
  vice-versa — nunca duas cores próximas em luminosidade), letra grossa o
  suficiente pra ler de dentro de um carro ou da calçada oposta, sem
  ornamento ou fonte fina demais que sinta bem em tela mas suma de longe.
  Não é uma medida numérica única (depende da distância real de leitura
  do local) — é orientação qualitativa pro dono do negócio levar pro
  fornecedor da placa.
- **Uniforme/aplicação em tecido** (se atendimento presencial) — a paleta
  definida em hex é pra tela; tecido, bordado e tinta serigráfica nem
  sempre reproduzem a cor exata. Orientação explícita no manual: "a cor
  que você vê aqui é a referência digital — quem for produzir a peça
  física (gráfica, bordadeira, serigrafia) deve aproximar pra uma cor
  Pantone equivalente; pequena variação de tom entre tela e tecido é
  normal e não é erro do fornecedor." Essa responsabilidade de conversão
  pra Pantone é de quem produz a peça física, não do pipeline — o manual
  não gera código Pantone, só avisa que a conversão é necessária.
- **Embalagem** (se produto) — logo em tamanho legível mesmo em embalagem
  pequena (respeitar o tamanho mínimo definido em §5), paleta aplicada de
  forma consistente com o resto do material pra reforçar reconhecimento
  na prateleira/balcão.

## 7. Anti-padrões — o que evitar

Lista curta e concreta (3-6 itens) do que NÃO fazer com a marca, específica
pro negócio — nunca uma lista genérica de designer. Exemplo de tom: "Não
estique a logo pra caber num espaço — prefira deixar ela menor mas
proporcional" em vez de "mantenha a proporção do logotipo".

## 8. Rodapé / próximos passos

- Se há lacunas abertas: convite claro pro dono conversar com o vendedor
  pra completar as partes que faltam.
- Botões/CTAs equivalentes aos da aba UI: exportar PDF, ver versão
  compartilhável, "gostei, seguir com essa marca".
