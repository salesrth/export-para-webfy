# Observabilidade — o que o pipeline precisa expor pra alguém melhorar o produto

> Métricas, não logs de debug. O objetivo aqui não é "o que aconteceu num
> lead específico" (isso é `historico_estagios` do estado de geração), é
> "o que está acontecendo em agregado, através de muitos leads, que sinaliza
> onde o produto (prompts, sinais coletados, gate de qualidade) precisa
> melhorar". Quem consome isso é quem opera/evolui o pipeline, não o
> vendedor nem o dono do negócio.

## 1. Taxa de aprovação direta no gate de qualidade

- **O que mede:** de todo manual que chega no agent-09 (Passo 10), qual
  fração sai `aprovado` já na 1ª tentativa, vs. `aprovado_com_ressalvas`,
  vs. precisou de 1-2 ciclos de correção, vs. escalou pra fila humana
  (3ª reprovação).
- **Por que importa:** taxa de aprovação direta caindo ao longo do tempo é
  sinal de regressão em algum agente upstream (mudança de prompt que
  piorou qualidade) — dispara investigação antes que vire volume grande de
  escalonamento pra fila humana.
- **Corte útil:** por categoria de negócio (ex: "padaria" aprova mais fácil
  que "clínica odontológica"?) — ajuda a priorizar onde melhorar prompt
  primeiro.

## 2. Taxa de reprovação por eixo

O agent-09 audita 4 eixos (`02-agentes/agent-09-revisor-qualidade.md`):
especificidade, fidelidade factual, isolamento entre leads, e completude do
racional. Cada `violation.type` mapeia pra um eixo:

| Eixo | `violation.type` |
|---|---|
| Especificidade | `generico` |
| Fidelidade factual | `fato_inventado` |
| Isolamento entre leads | `cruzamento_de_lead` |
| Completude do racional | `racional_ausente` |
| (achado auxiliar, não eixo próprio) | `cor_fora_paleta`, `enquadramento_incorreto` |

- **O que mede:** fração de reprovações atribuível a cada eixo, ao longo do
  tempo.
- **Por que importa:** eixos diferentes apontam pra correções diferentes.
  Muito `generico` → agent-10 (explicador didático) precisa de prompt mais
  específico. Muito `fato_inventado` → agent-02/03 estão inferindo além do
  que o sinal sustenta. `cruzamento_de_lead` alto é o mais grave de todos —
  qualquer taxa acima de zero merece alarme imediato, não só dashboard
  (é vazamento de dado entre negócios, não uma métrica de qualidade comum).

## 3. Tempo médio por etapa

- **O que mede:** tempo real (não estimado) de cada agente (01 a 10) e da
  geração de imagem, agregado em p50/p90/p99.
- **Por que importa:** dimensiona fila de processamento (ver
  `00-arquitetura/ARQUITETURA-AGENTES.md §5` pras estimativas originais) e
  detecta degradação de latência de provedor externo (geração de imagem é
  o componente com maior variância — se p99 estourar o timeout configurado
  em `04-integracao-webfy/SERVICO-GERACAO-DE-IMAGEM.md §4`, isso aparece
  aqui antes de virar reclamação de vendedor sobre "manual travado").

## 4. Distribuição de confiança por módulo de descoberta

- **O que mede:** para cada um dos 8 módulos (`propósito`, `posicionamento`,
  `público_e_nicho`, `personalidade`, `voz_e_tom`, `narrativa`,
  `tensão_do_fundador`, `síntese`), a distribuição de `confianca`
  (`alto`/`medio`/`baixo`/`ausente`) através de todos os leads processados.
- **Por que importa — é o sinal mais acionável de todos:** um módulo que
  vira `[LACUNA]` (`ausente`) na maioria dos leads não é "falha do agente",
  é sinal de que os SINAIS coletados na Etapa 1 (varredura da webfy) talvez
  precisem melhorar pra aquele tipo de dado. Ex: se `narrativa` está sempre
  ausente, isso é esperado por design (raramente está em sinal público
  estruturado — ver `02-agentes/agent-02-descoberta-automatica.md`); mas se
  `posicionamento` está frequentemente `baixo`/`ausente`, isso sugere que a
  varredura devia capturar mais reviews por negócio, ou GBP com descrição
  mais completa, antes de o problema ser tratado como "prompt ruim".
- **Corte útil:** por categoria de negócio e por `fonte_coleta` (versão do
  scraper) — permite atribuir queda de confiança a uma mudança específica
  na coleta, não só ao pipeline de IA.

## 5. Quantos leads escalam pra fila humana e por quê

- **O que mede:** volume de leads que atingem a 3ª reprovação
  (`estado_geracao.tentativas_geracao` no teto) e vão pra fila de
  suporte/operações do webfy (`02-agentes/agent-09-revisor-qualidade.md`),
  quebrado pelo `motivo_reprovacao` agregado dos 3 ciclos.
- **Por que importa:** é o teto de custo humano da automação — se esse
  número cresce, o ciclo automático de correção (2 tentativas antes de
  escalar) não está dando conta, e o motivo agregado indica se o problema é
  sistemático (mesmo tipo de achado repetindo) ou disperso (leads
  individualmente difíceis, sem padrão).
- **SLA:** ver `02-agentes/agent-09-revisor-qualidade.md` — o SLA por
  prioridade de fila ainda não está definido neste blueprint; esta métrica
  é o insumo que informa essa decisão futura (volume real de escalonamento
  antes de prometer um SLA que a operação não consegue sustentar).

## 6. Feedback do vendedor sobre a qualidade do manual

- **O que captura:** o vendedor precisa poder avaliar o manual gerado —
  positivo/negativo, com motivo curto opcional (texto livre, não
  obrigatório). É um sinal qualitativo direto de quem usa o manual como
  ferramenta de venda no dia a dia, diferente das métricas §1-§5 (que vêm
  do gate automático, não de julgamento humano).
- **Onde faz sentido capturar:** na própria aba "Manual de marca"
  (`04-integracao-webfy/ESPECIFICACAO-ABA-UI.md`, estado "Pronto"), perto
  dos botões de ação — não precisa ser modal separado nem fluxo à parte.
  O momento mais natural é perto de "Marcar como enviado" ou logo depois,
  quando o vendedor já teve contato real com o conteúdo.
- **Por que importa:** este feedback é o insumo mais direto pra iterar os
  prompts dos agentes (2 a 10) — sinaliza o que a métrica automática do
  gate não pega (ex: manual "aprovado" pelo agent-09 mas que o vendedor
  achou genérico ou fora do tom pro tipo de negócio). Esta versão do
  blueprint não decide COMO esse feedback vira ajuste de prompt/agente —
  só que o dado precisa existir e ser capturável, pra não depender de
  reclamação informal/anedótica de vendedor chegando por outro canal.
- **Escopo do dado:** fica de fora de `estado-geracao.schema.json` neste
  momento (schema não muda nesta rodada) — quando o webfy implementar a
  captura, o campo natural é uma extensão do estado por lead (associado
  ao `lead_id` e à versão do manual avaliada, já que o manual pode ter
  passado por regeração — ver `VERSIONAMENTO.md`), não um registro solto
  sem vínculo com qual versão foi avaliada.
- **Corte útil, quando o volume permitir:** cruzar feedback negativo com
  categoria de negócio e com a distribuição de confiança por módulo (§4)
  — um padrão de "feedback negativo concentrado em categoria X + módulo Y
  sempre baixo" é sinal mais forte que qualquer um dos dois sozinho.

## 7. Onde isso se conecta

- `01-contratos-de-dados/estado-geracao.schema.json` — campos-fonte:
  `gate_qualidade_ultimo_veredito`, `tentativas_geracao`,
  `confianca_por_modulo`, `historico_estagios`.
- `02-agentes/agent-09-revisor-qualidade.md` — origem de `violations[]` e
  do payload de escalonamento.
- `00-arquitetura/ESTRATEGIA-DE-TESTES.md` — regressão detectada por teste
  antes do deploy deveria, idealmente, nunca aparecer nesta observabilidade
  em produção; se aparecer, é sinal de que o caso de teste correspondente
  faltou.
