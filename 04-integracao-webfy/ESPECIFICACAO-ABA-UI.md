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

### 2.0 Sem add-on ativo (upsell)

- Estado que precede o 2.1 quando a conta do vendedor não tem o add-on
  premium ativo (ver gate de entitlement em `INTEGRACAO-CRM.md §0`) —
  substitui completamente o estado padrão, nunca aparece junto dele.
- Mostra: nome do negócio (contexto de que o manual existiria pra esse
  lead), texto de valor curto ("gere um manual completo de identidade
  visual pra usar como argumento extra na venda — disponível no plano
  [nome do add-on]"), e CTA **"Conhecer o add-on"** ou **"Ativar agora"**
  (destino exato — tela de billing/upgrade do webfy — é decisão de
  implementação, fora de escopo deste blueprint).
- **Nunca** um erro genérico, campo desabilitado sem explicação, ou botão
  que simplesmente não responde ao clique — falta de add-on é oportunidade
  de upsell, não falha. O tom é o mesmo dos outros textos de apoio da aba:
  direto, sem pressão agressiva.
- Se o vendedor ativar o add-on enquanto está com um lead aberto nesse
  estado, a tela reflete a mudança e passa pro estado 2.1 na próxima
  interação (não precisa recarregar a página inteira, mas também não
  precisa ser tempo real — um refresh normal da aba já resolve).

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

## 6. White-label total — nenhuma marca do webfy visível pro dono do negócio

O manual entregue ao dono do negócio é vendido como propriedade dele — não
pode carregar nenhuma marca, crédito ou menção visível de "webfy" em
lugar nenhum que o dono do negócio final enxerga. Isso cria uma tensão
específica: a tela de PIN (§5) é tecnicamente hospedada pela infraestrutura
do webfy e é a **primeira coisa** que o dono do negócio vê ao abrir o
link — se ela estampar "webfy" (logo, nome, rodapé, favicon, título de
aba do navegador), o white-label quebra antes mesmo do conteúdo carregar.

**Regra:** a tela de PIN, o manual web (dentro e fora do CRM) e o PDF
exportado são **visualmente neutros por padrão**, e **personalizáveis com
a identidade do vendedor/agência** que está usando o webfy pra vender —
nunca com a marca do webfy.

- **Campo `nome_exibicao_vendedor`:** todo ponto de contato do dono do
  negócio com o manual (tela de PIN, cabeçalho/rodapé do manual web,
  capa e rodapé do PDF, qualquer email ou mensagem automática que
  mencione o manual — ex: notificação de link gerado) usa esse campo
  como substituto de qualquer branding, no lugar de "webfy". Se o
  vendedor/agência não preencheu esse campo, o padrão é neutro genérico
  ("Manual de marca" sem atribuição de plataforma) — nunca cai pro nome
  "webfy" como fallback.
- **Onde isso se aplica, explicitamente:**
  - **Tela de PIN (§5):** sem logo/nome "webfy" em nenhum elemento —
    título da aba do navegador, cabeçalho da página, rodapé, favicon.
    Usa `nome_exibicao_vendedor` se preenchido, senão layout neutro.
  - **Manual web (link público e dentro do CRM):** mesma regra — o
    dono do negócio nunca vê "webfy" navegando pelas seções do manual.
    Dentro do CRM, a UI do vendedor (fora da visão do dono do negócio)
    pode seguir com a marca normal do webfy — a regra vale só pro que o
    **dono do negócio final** enxerga, não pra interface interna do
    vendedor.
  - **PDF exportado:** capa, cabeçalho/rodapé de cada página, e
    metadados do arquivo (autor/produtor do PDF) sem menção a "webfy".
  - **Email/mensagem automática:** qualquer notificação automática que
    mencione o manual (ex: "seu manual de marca está pronto", se o webfy
    tiver esse tipo de disparo) usa o mesmo `nome_exibicao_vendedor` no
    remetente/assinatura visível ao dono do negócio — nunca "equipe
    webfy".
- **O que NÃO muda:** a regra é sobre o que o **dono do negócio final**
  vê. A UI interna do vendedor dentro do CRM (a própria aba "Manual de
  marca", os estados 2.0-2.4 descritos acima) pode manter a marca webfy
  normalmente — é ferramenta de trabalho do vendedor, não material
  white-label entregue ao cliente.
