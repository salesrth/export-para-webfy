# Estratégia de testes — validar agente antes de mudar prompt/lógica em produção

> Espelha o padrão de `evals/` que a BNP já usa internamente (ex:
> `.claude/skills/voz-de-marca/evals` e `.claude/skills/motor-de-conteudo/
> evals` no repo da BNP — este pacote não tem acesso a esses arquivos, o que
> segue é o PADRÃO description, não uma cópia deles). Este pacote não
> contém código executável, então o que segue é a estratégia — o dev do
> webfy implementa os casos reais no stack real, mas a estrutura e o
> mínimo obrigatório abaixo são parte do blueprint.

## 1. O padrão (mesmo formato dos evals da BNP)

Cada agente do pipeline (`02-agentes/agent-01` a `agent-11`) tem um
conjunto de casos de teste: **input fixo + output esperado (ou critério de
aceitação, quando o output não é determinístico palavra por palavra)**.
Antes de qualquer mudança de prompt ou de lógica de um agente ir pra
produção, os casos rodam contra a versão nova. Regressão num caso que
passava antes **bloqueia o deploy** — não é aviso, é gate.

Diferença importante em relação a teste de código tradicional: como os
agentes 02-05, 07, 09, 10 e 11 usam modelo de raciocínio (não são
determinísticos palavra por palavra), o "output esperado" na maioria dos
casos é um **critério de aceitação verificável** (schema válido, campo
específico presente/ausente, confiança dentro de uma faixa esperada), não
um diff de texto exato. Só os agentes mecânicos (agent-01, agent-08 —
`model: modelo barato`, sem julgamento criativo) comportam comparação mais
próxima de determinística.

## 2. Estrutura de um caso de teste

```json
{
  "caso_id": "string, descritivo (ex: 'sem-sinal-nenhum')",
  "agente_alvo": "agent-01-ingestor-sinais",
  "input": "objeto de input pro agente (ex: input-sinais-negocio ou dump bruto)",
  "criterio_aceitacao": [
    "afirmação verificável 1 (ex: 'sinal_minimo_suficiente = false')",
    "afirmação verificável 2 (ex: 'erro.etapa = ingestao')"
  ],
  "motivo_do_caso": "por que esse caso existe / que regressão ele pega"
}
```

## 3. Casos de teste obrigatórios mínimos

Nenhum agente vai pra produção sem cobertura destes 5, no mínimo — cada um
mapeado pro(s) agente(s) que o executa:

### 3.1 Negócio sem nenhum sinal coletável

- **Input:** `input-sinais-negocio` com só `lead_id`, `nome_negocio`,
  `categoria`, `coletado_em` — todo o resto null/vazio (nenhum GBP, nenhuma
  rede social, nenhuma foto).
- **Agente(s):** `agent-01` (decide se há sinal mínimo — aqui há, por
  nome+categoria), `agent-02` (todos os 8 módulos, exceto talvez síntese
  básica, devem sair `confianca=ausente`).
- **Critério:** nenhum módulo recebe conteúdo inventado pra "preencher".
  `lacunas_abertas` no manual final reflete isso de forma honesta, não
  escondida.

### 3.2 Negócio com reviews com linguagem ofensiva/spam

- **Input:** `google_business_profile.reviews_amostra` com texto contendo
  ofensa, spam, ou linguagem imprópria.
- **Agente(s):** `agent-02` (síntese de personalidade/voz/posicionamento a
  partir de reviews).
- **Critério:** o conteúdo ofensivo/spam nunca aparece parafraseado nem
  citado no manual final — o agente precisa filtrar sinal de ruído/abuso
  antes de sintetizar, sem propagar a linguagem pro dono do negócio. Se o
  filtro reduzir o sinal disponível a ponto de a confiança cair, isso é o
  comportamento correto (confiança mais baixa, não confiança mantida com
  conteúdo impróprio).

### 3.3 `logo_url` quebrado/inacessível

- **Input:** `logo_url` preenchido apontando pra uma URL que retorna
  erro/timeout/imagem corrompida.
- **Agente(s):** `agent-06-logo-guardiao`.
- **Critério:** reporta `tem_logo_existente=false` de volta pro
  orquestrador (não trava o pipeline, não trata como "sem logo" desde o
  início — é falha técnica registrada, ver constraint já existente do
  agente).

### 3.4 Negócio com nome que colide com marca registrada conhecida

- **Input:** `nome_negocio` igual ou muito similar a uma marca registrada
  amplamente conhecida (ex: nome de rede nacional/internacional).
- **Agente(s):** `agent-09-revisor-qualidade` (é quem tem visão do manual
  completo pra sinalizar, não um agente criativo upstream que decidiria
  sozinho).
- **Critério:** **este é um caso de risco jurídico a SINALIZAR, não a
  resolver automaticamente.** O teste verifica que o gate de qualidade
  identifica a colisão e adiciona um achado/observação visível (ex: uma
  ressalva ou nota separada das `violations` padrão) — nunca que o pipeline
  bloqueia, corrige, ou toma decisão jurídica sozinho. Mesma postura de
  `[PENDENTE-JURIDICO]` usada em `04-integracao-webfy/SERVICO-GERACAO-DE-
  IMAGEM.md §7`: apontar, não resolver.

### 3.5 Negócio já processado antes (idempotência)

- **Input:** o mesmo `input-sinais-negocio` (sem nenhum sinal novo) enviado
  duas vezes pro pipeline completo.
- **Agente(s):** pipeline inteiro, mais especificamente a camada de
  orquestração/estado (fora do escopo dos agentes individuais, mas o teste
  cobre o comportamento agregado).
- **Critério:** rodar duas vezes **não deve gerar duas versões
  divergentes sem motivo**. Comportamento esperado: ou (a) a segunda
  chamada é reconhecida como duplicata e não incrementa `versao`/dispara
  nova geração de imagem (respeitando também o teto de custo de
  `04-integracao-webfy/SERVICO-GERACAO-DE-IMAGEM.md §6`), ou (b) se o
  produto decidir permitir reprocessamento manual explícito, as decisões
  centrais (origem da paleta, quantidade de módulos com `[LACUNA]`,
  isolamento por `lead_id`) saem equivalentes entre as duas rodadas, mesmo
  que o texto do racional varie palavra por palavra (não-determinismo de
  LLM é aceitável; decisão estrutural divergente sem sinal novo não é).

## 4. Quando adicionar um caso novo

Todo bug real encontrado em produção (reportado via skill `bug` no lado da
BNP, ou processo equivalente do webfy) vira um caso de teste novo antes de
ser considerado corrigido — impede que a mesma regressão volte
silenciosamente numa mudança de prompt futura.

## 5. Onde isso se conecta

- `02-agentes/agent-09-revisor-qualidade.md` — o gate que os casos 3.2, 3.4
  e o comportamento de honestidade de 3.1 verificam na prática.
- `00-arquitetura/OBSERVABILIDADE.md` — se uma métrica de produção piora
  sem um caso de teste correspondente ter pego a regressão antes, é sinal
  de que falta caso de teste, não só de que o prompt piorou.
- `00-arquitetura/VERSIONAMENTO.md` — critério de aceitação do caso 3.5
  depende diretamente das regras de regeração completa vs. pontual
  definidas lá.
