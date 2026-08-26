# Integração com o CRM do webfy

## 1. Quando a geração dispara

Duas opções possíveis — **recomendação: automática ao achar o lead**, com
detalhe abaixo.

### Opção A — Automática ao achar o lead (recomendada)

O pipeline dispara assim que a varredura captura um negócio novo e gera o
site automático, em paralelo (o manual não depende do site, e vice-versa).
Quando o vendedor abre o lead, o manual já está pronto ou em progresso —
zero fricção, e o manual vira mais um argumento de venda pronto na mão do
vendedor sem ele precisar lembrar de pedir.

**Por que recomendar essa:** o valor do manual como ferramenta de venda é
maior quanto mais cedo ele existir — o vendedor decide em segundos se usa
ou não, mas só se já estiver lá. Pedir manual sob demanda cria mais um
passo manual que o vendedor pode esquecer de fazer, reduzindo a adoção da
feature (o motivo de existir essa aba nova é aumentar o valor percebido da
venda — depender de lembrete manual do vendedor derruba esse objetivo).

**Custo a considerar:** roda o pipeline (e a geração de imagem) pra TODO
lead capturado, mesmo os que o vendedor nunca vai converter. Se o custo da
API de geração de imagem for alto, considerar rodar as fases 1-3 sempre
(barato, sem geração de imagem) e só disparar o agent-07 (as 3 opções de
logo, que é a parte cara) sob demanda quando o vendedor abrir a aba pela
primeira vez.

### Opção B — Sob demanda pelo vendedor

Só dispara quando o vendedor clica "Gerar manual de marca" na aba (ver
`ESPECIFICACAO-ABA-UI.md §2.1`). Mais barato (nunca processa lead que não
vira conversa de venda), mas depende do vendedor lembrar de usar a feature.

**Quando usar Opção B em vez de A:** se o volume de leads capturados for
muito maior que a capacidade de processamento/custo de imagem que o webfy
quer sustentar por mês — nesse caso, gerar sob demanda evita desperdiçar
processamento em leads que nunca serão trabalhados.

## 2. O que grava no card do lead

- **Estágio do manual** (espelha `estagio_atual` do estado de geração) —
  visível como uma tag/badge curta no card do lead na lista geral do CRM,
  não só dentro do detalhe.
- **Indicador de lacunas abertas** — ícone discreto se `lacunas_abertas_count
  > 0`, pra o vendedor saber de antemão que vale uma conversa extra com o
  dono antes de enviar.
- **Link público e status de PDF exportado** — pra evitar regenerar/reenviar
  à toa.
- **`status_entrega`** — alimenta o funil (ver §3).

## 3. Novo status de funil sugerido

Inserir um status opcional no funil de vendas existente, entre "site gerado"
e "proposta enviada":

```
Lead capturado → Site gerado → [Manual de marca pronto] → Contato iniciado
  → Proposta enviada → Fechado
```

`[Manual de marca pronto]` não é obrigatório passar por ele (um vendedor
pode pular direto pro contato sem usar o manual), mas fica disponível como
filtro/coluna no funil — permite ao gestor comercial medir se leads com
manual pronto convertem mais que leads sem, dado real de valor da feature
pra decisão futura de investimento nela.
