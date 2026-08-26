# Serviço de geração de imagem — interface pluggable

> A escolha de QUAL API de geração de imagem usar é decisão de configuração
> do webfy (custo/qualidade/latência), não deste blueprint. O que segue é o
> contrato de entrada/saída que o agent-07 espera dessa interface —
> qualquer provedor por trás dela precisa satisfazer este contrato.

## 1. Por que é pluggable

O mercado de geração de imagem muda rápido (preço, qualidade, latência).
Acoplar o pipeline a um provedor específico trava o webfy numa decisão que
vai precisar ser revisitada. A interface abaixo isola o agent-07 (que monta
o brief) do provedor real (que renderiza), permitindo trocar de provedor
sem tocar em nenhum agente do pipeline.

## 2. Formato do brief enviado (request)

```json
{
  "lead_id": "string",
  "opcao_id": "string",
  "conceito": "descrição textual curta do conceito do logo",
  "paleta_hex": ["#8a5a2e", "#e8c07d"],
  "estilo_composicao": "geometrico | organico | manuscrito | tipografico_puro",
  "elementos_obrigatorios": ["nome do negócio, se for wordmark", "símbolo, se houver"],
  "o_que_evitar": ["gradiente pesado", "clichê genérico de estoque", "elementos ilegíveis em tamanho pequeno"],
  "formato_saida_desejado": {
    "proporcao": "1:1",
    "fundo": "transparente | branco",
    "resolucao_minima_px": 1024
  }
}
```

Este é o brief que sai de `logo.alternativas_geradas[].brief_enviado_api_imagem`
no schema de output — o provedor real recebe este JSON (ou uma tradução dele
pro formato de prompt específico da API escolhida).

## 3. Formato esperado de retorno (response)

```json
{
  "opcao_id": "string (eco do request)",
  "status": "gerada | falhou",
  "imagem_url": "string (uri) | null",
  "metadata": {
    "provedor": "string, nome do provedor usado",
    "tempo_geracao_ms": "integer",
    "seed_ou_id_geracao": "string, pra rastreabilidade/regeração determinística se o provedor suportar"
  },
  "motivo_falha": "string | null"
}
```

Um retorno com `status=falhou` não trava o pipeline — o agent-08 monta o
manual com essa opção marcada como indisponível, e o agent-09 confere que
a lacuna aparece explícita, não escondida (ver `02-agentes/agent-09-revisor-qualidade.md`).

## 4. Requisitos não-funcionais da interface

- **Timeout configurável** — recomendação: 60-90s por imagem antes de
  marcar `falhou` e seguir sem travar o pipeline inteiro.
- **As 3 chamadas (uma por opção) devem poder rodar em paralelo** — não há
  dependência entre as 3 opções de logo.
- **Idempotência por `opcao_id` + `seed`** — permitir regenerar a MESMA
  opção (ex: botão "Regenerar logo" na UI) sem precisar recriar o conceito
  do zero, se o provedor suportar seed/reprodutibilidade.

## 5. Candidatos de mercado a avaliar (não prescritivo — decidir depois por custo/qualidade)

Lista curta pra referência de pesquisa, não uma recomendação cravada:

- **Modelos de geração de imagem via API de grandes provedores de IA
  generativa** (ex: linha de modelos de imagem da OpenAI, da Google, ou
  similares) — avaliar qualidade de texto/tipografia dentro da imagem, que
  costuma ser o ponto fraco em geração de logo.
- **Stable Diffusion / SDXL hospedado (self-host ou API gerenciada)** —
  mais controle de custo em volume alto, mas exige mais trabalho de
  engenharia de prompt pra qualidade consistente de logo.
- **APIs especializadas em geração de logo/vetor** (categoria de produto
  específica, existem várias no mercado) — avaliar se entregam SVG
  diretamente, o que resolveria o problema de vetorização que o agent-06
  precisa avaliar manualmente em logos existentes.

Critério de decisão sugerido pro webfy: custo por imagem em volume (o
pipeline roda pra cada lead capturado, potencialmente milhares/mês),
qualidade de tipografia embutida na imagem, e se entrega formato vetorial
nativo (reduz trabalho de pós-processamento).

## 6. Teto de custo — regra dura

**No máximo 1 geração automática completa (as 3 opções de logo) por
`lead_id`.** É a chamada que sai do Passo 7 do pipeline
(`CHRONOLOGIA.md`), disparada uma única vez por lead. Qualquer geração
adicional — regenerar as 3 de novo, regenerar só 1 opção, ou reprocessar
por causa de reprovação do gate de qualidade — **exige clique explícito do
vendedor** (botão "Regenerar logo" na UI, `04-integracao-webfy/
ESPECIFICACAO-ABA-UI.md §2.3`, ou o agent-11 disparado por ação humana). O
serviço de geração de imagem nunca deve aceitar uma segunda chamada
automática pro mesmo `lead_id` sem esse sinal de clique explícito no
request — implementações devem validar isso no nível da interface, não só
confiar que o agent-07 vai respeitar a regra sozinho (defesa em
profundidade contra custo de API rodando sem controle em escala).

## 7. [PENDENTE-JURIDICO] Direitos autorais do logo gerado por IA

A titularidade dos direitos autorais sobre um logo gerado por modelo de IA
generativa **não está resolvida juridicamente** neste blueprint — é uma
pendência formal que precisa de análise de advogado antes do lançamento da
feature. Isso afeta pelo menos três pontos:

- Se o dono do negócio pode reivindicar uso comercial exclusivo do logo
  gerado (registro de marca, por exemplo) sem risco de contestação sobre a
  autoria da geração.
- Se o webfy, o provedor de IA usado, ou terceiros retêm algum direito
  sobre a imagem gerada, dependendo dos termos de uso da API escolhida
  (candidatos listados em §5) — cada provedor tem cláusulas diferentes de
  propriedade sobre output gerado, que precisam ser revisadas uma a uma.
- Se há elemento derivativo problemático no output do modelo (ex:
  similaridade não intencional com marca registrada de terceiro — ver
  também `00-arquitetura/ESTRATEGIA-DE-TESTES.md` pro caso de teste
  correspondente).

Até essa análise jurídica ser concluída, nenhum agente, texto de UI ou
material do manual deve afirmar ou implicar que o direito autoral do logo
gerado pertence automaticamente e sem ressalva ao dono do negócio.
