# Cronologia completa — do lead sem site ao manual entregue

> Passo a passo cronológico de ponta a ponta. Cada passo diz: o que
> acontece, qual agente/etapa executa, o que entra, o que sai, tempo
> estimado, e a bifurcação "tem logo" vs. "não tem logo" quando aplicável.
> Ver `00-arquitetura/ARQUITETURA-AGENTES.md` pro detalhe de orquestração
> por trás de cada passo.

## 0. Pré-condição (já existe na webfy, fora do escopo deste pacote)

A plataforma webfy já varre a web procurando negócios locais sem site, já
gera automaticamente um site pra esse negócio, e já tem um vendedor que
vende esse site pronto pro dono do negócio, com o lead caindo num CRM
próprio. Este pacote adiciona SÓ a geração automática do manual de marca em
cima desse fluxo existente — não substitui nem duplica nenhuma dessas
etapas.

---

## Passo 1 — Varredura encontra o negócio

- **O que acontece:** a varredura da webfy identifica um negócio local sem
  site e coleta os sinais públicos disponíveis (GBP, redes sociais, site
  antigo se houver, fotos).
- **Etapa:** infraestrutura existente da webfy (fora deste blueprint).
- **Entra:** nada (início do fluxo).
- **Sai:** dump bruto de sinais coletados, associado a um `lead_id`.
- **Tempo:** já acontece hoje, sem mudança neste passo.

## Passo 2 — Ingestão e normalização dos sinais

- **O que acontece:** os sinais brutos são normalizados pro contrato formal
  que o pipeline de manual consome.
- **Etapa:** `agent-01-ingestor-sinais`.
- **Entra:** dump bruto da varredura.
- **Sai:** `input-sinais-negocio` validado + decisão se há sinal mínimo
  suficiente pra prosseguir.
- **Tempo:** 5-15s.
- **Bifurcação que nasce aqui:** o campo `logo_url` é preenchido ou fica
  `null` — essa decisão determina toda a bifurcação dos passos 5-6.

## Passo 3 — Descoberta automática de identidade (8 módulos)

- **O que acontece:** os 8 módulos do método de descoberta (adaptado de
  `.claude/skills/descoberta-marca/SKILL.md`) são inferidos a partir dos
  sinais, cada um com score de confiança. Sem entrevista humana.
- **Etapa:** `agent-02-descoberta-automatica`.
- **Entra:** `input-sinais-negocio`.
- **Sai:** os 8 módulos (propósito, posicionamento, público, personalidade,
  voz, narrativa, tensão do fundador, síntese), cada um `alto/medio/baixo/
  ausente` + conteúdo ou `[LACUNA]`.
- **Tempo:** 15-40s.
- **Nota:** "Tensão do fundador" fica `ausente` quase sempre neste passo —
  é esperado, não é falha (ver Passo 9).

## Passo 4 — Posicionamento consolidado

- **O que acontece:** propósito + posicionamento + público + personalidade
  viram um pacote objetivo (frase de posicionamento + traços de
  personalidade) que os agentes visuais consomem.
- **Etapa:** `agent-03-posicionamento`.
- **Entra:** saída do Passo 3.
- **Sai:** posicionamento sintetizado + perfil de personalidade.
- **Tempo:** 20-40s. Roda em paralelo com o início dos Passos 5 e 6.

## Passo 5 — Bifurcação: existe logo?

### 5a. SIM, tem logo existente (`logo_url` preenchido)

- **O que acontece:** o logo é tratado como âncora — cores, estilo
  tipográfico, qualidade e regras de uso são extraídos dele, nunca
  redesenhado por decisão própria do agente.
- **Etapa:** `agent-06-logo-guardiao`.
- **Entra:** `logo_url`.
- **Sai:** cores dominantes em hex, estilo tipográfico aproximado, nível de
  qualidade (vetorizável/raster boa/raster baixa), regras de uso completas.
- **Tempo:** 20-45s (inclui leitura de imagem).

### 5b. NÃO, sem logo existente (`logo_url` nulo)

- **O que acontece:** este passo é pulado inteiramente. O pipeline segue
  direto pro Passo 6 sem insumo de logo.
- **Tempo:** 0s (não roda).

## Passo 6 — Sistema visual (paleta + tipografia)

- **O que acontece:** a paleta funcional e a dupla tipográfica são
  definidas.
- **Etapa:** `agent-04-sistema-cor` + `agent-05-tipografia-e-composicao`,
  em paralelo entre si.
- **Entra:** perfil de personalidade (Passo 4) + saída do Passo 5a, se
  existir.
- **Sai:** paleta com hex/hover/racional parcial; dupla tipográfica com
  regras de composição.
- **Tempo:** 20-60s.
- **Bifurcação:** com logo (5a), a paleta deriva das cores extraídas do
  logo; sem logo (5b), deriva do posicionamento/personalidade.

## Passo 7 — Geração das 3 alternativas de logo

- **O que acontece:** 3 briefs estruturados de logo novo são compostos e
  enviados ao serviço externo de geração de imagem.
- **Etapa:** `agent-07-logo-alternativas` + serviço externo plugável (ver
  `04-integracao-webfy/SERVICO-GERACAO-DE-IMAGEM.md`).
- **Entra:** posicionamento (Passo 4), paleta (Passo 6), logo existente se
  houver (Passo 5a).
- **Sai:** 3 opções de logo com conceito, racional, brief e imagem
  (ou `falhou`).
- **Tempo:** 20-90s (depende do provedor de imagem; as 3 rodam em
  paralelo entre si).
- **Bifurcação:**
  - Com logo existente (5a): as 3 opções são enquadradas como **upsell
    opcional** ("caso queira trocar").
  - Sem logo (5b): as 3 opções são enquadradas como **escolha inicial
    obrigatória** (é o único caminho pra o negócio ter uma logo).

## Passo 8 — Composição do manual

- **O que acontece:** tudo que foi decidido nos passos 3-7 é consolidado no
  objeto final do manual, incluindo aplicações práticas e anti-padrões.
- **Etapa:** `agent-08-compositor-manual`.
- **Entra:** saídas dos agentes 2 a 7.
- **Sai:** manual quase completo (`output-manual-marca` sem `racional`
  final nem `gate_qualidade`).
- **Tempo:** 10-20s.

## Passo 9 — Explicação didática

- **O que acontece:** cada elemento gráfico recebe o racional causal
  completo (mecanismo → percepção → aplicação → recomendação) em
  linguagem acessível pro dono do negócio, fundamentado em psicologia de
  design genérica — nunca no livro proprietário da BNP.
- **Etapa:** `agent-10-explicador-didatico`.
- **Entra:** saída do Passo 8.
- **Sai:** manual com todo `racional` preenchido.
- **Tempo:** 10-30s.
- **Nota sobre "tensão do fundador":** se esse módulo permaneceu `ausente`
  desde o Passo 3, ele segue como lacuna aqui — só é preenchido se, depois
  da entrega inicial, o vendedor fizer uma micro-conversa de alinhamento
  com o dono do negócio (fluxo manual, fora deste pipeline automático) e
  disparar uma regeração pontual do módulo.

## Passo 10 — Gate de qualidade

- **O que acontece:** o manual é auditado contra 4 eixos: especificidade,
  fidelidade factual, isolamento entre leads, e completude do racional.
- **Etapa:** `agent-09-revisor-qualidade`.
- **Entra:** manual completo (Passo 9) + `input-sinais-negocio` original.
- **Sai:** veredito `aprovado` / `aprovado_com_ressalvas` / `reprovado`.
- **Tempo:** 15-30s.
- **Bifurcação:**
  - Aprovado (com ou sem ressalvas): segue pro Passo 11.
  - Reprovado: volta pro agente responsável pelo achado (máx. 2 ciclos
    automáticos); no 3º reprovado, escala pra fila humana em vez de tentar
    de novo sozinho.

## Passo 11 — Manual disponível na aba do vendedor

- **O que acontece:** o estado de geração muda pra `pronto`; a aba "Manual
  de marca" no CRM do webfy passa a renderizar o conteúdo completo,
  navegável por seção, com botões de exportar PDF e copiar link público.
- **Etapa:** UI da aba (`04-integracao-webfy/ESPECIFICACAO-ABA-UI.md`).
- **Entra:** manual aprovado.
- **Sai:** tela "Pronto", com banner de lacunas se houver.
- **Tempo:** imediato após o Passo 10 (renderização, não processamento).

## Passo 12 — Vendedor mostra/envia o manual pro dono do negócio

- **O que acontece:** o vendedor usa o manual como argumento de venda
  extra — mostra na tela, exporta PDF, ou envia o link público — durante o
  contato com o dono do negócio.
- **Etapa:** ação humana do vendedor, fora do pipeline automático.
- **Entra:** manual pronto.
- **Sai:** decisão do vendedor de marcar como "enviado".
- **Tempo:** variável, depende do ciclo de vendas.

## Passo 13 — CRM registra a entrega

- **O que acontece:** ao clicar "Marcar como enviado", o CRM grava
  `status_entrega=enviado_pelo_vendedor`, atualiza o card do lead, e o
  status opcional de funil "Manual de marca pronto" fica disponível como
  sinal de progresso da venda.
- **Etapa:** integração CRM (`04-integracao-webfy/INTEGRACAO-CRM.md`).
- **Entra:** ação do Passo 12.
- **Sai:** estado de geração atualizado, card do lead atualizado.
- **Tempo:** imediato.

---

## Diagrama do fluxo completo (texto)

```
[Varredura acha negocio sem site]
              |
              v
   [Passo 2: agent-01 ingestao/normalizacao] --(sinal insuficiente)--> [aborta, reporta erro]
              |
              v
   [Passo 3: agent-02 descoberta automatica - 8 modulos, c/ confianca]
              |
              v
   [Passo 4: agent-03 posicionamento consolidado] -----------------------+
              |                                                          |
              v                                                          |
   +-------- BIFURCACAO: logo_url existe? ---------+                    |
   |                                                |                    |
  SIM                                              NAO                   |
   |                                                |                    |
   v                                                |                    |
[Passo 5a: agent-06 logo-guardiao                   |                    |
 extrai cor/estilo/qualidade/regras]                |                    |
   |                                                |                    |
   +--------------------+---------------------------+                    |
                         |                                                |
                         v                                                |
        [Passo 6: agent-04 sistema-cor + agent-05 tipografia] <----------+
                         |
                         v
        [Passo 7: agent-07 logo-alternativas -> servico externo de imagem]
           |                                          |
        tem logo: 3 opcoes = upsell opcional     sem logo: 3 opcoes = escolha inicial
           |                                          |
           +-------------------+----------------------+
                                |
                                v
              [Passo 8: agent-08 compositor-manual]
                                |
                                v
              [Passo 9: agent-10 explicador didatico
               (racional mecanismo->percepcao->aplicacao->recomendacao)]
                                |
                                v
              [Passo 10: agent-09 revisor de qualidade]
                     |                        |
                aprovado /              reprovado (max 2 ciclos automaticos,
              aprov. c/ ressalvas          3o vai pra fila humana)
                     |                        |
                     v                        v
    [Passo 11: aba "Manual de marca" no CRM  [volta ao agente responsavel
     mostra Pronto, com banner de lacunas]    pelo achado]
                     |
                     v
    [Passo 12: vendedor mostra/exporta/envia o manual pro dono do negocio]
                     |
                     v
    [Passo 13: CRM grava status_entrega, funil ganha marco opcional]
```
