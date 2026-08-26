# Meta-arquivos — estado de geração por lead

> Adaptação do `.agent/state.json` da BNP. Lá, o estado rastreia progresso de
> TASKs de projeto (um roadmap fixo, poucas dezenas de itens, editado por
> sessões humanas). Aqui, o estado rastreia progresso de GERAÇÕES DE MANUAL,
> potencialmente milhares em paralelo, um registro por `lead_id`, editado só
> por agentes automáticos. A analogia é estrutural (mesma função: "onde
> paramos, o que falta, o que ficou pendente"), não literal — não é o mesmo
> arquivo reaproveitado.

## 1. O que é

Um registro de estado por negócio/lead, no formato de
`01-contratos-de-dados/estado-geracao.schema.json`. Na prática do webfy isso
provavelmente vive numa tabela do banco (uma linha por `lead_id`), não num
arquivo JSON solto — o schema é o contrato de campos, não uma prescrição de
"deve ser um arquivo".

## 2. Por que existe separado do output do manual

O manual (`output-manual-marca.schema.json`) é o PRODUTO — o que o dono do
negócio vê. O estado é o PROCESSO — o que o vendedor e a própria plataforma
precisam saber pra decidir a próxima ação (mostrar spinner? escalar lacuna
pro vendedor? já foi enviado, não reenviar à toa?). Misturar os dois faria o
manual carregar campos operacionais irrelevantes pro cliente final.

## 3. Campos centrais e o que cada um resolve

| Campo | Resolve |
|---|---|
| `estagio_atual` | Que etapa do pipeline está rodando agora — alimenta a barra de progresso da aba UI (`04-integracao-webfy/ESPECIFICACAO-ABA-UI.md`). |
| `confianca_por_modulo` | Lookup rápido pra UI sinalizar quais dos 8 módulos vieram fracos, sem precisar abrir o manual inteiro. |
| `logo_existente_detectado` | Decide qual dos dois textos de enquadramento a UI mostra pro vendedor ("achamos o logo de vocês" vs. "geramos 3 opções pra escolher"). |
| `lacunas_abertas_count` | Quando > 0, a UI mostra o estado "pronto com pendências" em vez de "pronto" puro — nunca esconde lacuna. |
| `gate_qualidade_ultimo_veredito` | Se `reprovado`, o manual NUNCA aparece como disponível pro vendedor, mesmo que estagio_atual diga "compondo_manual" tenha terminado — é o freio duro. |
| `ciclos_qualidade_atual` | Circuit breaker do gate de qualidade: ao atingir o teto de 2 ciclos automáticos (3ª reprovação), para de tentar sozinho e cai pra fila humana em vez de regenerar em loop. Reseta a cada geração nova — não é o mesmo contador de `versao_manual_atual`. |
| `status_entrega` | Rastreado separado do estágio técnico — "pronto" e "entregue" são coisas diferentes; um manual pode ficar pronto e nunca ser enviado, e isso é visível. |
| `historico_estagios` | Trilha de auditoria mínima — não é log completo, é o rastro de transições pra debugar "por que esse lead travou". |

## 4. Regras de escrita (equivalente às regras de state.json da BNP)

1. **Atualize o estágio imediatamente após cada agente terminar** — nunca em
   lote no fim do pipeline. Se o processo cair no meio, o estado precisa
   refletir onde parou de verdade.
2. **`gate_qualidade_ultimo_veredito=reprovado` é definitivo até nova
   tentativa.** Nenhuma parte da UI ou do CRM deve tratar um manual reprovado
   como disponível, independente do que `estagio_atual` diga.
3. **Nunca decremente `lacunas_abertas_count` fora de uma regeração real.**
   Marcar lacuna como "resolvida" sem reprocessar o módulo correspondente é
   o equivalente a preencher achismo — proibido pela regra central do método
   (`.claude/skills/descoberta-marca/SKILL.md` regra 5, herdada aqui).
4. **`versao_manual_atual` só incrementa quando o `output-manual-marca` muda
   de fato** (nova geração completa ou regeração pontual de logo) — não a
   cada toque no estado.

## 5. Onde isso se conecta

- CRM (`04-integracao-webfy/INTEGRACAO-CRM.md`): o card do lead lê
  `estagio_atual` e `status_entrega` pra decidir o que mostrar ao vendedor.
- UI (`04-integracao-webfy/ESPECIFICACAO-ABA-UI.md`): consome o schema
  inteiro pra renderizar os 5 estados de tela.
- Gate de qualidade (`02-agentes/agent-09-revisor-qualidade.md`): é o único
  agente autorizado a escrever `gate_qualidade_ultimo_veredito`.
