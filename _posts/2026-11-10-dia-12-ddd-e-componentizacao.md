---
title: "Dia 12 — DDD e componentização: o idioma de produto que eu já falava sem saber"
date: 2026-11-10
categories: [desafio-fde]
tags: [fde, ddd, linguagem-ubiqua, bounded-context, camada-anticorrupcao, camada-semantica, componentizacao]
excerpt: "Uma sonda de 125 milhões de dólares se perdeu em Marte porque dois times usavam a mesma palavra com unidades diferentes. Hoje o assunto é linguagem: por que o glossário é o primeiro artefato de um deployment, como a palavra 'pagamento' muda de sentido três vezes dentro da mesma empresa, por que um bom servidor MCP de ERP é uma camada anticorrupção, o que um benchmark de 2023 mostrou sobre agentes em cima de tabela crua e por que só se reaproveita peça que tem contrato."
---

## O que estamos construindo

Desde o Dia 3 eu uso um único caso para não deixar nada no abstrato: um **agente de conciliação financeira**.

Conciliar é casar duas listas que deveriam bater e quase nunca batem de primeira. De um lado, o que entrou no banco: PIX, boleto, TED, recebimento de cartão. Do outro, o que a empresa cobrou: as cobranças e notas fiscais emitidas no ERP. Para cada recebimento, alguém precisa dizer a qual cobrança ele corresponde, dar baixa e seguir para o próximo.

O trabalho do agente cabe em quatro passos: buscar os recebimentos no extrato, procurar a cobrança candidata no ERP, propor o casamento com uma justificativa e, quando não tiver segurança, escalar para uma pessoa. Ele nunca efetiva uma baixa sozinho.

O que a série já construiu:

| Dia | O que entrou no agente |
|---|---|
| 3 | A anatomia: o modelo só fala, quem executa é o harness |
| 4 | O que ele vê: system prompt, poucos exemplos e três tools sem sobreposição |
| 5 | Quando ele para: os quatro critérios de parada e o graph que separa código, LLM e humano |
| 6 | Como ele fala com o banco e com o ERP: tools, idempotência e MCP |
| 7 | Como ele aguenta o mundo real: fila, retry, DLQ, human-in-the-loop e a conta de custo |
| 8 | Como eu provo, com número, que ele pode trabalhar |
| 9 | O que impede ele de fazer besteira, e como eu enxergo o que ele fez |
| 10 | Por que alguém pagaria por ele: processo, exceção e ROI |
| 11 | Quem precisa dizer sim, o que está no contrato e como se cobra |
| 12 | Hoje: as palavras que ele usa, as fronteiras entre as áreas e as peças com contrato |

O caso é inspirado num processo financeiro real que eu conheço bem, mas tudo aqui é anônimo e os números dos exemplos são fictícios.

---

Em setembro de 1999, a NASA perdeu a Mars Climate Orbiter, uma sonda de cerca de 125 milhões de dólares, depois de mais de nove meses de viagem até Marte. Na manobra de chegada, ela passou baixo demais pela atmosfera do planeta e nunca mais deu sinal.

O relatório da comissão de investigação, publicado em novembro daquele ano, chegou a uma causa que parece piada e não é. Um software em terra, mantido por um time, entregava o impulso dos propulsores em libra-força segundo. O software de navegação, mantido por outro time, lia aquele número como se estivesse em newton-segundo, que era o que a especificação de interface mandava. A diferença entre as duas unidades é de 4,45 vezes. Durante meses, cada pequena correção de rota foi subestimada nessa proporção, e ninguém conferiu a unidade no ponto em que o dado passava de um time para o outro.

Os dois times estavam certos dentro do próprio mundo. O número era o mesmo, o campo tinha o mesmo nome, e a palavra "impulso" significava uma coisa para quem escrevia e outra para quem lia. A sonda se perdeu na fronteira.

Eu começo por essa história porque o assunto de hoje é exatamente esse: **o que acontece com uma palavra quando ela atravessa a fronteira entre duas áreas.** E num deployment de agente isso deixa de ser detalhe de arquitetura, porque o agente é feito de palavras. O system prompt é texto. A descrição de cada tool é texto. O nome de cada campo que volta do ERP é lido por um modelo que não tem como levantar a mão e perguntar "mas o que vocês querem dizer com isso?".

O nome técnico do assunto é **Domain-Driven Design**, e eu confesso que demorei para levar a sério, porque o termo tem cara de coisa de arquiteto de software com quadro branco cheio de caixinha. Quando eu finalmente li com calma, encontrei uma coisa que eu conheço há muito tempo. Antes de dados e IA, a minha formação passou por Letras, e o problema central do DDD é um problema de semântica: decidir o que cada palavra significa, para quem, e onde esse significado deixa de valer.

## 1. DDD em uma frase, e por que ela importa mais com agente

O *Domain-Driven Design* foi organizado por Eric Evans num livro de 2003, e a ideia central cabe numa frase: **o software deve ser construído a partir do modelo do negócio, e não a partir da tecnologia.** Quem desenvolve senta com quem conhece o negócio, aprende como essa pessoa pensa e fala, e o código passa a refletir esse jeito de pensar, com os mesmos nomes e as mesmas regras.

Parece óbvio escrito assim, e quase ninguém faz. O caminho natural de quem é técnico é o contrário: começar pela tabela, pela API, pelo framework, e depois tentar encaixar o negócio no que já foi construído. O resultado é o sistema que todo mundo já viu, em que o campo se chama `flag_2`, a regra mais importante está num `if` escondido e só uma pessoa sabe por que o status `7` significa "cancelado, mas com crédito".

Com agente, esse problema piora, por um motivo simples. No software tradicional, a ambiguidade do nome fica escondida dentro do código, e o código não interpreta nada: ele faz o que está escrito. Num agente, o modelo **lê** os nomes e decide a partir deles. Se a tool devolve um campo `status: "B"`, o modelo vai tentar adivinhar o que `B` quer dizer, e às vezes vai adivinhar com muita confiança.

Em março de 2024, na abertura de uma conferência dedicada ao tema, o próprio Evans disse uma frase que eu achei certeira: na linguagem de quem pratica DDD, **um modelo de linguagem treinado é um bounded context**. Ele estava falando de modelos ajustados para um domínio específico, mas a ideia vale também para quem só escreve prompt: o agente opera dentro de um vocabulário, e esse vocabulário precisa ter fronteira. Vou explicar o que é bounded context na seção 4. Antes, é preciso entender onde vale a pena gastar esforço.

## 2. Domínio e subdomínios: onde colocar o melhor time

O **domínio** é a razão de o negócio existir. No nosso caso, é a gestão financeira dos recebimentos de uma empresa: saber quem pagou, quanto, a que se refere, e quanto dinheiro existe de verdade no caixa.

Dentro do domínio há partes que valem muito mais do que outras, e o DDD separa essas partes em três tipos.

| Tipo | O que é | Na conciliação | O que fazer |
|---|---|---|---|
| **Core** | O que diferencia, e onde está a vantagem | Casar recebimento com cobrança nos casos difíceis, e transformar isso em visão de caixa | Construir, com o melhor time e o melhor cuidado |
| **Suporte** | Necessário, mas não diferencia | Cadastro de clientes, regras de alçada, a fila de exceções | Construir simples, ou adaptar algo pronto |
| **Genérico** | Problema resolvido no mercado | Autenticação, emissão de nota fiscal, leitura de extrato em formato padrão | Comprar ou usar pronto, sem culpa |

Essa tabela parece burocrática, mas ela responde a uma pergunta que todo projeto de agente enfrenta na segunda semana: **onde gastar o tempo do FDE?**

A tentação é gastar onde é divertido. Montar um parser próprio de extrato bancário é um problema técnico gostoso, com formato estranho, campo que muda de banco para banco e uma vitória clara no final. Só que extrato é genérico: existem padrões de arquivo, APIs de banco e agregadores que resolvem isso. Cada semana gasta ali é uma semana a menos no core, que é o pagador divergente, o pagamento agrupado de três cobranças num PIX só, o desconto não combinado que deixa o valor três reais abaixo do esperado. É ali que o agente ganha ou perde a conta do Dia 10.

Tem um efeito comercial também. Quando você mostra ao cliente que identificou o que é core e que vai usar pronto o que é genérico, você está dizendo que entende o negócio dele melhor do que a vontade de construir coisa. Isso é argumento de escopo, e é exatamente o tipo de corte que deixou o SOW do Dia 11 mais crível.

## 3. Linguagem ubíqua: o primeiro artefato é um glossário

A **linguagem ubíqua** é a prática de usar o **mesmo termo, com o mesmo significado**, em todo lugar: na conversa com o negócio, no documento de produto, no código e, agora, também no prompt e nas tools do agente.

O Martin Fowler escreveu um verbete curto sobre isso em 2006, e duas frases dele resumem o espírito. A primeira é que a linguagem precisa ser *rigorosa*, porque *"o software não lida bem com ambiguidade"*. A segunda vem do Evans e é a que mais me interessa: quem conhece o negócio *deve reclamar* de termos que soam estranhos ou que não dão conta do que ele entende do domínio. Ou seja, o glossário não é algo que o time técnico escreve e o negócio aprova; ele é negociado, e a palavra final sobre o significado é de quem vive o processo.

O exemplo clássico no nosso caso é a **baixa**. Para o financeiro, dar baixa é registrar que uma cobrança foi liquidada. Se o código chama isso de `settlement_update`, de `close_invoice` ou de `status = 'L'`, nasce uma tradução, e toda tradução é um lugar onde o significado pode se perder. Quando o analista diz "essa baixa foi parcial", alguém precisa saber que isso corresponde a um registro em que o valor recebido é menor que o da cobrança e a cobrança continua aberta pela diferença. Se ninguém escreveu isso, cada pessoa vai implementar a sua versão.

Com agente, isso tem consequência medida. Em setembro de 2025, a Anthropic publicou um guia de engenharia sobre como escrever tools para agentes, e duas recomendações dele são linguagem ubíqua com outro nome. A primeira é descrever a tool *como você descreveria para alguém novo no time*, explicitando o que para você é óbvio, inclusive o jargão. A segunda é trocar identificadores técnicos por nomes que fazem sentido: segundo o texto, apenas resolver códigos alfanuméricos arbitrários em linguagem com significado melhorou a precisão do modelo em tarefas de busca, reduzindo alucinação.

Por isso eu passei a considerar o glossário o **primeiro artefato de qualquer deployment**, antes do diagrama de arquitetura e antes do primeiro prompt. Ele é barato, ele é feito nas mesmas entrevistas do discovery do Dia 10, e ele resolve problemas que de outro jeito só aparecem no shadow mode.

O formato que eu uso tem cinco colunas, e a quarta é a que faz o trabalho:

| Termo | Significa | Em qual contexto | Não confundir com | Nome no código e no prompt |
|---|---|---|---|---|
| Recebimento | Crédito que de fato apareceu no extrato do banco | Financeiro | Comprovante enviado pelo cliente | `Recebimento` |
| Cobrança | O valor que a empresa emitiu para alguém pagar | Financeiro | Nota fiscal (que é o documento fiscal) | `Cobranca` |
| Título | O nome que o ERP dá para a cobrança | Sistema legado | Nada: é sinônimo, e só existe na borda | traduzido para `Cobranca` |
| Baixa | Registro de que a cobrança foi liquidada, no todo ou em parte | Financeiro | Conciliação (que é o casamento, a baixa vem depois) | `baixar_cobranca`, só humano |
| Baixa parcial | Recebimento menor que a cobrança, que continua aberta pela diferença | Financeiro | Desconto (que encerra a cobrança) | `status: parcial` |
| Pagador | Quem fez a transferência | Banco | Cliente (quem deve) | `pagador_documento` |
| Cliente | Quem tem o contrato e a dívida | Comercial e Financeiro | Pagador | `cliente_id` |
| Pago, no cartão | Venda aprovada; o dinheiro chega em outra data, já descontada a taxa | Comercial | Liquidado | `venda_aprovada` |
| Liquidado | Dinheiro do cartão que entrou de fato na conta | Financeiro | Pago, no cartão | `liquidacao` |
| Exceção | Recebimento que o agente não casou com segurança e mandou para uma pessoa | Conciliação | Erro (o agente errar é outra coisa) | `Excecao` |

Repare na última coluna: o mesmo nome aparece na tabela do banco de dados, no esquema da tool e no system prompt. Quando o analista disser "pagador divergente", o modelo vai encontrar `pagador_documento` diferente do documento do `cliente_id`, e a frase do analista, a regra do prompt e o dado da tool vão estar falando da mesma coisa.

## 4. Bounded context: a palavra "pagamento" muda de sentido três vezes

Aqui o DDD dá o passo que, para mim, é o mais inteligente de todo o método.

A tentação de quem faz um glossário é querer um significado único para cada palavra, válido para a empresa inteira. O Evans diz, e o Fowler repete, que isso não funciona: *a unificação total do modelo de domínio de um sistema grande não vai ser viável nem compensar o custo*. O exemplo do Fowler é de uma empresa de energia elétrica em que a palavra "medidor" significava coisas sutilmente diferentes para áreas diferentes: para uns era a ligação entre a rede e um endereço, para outros a ligação entre a rede e um cliente, para outros ainda o aparelho físico. Em conversa, isso se resolve sozinho. Num computador, não.

A saída é o **bounded context**, ou contexto delimitado: **a fronteira dentro da qual um termo significa uma coisa só.** Em vez de um modelo único para a empresa toda, o sistema é dividido em contextos, cada um com o seu modelo coerente, e a tradução entre eles fica explícita.

Quem estudou filosofia da linguagem vai reconhecer o eco. Nas *Investigações Filosóficas*, o Wittgenstein escreveu que, para uma grande classe de casos, o significado de uma palavra é o seu uso na linguagem. Ele tomou o cuidado de dizer "uma grande classe", e não "todos", e eu mantenho a ressalva. Mas a intuição serve muito bem aqui: "pagamento" não tem um significado guardado em algum lugar. Tem os usos que cada área faz da palavra, e cada uso tem as suas regras.

No caso da conciliação, ela muda de sentido pelo menos três vezes:

<figure>
  <img src="{{ '/assets/images/desafio-fde/dia-12-pagamento-tres-contextos.svg' | relative_url }}" alt="Três colunas, uma por contexto, mostrando o que a palavra pagamento significa em cada um. Atendimento, em laranja: pagamento é o cliente dizendo que pagou; a evidência é um comprovante em PDF ou print enviado por e-mail ou WhatsApp; o risco, se o agente usar esse sentido, é dar baixa num comprovante que pode estar errado, repetido ou ser falso, e um PDF enviado por terceiro também é porta de injeção de instrução. Financeiro, em verde, com destaque: pagamento é o crédito que apareceu no extrato do banco; a evidência é o lançamento bancário com valor, data e pagador; é o sentido que vale para conciliar, e o único que autoriza propor uma baixa. Fiscal, em cinza: pagamento é um evento que precisa ter a nota fiscal correspondente, no valor e na competência certos; a evidência é a nota emitida e a sua relação com o recebimento; o risco, se o agente usar esse sentido, é tratar como resolvido algo que está pago mas sem nota, ou com nota em valor diferente. Uma faixa escura no pé diz: três verdades sobre o mesmo dinheiro, e o agente de conciliação mora no contexto financeiro; os outros dois sentidos entram por tradução explícita, nunca crus.">
  <figcaption>Nenhum dos três está errado. O erro é o agente não saber em qual deles está.</figcaption>
</figure>

Para o **atendimento**, pagamento é o que o cliente diz: "já paguei, segue o comprovante". Para o **financeiro**, pagamento é o crédito que apareceu no extrato. Para o **fiscal**, pagamento é um evento que precisa ter uma nota correspondente, no valor certo e na competência certa. Os três estão corretos dentro do próprio contexto, e são três verdades diferentes sobre o mesmo dinheiro.

Agora imagine o system prompt com a instrução "verifique se o cliente já pagou antes de escalar". Se o agente tiver acesso à caixa de entrada do atendimento, um comprovante em PDF anexado a um e-mail vai parecer uma resposta perfeitamente razoável. E é aqui que a ambiguidade de palavra vira problema de segurança: no Dia 9 eu mostrei que documento enviado por terceiro é a porta clássica de injeção indireta de instrução. Um comprovante forjado, ou só repetido de um mês anterior, viraria baixa proposta com justificativa convincente.

A correção não é um prompt mais esperto. É **decidir em qual contexto o agente mora.** O agente de conciliação mora no contexto financeiro, e lá pagamento é crédito no extrato. O comprovante do atendimento pode entrar, mas entra traduzido como o que ele é, uma *alegação de pagamento*, que serve para procurar o recebimento correspondente e nunca para substituí-lo.

E "pagamento" não é o único caso. Eu encontrei pelo menos mais três no mesmo processo:

- **Cliente.** Para o comercial, é quem assinou o contrato. Para o banco, só existe o pagador, que muitas vezes é outra pessoa: a filial, o sócio, a empresa do mesmo grupo. A categoria inteira de exceção que eu chamei de *pagador divergente* no Dia 10 nasce dessa diferença de sentido.
- **Valor.** O valor da cobrança, o valor recebido e o valor líquido do cartão depois da taxa são três números diferentes, e os três costumam aparecer na mesma tela com o nome "valor".
- **Pago, no cartão.** Para o comercial, a venda aprovada já está paga. Para o financeiro, o dinheiro só existe na data da liquidação, semanas depois e com desconto. É a Mars Climate Orbiter em versão contábil: o mesmo número, a mesma palavra, e uma diferença que só aparece quando alguém soma.

## 5. Context map: como os contextos conversam

Se cada contexto tem o seu modelo, alguém precisa desenhar como eles se relacionam. Esse desenho se chama **context map**, e o DDD tem um catálogo de tipos de relação. O repositório aberto da comunidade DDD Crew mantém uma boa referência visual deles, e três já resolvem quase tudo num deployment.

**Cliente–fornecedor.** Um contexto consome o que o outro produz, e o consumidor tem voz no que é produzido. No nosso caso, a conciliação consome as cobranças que o faturamento emite. Se o faturamento passar a emitir PIX Cobrança com `txid`, aquela concessão de escopo do Dia 11, a vida da conciliação melhora muito. Essa é uma conversa entre contextos, e ela só acontece quando alguém desenha a relação.

**Kernel compartilhado.** Dois contextos dividem um pedaço do modelo, por exemplo o cadastro de clientes. É útil e perigoso, porque qualquer mudança nesse pedaço exige acordo dos dois lados. Na prática, todo kernel compartilhado precisa de um dono com nome. Sem isso, ele vira a tabela que todo mundo altera e ninguém mantém.

**Camada anticorrupção.** Esta é a que mais importa para quem faz deployment, e merece uma seção própria.

<figure>
  <img src="{{ '/assets/images/desafio-fde/dia-12-context-map.svg' | relative_url }}" alt="Mapa de contextos do caso da conciliação. No centro, em verde, o contexto de conciliação, marcado como core, onde mora o agente, com o modelo canônico: Recebimento, Cobrança, Proposta e Exceção. À esquerda, dois sistemas externos em cinza: o banco e o adquirente de cartão, com extrato e liquidação; e o ERP legado, com títulos, códigos de situação de uma letra e campos de nome curto. Entre cada sistema externo e o centro há uma caixa laranja de camada anticorrupção, implementada como servidor MCP: a do banco traduz lançamento bancário em Recebimento; a do ERP traduz título em Cobrança e código de situação em status com nome. À direita, dois contextos internos: atendimento, ligado ao centro por uma seta tracejada com a etiqueta alegação de pagamento, nunca baixa; e fiscal, ligado ao centro numa relação de cliente e fornecedor, com a etiqueta recebimento conciliado, com nota. Abaixo, uma faixa liga faturamento à conciliação, com a etiqueta cliente e fornecedor: cobrança emitida, de preferência com txid. Uma faixa escura no pé diz: o legado não entra cru no modelo do agente; toda fronteira tem um tradutor com dono.">
  <figcaption>O agente só enxerga o centro. Tudo que vem de fora chega traduzido.</figcaption>
</figure>

## 6. A camada anticorrupção, ou o que um bom servidor MCP de ERP faz

A **camada anticorrupção** (ACL, de *anti-corruption layer*) também foi descrita pelo Evans no livro de 2003. É uma camada que fica entre o seu sistema e um sistema que você não controla, normalmente um legado, e **traduz o modelo de lá para o seu**, para que a bagunça de lá não contamine o que você está construindo.

O Azure Architecture Center, da Microsoft, tem uma página sobre o padrão que eu recomendo, porque ela é honesta sobre o custo. A camada acrescenta latência, é mais um serviço para manter e monitorar, e precisa de observabilidade própria, com identificador de correlação e log estruturado, para que dê para diagnosticar quando a tradução falha. A página também faz duas recomendações que eu levo a sério. A primeira é que a ACL deve conter **tradução, e não regra de negócio**. A segunda é que, como ela fica entre sistemas com níveis de confiança diferentes, ela é um bom lugar para **validar e limpar a entrada**.

O servidor MCP do ERP que eu desenhei no Dia 6 é exatamente isso, embora eu ainda não tivesse esse nome para ele. Um ERP de dez anos devolve uma linha mais ou menos assim:

```
NR_TITULO   VL_TITULO   DT_VENC    CR_SIT  COD_PORT  VL_ABAT
000881      1500.00     20261105   A       341       0.00
```

E o agente deveria receber isto:

```json
{
  "cobranca_id": "881",
  "cliente_id": "C-2210",
  "valor": { "centavos": 150000, "moeda": "BRL" },
  "vencimento": "2026-11-05",
  "status": "aberta",
  "saldo_em_aberto": { "centavos": 150000, "moeda": "BRL" }
}
```

A diferença parece cosmética, e ela decide a qualidade do agente. Na versão crua, o modelo precisa adivinhar que `CR_SIT = A` quer dizer "aberta" e não "aprovada", que `COD_PORT 341` é um código de banco e não um código de produto, e que a data está num formato sem separador. Na versão traduzida, nada disso é pergunta. E repare no valor: ele vem em centavos, com a moeda explícita. É a lição da sonda aplicada a dinheiro, com a unidade dentro do dado em vez de numa especificação que alguém precisa lembrar de ler.

Três regras que eu adotei para a ACL de um agente:

1. **Ela devolve o modelo canônico, e só ele.** Nenhum campo do legado passa direto "porque pode ser útil". Se pode ser útil, ele ganha um nome do glossário e entra traduzido.
2. **Código desconhecido é erro explícito, nunca adivinhação.** Se o ERP devolver um `CR_SIT = X` que ninguém mapeou, a ACL não chuta: ela devolve um erro claro, e o caso vai para a fila de exceção. Silêncio na tradução é o tipo de falha que só aparece no fechamento.
3. **Ela não decide nada.** Regra de alçada, regra de casamento e regra de quando escalar ficam no graph e no guardrail, como no Dia 5 e no Dia 9. A ACL traduz. Quando ela começa a decidir, vira um segundo sistema de regras escondido dentro de um conector.

Existe uma alternativa à ACL que o catálogo chama de **conformista**: aceitar o modelo do legado como ele é, e usar `CR_SIT` dentro do seu próprio sistema. Às vezes é a decisão certa, por exemplo num protótipo de duas semanas que vai ser jogado fora. Mas com agente, o conformista paga um preço extra, porque cada nome ruim do legado vai parar no contexto do modelo e vira uma chance de erro em cada chamada.

## 7. Os blocos táticos, e por que evento de domínio é gatilho de agente

O DDD tem também um vocabulário mais próximo do código. Para quem faz deployment, basta reconhecer quatro peças, e a quarta tem uma consequência direta para agentes.

**Entidade** é algo com identidade própria, que continua sendo a mesma coisa mesmo quando os atributos mudam. Um cliente é uma entidade: ele muda de endereço e continua sendo o mesmo cliente.

**Objeto de valor** é algo definido só pelos atributos. Um valor de R$ 1.500,00 é um objeto de valor: dois "R$ 1.500,00" são iguais, e não faz sentido perguntar "qual dos dois". É aqui que a unidade mora. Se o tipo `Dinheiro` sempre carrega centavos e moeda, ninguém consegue somar real com dólar nem centavo com real por descuido.

**Agregado** é um grupo de objetos que precisa ficar consistente junto, com uma porta de entrada única. A cobrança com as suas parcelas é um agregado: você não dá baixa numa parcela sem que a cobrança saiba, porque o saldo em aberto é uma regra do conjunto. O Fowler tem um verbete curto sobre o conceito que vale a leitura.

**Evento de domínio** é algo que aconteceu e que importa para o negócio, escrito no passado: `PagamentoRecebido`, `CobrancaEmitida`, `ConciliacaoProposta`, `ExcecaoAberta`, `BaixaDesfeita`.

O evento de domínio é o **gatilho natural de um agente**. Em vez de um agente que roda a cada hora varrendo o extrato inteiro para ver se tem algo novo, você tem uma regra que se lê como o negócio fala: *quando `PagamentoRecebido`, tentar conciliar; quando `ExcecaoAberta`, notificar a fila do analista; quando `BaixaDesfeita`, registrar como caso de eval.* Isso tem três vantagens que se somam ao que a série já construiu. O custo cai, porque o agente só roda quando há trabalho. O trace fica legível, porque cada execução começa com um fato de negócio que qualquer pessoa entende. E a última regra fecha um ciclo do Dia 8: cada baixa desfeita vira, automaticamente, um caso novo na suíte de eval.

## 8. Ontologia, camada semântica e por que agente rende mais em cima delas

Tem uma ponte aqui que eu acho a parte mais útil do dia para quem vem de dados, como eu.

O Foundry, plataforma da Palantir, organiza os dados do cliente no que eles chamam de **Ontologia**. A documentação pública define a Ontologia como uma *camada operacional da organização*: os dados existentes são mapeados em **objetos, propriedades e ligações**, como armazém, pedido, cliente e as relações entre eles, e a Ontologia funciona como um *gêmeo digital* da empresa. Além desses elementos semânticos, ela guarda as **ações** que podem ser feitas sobre os objetos. É, na prática, um modelo de domínio no sentido do DDD, montado em cima de dados em vez de código.

Quem trabalha com BI já conhece um parente próximo: o **modelo semântico**, aquele em que as tabelas viram medidas, dimensões e relacionamentos com nome de negócio. A tabela `fct_tr_041` vira "Receita líquida", e a métrica é calculada do mesmo jeito em todo relatório. Eu passei anos construindo esse tipo de camada para pessoas lerem dashboards. A novidade é que agora quem mais se beneficia dela é o modelo de linguagem.

Existe um número sobre isso, e ele é interessante. Em novembro de 2023, três pesquisadores da data.world publicaram um benchmark sobre perguntas de negócio feitas a um banco de dados corporativo do setor de seguros. Com o GPT-4 respondendo direto em cima das tabelas SQL, sem exemplos, a taxa de acerto foi de cerca de **16%**. Quando as mesmas perguntas foram feitas sobre uma representação do mesmo banco em grafo de conhecimento, com os conceitos de negócio explícitos, a taxa subiu para cerca de **54%**. Nas perguntas mais difíceis, com esquema complexo, o acerto em cima do SQL cru foi zero.

Eu leio esse número com três ressalvas. É um benchmark só, num domínio só, com um modelo de 2023, e os modelos de hoje acertam bem mais. Os autores trabalham numa empresa que vende exatamente esse tipo de camada. E 54% ainda é pouco para colocar em produção. Mas a direção é a mesma de tudo que este artigo defende: **um agente em cima de uma camada semântica limpa erra menos do que um agente em cima de tabela crua**, e a diferença vem do mesmo lugar de onde veio a perda da sonda. O modelo não precisa adivinhar o que cada coisa significa.

No Dia 11 eu citei a apresentação da AWS sobre a sua equipe de FDE, em que o ativo mais valioso deixado no cliente seria justamente a camada semântica. Juntando as peças, o glossário da seção 3, o modelo canônico da ACL e a camada semântica são três formas do mesmo artefato. E ele sobrevive ao agente: se amanhã o modelo mudar, o framework mudar ou o próprio agente for desligado, a camada continua servindo para pessoas, relatórios e o próximo caso de uso.

## 9. Componentização: só se reaproveita o que tem contrato

A segunda metade do dia sai da linguagem e vai para a forma da solução. A ideia é simples de dizer: **quebre a solução em peças, e dê a cada peça um contrato com quatro elementos — entrada, saída, erros e dono.**

Os três primeiros são técnicos e já apareceram na série, quando eu falei do esquema de cada tool no Dia 6. O quarto é o que quase sempre falta. Uma peça sem dono não tem quem decida se ela pode mudar, quem acorda quando ela quebra e quem aprova uma versão nova. No Dia 11, eu disse que o teste de saída de um deployment é "dá para rodar sem o fornecedor?". O dono de cada componente é a resposta concreta a essa pergunta.

Num deployment de agente, os componentes costumam ser estes, e cada um tem um tipo de contrato diferente:

| Componente | O contrato |
|---|---|
| **Conector / servidor MCP** | As tools que expõe, a autenticação, os limites de chamada |
| **Tool** | Os parâmetros, o retorno, os erros e a idempotência |
| **Prompt** | O papel, as regras, o formato de saída e a versão |
| **Suíte de eval** | Os casos, os graders e o limiar de aprovação |
| **Política de guardrail** | Os limites de ação, de dado e de custo |
| **Esquema canônico** | O modelo limpo do domínio: `Recebimento`, `Cobranca`, `Proposta`, `Excecao` |

Aplicando ao nosso caso, a conciliação se quebra em sete peças:

<figure>
  <img src="{{ '/assets/images/desafio-fde/dia-12-componentes-com-contrato.svg' | relative_url }}" alt="Sete cartões, um por componente do agente de conciliação, cada um com entrada, saída, erro e dono. Um, conector do banco, servidor MCP: entra conta e período de até 31 dias; sai lista de Recebimento; erra com período longo demais ou banco indisponível, com retry; dono, integração com aval da tesouraria. Dois, camada anticorrupção do ERP, servidor MCP, destacado em laranja: entra cliente, valor e vencimento; sai lista de Cobrança no modelo canônico; erra com código desconhecido como erro explícito, nunca adivinhação; dono, o dono do dado mais o FDE. Três, tool conciliacao_propor: entra recebimento, cobrança, justificativa e confiança; sai proposta com id idempotente; erra com duplicata ou valor divergente; dono, o FDE. Quatro, prompt do agente: entra recebimento e candidatas; sai proposta ou escalonamento no formato fixo; erra com saída fora do esquema, rejeitada pelo validador; dono, FDE com as regras assinadas pela champion; versionado. Cinco, suíte de eval: entra uma versão de prompt, modelo e tools; sai acerto por categoria e taxa de baixa errada; erra quando fica abaixo do limiar e bloqueia o deploy; dono, os analistas, autores dos casos. Seis, política de guardrail: entra uma ação proposta; sai permitir, bloquear ou pedir humano; erra com bloqueio registrado no trace; dono, segurança. Sete, esquema canônico, em verde, destacado: define Recebimento, Cobrança, Proposta e Exceção, com dinheiro sempre em centavos e moeda; erra quando há mudança sem versão; dono, o glossário, com o financeiro como autoridade sobre o significado. Uma faixa escura no pé diz: a peça que tem contrato vai para o próximo cliente; a que não tem fica no primeiro, para sempre.">
  <figcaption>O quarto campo de cada cartão, o dono, é o que transforma um projeto em algo que roda sem você.</figcaption>
</figure>

Duas coisas me chamam a atenção nessa quebra.

A primeira é que **o prompt é um componente como os outros**, com entrada, saída, versão e dono. Ainda é comum ver prompt tratado como texto solto, editado direto em produção por quem estiver com pressa. Num agente que mexe com dinheiro, isso equivale a alterar código sem commit. Com contrato, uma mudança no prompt passa pela suíte de eval do componente 5 antes de ir para produção, exatamente como qualquer outra mudança.

A segunda é que **os componentes que vão para o próximo cliente são os que têm contrato**. O conector do banco, a ACL de um ERP comum no mercado, a política de guardrail e o esquema canônico de recebimento são peças que um segundo projeto de conciliação usaria quase sem mudança. Já uma solução sem contrato, em que o parser, a regra e o prompt estão misturados num arquivo só, funciona no primeiro cliente e precisa ser refeita do zero no segundo. Eu gosto de pensar nela como uma estrada de terra: dá para passar, mas você vai pagar o caminho de novo cada vez que precisar usá-lo.

É essa a ponte com o Dia 13. Quando eu falar de N→1, de transformar muitos projetos parecidos numa coisa só, o pré-requisito vai ser este: só vira produto o que foi construído em peças com contrato.

## 10. Onde o DDD atrapalha

Eu não quero terminar como se o DDD fosse resposta para tudo, porque ele não é, e eu já vi ele atrapalhar.

O primeiro problema é **excesso de cerimônia.** O DDD tem um vocabulário grande, e é fácil um time passar semanas discutindo se algo é entidade ou objeto de valor num sistema que é, honestamente, um formulário com uma tabela. Para um agente que lê três campos e responde uma pergunta, um glossário de uma página e um esquema canônico simples bastam. O valor do método está em linguagem ubíqua e bounded context. O resto é ferramenta para quando a complexidade aparecer de verdade.

O segundo é **o glossário que vira documento de compliance.** Se o glossário mora num arquivo que ninguém abre, ele não é linguagem ubíqua: é um documento a mais. O teste que eu uso é ver se o termo do glossário aparece no prompt, no nome da tool e no nome do campo. Se aparece só no documento, ele não está sendo usado.

O terceiro é **desenhar fronteira sem olhar para as pessoas.** Os contextos de um sistema costumam acompanhar os times que conversam entre si, e as fronteiras que funcionam são, muitas vezes, as mesmas do organograma. Um context map bonito que corta um time no meio vai ser contornado na primeira semana. Aqui o assunto do Dia 11 volta: cada fronteira do mapa é também uma fronteira entre pessoas com interesses diferentes, e o tradutor de cada fronteira precisa de um dono que as duas partes aceitem.

## Prática: o glossário, o mapa e as peças

Se você está desenhando um deployment agora, três exercícios cabem numa manhã.

**1. Glossário de dez termos.** Use o formato de cinco colunas da seção 3, e marque os termos que mudam de sentido entre áreas. Se você não encontrou nenhum, as entrevistas foram rasas: volte e pergunte para duas áreas diferentes o que cada uma entende por "pago", "cliente" ou "concluído".

**2. Context map.** Desenhe os contextos do processo, os sistemas externos, e o que é traduzido em cada fronteira. Marque onde entra uma camada anticorrupção e quem é o dono dela. Se uma seta não tem tradutor, pergunte o que acontece quando o sistema de origem mudar um código.

**3. Componentes com contrato.** Quebre a solução em cinco a sete peças, com entrada, saída, erro e dono. A peça que ficar sem dono é a primeira pergunta da próxima reunião.

E três frases que eu quero ter prontas na próxima conversa de desenho:

- *"Antes de desenhar o agente, eu faria o glossário do domínio. O prompt é feito de linguagem."*
- *"Pagamento significa a mesma coisa para o financeiro e para o fiscal? Acho que temos dois contextos aqui."*
- *"Esse conector deveria ser uma camada anticorrupção. O legado não entra cru no modelo do agente."*

## TLDR

- **A Mars Climate Orbiter se perdeu numa fronteira**, porque dois times usavam o mesmo número com unidades diferentes e ninguém conferiu na passagem. Agente tem o mesmo risco, multiplicado, porque é feito de palavras.
- **DDD é construir a partir do modelo do negócio**, com os nomes e as regras de quem vive o processo, e não a partir da tabela ou do framework.
- **Com agente, ambiguidade de nome vira decisão errada**, porque o modelo lê o nome e age a partir dele.
- **Separe core, suporte e genérico.** O tempo do FDE vai para o core; o genérico se compra pronto.
- **O primeiro artefato de um deployment é o glossário**, com uma coluna de "não confundir com" e o nome que o termo terá no código e no prompt.
- **Nomes com significado melhoram o agente de forma medida**: a Anthropic relata ganho de precisão só por trocar códigos arbitrários por linguagem.
- **Bounded context é a fronteira onde uma palavra significa uma coisa só.** Unificar o modelo da empresa inteira não é viável, e o DDD não tenta.
- **"Pagamento" muda de sentido três vezes**: alegação no atendimento, crédito no extrato no financeiro, evento com nota no fiscal. O agente precisa saber em qual contexto mora.
- **Comprovante de cliente é alegação, não pagamento**, e também é porta de injeção de instrução.
- **Context map tem três relações que resolvem quase tudo**: cliente–fornecedor, kernel compartilhado e camada anticorrupção.
- **Um bom servidor MCP de ERP é uma camada anticorrupção**: devolve só o modelo canônico, trata código desconhecido como erro e não decide nada.
- **Dinheiro em centavos e com moeda explícita** é a lição da sonda aplicada a finanças: a unidade mora dentro do dado.
- **Evento de domínio é o gatilho natural de agente.** "Quando `PagamentoRecebido`, conciliar" custa menos e gera um trace que qualquer pessoa entende.
- **Agente rende mais em cima de camada semântica.** Num benchmark de 2023, o acerto subiu de 16% para 54% só com a camada. Leia como direção, não como promessa.
- **Glossário, modelo canônico e camada semântica são o mesmo artefato**, e ele sobrevive à troca de modelo, de framework e do próprio agente.
- **Contrato de componente tem quatro campos: entrada, saída, erro e dono.** O dono é o que mais falta.
- **O prompt é um componente**, com versão, dono e eval antes de ir para produção.
- **Só se reaproveita o que tem contrato.** Peça sem contrato funciona no primeiro cliente e é refeita no segundo.
- **DDD atrapalha quando vira cerimônia**, quando o glossário não aparece no código e quando a fronteira ignora as pessoas.

## Fontes

**Leituras**

- [Mars Climate Orbiter Mishap Investigation Board — Phase I Report, NASA (nov/1999)](https://llis.nasa.gov/llis_lib/pdf/1009464main1_0641-mr.pdf) — o relatório oficial da perda da sonda: o dado de impulso entregue em libra-força segundo onde a especificação de interface pedia newton-segundo, e a falta de verificação na passagem entre os times.
- [Bounded Context — Martin Fowler (2014)](https://martinfowler.com/bliki/BoundedContext.html) — o exemplo do "medidor" na empresa de energia e a citação de Evans sobre a inviabilidade de unificar o modelo de um sistema grande.
- [Ubiquitous Language — Martin Fowler (2006)](https://martinfowler.com/bliki/UbiquitousLanguage.html) — a linguagem comum e rigorosa entre quem desenvolve e quem usa, e a ideia de que o especialista do negócio deve reclamar de termos inadequados.
- [DDD Aggregate — Martin Fowler](https://martinfowler.com/bliki/DDD_Aggregate.html) e [Domain Event — Martin Fowler](https://martinfowler.com/eaaDev/DomainEvent.html) — os dois blocos táticos que mais importam para agentes, explicados em poucos parágrafos.
- [Domain-Driven Design Reference — Eric Evans](https://www.domainlanguage.com/ddd/reference/) — o resumo oficial e gratuito dos padrões do livro de 2003, escrito pelo próprio autor. Bom para consulta rápida.
- [Eric Evans Encourages DDD Practitioners to Experiment with LLMs — InfoQ (mar/2024)](https://www.infoq.com/news/2024/03/Evans-ddd-experiment-llm/) — a cobertura da palestra de abertura do Explore DDD 2024, em que Evans disse que um modelo de linguagem treinado é um bounded context.
- [Context Mapping — DDD Crew, GitHub](https://github.com/ddd-crew/context-mapping) — a referência aberta e visual dos tipos de relação entre contextos, incluindo conformista e camada anticorrupção.
- [Anti-Corruption Layer pattern — Azure Architecture Center, Microsoft](https://learn.microsoft.com/en-us/azure/architecture/patterns/anti-corruption-layer) — o padrão descrito com os custos à mostra: latência, manutenção, observabilidade, validação de entrada na fronteira e a recomendação de não colocar regra de negócio na camada.
- [Writing effective tools for agents — with agents, Anthropic (set/2025)](https://www.anthropic.com/engineering/writing-tools-for-agents) — descrever a tool como para alguém novo no time, prefixar nomes por sistema e trocar identificadores técnicos por nomes com significado, com o ganho de precisão relatado.
- [Ontology overview — documentação do Palantir Foundry](https://www.palantir.com/docs/foundry/ontology/overview/) — a Ontologia como camada operacional da organização, com objetos, propriedades, ligações e ações, funcionando como gêmeo digital.
- [A Benchmark to Understand the Role of Knowledge Graphs on Large Language Model's Accuracy for Question Answering on Enterprise SQL Databases — Sequeda, Allemang e Jacob, arXiv (nov/2023)](https://arxiv.org/abs/2311.07509) — o salto de cerca de 16% para cerca de 54% de acerto quando as perguntas passam a ser feitas sobre uma representação semântica do banco. Os autores trabalham numa empresa do setor: leia com isso em mente.
- [Philosophical Investigations — verbete da Wikipedia](https://en.wikipedia.org/wiki/Philosophical_Investigations) — o contexto do §43 de Wittgenstein, com a ressalva de que a observação vale "para uma grande classe de casos", e não para todos.

**Em português**

- [O que é Domain-Driven Design — Full Cycle](https://fullcycle.com.br/domain-driven-design/) — uma boa introdução ao DDD em português.
- [Aprenda DDD do jeito certo — Full Cycle, YouTube](https://www.youtube.com/watch?v=eUf5rhBGLAk) — uma hora de vídeo. Se o tempo for curto, os trechos sobre linguagem ubíqua, bounded contexts e mapas de contexto somam cerca de 25 minutos.

**Para depois**

- *Domain-Driven Design Distilled*, de Vaughn Vernon — a versão curta e acessível do livro do Evans, para quem quer o método inteiro sem as quase 600 páginas do original.

**Nota sobre as fontes.** Os nomes de campo do ERP, os códigos de situação e os valores deste artigo são fictícios e servem para mostrar a forma da tradução, não um sistema real. O número do benchmark de 2023 vem de um único estudo, num único domínio, com um modelo que já foi superado, e deve ser lido como direção. O glossário, o context map e os componentes são do caso fictício da série, inspirado num processo financeiro real que eu conheço bem.

**[→ Acompanhe os próximos artigos](/blog/)**
