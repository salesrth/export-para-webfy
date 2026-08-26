# Versionamento — o que acontece quando o negócio muda depois

> O manual não é um documento gerado uma vez e congelado. Este arquivo
> define quando uma mudança vira versão nova, quando vira só um módulo
> atualizado, e qual decisão do cliente é sagrada demais pra sobrescrever
> sem confirmação. Ver `CHRONOLOGIA.md` Passo 14 pro fluxo cronológico e
> `02-agentes/agent-11-regenerador-modulo.md` pro agente que executa a
> regeração pontual.

## 1. O que é uma versão

`output-manual-marca.versao` (schema) e `estado_geracao.versao_manual_atual`
(estado) andam juntos: toda vez que o conteúdo do manual muda de fato — não
o estado do pipeline, o CONTEÚDO — a versão incrementa em 1. `v1` é sempre a
primeira geração completa que passou pelo gate (Passo 10). Não há versão
"0.1" nem numeração semântica — é um contador inteiro simples, porque não
existe distinção formal entre "mudança pequena" e "mudança grande" no
schema: as duas incrementam o mesmo contador. A distinção que importa é
outra (§2): regeração completa vs. regeração pontual de módulo.

## 2. Regeração completa vs. regeração pontual

| | Regeração completa | Regeração pontual de módulo |
|---|---|---|
| **Dispara** | `agent-01` até `agent-09` de novo, pipeline inteiro | `agent-11-regenerador-modulo`, só o módulo alvo + racional + checagem do gate nesse trecho |
| **Quando usar** | Mudança estrutural no sinal de origem: nome do negócio corrigido, categoria reclassificada, negócio mudou de nicho/endereço de forma que invalida premissas de vários módulos ao mesmo tempo | Ajuste localizado: "tensão do fundador" preenchida após micro-conversa do vendedor, pedido pontual de ajuste de paleta, correção de um fato específico apontado pelo dono do negócio |
| **O que preserva** | Nada — tudo é reprocessado a partir do sinal atualizado | Todo o resto do manual fica intacto; só o módulo alvo (e o racional dele) muda |
| **Quem dispara** | Ação humana explícita (vendedor confirma a mudança estrutural) ou correção de erro grave achada pelo gate — nunca automático por reprovação isolada | Ação humana explícita (vendedor/dono pede o ajuste) |
| **Efeito na versão** | Incrementa `versao` | Incrementa `versao` |

Regra prática: se a mudança de sinal derruba a premissa de 3+ módulos ao
mesmo tempo, é regeração completa. Se afeta 1 módulo isolado, é regeração
pontual — mesmo que o "motivo" pareça grande pro dono do negócio (trocar a
paleta inteira ainda é regeração pontual do módulo `paleta`, não do
manual inteiro).

## 3. Histórico de versões acessível pro vendedor

Cada geração de manual — completa ou pontual — fica registrada e acessível
via `estado_geracao.link_historico_versoes` (ver
`01-contratos-de-dados/estado-geracao.schema.json`). O vendedor precisa
poder ver: quantas versões existem, quando cada uma foi gerada, e (no
mínimo) se foi regeração completa ou pontual — útil quando o dono do
negócio pergunta "o que exatamente vocês mudaram". A implementação real
(tabela, storage de blob versionado) é decisão do webfy; o contrato aqui é
só "o vendedor tem que conseguir ver isso", não uma prescrição de schema
completo de histórico.

## 4. Módulo que NUNCA regenera sozinho sem confirmação humana

**O logo escolhido e ativado pelo dono do negócio.** Quando o negócio já
tinha logo existente e decide trocar por uma das 3 alternativas geradas —
ou quando não tinha logo e escolhe uma das 3 como definitiva — essa escolha
é uma decisão do cliente, não um estado técnico do pipeline. Regenerar o
módulo de logo automaticamente depois disso destruiria essa decisão sem o
cliente saber.

Isso vale mesmo dentro do gatilho 3 do Passo 14 (mudança estrutural — nome/
categoria do negócio mudou): o restante do manual pode reprocessar
normalmente, mas o logo ativo fica intocado até confirmação humana
explícita e separada ("sim, pode gerar um logo novo mesmo já tendo um
escolhido"), nunca incluída de graça dentro de uma regeração completa
disparada por outro motivo.

Pré-condição de implementação: a plataforma webfy precisa persistir, em
algum lugar (CRM, estado de geração, ou equivalente — este blueprint não
crava o campo exato), o sinal de "esta alternativa foi escolhida/ativada
pelo dono do negócio". Enquanto esse sinal existir marcado, nenhuma
regeração automática do módulo `logo` pode rodar — nem completa, nem
pontual — sem esse "sim" explícito adicional. Ver também o teto de custo
de geração de imagem em `04-integracao-webfy/SERVICO-GERACAO-DE-IMAGEM.md
§6`, que já bloqueia qualquer 2ª geração automática independente desta
regra — as duas regras se reforçam, mas resolvem problemas diferentes
(custo vs. destruição de decisão do cliente).

## 5. Onde isso se conecta

- `CHRONOLOGIA.md` Passo 14 — os 3 gatilhos que levam até aqui.
- `02-agentes/agent-11-regenerador-modulo.md` — o agente que executa §2
  coluna "pontual".
- `00-arquitetura/ESTRATEGIA-DE-TESTES.md` — caso de teste de idempotência
  cobre o cenário "rodar duas vezes não deveria criar 2 versões
  divergentes sem motivo".
- `01-contratos-de-dados/estado-geracao.schema.json` — campos
  `versao_manual_atual` e `link_historico_versoes`.
