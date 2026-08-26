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
    da primeira geração bem-sucedida). Na primeira vez que este botão é
    usado, gera também o PIN de acesso (ver §5) — o link sozinho não abre
    o manual.
  - **Ver PIN de acesso** — mostra o PIN atual de 6 dígitos pro vendedor
    copiar/repassar junto com o link (ver §5). Sempre visível pro vendedor,
    nunca escondido dele — só o dono do negócio final não vê o PIN em
    lugar nenhum da UI pública, só recebe pelo vendedor.
  - **Regenerar logo** — dispara só o agent-07 de novo (não o pipeline
    inteiro), útil se o vendedor/dono não gostou das 3 opções. **Este botão
    é a única forma de disparar uma nova geração completa de logo depois da
    primeira** — o pipeline nunca gera de novo sozinho (teto de custo, ver
    `SERVICO-GERACAO-DE-IMAGEM.md §6`). Clicar aqui é o "clique explícito
    do vendedor" que a regra exige.
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

## 5. Acesso por PIN ao link público

O link público não é mais um link direto sem proteção — a página pede um
PIN antes de renderizar qualquer conteúdo do manual. Não é um sistema de
contas completo, é fricção mínima: sem cadastro, sem senha reutilizável,
sem email de verificação.

- **Geração:** o PIN é criado junto com o link, na primeira vez que o
  vendedor clica "Copiar link público" (`estado_geracao.pin_acesso`, 6
  dígitos numéricos). Um `lead_id` tem sempre 1 PIN ativo por vez.
- **Entrega:** o vendedor repassa o PIN ao dono do negócio pelo mesmo canal
  que usar pra mandar o link (WhatsApp, email) — a UI mostra o PIN atual
  pro vendedor a qualquer momento via "Ver PIN de acesso" (§2.3).
- **Tela de acesso:** ao abrir o link público, o visitante vê só um campo
  de PIN + botão "Acessar manual". Nenhum conteúdo do manual é enviado ao
  navegador antes da validação — o PIN errado não deve nem revelar se o
  link é válido além do próprio erro genérico.
- **Tentativas:** recomendação de teto **5 tentativas erradas consecutivas**
  antes de bloquear (`estado_geracao.pin_bloqueado=true`). Bloqueio é por
  link/lead, não por IP — evita que um visitante legítimo com PIN certo
  fique travado por causa de tentativas erradas de outra pessoa, mas também
  não deveria ser tão frouxo a ponto de permitir força bruta de 6 dígitos
  (recomendação: rate-limit adicional por IP como camada extra, decisão de
  implementação do webfy).
- **Reenvio/reset:** o vendedor pode gerar um novo PIN a qualquer momento
  (ação "Gerar novo PIN", disponível mesmo sem estar bloqueado) — isso
  invalida o PIN anterior, zera `pin_tentativas_falhas` e limpa
  `pin_bloqueado`. É o fluxo padrão tanto pra bloqueio quanto pra "dono do
  negócio esqueceu o PIN": o vendedor não reenvia o PIN antigo, gera um
  novo e reenvia esse.
- **Escopo do PIN:** protege o link público (visão do dono do negócio fora
  do CRM). Não afeta a aba dentro do CRM do webfy, que já tem seu próprio
  controle de acesso de vendedor/conta.
