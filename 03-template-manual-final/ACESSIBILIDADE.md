# Acessibilidade — requisitos pra página web e PDF do manual

> Vale pros 2 formatos que carregam o mesmo conteúdo (página web interativa
> na aba do webfy/link público, e PDF exportado — ver
> `03-template-manual-final/ESTRUTURA-MANUAL.md`). O link público, além
> disso, agora passa por uma tela de PIN antes do conteúdo
> (`04-integracao-webfy/ESPECIFICACAO-ABA-UI.md §5`) — essa tela também
> precisa cumprir os requisitos abaixo, não só o manual em si.

## 1. Contraste mínimo — AA 4.5:1 pro texto sobre as cores recomendadas

Toda combinação de texto sobre fundo que o manual EXIBE (não só recomenda
pro negócio usar depois) precisa manter contraste mínimo **4.5:1** pra
texto normal e **3:1** pra texto grande (≥18pt ou ≥14pt bold), conforme
WCAG 2.1 AA. Isso já é parte do critério que `agent-04-sistema-cor` usa pra
derivar neutros de texto/fundo (`02-agentes/agent-04-sistema-cor.md`,
passo 2) — este documento estende a mesma régua pra própria interface que
RENDERIZA o manual, não só pro conteúdo recomendado:

- A tabela de paleta (nome do papel + amostra + hex) precisa ser legível
  mesmo se a cor de fundo do negócio for escura — não assuma fundo claro
  fixo da própria UI.
- Texto de racional (mecanismo/percepção/aplicação/recomendação) nunca
  renderiza em cima da cor de amostra diretamente — sempre em área neutra
  da própria interface, separada da amostra de cor.

## 2. Tamanho mínimo de fonte

- **Web:** corpo de texto nunca menor que 16px equivalente (1rem com base
  padrão do navegador); texto de apoio/legenda não abaixo de 14px. Título
  de seção escalável, nunca fixo em px que ignore zoom do usuário.
- **PDF:** corpo nunca menor que 10pt; preferencialmente 11-12pt pra leitura
  confortável em tela pequena (o dono do negócio provavelmente abre o PDF
  no celular). Título de seção pelo menos 16pt.
- Nenhum dos dois formatos usa tamanho de fonte fixo que impeça o usuário
  de aumentar zoom (web) ou usar leitor de tela com reflow (PDF tagueado).

## 3. Texto alternativo nas imagens de logo

Toda imagem de logo — o existente (se houver) e as 3 alternativas geradas
— precisa de texto alternativo descritivo, nunca `alt=""` nem `alt="logo"`
genérico:

- **Logo existente:** alt descreve o que está visualmente presente (ex:
  "Logo atual da Padaria Bom Trigo: círculo marrom com as letras BT
  entrelaçadas").
- **Cada alternativa gerada:** alt inclui o conceito por trás dela, não só
  a descrição visual crua (ex: "Opção de logo 2: símbolo de espiga de trigo
  ao lado do monograma BT, mesma paleta marrom e trigo-claro" — não apenas
  "imagem gerada por IA").
- Texto alternativo é responsabilidade de quem compõe o manual final
  (`agent-08-compositor-manual`, a partir do `conceito` já produzido pelo
  `agent-06`/`agent-07`) — não é um campo à parte a inventar do zero,
  reaproveita o racional que já existe.

## 4. Navegação por teclado na versão web

- Toda ação disponível por clique (navegar entre seções, exportar PDF,
  copiar link, regenerar logo, marcar como enviado) precisa ter equivalente
  acessível por teclado (Tab/Shift+Tab pra navegar, Enter/Espaço pra
  ativar), com indicador de foco visível (nunca `outline: none` sem
  substituto visual).
- Navegação entre seções do manual (capa, sobre a marca, paleta,
  tipografia, logo, aplicações, anti-padrões, próximos passos — ver
  `ESTRUTURA-MANUAL.md`) segue ordem lógica de leitura no DOM/tab order,
  igual à ordem visual.
- **Tela de PIN** (`04-integracao-webfy/ESPECIFICACAO-ABA-UI.md §5`): campo
  de PIN recebe foco automático ao carregar a página, erro de PIN incorreto
  é anunciado via região `aria-live` (não só mudança visual de cor), e o
  botão "Acessar manual" é alcançável e ativável só por teclado, sem
  depender de mouse/touch.
- Marcação semântica mínima: landmarks (`<nav>`, `<main>`, seções com
  heading hierárquico correto — nunca pular de `h1` pra `h3`), pra quem usa
  leitor de tela navegar por seção sem precisar ler linearmente o documento
  inteiro.

## 5. PDF exportado — requisitos equivalentes

O PDF gerado a partir do mesmo conteúdo precisa manter: texto real
selecionável (nunca a página inteira como imagem rasterizada), ordem de
leitura tagueada correspondendo à ordem visual, e os textos alternativos do
§3 preservados como alt text de PDF acessível (PDF/UA), não perdidos na
exportação.

## 6. Onde isso se conecta

- `02-agentes/agent-04-sistema-cor.md` — já aplica AA 4.5:1 na decisão da
  paleta; este documento garante que a mesma régua vale pra interface que
  renderiza, não só pro conteúdo recomendado.
- `02-agentes/agent-08-compositor-manual.md` — ponto onde o texto
  alternativo das imagens de logo é composto a partir do racional/conceito
  já existente.
- `04-integracao-webfy/ESPECIFICACAO-ABA-UI.md §5` — tela de PIN, coberta
  pelos requisitos de teclado e contraste deste documento.
