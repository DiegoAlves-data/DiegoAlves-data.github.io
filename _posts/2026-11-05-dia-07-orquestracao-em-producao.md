---
title: "Dia 7 — Orquestração em produção: o que separa um agente de notebook de um agente de verdade"
date: 2026-11-05
categories: [desafio-fde]
tags: [fde, producao, orquestracao, human-in-the-loop, custo]
excerpt: "No notebook, o agente concilia 20 lançamentos com você olhando. Em produção, são milhares no dia 5 do mês, de madrugada, sem ninguém olhando, com a API do banco fora do ar. Os dez conceitos que separam os dois mundos, os padrões de human-in-the-loop, onde o agente mora na cloud do cliente e a conta de custo que mostra que o dinheiro quase nunca está no token."
---

## O que estamos construindo

Desde o Dia 3 eu uso um único caso para não deixar nada no abstrato: um **agente de conciliação financeira**.

Conciliar é casar duas listas que deveriam bater e quase nunca batem de primeira. De um lado, o que entrou no banco: PIX, boleto, TED, recebimento de cartão. Do outro, o que a empresa cobrou: as cobranças e notas fiscais emitidas no ERP. Para cada recebimento, alguém precisa dizer a qual cobrança ele corresponde, dar baixa e seguir para o próximo.

O trabalho do agente cabe em quatro passos: buscar os recebimentos no extrato, procurar a cobrança candidata no ERP, propor o casamento com uma justificativa e, quando não tiver segurança, escalar para uma pessoa. Ele nunca efetiva uma baixa sozinho. Propõe, um validador confere, e acima de um valor um humano aprova.

O que a série já construiu até aqui:

| Dia | O que entrou no agente |
|---|---|
| 3 | A anatomia: o modelo só fala, quem executa é o harness |
| 4 | O que ele vê: system prompt, poucos exemplos e três tools sem sobreposição |
| 5 | Quando ele para: os quatro critérios de parada e o graph que separa código, LLM e humano |
| 6 | Como ele fala com o banco, o ERP e o time financeiro: tools, idempotência e MCP |
| 7 | Hoje: como ele aguenta o mundo real, com volume, falhas e gente envolvida |

O caso é inspirado num processo financeiro real que eu conheço bem, mas tudo aqui é anônimo e os números dos exemplos são fictícios.

---

Repita comigo: funcionar no notebook não é funcionar.

No notebook, o agente da conciliação roda vinte lançamentos. Você olhando. A API do banco respondendo em 200 milissegundos. Se algo dá errado, você roda de novo.

Em produção, ele roda seis mil lançamentos por mês, boa parte concentrada nos primeiros dias úteis. De madrugada. Sem ninguém olhando. E, em algum momento, a API do banco vai ficar fora do ar, o provedor do modelo vai limitar as chamadas e o servidor vai reiniciar no meio de uma tarefa.

A distância entre esses dois mundos tem nome: **orquestração**. É tudo o que fica em volta do agente para que ele trabalhe com volume, sobreviva às falhas e passe a bola para uma pessoa na hora certa.

Quem faz deployment não precisa implementar tudo isso com as próprias mãos. Mas vai ouvir esses termos em toda conversa com a TI do cliente, e precisa saber fazer as perguntas certas. O objetivo de hoje é esse: entender cada peça bem o bastante para perguntar sobre ela.

## 1. Um agente em produção é um sistema distribuído disfarçado

A Temporal, empresa que faz software de orquestração e atende empresas como Stripe, Netflix e Coinbase, descreve o problema numa frase da própria documentação: os loops de agente são longos e guardam estado, e as chamadas dentro deles falham. O pedido ao modelo dá timeout. A API do outro lado cai. O processo reinicia.

Todo "agente simples" encadeia chamadas de LLM, espera APIs, às vezes espera gente e ramifica de um jeito que ninguém previu. Isso é, na definição mais clássica, um sistema distribuído: várias partes, em lugares diferentes, conversando por uma rede que falha.

Nos anos 1990, Peter Deutsch e outros engenheiros da Sun listaram as "falácias da computação distribuída", as premissas falsas que todo mundo assume no começo. A primeira é "a rede é confiável". A segunda, "a latência é zero". Trinta anos depois, elas continuam sendo o erro número um de quem leva um protótipo de IA para produção.

A diferença entre os dois mundos fica assim:

| | No notebook | Em produção |
|---|---|---|
| Volume | 20 lançamentos | 6 mil por mês, com pico no começo do mês |
| Quem está olhando | Você | Ninguém, às 3h da manhã |
| Quando algo falha | Você roda de novo | O sistema precisa decidir sozinho: tentar de novo, esperar ou chamar alguém |
| Onde roda | No seu computador | Na conta de nuvem do cliente, com as regras dele |
| Quem decide no caso difícil | Você, na hora | Uma pessoa do financeiro, que pode demorar dois dias para responder |
| O que custa | Quase nada | Token, infraestrutura e, principalmente, tempo de gente |

Cada linha dessa tabela vira uma peça de arquitetura. Este é o mapa completo:

<figure>
  <img src="{{ '/assets/images/desafio-fde/dia-07-diagrama-de-producao.svg' | relative_url }}" alt="Diagrama de produção do agente de conciliação. Na linha principal, seis caixas ligadas por setas cinzas: 1 gatilho (webhook de PIX ou agenda de hora em hora), 2 fila (lançamentos esperando a vez), 3 worker (o agente, o graph do Dia 5), 4 tools (banco, ERP e LLM), 5 validador (regras do cliente em código) e 6 registro (proposta gravada com rastro). Linhas tracejadas laranjas saem da fila (pico do dia 5?), do worker (timeout?) e das tools (API fora? limite 429?) e descem para uma caixa de retry com backoff e jitter, que espera 1, 2, 4, 8 segundos e devolve o item para a fila. Do retry, quando falhou N vezes, uma seta desce para a dead letter queue, e dela para uma caixa onde um humano investiga, com alarme, causa e reprocessamento. Do validador, quando o resultado é inválido, duvidoso ou acima de R$ 5 mil, uma seta desce para a caixa de human-in-the-loop, onde uma pessoa aprova, corrige ou decide; quando aprovado, sobe para o registro. A regra do desenho: todo lançamento termina registrado, nas mãos de uma pessoa ou na DLQ, e nenhum some em silêncio.">
  <figcaption>O agente é uma caixa entre seis. As outras cinco existem porque o mundo real falha.</figcaption>
</figure>

A regra que está embaixo do desenho é a mais importante do artigo: **todo lançamento termina em um de três lugares**. Registrado, nas mãos de uma pessoa ou na DLQ. Nenhum some em silêncio. Para um diretor financeiro, isso vale mais do que qualquer taxa de acerto, porque é a garantia de que nada foi esquecido.

## 2. Os dez conceitos, numa página

Antes de entrar em cada um, a cola. Se você só ler esta tabela, já vai conseguir acompanhar uma reunião de arquitetura.

| Conceito | O que é | Por que importa para o negócio |
|---|---|---|
| **Fila** | O trabalho não roda na hora do pedido: entra numa fila e workers processam | Absorve o pico do começo do mês e permite tentar de novo |
| **Retry com backoff** | Tentar de novo esperando 1s, 2s, 4s, 8s... com um pouco de aleatoriedade | Falha temporária se resolve sozinha, sem acordar ninguém |
| **Idempotência** | Repetir a chamada gera o mesmo efeito que fazer uma vez | Sem ela, o retry vira baixa em dobro |
| **Dead letter queue (DLQ)** | O que falhou N vezes vai para uma fila separada, onde uma pessoa olha | Nada se perde em silêncio |
| **Timeout e circuit breaker** | Desistir depois de X segundos; se um sistema está fora, parar de chamá-lo por um tempo | O processo degrada com elegância, em vez de travar inteiro |
| **Rate limit** | Limite de chamadas por minuto, do provedor do modelo e das APIs do cliente | É ele que define quantos lançamentos por hora o agente consegue processar |
| **Durable execution** | O fluxo grava cada passo; se cair, retoma de onde parou | Tarefas longas e esperas de dias por aprovação não perdem estado |
| **Human-in-the-loop (HITL)** | O ponto onde uma pessoa aprova, corrige ou decide | É onde o risco é controlado, e também onde está a maior parte do custo |
| **Versionamento de prompt e modelo** | Trocar o prompt ou o modelo é um deploy, com teste antes | Evita que o agente piore sem ninguém perceber |
| **Ambiente e dado do cliente** | Segregação, mínimo privilégio, segredos num cofre, logs sem dado pessoal, LGPD | É o que trava projeto na TI e no jurídico |

Agora, um por um.

## 3. Fila: o amortecedor do dia 5

Quem trabalha com financeiro conhece o calendário. Boa parte dos recebimentos de um mês cai nos primeiros dias úteis: vencimento de boleto, folha, repasse. Num cliente com seis mil lançamentos por mês, não é exagero imaginar dois mil chegando em dois dias.

Se o agente processasse cada lançamento na hora em que ele chega, o sistema teria de ser dimensionado para o pico, e ficaria parado o resto do mês. Pior: quando o pico passasse do que as APIs aguentam, os pedidos simplesmente falhariam.

A fila resolve as duas coisas. O gatilho (um webhook do banco avisando que entrou um PIX, ou uma agenda que roda de hora em hora) só coloca o lançamento na fila. Do outro lado, um número controlado de **workers**, que são cópias do agente rodando em paralelo, vai tirando itens e processando no ritmo que o sistema aguenta.

A analogia que eu uso com quem não é técnico: é a senha do banco. Ninguém é atendido na hora em que entra, mas ninguém é mandado embora. E se o caixa precisar sair por cinco minutos, as senhas continuam esperando.

Três coisas a fila dá de graça:

- **Pico vira atraso, não erro.** No dia 5 a fila cresce; no dia 8 ela esvazia.
- **Tentar de novo fica fácil.** Se o processamento falhar, o item volta para a fila.
- **Dá para medir.** O tamanho da fila e a idade do item mais antigo são os dois números que dizem se o sistema está dando conta.

## 4. Retry com backoff e jitter: tentar de novo sem piorar as coisas

A maior parte das falhas em produção é passageira. A API do banco engasga por trinta segundos, o provedor do modelo devolve "muitas requisições, tente mais tarde". Tentar de novo resolve.

O problema é **como** tentar de novo. Se cem workers falharem ao mesmo tempo e todos tentarem de novo imediatamente, a API que estava se recuperando recebe cem chamadas no mesmo segundo e cai de novo.

Por isso existe o **backoff exponencial**: esperar 1 segundo antes da segunda tentativa, 2 antes da terceira, 4 antes da quarta, 8 antes da quinta. Cada espera dobra.

Só que isso não basta. Em 2015, Marc Brooker, engenheiro da AWS, publicou um estudo mostrando que o backoff sozinho mantém as tentativas agrupadas: os cem workers continuam tentando juntos, só que no segundo 1, depois no 2, depois no 4. A conclusão dele virou padrão no mercado: "a solução não é remover o backoff, é adicionar jitter". Jitter é um pouco de aleatoriedade. Cada worker sorteia quanto vai esperar dentro da janela.

<figure>
  <img src="{{ '/assets/images/desafio-fde/dia-07-backoff-com-jitter.svg' | relative_url }}" alt="Dois gráficos de barras com o tempo de 0 a 16 segundos no eixo horizontal e uma linha tracejada marcando o que a API aguenta. No primeiro, sem jitter, aparecem quatro barras altas, em 1, 2, 4 e 8 segundos, com 100, 85, 70 e 55 chamadas, todas bem acima da linha: as tentativas chegam no mesmo instante e derrubam a API de novo. No segundo, com jitter, as mesmas tentativas aparecem como muitas barras baixas espalhadas ao longo dos 16 segundos, todas abaixo ou na linha: a carga se espalha e a API consegue voltar.">
  <figcaption>O mesmo número de tentativas. A diferença é só quando cada uma acontece.</figcaption>
</figure>

Tem mais duas regras que separam um retry bem feito de um retry perigoso.

**Só se tenta de novo o que pode dar certo na segunda vez.** Erro de "serviço indisponível" ou de limite de chamadas (o famoso 429) é passageiro: vale tentar. Erro de "pedido inválido" (400) não vai mudar sozinho: tentar de novo é desperdício. E quando a API diz quanto tempo esperar, como a da Anthropic faz no cabeçalho `retry-after`, a espera certa é a que ela pediu.

**Retry em uma camada só.** O mesmo artigo da AWS mostra a conta que assusta: se cada uma de cinco camadas de um sistema tentar três vezes, uma falha no fundo vira 243 vezes mais carga no banco de dados. No agente, isso acontece fácil: o SDK do modelo já tenta de novo, o worker tenta de novo, e a fila devolve o item. O livro de SRE do Google recomenda ir além e ter um **orçamento de retries**: por exemplo, no máximo 60 tentativas extras por minuto no processo inteiro. Acabou o orçamento, a chamada falha e segue o fluxo de erro.

## 5. Idempotência e DLQ: o que garante que nada duplica e nada some

Retry só é seguro se a operação for **idempotente**: fazer duas vezes tem de dar o mesmo resultado que fazer uma. Esse foi o tema do Dia 6, e não vou repetir tudo. O resumo é que a tool `conciliacao_propor` usa uma chave montada pelo servidor a partir do lançamento e da cobrança. Se a mesma proposta chegar duas vezes, a segunda devolve a primeira, sem gravar nada.

A frase que eu levaria para qualquer reunião: **sem idempotência, retry vira cobrança duplicada**. Antes de aprovar qualquer política de retry, pergunte quais operações escrevem algo e como cada uma se protege de repetição.

Agora, o outro lado. Às vezes tentar de novo não resolve. O lançamento tem um campo corrompido. A cobrança foi apagada do ERP. O agente falha na primeira, na segunda, na quinta tentativa.

Esse item não pode ficar girando na fila para sempre, e também não pode desaparecer. Ele vai para a **dead letter queue**, a fila de mensagens mortas. Na AWS, por exemplo, isso é configurado com um número: depois de N recebimentos sem sucesso, a mensagem é movida para a DLQ. A documentação deles lista o que fazer lá dentro: olhar os logs, entender por que falhou e, depois de corrigir, devolver o item para a fila original.

Três detalhes que fazem a DLQ funcionar de verdade:

- **Ela precisa de alarme.** Uma DLQ que ninguém olha é só um lugar mais organizado para perder coisas.
- **Ela precisa de dono.** Quem recebe o alarme? Em quanto tempo responde? Isso é conversa com o cliente, não com o time técnico.
- **A DLQ não é a fila de exceções do Dia 5.** A fila de exceções recebe casos de negócio que o agente decidiu escalar: o cliente sem cadastro, o pagamento agrupado que não fecha. A DLQ recebe falhas técnicas: o que quebrou. Misturar as duas faz o time financeiro receber erro de sistema e a TI receber dúvida de conciliação.

## 6. Timeout e circuit breaker: saber desistir

Toda chamada externa precisa de um prazo. Sem ele, um worker pode ficar esperando para sempre uma API que travou, e com cem workers travados o processo inteiro para. A AWS diz que a prática padrão dentro da Amazon é pôr timeout em toda chamada remota. O livro do Google vai além: o prazo deve ser passado adiante, para que ninguém lá no fundo gaste trabalho com um pedido que já foi abandonado lá em cima.

O timeout resolve uma chamada. O **circuit breaker** resolve o padrão. O nome vem do disjuntor da sua casa: quando a carga passa do limite, ele desarma e protege o resto da instalação. O padrão foi popularizado por Michael Nygard no livro *Release It!* e descrito por Martin Fowler em 2014.

<figure>
  <img src="{{ '/assets/images/desafio-fde/dia-07-circuit-breaker.svg' | relative_url }}" alt="Diagrama de estados do circuit breaker com três caixas. Fechado: as chamadas passam normalmente e as falhas são contadas. Depois de 5 falhas seguidas, passa para Aberto: nem tenta chamar, falha na hora e devolve o item para a fila com atraso. Depois de 30 segundos, passa para Meio-aberto: deixa passar uma chamada de teste. Se o teste falha, volta para Aberto. Se o teste dá certo, volta para Fechado, o normal. Nota: sem disjuntor, 100 workers martelam a API caída e atrapalham a volta dela; com ele, o agente espera sem gastar token e o resto do processo continua. Os números são exemplo e cada cliente calibra.">
  <figcaption>O disjuntor protege o sistema do cliente do próprio agente.</figcaption>
</figure>

Em linguagem de negócio, o circuit breaker é o que faz o sistema **degradar com elegância**. A API do banco caiu? A conciliação daquela conta pausa, os itens esperam na fila, e as outras contas seguem normalmente. Quando o banco volta, o disjuntor percebe sozinho e o trabalho retoma. Ninguém acorda de madrugada.

E tem um bônus que é específico de agente: enquanto o disjuntor está aberto, o agente não chama o modelo para raciocinar sobre um lançamento que ele não vai conseguir processar. Isso é token que não se gasta.

## 7. Rate limit: o que define a velocidade real

Todo sistema externo tem um limite de chamadas. O provedor do modelo tem. A API do banco do cliente tem. O ERP tem.

Na API da Anthropic, por exemplo, o limite é medido em três dimensões ao mesmo tempo: requisições por minuto, tokens de entrada por minuto e tokens de saída por minuto. Ele funciona como um balde que se enche continuamente, e não como uma cota que zera de minuto em minuto. Na prática, isso quer dizer que até uma rajada curta de pedidos pode estourar o limite.

Uma conta rápida para ver onde está o gargalo. Suponha dois mil lançamentos no pico, cada um com quatro passos de cerca de 6 mil tokens de entrada. São 48 milhões de tokens. Espalhados em oito horas, dá uns 100 mil tokens por minuto, o que fica dentro do limite dos planos de entrada dos provedores grandes.

Agora a API do banco. Suponha que ela aceite 60 chamadas por minuto (um número hipotético, mas na ordem de grandeza que eu já vi) e que cada lançamento precise de duas chamadas. Os mesmos dois mil lançamentos levam mais de uma hora só esperando o banco.

A lição: **o gargalo quase nunca é o LLM**. É o sistema legado do cliente. Por isso a primeira pergunta de dimensionamento não é "qual modelo vamos usar", e sim "qual é o limite de cada API que o agente vai chamar, e ele é compartilhado com quem?".

Duas alavancas ajudam. Uma é o cache: na Anthropic, para a maioria dos modelos, tokens lidos do cache não contam no limite de entrada, então um system prompt longo e repetido custa bem menos limite. A outra é limitar o número de workers ao que o sistema mais lento aguenta, e não ao que o servidor aguenta.

## 8. Durable execution: o fluxo que não esquece onde parou

Volte ao caso. O agente buscou o extrato, buscou a cobrança, está no meio da proposta. O servidor reinicia, por uma atualização de segurança, uma queda ou um ajuste de capacidade.

Num programa comum, tudo o que estava na memória se perde. O lançamento volta para a fila e o agente recomeça do zero: paga de novo os tokens e as chamadas dos passos que já tinha feito. E se aquele lançamento estava esperando a aprovação de uma pessoa havia dois dias, essa espera some.

**Durable execution** é a técnica que resolve isso. O resultado de cada passo concluído (cada chamada de modelo, cada chamada de tool) é gravado num histórico fora do servidor. Se o processo cair, outro worker lê esse histórico, pula o que já foi feito e retoma exatamente de onde parou.

<figure>
  <img src="{{ '/assets/images/desafio-fde/dia-07-durable-execution.svg' | relative_url }}" alt="Diagrama em duas linhas com cinco passos cada. Sem durable execution: buscar extrato (feito), buscar cobrança (feito), servidor reinicia (tudo estava na memória), recomeça do passo 1 (tokens e chamadas pagos de novo) e a espera se perde (quem estava aguardando aprovação some da fila). Com durable execution, em que cada passo concluído é gravado num histórico fora do servidor: buscar extrato (feito e gravado), buscar cobrança (feito e gravado), servidor reinicia (o histórico está salvo), outro worker lê o histórico e não repete os passos 1 e 2, e retoma no passo 3, com a espera de 2 dias pela aprovação continuando. Nota: o que é gravado é o resultado de cada chamada de LLM e de tool; por isso a escrita do passo 3 ainda precisa ser idempotente, porque se a queda for no meio dela, o passo 3 roda de novo.">
  <figcaption>A queda é a mesma. O que muda é se alguém anotou o caminho.</figcaption>
</figure>

A documentação da Temporal resume a promessa: o fluxo retoma sozinho depois de uma queda, de um timeout de rede ou de uma espera de vários dias por uma pessoa que precisa aprovar um passo. No vocabulário deles, o loop do agente é um *workflow* e cada chamada de modelo ou de tool é uma *activity*, que tem retry próprio e cujo resultado fica gravado. Hoje existem integrações prontas com o OpenAI Agents SDK, o LangGraph, o Google ADK e outros frameworks. Na conferência anual da Temporal, em maio de 2026, a própria OpenAI subiu ao palco para contar como a ferramenta virou plataforma de vários times dela.

A Temporal não é o único caminho. A AWS tem o Step Functions, a Azure tem o Durable Functions, e dá para chegar perto disso com um banco de dados e disciplina. O que importa é o princípio, e ele aparece de outro jeito no *12-Factor Agents*, um guia aberto sobre como construir agentes confiáveis:

- **Fator 5, unificar o estado de execução e o estado de negócio.** Em vez de guardar "em que passo estou" num lugar e "o que já aconteceu" em outro, guarde tudo junto, de preferência como o próprio histórico do agente. Uma fonte só de verdade, fácil de salvar, de recuperar e de auditar.
- **Fator 6, iniciar, pausar e retomar com APIs simples.** O agente tem de poder parar para esperar uma pessoa ou uma operação longa, e ser retomado por um evento externo, como um webhook de aprovação.
- **Fator 12, fazer do agente um *stateless reducer*.** O agente não guarda nada escondido na memória: recebe o estado atual mais um evento novo e devolve o próximo estado. É o que torna possível retomar em qualquer servidor.

Um detalhe que o diagrama mostra e que pouca gente percebe: durable execution não dispensa idempotência. O que fica gravado é o resultado de cada passo **concluído**. Se a queda acontecer no meio da escrita, aquele passo roda de novo. A chave de idempotência do Dia 6 continua sendo a última linha de defesa.

## 9. Human-in-the-loop: onde a pessoa entra, e por quê

O ponto onde uma pessoa aprova, corrige ou decide não é um detalhe de interface. É uma decisão de arquitetura, de risco e de custo ao mesmo tempo.

A regra que eu sigo: **defina o ponto de HITL por valor e risco, não por tipo de tarefa**. "Toda conciliação de boleto passa por humano" é uma regra por tipo, e gasta atenção onde não precisa. "Toda conciliação acima de R$ 5 mil, ou com divergência de valor, passa por humano" é uma regra por risco, e põe a atenção onde está o prejuízo possível.

Existem quatro padrões, e eles combinam entre si:

| Padrão | Como funciona | Quando usar |
|---|---|---|
| **Shadow** | O agente roda em paralelo e registra o que faria. O humano continua decidindo tudo, como sempre | Antes de tudo. É o período de prova, e é o tema do Dia 8 |
| **Aprovação prévia** | O agente propõe, a pessoa aprova antes de qualquer efeito | Alto valor ou alto risco. No caso, baixas acima do limite do cliente |
| **Escalonamento por exceção** | Só vão para a pessoa os casos em que o agente tem dúvida ou que violam uma regra | É o regime normal da conciliação: a tool `humano_escalar_excecao` do Dia 6 |
| **Revisão por amostragem** | O agente executa e a pessoa revisa depois uma amostra de 5% a 10% | Baixo risco e alto volume, quando o agente já provou que acerta |

A lição 16 do curso aberto *AI Agents for Beginners*, da Microsoft, descreve a aprovação prévia exatamente nesse formato: ações arriscadas, como emitir um reembolso, pausam para aprovação; o agente propõe, a pessoa aprova ou rejeita, e o fluxo retoma. É aqui que o durable execution da seção anterior se paga: a pausa pode durar dois dias sem perder nada.

Eu enxergo esses padrões como uma escada, e não como um cardápio.

<figure>
  <img src="{{ '/assets/images/desafio-fde/dia-07-escada-hitl.svg' | relative_url }}" alt="À esquerda, uma escada de quatro degraus: 1 Shadow, o agente roda e o humano decide tudo; 2 Aprovação prévia, o humano aprova cada proposta; 3 Exceção mais limite, só a dúvida e o que passa de R$ 5 mil; 4 Amostragem, o humano revisa 5% a 10% depois. À direita, um gráfico de linha ilustrativo com a porcentagem dos lançamentos que passam por uma pessoa ao longo de 12 semanas: 100% nas semanas 1 a 4 (degraus 1 e 2), caindo para 38% na semana 5 e descendo até 9% na semana 12 (degraus 3 e 4). Nota: a curva caindo é a prova de que a automação está andando sozinha; se ela para de cair, o problema está no agente, não nas pessoas.">
  <figcaption>Cada degrau se conquista com dado do degrau anterior. A curva é ilustrativa.</figcaption>
</figure>

### A métrica que prova que a automação anda sozinha

Se eu pudesse acompanhar um único número de um agente em produção, seria a **taxa de escalonamento**: a porcentagem dos casos que precisaram de uma pessoa.

Ela tem de cair com o tempo. Cai porque o agente melhora a cada caso difícil que vira exemplo, porque as regras do cliente ficam mais claras e porque o limite de aprovação sobe conforme a confiança aumenta.

Essa curva é, para mim, o critério mais honesto para dizer que um deployment terminou. Não é "o agente está no ar". É "a taxa de escalonamento caiu até um patamar combinado com o cliente e ficou lá". Se a curva para de cair, não adianta contratar mais gente para revisar: o problema está no agente, e é lá que se mexe.

## 10. Versionamento de prompt e modelo: trocar é fazer deploy

Esta é a armadilha mais silenciosa. Alguém ajusta uma frase do system prompt para resolver um caso. Ou o provedor lança uma versão nova do modelo e alguém troca, porque "é melhor". Ninguém testa. Duas semanas depois, a taxa de acerto em pagamentos agrupados caiu de 90% para 70%, e ninguém sabe por quê.

A regra: **prompt e modelo são código**. Ficam versionados no repositório, cada mudança tem dono e data, e nenhuma vai para produção sem passar por uma bateria de testes com casos reais, os *evals*, que são o tema de amanhã. A lição 10 do *AI Agents for Beginners* insiste no mesmo ponto: como agentes não são determinísticos e mudam com o tempo, a avaliação tem de ser contínua, antes do deploy e depois dele.

Três práticas simples:

- **Fixe a versão do modelo.** Use o identificador exato da versão, e não um apelido que muda sozinho.
- **Grave a versão em cada registro.** Toda proposta de conciliação leva junto qual prompt e qual modelo a geraram. Quando algo der errado, você sabe exatamente o que estava rodando.
- **Troque aos poucos.** A versão nova roda primeiro numa parte pequena do volume, ou em shadow, e só depois assume tudo.

## 11. Onde o agente mora: a cloud do cliente

Em finanças, a regra quase sempre é a mesma: **o dado não sai do ambiente do cliente**. Extrato, CNPJ, CPF de sócio, valores. Nada disso pode ir para um servidor de fornecedor, nem para o notebook de quem faz o deployment.

Então o agente vai até o dado. Ele roda na conta de nuvem do cliente (AWS, Azure ou GCP), dentro da rede privada dele, com as credenciais e as políticas de segurança dele.

<figure>
  <img src="{{ '/assets/images/desafio-fde/dia-07-cloud-do-cliente.svg' | relative_url }}" alt="Diagrama com uma grande área tracejada chamada conta de nuvem do cliente (AWS, Azure ou GCP, na região escolhida por ele). Dentro, uma rede privada sem saída aberta para a internet contém a fila, o worker com o agente e o validador com registro, com a nota de que o worker tem uma identidade própria que só lê o extrato e só grava propostas, o mínimo privilégio. Ainda dentro da conta: o modelo, acessado por um provedor aprovado (Bedrock, Azure OpenAI ou Vertex) por endpoint privado; o ERP e o banco do cliente, acessados com as credenciais e a política dele; um cofre de segredos, sem nenhuma senha no código; logs e traces sem CPF, CNPJ ou nomes; e a fila de exceções do time financeiro dele. Fora da área, o time de deployment entra por um papel temporário, auditado e aprovado pela TI, e a infraestrutura como código descreve o ambiente em arquivos versionados que recriam tudo igual e ficam com o cliente. Nota: extrato, CNPJ, CPF, valores e o texto dos prompts e respostas nunca atravessam a linha tracejada; cuidado com opções globais de processamento, que podem mandar o pedido para outra região.">
  <figcaption>Os nomes dos serviços mudam de nuvem para nuvem. As seis caixas, não.</figcaption>
</figure>

E o modelo? Ele é chamado por um **provedor aprovado**, que roda dentro da mesma nuvem do cliente. Os dois mais comuns:

- **Amazon Bedrock.** A documentação da AWS diz que os provedores de modelo não têm acesso às contas onde os modelos rodam, e por isso não têm acesso aos logs nem aos prompts e respostas dos clientes.
- **Azure OpenAI**, hoje parte do Microsoft Foundry. A documentação da Microsoft diz que os prompts e as respostas não ficam disponíveis para a OpenAI nem para outros clientes, e não são usados para treinar modelos.

Aqui mora uma pegadinha que vale ouro numa reunião com o jurídico. A própria Microsoft avisa que, nas modalidades de implantação chamadas *Global* e *DataZone*, os pedidos podem ser processados em outras geografias. E o processamento em lote da Azure é, por definição, do tipo Global. Ou seja: a opção mais barata pode ser justamente a que leva o dado para fora do país. Pergunte sempre onde o pedido é **processado**, e não só onde o dado fica guardado.

O resto do ambiente segue princípios que a LGPD já escreveu em lei. O artigo 6º fala em **necessidade**, que é limitar o tratamento ao mínimo necessário, e em **segurança**. O artigo 46 obriga a adotar medidas técnicas e administrativas para proteger os dados pessoais. Traduzindo para arquitetura:

- **Segregação.** Uma conta ou um ambiente por cliente. Nada compartilhado.
- **Mínimo privilégio.** O agente tem uma identidade própria, que só lê o extrato e só grava propostas. Não apaga nada, não lê o que não precisa.
- **Segredos num cofre.** Nenhuma senha ou chave de API no código ou em arquivo de configuração.
- **Logs sem dado pessoal.** O log diz "lançamento lanc_8812 conciliado com cob_0457", e não o nome e o CPF do pagador. Logs circulam muito mais do que bancos de dados.

Por fim, a **infraestrutura como código** (IaC). Em vez de configurar o ambiente clicando no console, ele é descrito em arquivos versionados que recriam tudo igual, quantas vezes for preciso. Para quem faz deployment, isso tem dois valores. Dá para montar o ambiente de teste idêntico ao de produção. E os arquivos ficam com o cliente quando o projeto acaba: são um dos artefatos que provam que o trabalho é dele, e não de quem construiu.

Pela minha experiência, esta seção é a que mais atrasa projeto. Não por ser difícil, mas por ser a última em que alguém pensa. Quem já trabalhou com dados de um cliente grande sabe que liberar um acesso pode levar semanas e que nenhuma ferramenta nova entra sem passar por segurança, jurídico e compras. Eu aprendi a pôr essa conversa na primeira semana, e não na última.

## 12. Batch ou tempo real? Pergunte qual latência o negócio precisa

Existem duas formas de disparar a conciliação.

| | Em lote (batch) | Por evento (tempo real) |
|---|---|---|
| Como funciona | Roda a cada hora ou toda noite, sobre tudo o que chegou | Cada PIX recebido dispara o agente por um webhook |
| Velocidade | Horas | Segundos ou minutos |
| Custo | Menor. Os provedores dão desconto para processamento em lote: na Anthropic, 50%, com a maior parte dos lotes concluída em menos de uma hora | Maior. Preço cheio e infraestrutura sempre ligada |
| Complexidade | Baixa. Tolera fila e falha com folga | Alta. Exige fila, idempotência e monitoramento afinados |
| Faz sentido quando | O fechamento é diário ou mensal | Existe uma decisão que depende da baixa na hora, como liberar um pedido assim que o pagamento entra |

A tentação é sempre o tempo real, porque parece mais moderno. A pergunta que desfaz a tentação é simples: **qual latência o negócio realmente precisa?**

Se o financeiro olha a conciliação uma vez por dia, rodar de hora em hora já é dez vezes mais rápido do que o processo de hoje, e custa metade. Se existe um pedido de cliente travado esperando a baixa do PIX, aí sim o evento se paga. Muitas vezes a resposta é um híbrido: tempo real para os poucos casos que destravam alguma coisa e lote para o resto.

E lembre da pegadinha da seção anterior: confira onde o lote é processado antes de escolher.

## 13. Custo por tarefa: a conta que todo FDE faz

Esta é a conta que eu faço antes de qualquer proposta:

```text
custo por tarefa = passos × (tokens de entrada × preço de entrada
                           + tokens de saída × preço de saída)
                 + infraestrutura
                 + revisão humana
```

Com números do caso, e preços hipotéticos de US$ 3 por milhão de tokens de entrada e US$ 15 por milhão de saída:

```text
4 passos × (6.000 × 3/1.000.000 + 500 × 15/1.000.000)
= 4 × (0,018 + 0,0075)
≈ US$ 0,10 por lançamento
```

Seis mil lançamentos por mês dão uns US$ 600, perto de R$ 3,4 mil com um câmbio hipotético de R$ 5,50. Parece muito, até você comparar com o processo manual.

Agora a parte que muda tudo: a revisão humana. Suponha que cada caso escalado leve 4 minutos de uma pessoa, a um custo hipotético de R$ 90 por hora, já com encargos.

<figure>
  <img src="{{ '/assets/images/desafio-fde/dia-07-custo-por-tarefa.svg' | relative_url }}" alt="Gráfico de barras horizontais com o custo mensal em reais para 6 mil lançamentos, com preços e salário hipotéticos. Tudo manual: R$ 36.000, 400 horas de gente. Agente com 30% escalados: R$ 14.200, sendo R$ 3.400 de modelo e o resto de 120 horas de revisão. Agente com 10% escalados: R$ 7.000, com 40 horas de revisão. Agente com 3% escalados: R$ 4.480, com 12 horas de revisão. O custo do modelo é fixo em cerca de R$ 3,4 mil; o tempo de pessoa é calculado a R$ 90 por hora e 4 minutos por revisão.">
  <figcaption>A barra escura quase não muda. A laranja decide o projeto.</figcaption>
</figure>

| Cenário | Horas de gente por mês | Revisão | Modelo | Total |
|---|---|---|---|---|
| Tudo manual (4 min por lançamento) | 400 h | R$ 36.000 | — | R$ 36.000 |
| Agente com 30% escalados | 120 h | R$ 10.800 | R$ 3.400 | R$ 14.200 |
| Agente com 10% escalados | 40 h | R$ 3.600 | R$ 3.400 | R$ 7.000 |
| Agente com 3% escalados | 12 h | R$ 1.080 | R$ 3.400 | R$ 4.480 |

Três leituras dessa tabela.

**Primeira: o token é o custo fixo; a taxa de escalonamento é o custo variável.** O modelo custa o mesmo nos três cenários com agente. O que separa R$ 14 mil de R$ 4 mil é só quantos casos passam por uma pessoa. No começo do projeto, com 30% escalados, a revisão custa três vezes o modelo.

**Segunda: cada ponto percentual de escalonamento vale 4 horas de gente por mês.** É uma conta que o diretor financeiro entende na hora, e é por isso que a curva da seção 9 é o melhor slide de acompanhamento que existe.

**Terceira: otimizar token é a última coisa a fazer.** Trocar para um modelo mais barato, usar cache ou rodar em lote pode cortar a barra escura pela metade. Mas um agente que erra mais e escala mais do que o caro custa mais no total. Em agente financeiro, o custo relevante raramente é o token.

(Todos os números desta seção são hipotéticos. A estrutura da conta é o que vale.)

## Prática: o diagrama do caso e os três pontos onde ele quebra primeiro

O diagrama da seção 1 é o desenho de produção da conciliação. Antes de entregar qualquer desenho como esse, eu faço um exercício: onde ele quebra primeiro? Não "se" quebra, mas onde.

No caso, aposto nestes três.

### 1. O dia 5, quando o pico encontra o limite da API do banco

É a soma das seções 3 e 7. Dois mil lançamentos entram na fila em poucas horas, os workers sobem e a API do banco, com seus 60 pedidos por minuto, começa a devolver 429. Sem backoff com jitter e sem circuit breaker, os workers martelam a API, o banco pode bloquear a credencial por excesso de chamadas, e a conciliação do mês para no dia em que mais importa.

O que eu pergunto antes de subir: qual é o limite da API do banco, ele é compartilhado com outros sistemas do cliente, e o que acontece se a credencial for bloqueada?

### 2. O retry em duas camadas que dobra uma escrita

O SDK do modelo tenta de novo. O worker tenta de novo. A fila devolve o item. Se a tool de escrita não for idempotente, uma única instabilidade de rede gera duas propostas para o mesmo recebimento, e uma delas pode passar pela aprovação de uma pessoa diferente.

O que eu pergunto: em quantas camadas existe retry hoje? Quais operações escrevem? Qual é a chave de idempotência de cada uma?

### 3. A fila de exceções, o único componente sem autoscaling

Este é o que mais me preocupa, porque não é técnico. Dá para subir mais workers com um clique. Não dá para subir mais gente no financeiro. No dia 5, com 30% de escalonamento, são 600 casos esperando uma pessoa, no mesmo dia em que essa pessoa já está afogada no fechamento.

Se a fila de exceções cresce mais rápido do que esvazia, o agente não economizou trabalho: só mudou o trabalho de lugar. O que eu pergunto: quem atende a fila de exceções, quantos casos por dia essa pessoa aguenta, e o que acontece com um caso que passa do prazo?

Esse terceiro ponto é também o motivo de a taxa de escalonamento ser a métrica principal. Ela não mede só custo. Mede se o desenho cabe na vida real do time do cliente.

### Para fazer com o seu processo

Pegue o processo que você vem usando desde o Dia 3 e desenhe o caminho de produção dele, com as seis caixas: gatilho, fila, worker, tools, validador e registro. Depois, para cada seta, escreva o que acontece quando ela falha. Se a resposta for "não sei", você achou uma pergunta para fazer ao cliente.

Por fim, responda as dez perguntas abaixo. Elas são, na prática, a pauta da primeira reunião com a TI do cliente:

1. Qual é o volume por mês, e como ele se distribui ao longo do mês?
2. Qual latência o negócio realmente precisa: minutos, horas ou o dia seguinte?
3. Qual é o limite de chamadas de cada API que o agente vai usar, e ele é compartilhado?
4. Quais operações escrevem algo, e como cada uma se protege de repetição?
5. Para onde vai o que falhar N vezes, e quem recebe o alarme?
6. Onde a pessoa entra, por qual critério de valor e risco, e em quanto tempo ela responde?
7. Se o servidor cair no meio de uma espera de dois dias, o que se perde?
8. Como se troca o prompt ou o modelo sem ninguém perceber a diferença, para pior?
9. Em qual conta e região o agente roda, e onde o modelo processa o pedido?
10. Quanto custa um caso, somando token, infraestrutura e o tempo de gente?

## TLDR

- **Funcionar no notebook não é funcionar.** Em produção, o agente vira um sistema distribuído: rede que falha, APIs que caem, servidor que reinicia, gente que demora.
- **Todo item termina em um de três lugares:** registrado, com uma pessoa ou na DLQ. Nenhum some em silêncio.
- **Fila** transforma pico em atraso, e não em erro.
- **Retry com backoff exponencial e jitter**, só para erro passageiro, em uma camada só e com orçamento. Sem jitter, as tentativas chegam juntas e derrubam o sistema de novo.
- **Sem idempotência, retry vira cobrança duplicada.** E a DLQ precisa de alarme e de dono.
- **Timeout e circuit breaker** fazem o sistema degradar com elegância e protegem o sistema do cliente do próprio agente.
- **O gargalo quase nunca é o LLM.** É o limite da API legada do cliente.
- **Durable execution** grava cada passo e retoma de onde parou, inclusive depois de esperas de dias. Não dispensa idempotência.
- **HITL por valor e risco, não por tipo de tarefa.** Shadow, aprovação prévia, exceção e amostragem são degraus de uma escada.
- **A taxa de escalonamento é a métrica-chave.** Ela tem de cair, e é ela que mostra que a automação anda sozinha.
- **Prompt e modelo são código.** Trocar qualquer um é deploy, com eval antes.
- **O agente vai até o dado**, na conta do cliente, com provedor aprovado, mínimo privilégio, cofre de segredos e logs sem dado pessoal. Pergunte onde o pedido é processado.
- **Batch ou tempo real:** pergunte qual latência o negócio precisa. Lote costuma custar metade.
- **O custo relevante raramente é o token.** É o tempo de gente, e ele é governado pela taxa de escalonamento.

## Fontes

**Leituras**

- [Durable AI — documentação da Temporal](https://docs.temporal.io/ai) — durable execution para agentes: retomar depois de queda, timeout ou espera de dias por aprovação humana, e as integrações com OpenAI Agents SDK, LangGraph e outros.
- [Use cases and design patterns — documentação da Temporal](https://docs.temporal.io/evaluate/use-cases-design-patterns) — a descrição dos loops de agente como longos, com estado e cheios de chamadas que falham, e o modelo de workflow e activities.
- [Temporal, OpenAI, and the future of Agentic AI — Temporal](https://temporal.io/resources/on-demand/temporal-openai-and-the-future-of-agentic-ai) — a OpenAI falando sobre o uso da Temporal em produção, na conferência Replay 2026.
- [12-Factor Agents — HumanLayer](https://github.com/humanlayer/12-factor-agents) — em especial os fatores 5 (unificar estado de execução e de negócio), 6 (iniciar, pausar e retomar com APIs simples) e 12 (o agente como *stateless reducer*).
- [AI Agents for Beginners, lição 10: AI Agents in Production — Microsoft](https://github.com/microsoft/ai-agents-for-beginners/blob/main/10-ai-agents-production/README.md) — observabilidade, avaliação contínua e controle de custo.
- [AI Agents for Beginners, lição 16: Deploying Scalable Agents — Microsoft](https://github.com/microsoft/ai-agents-for-beginners/blob/main/16-deploying-scalable-agents/README.md) — estado fora do processo, concorrência limitada, backoff, dead letter e aprovação humana para ações arriscadas.
- [Exponential Backoff and Jitter — Marc Brooker, AWS Architecture Blog (2015)](https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/) — o estudo que mostrou por que backoff sem jitter não basta.
- [Timeouts, retries, and backoff with jitter — Amazon Builders' Library](https://builder.aws.com/content/3EumjoZascWd1oZiEgL8ORlv3qE/timeouts-retries-and-backoff-with-jitter) — timeout em toda chamada remota, retry em uma camada só e a conta das 243 vezes.
- [Addressing Cascading Failures — Site Reliability Engineering, Google](https://sre.google/sre-book/addressing-cascading-failures/) — orçamento de retries, prazos propagados e como retries derrubam sistemas.
- [CircuitBreaker — Martin Fowler (2014)](https://martinfowler.com/bliki/CircuitBreaker.html) — os três estados do disjuntor e a referência ao *Release It!*, de Michael Nygard.
- [Using dead-letter queues in Amazon SQS — AWS](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-dead-letter-queues.html) — `maxReceiveCount`, alarmes e reprocessamento.
- [Rate limits — documentação da Claude API](https://platform.claude.com/docs/en/api/rate-limits) — limites por requisição, tokens de entrada e de saída, o algoritmo de balde, o erro 429, o cabeçalho `retry-after` e o efeito do cache.
- [Batch processing — documentação da Claude API](https://platform.claude.com/docs/en/build-with-claude/batch-processing) — o desconto de 50% para processamento em lote.
- [Data protection — Amazon Bedrock](https://docs.aws.amazon.com/bedrock/latest/userguide/data-protection.html) — provedores de modelo sem acesso a prompts e respostas.
- [Data, privacy, and security for Azure OpenAI — Microsoft Learn](https://learn.microsoft.com/en-us/azure/ai-foundry/responsible-ai/openai/data-privacy) — o que fica e o que não fica com a OpenAI, e onde os pedidos Global, DataZone e em lote são processados.
- [Lei Geral de Proteção de Dados (Lei 13.709/2018)](https://www.planalto.gov.br/ccivil_03/_ato2015-2018/2018/lei/l13709.htm) — artigos 6º (necessidade e segurança) e 46 (medidas de segurança).

**[→ Acompanhe os próximos artigos](/blog/)**

