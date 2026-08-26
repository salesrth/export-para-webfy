# Especificação da aba UI — manual de marca dentro do webfy

> Spec de UX, não de componente/framework. O dev do webfy adapta ao design
> system já existente da plataforma — o que importa aqui são os estados de
> tela, as transições entre eles e onde a aba se encaixa no fluxo do
> vendedor.

## 1. Onde a aba se encaixa

A aba vive dentro da visão de detalhe de um lead/negócio já capturado pela
varredura (mesma tela onde o vendedor já vê o site gerado automaticamente).
Nome sugerido da aba: **"Manual de marca"**, ao lado de abas existentes tipo
"Site" e "Dados do lead".

## 2. Estados de tela

### 2.1 Antes de gerar

- Estado padrão pra lead recém-capturado sem manual ainda (`estagio_atual`
  = `aguardando_sinais` ou `sinais_coletados`).
- Mostra: nome do negócio, prévia mínima (categoria, se tem logo detectado
  ou não), e botão **"Gerar manual de marca"**.
- Texto de apoio explicando o valor pro vendedor: "gera um manual completo
  de identidade visual pra usar como argumento extra na venda do site."

### 2.2 Gerando / progresso

- Disparado ao clicar "Gerar manual de marca" (ou automaticamente — ver
  `INTEGRACAO-CRM.md` pra recomendação de quando disparar).
- Barra de progresso com as 4 fases (`00-arquitetura/ARQUITETURA-AGENTES.md
  §1`), lendo `estagio_atual` do estado de geração.
- Tempo estimado visível ("normalmente pronto em 2 a 5 minutos").
- Não bloqueia navegação — vendedor pode sair da aba e voltar; o processo
  continua em background.

### 2.3 Pronto

- `estagio_atual=pronto` e `gate_qualidade_ultimo_veredito` em
  `aprovado`/`aprovado_com_ressalvas`.
- Renderiza o manual completo navegável por seção (mesma estrutura de
  `03-template-manual-final/ESTRUTURA-MANUAL.md`: paleta, tipografia, logo,
  aplicações).
- Se `lacunas_abertas_count > 0`: banner discreto no topo ("este manual tem
  N partes que podem ficar melhores com uma conversa rápida com o dono do
  negócio") — nunca escondido, nunca bloqueante.
- Botões:
  - **Exportar PDF** — gera o PDF do conteúdo atual da tela.
  - **Copiar link público** — copia a URL compartilhável (só ativa depois
    da primeira geração bem-sucedida).
  - **Regenerar logo** — dispara só o agent-07 de novo (não o pipeline
    inteiro), útil se o vendedor/dono não gostou das 3 opções.
  - **Marcar como enviado** — grava `status_entrega=enviado_pelo_vendedor`
    e atualiza o CRM (ver `INTEGRACAO-CRM.md`).

### 2.4 Erro / lacunas pendentes que bloqueiam

- `estagio_atual=falhou`, ou `gate_qualidade_ultimo_veredito=reprovado` após
  o teto de tentativas automáticas (`tentativas_geracao` no limite).
- Mensagem honesta pro vendedor: o que faltou (ex: "sinais insuficientes
  pra gerar um manual de qualidade — esse negócio tem poucas informações
  públicas") + ação disponível: **"Tentar novamente"** ou, se o motivo for
  sinal insuficiente, **"Preencher manualmente"** (abre um formulário curto
  pro vendedor inserir os sinais que faltaram, ex: se ele sabe o nome de
  verdade da padaria mas o GBP não tinha descrição).
- Nunca mostra um manual reprovado como se estivesse pronto.

## 3. Onde essa aba se encaixa no fluxo do vendedor

```
Vendedor abre lead → vê site já gerado → abre aba "Manual de marca"
  → gera (ou já está pronto, se disparo foi automático)
  → revisa o manual, decide se envia como está ou pede micro-conversa
    com o dono pra fechar as lacunas
  → clica "Marcar como enviado" ao mostrar/mandar pro dono do negócio
  → CRM registra o evento, novo status de funil disponível
```

## 4. Regra de honestidade da UI (herdada do método BNP)

Nunca a UI deve apresentar um módulo com confiança `baixo`/`ausente` como
se fosse informação certa. Toda lacuna aparece marcada visualmente (ex:
ícone de alerta discreto, não vermelho de erro — é uma lacuna esperada do
processo automático, não uma falha).
