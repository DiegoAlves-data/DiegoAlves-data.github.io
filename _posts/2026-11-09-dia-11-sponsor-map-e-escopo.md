---
title: "Dia 11 — Sponsor map, SOW e preço: quem paga, quem decide e quem pode sabotar"
date: 2026-11-09
categories: [desafio-fde]
tags: [fde, sponsor-map, meddicc, sow, precificacao, nrr, gestao-da-mudanca]
excerpt: "Ontem a conta fechou. Hoje é o dia de descobrir que conta fechada não aprova projeto. Os cinco papéis de um deployment e por que o dono do dado é o que mais trava, o MEDDICC traduzido para quem não é de vendas, um SOW de uma página com aceite numérico e critério de saída, cinco modelos de preço lidos pelo risco, a NRR como prova de que o modelo funciona, e o usuário que pode matar o agente sem nunca dizer não."
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
| 11 | Hoje: quem precisa dizer sim, o que está no contrato e como se cobra |

O caso é inspirado num processo financeiro real que eu conheço bem, mas tudo aqui é anônimo e os números dos exemplos são fictícios.

---

No artigo de ontem eu cheguei a uma conta que fecha: payback em oito meses, com o cenário pessimista na mesa e a origem de cada premissa declarada, do jeito que um controller costuma exigir antes de levar o número adiante. O problema é que uma conta bem feita, sozinha, não aprova projeto nenhum.

Quem assina o contrato é uma pessoa, e essa pessoa depende de outras cinco, cada uma com um motivo diferente para querer, temer ou simplesmente ignorar o projeto. O CFO quer saber se o número vai mexer. A gerente que trouxe o projeto quer o crédito quando der certo. O analista que faz o trabalho hoje quer saber se ainda vai ter emprego em março, a segurança quer garantir que nada de ruim aconteça, e o administrador do ERP quer, acima de tudo, que ninguém encoste no sistema dele.

**Quem decide, quem paga e quem é impactado quase nunca são a mesma pessoa.** Esse é o assunto de hoje: a política do deployment, e como ela vira três documentos — um mapa de pessoas, um contrato de escopo e uma tabela de preço.

Esse terreno não é novo para mim. Há anos eu trabalho numa função em que quase nada do que eu entrego depende só de mim: o dado costuma estar em outra área, a decisão costuma ser de outra pessoa, e quem usa o resultado raramente é quem fez o pedido. Quem trabalha com dado aprende cedo que acesso se negocia, que prazo depende de política interna e que o melhor modelo do mundo não sobrevive a um gestor que não foi consultado.

O que me faltava era vocabulário para isso. Fui buscar o de quem vende software de grande porte para grandes empresas e descobri que ele organiza bem várias coisas que eu já fazia por instinto, além de deixar claras outras que eu fazia errado.

## 1. Por que agente que funciona é cancelado

Em junho de 2025 a Gartner publicou uma previsão que pegou mal no mercado: **mais de 40% dos projetos de IA agêntica serão cancelados até o fim de 2027.** Os motivos apontados foram custo crescente, valor de negócio pouco claro e controle de risco insuficiente.

Nenhum dos três motivos tem a ver com o modelo ter errado.

Custo crescente é falta de escopo: o projeto não tinha borda e foi engordando. Valor pouco claro é falta de número combinado: ninguém escreveu, antes de começar, o que contaria como sucesso. Controle de risco insuficiente é o blocker que entrou tarde e, com razão, travou o go-live.

É o mesmo diagnóstico do relatório do MIT que eu citei ontem, por outro caminho. O projeto não morre no harness. Morre na sala de reunião, numa terça-feira, quando alguém pergunta "mas quem pediu isso mesmo?" e a pessoa que pediu foi promovida para outra área.

Tem uma frase do ensaio da a16z sobre FDE que eu releio sempre que penso nisso. Entre as boas práticas, a última é **estar lá pessoalmente**, e o motivo dado não é técnico: a presença ajuda a *"entender as dinâmicas internas de poder e conduzir a adoção de ferramentas novas"*. Um fundo de venture capital escrevendo, num texto sobre engenharia, que parte do trabalho é ler o poder dentro do cliente. Isso diz bastante sobre onde o trabalho de verdade acontece.

## 2. Os cinco papéis de um deployment

O **sponsor map** é a lista das pessoas que podem fazer o projeto acontecer ou morrer, organizada por papel, não por cargo. Cinco papéis cobrem quase tudo.

<figure>
  <img src="{{ '/assets/images/desafio-fde/dia-11-sponsor-map.svg' | relative_url }}" alt="Cinco cartões lado a lado, um por papel do sponsor map, aplicados ao caso da conciliação. Cada cartão responde a quatro perguntas: o que ganha, o que perde, o que precisa ver e quando entra. Sponsor, o CFO ou diretor financeiro: ganha fechar em D mais cinco e caixa previsível; perde orçamento e credibilidade se a conta não fechar; precisa ver a conta da fase um, em faixa e não em número; entra na semana zero e em cada marco de aceite. Champion, em verde, a gerência de recebíveis: ganha visibilidade e um projeto com o nome dela; perde capital político se der errado em público; precisa ver uma vitória pequena já no primeiro mês; entra antes de todo mundo, porque é quem abre as portas. Usuário impactado, em laranja, os analistas de conciliação: ganha o fim da fila de exceção e do retrabalho; perde ou teme perder o emprego e o saber que só ele tem; precisa ver o agente errando e ele sendo quem corrige; entra no discovery, como autor dos casos de eval; risco, sabotagem silenciosa. Blocker, TI, segurança e jurídico: não ganha nada visível, só o risco que não aconteceu; perde tudo se vazar dado ou uma baixa sair errada; precisa ver identidade própria, menor privilégio e trace; entra na semana um, nunca na véspera do go-live. Dono do dado, com borda laranja forte, o administrador do ERP e a tesouraria: ganha quase nada, e é por isso que ninguém lembra dele; perde a estabilidade do sistema que é dele; precisa ver acesso só de leitura, limite de chamadas e plano de volta; entra na semana um, antes do primeiro pedido de acesso; é o mais esquecido. Uma faixa escura no pé resume: o sponsor assina, o champion empurra, o blocker decide se vai ao ar, o usuário decide se o agente vai ser usado e o dono do dado decide se ele vai sequer existir.">
  <figcaption>O mapa não é o organograma. Uma pessoa pode ter dois papéis, e um papel pode ter três pessoas.</figcaption>
</figure>

Vou passar por cada um com as quatro perguntas que importam: **o que ganha, o que perde, o que precisa ver para dizer sim, e quando entra.** A última é a que quase todo mundo esquece, e é a que mais custa.

### Sponsor: quem paga

O *economic buyer*. Num caso de conciliação, o CFO ou o diretor financeiro. É quem tem o orçamento e quem responde pelo número que o projeto promete mexer.

O que ele quer é **o número.** Arquitetura, demonstração e protocolo de integração não entram na conversa; ele quer saber se o fechamento vai sair em D+5 em vez de D+9 e quanto custa chegar lá.

O que ele perde se der errado é maior do que parece. Não é só o dinheiro: é a credibilidade de quem aprovou um projeto de IA que não entregou, num ano em que o conselho pergunta de IA em toda reunião. Ele precisa ver a conta da fase 1, aquela do Dia 10, **em faixa**, com o cenário pessimista na mesa. E entra duas vezes: no começo, para aprovar, e em cada marco de aceite, para confirmar que o número está andando.

O erro mais comum com o sponsor é falar com ele demais. Cada reunião desnecessária com quem assina é um saque na conta de atenção. Ele deve ver pouco, e o que vê deve ter número.

### Champion: quem quer que dê certo

É a pessoa que te trouxe, que defende o projeto nas reuniões em que você não está e que abre as portas que você não sabe que existem. No caso, alguém como a gerência de contas a receber.

O que ela ganha é **visibilidade e carreira**, e não vejo cinismo nisso: é o que move o papel. Um projeto que dá certo com o nome dela vira argumento de promoção, e é por isso que ela aceita gastar capital político com você. Se der errado, é justamente esse capital que ela perde, e na frente de todo mundo.

Por isso ela precisa de uma **vitória pequena e cedo**, de preferência no primeiro mês, e não no sexto: algo que possa mostrar ao chefe sem precisar explicar muito. Foi esse o motivo de, no Dia 10, o quadrante "volume alto, erro barato" ter virado o primeiro passo (classificar e rotear a exceção, sem decidir nada, nas primeiras semanas). Essa vitória rápida serve menos ao projeto e mais à champion, que precisa de algo concreto para continuar defendendo a ideia.

E aqui mora um risco que nenhuma tabela de papel mostra: **o projeto de um champion só é tão estável quanto o cargo dele.** Se ela for promovida, transferida ou sair da empresa no mês 3, o projeto fica órfão. Quem vende software grande chama isso de estar *single-threaded*: a conta inteira pendurada num fio. A defesa é ter mais de um fio desde cedo: o sponsor precisa conhecer o projeto pelo nome, e precisa haver pelo menos uma segunda pessoa que o defenderia sozinha.

### Usuário impactado: quem faz o trabalho hoje

Os analistas que conciliam na mão. São eles que conhecem as 900 exceções de cabeça, que sabem que o cliente X paga sempre pela conta da filial e que a planilha da aba "pendentes" tem uma coluna que só uma pessoa entende.

O que eles ganham, se der certo, é real: o fim da fila de exceção, menos retrabalho, menos sexta-feira até mais tarde no fechamento. O que eles temem é mais real ainda: **o emprego, e o valor do saber que só eles têm.** Quem é a única pessoa que entende as exceções tem uma forma de segurança no trabalho. Um agente que aprende as exceções tira essa segurança, mesmo que ninguém seja demitido.

E o risco de ignorar esse papel tem nome: **sabotagem silenciosa.** Ninguém vai dizer não. Ninguém vai levantar a mão na reunião. O analista só vai continuar mantendo a planilha paralela "por segurança", vai corrigir toda proposta do agente mesmo quando ela está certa, vai rotular os casos de eval com pressa, e o projeto vai morrer de "baixa adoção" seis meses depois, num relatório em que ninguém tem culpa.

Esse papel é tão importante que tem uma seção inteira mais abaixo, a 8.

### Blocker: quem pode dizer não

TI, segurança da informação, jurídico, compliance, às vezes a auditoria interna. O nome do papel é injusto, e eu prefiro deixar isso claro logo: **o blocker costuma ter razão.**

O incentivo dele é assimétrico de um jeito brutal. Se o projeto der certo, ninguém lembra que ele aprovou. Se vazar um dado de cliente ou uma baixa sair errada, é o nome dele no relatório de incidente. Ele não ganha nada visível com o seu projeto: só o risco que não aconteceu. Querer risco zero, nessa posição, não é burocracia. É racional.

O que ele precisa ver é exatamente o que o Dia 9 construiu: **o agente com identidade própria, acesso com menor privilégio, guardrail em código, trace auditável e um plano de desligamento em cinco minutos.** O checklist de go-live do Dia 9, na prática, é um documento escrito para o blocker.

E o momento em que ele entra é tudo. Blocker que conhece o projeto na véspera do go-live trava. Blocker que entra na semana 1, ajuda a desenhar o acesso e vê o trace funcionando no shadow vira, muitas vezes, o maior defensor técnico do projeto, porque ele finalmente viu alguém levar segurança a sério desde o começo. **A diferença entre blocker e aliado é quase sempre calendário.**

### Dono do dado: o mais esquecido, e o que mais trava

Esse é o papel que me fez repensar como eu monto projeto.

O dono do dado é quem administra o sistema de onde o agente precisa ler. No caso, o administrador do ERP, a tesouraria que controla o acesso ao banco, às vezes a pessoa que mantém a planilha que virou sistema. Ele não é o comprador. Não é o usuário. Não está na cadeia de aprovação. Por isso ele **não aparece em nenhuma reunião**, e ninguém pensa nele até a semana 5, quando o projeto pede uma credencial de API para o ERP e descobre que existe uma fila de mudança com comitê quinzenal.

Por que ele é o que mais trava? Porque o incentivo dele é o pior de todos. O blocker, pelo menos, tem o papel de avaliar. O dono do dado só tem a desvantagem: se o seu agente fizer chamadas demais e derrubar o ERP no dia do fechamento, quem é acordado às três da manhã é ele. **Ele não ganha quase nada, e é por isso que ninguém lembra dele.**

O que ele precisa ver é concreto: acesso **só de leitura** na fase 1, **limite de chamadas** por minuto, horário de execução fora do pico, e um plano de volta se algo der errado. São exatamente as coisas que o Dia 6 (idempotência) e o Dia 7 (retry com backoff, circuit breaker) desenharam. Elas não são só boa engenharia. São o argumento que convence o dono do dado.

A frase que eu passei a usar na primeira reunião: **"Quem é o dono do dado aqui? Eu queria essa pessoa na primeira semana."** Ela costuma gerar um silêncio curto, e depois alguém diz um nome que não estava na lista de convidados. Esse silêncio é o projeto se salvando.

### Três observações sobre o mapa

**Uma pessoa pode ter dois papéis.** Numa empresa média, o CFO pode ser sponsor e champion ao mesmo tempo. Numa pequena, o analista pode ser o usuário impactado e o dono da planilha. Mapeie o papel, não o crachá.

**O mapa muda.** A champion é promovida. O blocker é trocado. Um novo diretor de tecnologia chega com outra prioridade. O sponsor map é um documento vivo, revisado a cada marco, não um slide da semana 1.

**Quem está fora do mapa também importa.** O sindicato, o contador terceirizado que recebe o fechamento, o auditor externo que vai perguntar como uma baixa foi feita. Não são papéis do projeto, mas qualquer um deles pode virar blocker tarde. Vale uma linha no mapa com "quem mais pode se importar".

## 3. MEDDICC: a checklist de quem vende contrato grande

Quem vem de dados e de engenharia costuma torcer o nariz para metodologia de vendas. Eu também torcia. Mudei de ideia quando percebi que o MEDDICC não é uma técnica de persuasão: **é uma lista de perguntas que, se você não souber responder, o seu projeto tem um buraco.**

A origem é dos anos 1990, na PTC, uma empresa americana de software de engenharia. Dick Dunkel e Jack Napoli, que trabalhavam no time de vendas sob John McMahon, analisaram por que os negócios eram ganhos, perdidos ou escorregavam de trimestre, e chegaram a um punhado de elementos que se repetiam. O acrônimo cresceu com o tempo (MEDDIC, MEDDICC, MEDDPICC) e se espalhou pelo software corporativo levado por quem passou pela PTC. As histórias sobre o tamanho do crescimento que ele gerou variam muito conforme a fonte, e a maioria das fontes vende treinamento do método, então eu fico com o que interessa: a lista.

Sete letras, aplicadas ao caso da conciliação:

| Letra | O que é | No caso da conciliação |
|---|---|---|
| **M**etrics | O número que vai mexer | Fechamento de D+9 para D+5; 0% → 80% conciliado sem toque humano |
| **E**conomic buyer | Quem tem o dinheiro e a palavra final | O CFO. Não a gerente, não o comitê |
| **D**ecision criteria | Como a decisão vai ser julgada | Payback abaixo de 12 meses, aprovação de segurança, dado dentro da nuvem do cliente |
| **D**ecision process | O rito: quem aprova o quê, em que ordem | Comitê de TI quinzenal → jurídico revisa o contrato de dados → compras pede três cotações → CFO assina |
| **I**dentify pain | A dor, em fato do passado | O último fechamento saiu em D+11 e atrasou o relatório para o conselho |
| **C**hampion | Quem defende quando você não está | A gerência de recebíveis |
| **C**ompetition | As alternativas | Contratar mais dois analistas; esperar o módulo de IA do ERP; o time interno tentar; **não fazer nada** |

Três letras dessas merecem mais que uma linha.

**Decision process é diferente de decision criteria, e é onde mora o atraso.** O critério diz *o que* precisa ser verdade. O processo diz *quem* precisa confirmar e *em que ordem*. Eu já vi projeto com critério cumprido esperar dois meses porque ninguém sabia que compras exigia três cotações para qualquer contrato acima de um valor. Se você não sabe o rito, você não sabe o prazo. E se você não sabe o prazo, o seu cronograma de 14 semanas é ficção a partir da semana 2.

**Competition inclui "não fazer nada", e ele é o concorrente que mais ganha.** Não fazer nada é barato, não tem risco de carreira e ninguém é demitido por ter mantido o processo que já existia. Para cada projeto de IA, a alternativa mais forte é a planilha atual com uma pessoa a mais. Se a sua conta não bate isso com folga, o cliente vai escolher, com razão, não fazer nada. E vale outra alternativa que aparece cada vez mais: "o fornecedor do ERP disse que vai lançar um módulo de IA no ano que vem". Essa promessa compete com você de graça, sem precisar entregar nada.

**Identify pain se responde com o passado, não com o futuro.** É o Mom Test do Dia 10 voltando. "A conciliação é demorada" não é dor identificada. "O fechamento de março saiu em D+11, o relatório do conselho atrasou e o CFO teve que explicar por quê" é dor identificada, com data, consequência e uma pessoa que se lembra.

O uso que eu passei a fazer do MEDDICC não é como roteiro de venda. É como **checklist de risco do projeto.** Cada letra que eu não consigo preencher é uma pergunta para a próxima reunião. Se eu chego na semana 3 sem saber o decision process, eu sei que o meu prazo está errado. Se eu não sei quem é o economic buyer, eu sei que posso estar construindo para alguém que não tem orçamento.

## 4. Política tem calendário

Juntando os dois últimos blocos, a pergunta prática fica assim: **em cada marco do projeto, quem precisa dizer sim?**

<figure>
  <img src="{{ '/assets/images/desafio-fde/dia-11-marcos-e-quem-diz-sim.svg' | relative_url }}" alt="Seis caixas em sequência, uma por marco do SOW, cada uma com a entrega verificável e quem precisa dizer sim. Um, discovery, semanas um e dois: entrega o mapa de processo e a baseline medida; dizem sim a champion e o dono do dado, com o acesso de leitura. Dois, protótipo, semanas três e quatro: entrega a suíte de eval com casos reais rotulados; diz sim o usuário impactado, autor dos casos. Três, shadow, semanas cinco a oito: entrega a conta medida, com acerto, tempo e override; dizem sim o sponsor, sobre a conta, e o blocker, sobre o acesso. Quatro, go-live, semana nove: entrega o agente em produção com humano efetivando a baixa; dizem sim sponsor e blocker, contra o aceite escrito. Cinco, operação assistida, semanas dez a treze: entrega oitenta por cento sem toque humano e a adoção medida; diz sim o usuário impactado, que usa ou contorna. Seis, saída, semana catorze, em caixa escura: entrega infraestrutura como código, evals, servidores MCP e runbooks; diz sim o dono treinado, que assume sem a gente. Uma chave verde abaixo dos três primeiros marcos indica fase um, com preço fixo, que entrega a conta; uma chave laranja abaixo dos três últimos indica fase dois, com marcos e recorrente, que entrega a operação. Uma faixa no pé diz que, se a conta da fase um não fechar, o projeto acaba cedo e barato, e isso está escrito no SOW desde o primeiro dia; e que essa não é uma cláusula de proteção do fornecedor, mas a que mais facilita o sim do sponsor, porque ele aprova um experimento e não um ato de fé.">
  <figcaption>Cada marco tem uma entrega que se verifica e uma pessoa que pode travar. As duas coisas vão para o contrato.</figcaption>
</figure>

Olhando assim, três coisas ficam óbvias, e nenhuma delas era óbvia para mim antes de desenhar.

**O dono do dado é o primeiro a precisar dizer sim**, não o último. Sem acesso de leitura na semana 1, não existe baseline medida na semana 2, e sem baseline o resto do projeto não tem régua.

**O usuário impactado aparece duas vezes**, e nas duas ele tem poder de veto informal. No protótipo, porque é ele quem rotula os casos de eval: rótulo apressado gera eval mentiroso. Na operação assistida, porque é ele quem usa ou contorna. Um papel que aparece duas vezes no calendário merece ser tratado como sócio, não como público.

**O último sim não é do sponsor.** É do dono treinado, a pessoa do cliente que vai assumir o sistema. Se ela não disser "eu consigo operar isso sem vocês", o projeto não terminou, por mais que o número tenha mexido.

## 5. O SOW de uma página

O SOW, *statement of work*, é o documento que diz o que vai ser entregue, como vai ser medido e quando acaba. Em consultoria tradicional ele costuma ser longo e vago. Em deployment, eu defendo o contrário: **curto e numérico.** Se não cabe em uma página, tem coisa demais ou coisa de menos.

Seis elementos são obrigatórios. Escrevi o do caso da série, com números fictícios, consistentes com os dias anteriores:

```
SOW · Agente de conciliação de recebíveis · Fase 1 + Fase 2

1. NÚMERO-ALVO E COMO SE MEDE
   % de recebimentos conciliados sem toque humano: de 0% para 80% em 90 dias.
   Fechamento mensal: de D+9 para D+5 até o 2º fechamento após o go-live.
   Baseline: 4 semanas de log do ERP + 30 casos cronometrados (semanas 1–2).
   Fonte do dado: relatório de baixas do ERP. Não é dashboard nosso.

2. ESCOPO E O QUE ESTÁ FORA
   Dentro: recebimentos de 1 banco (PIX, boleto, TED) contra cobranças do ERP.
           O agente propõe; uma pessoa efetiva a baixa.
   Fora:   contas a pagar. Recebíveis de cartão. Um segundo banco.
           Baixa automática sem aprovação humana. Limpeza do histórico
           anterior ao go-live. Migração ou customização do ERP.
   Antes do agente: o cliente passa a emitir PIX Cobrança com txid.
           Isso reduz o volume que o agente precisa conciliar, e o preço.

3. CRITÉRIO DE ACEITE (go-live)
   Em shadow, sobre ≥ 2.000 recebimentos reais:
   precisão das propostas ≥ 99,5% · zero baixa errada acima de R$ 1.000
   escalonamento ≤ 20% · regressão 100% · custo ≤ R$ 0,50 por lançamento.
   Medido pela suíte de eval entregue no marco 2.

4. MARCOS
   Discovery (sem 1–2) → Protótipo (3–4) → Shadow (5–8) → Go-live (9)
   → Operação assistida (10–13) → Saída (14).
   Fase 1 termina no fim do shadow, com a conta medida.
   Se a conta não fechar, o projeto encerra ali, sem multa.

5. PARTICIPAÇÃO DO CLIENTE
   Acesso de leitura ao ERP e ao extrato até o dia 5.
   Dono do dado nomeado, com 4h/semana nas semanas 1–4.
   2 analistas com 3h/semana para rotular casos (semanas 3–8).
   Uma pessoa com autoridade para assinar o aceite.
   Atraso de acesso desloca o cronograma na mesma medida.

6. CRITÉRIO DE SAÍDA
   O squad sai quando o dono treinado opera 2 fechamentos seguidos
   sem chamar a gente, com os runbooks entregues.
   Ficam no cliente: IaC na nuvem dele, servidores MCP, suíte de eval
   com o runner, runbooks de incidente e de troca de modelo.
```

Agora, o que eu aprendi escrevendo isso, item por item.

**O número-alvo vem com a fonte do dado, e a fonte é do cliente.** "Fonte do dado: relatório de baixas do ERP. Não é dashboard nosso." Essa linha resolve uma briga antes de ela começar. Se o número é medido num painel que o fornecedor construiu, toda discussão de resultado vira discussão sobre o painel. Se é medido no relatório que o cliente já usa, não tem o que discutir.

**O "fora" tem o mesmo tamanho do "dentro", de propósito.** Esse é o item que mais protege os dois lados, e o que mais gente escreve com pressa. Tudo que não está escrito como fora vai ser pedido como dentro, normalmente na semana 7, normalmente por alguém que não estava na reunião de escopo. "Já que o agente lê o banco, dá para fazer o contas a pagar também?" Dá. É a fase 3, com outro preço. Escrever isso no dia 1 transforma o pedido de escopo extra numa conversa de expansão, e não numa conversa de favor.

**Uma linha do escopo reduz a minha própria receita, e está ali de propósito.** "Antes do agente: o cliente passa a emitir PIX Cobrança com txid." No Dia 10 eu mostrei que o `txid` transforma boa parte da conciliação numa consulta de banco de dados, sem modelo nenhum. Botar isso no SOW diminui o volume que o agente concilia e, se o preço for por conciliação, diminui o que eu recebo. Eu faria assim mesmo. Um cliente que percebe que você cortou o seu próprio escopo para resolver o problema dele mais barato confia no resto do documento. Essa confiança vale mais do que as conciliações que eu deixei de cobrar.

**O critério de aceite não é negociado depois do resultado.** É a tabela do gate do Dia 8, copiada para o contrato. Ela precisa estar assinada antes do shadow começar. Se não estiver, todo resultado vira negociação: 99,1% é bom o suficiente? Depende de quem está mais cansado na reunião. O guia de FDE da Webfuse coloca isso no quarto passo do que ele chama de *operating loop*: **construir o eval antes do agente**, com casos históricos reais de resposta conhecida, porque ele se torna "a definição compartilhada de funcionando". Eu leio "compartilhada" como "assinada".

**A participação do cliente é a cláusula que mais se ignora e a que mais atrasa.** Ela define o prazo para liberar o acesso, as horas reservadas do dono do dado e o tempo dos analistas para rotular casos, e termina com a linha mais importante do bloco: **"atraso de acesso desloca o cronograma na mesma medida".** Sem ela, o fornecedor absorve todos os atrasos do cliente e entrega tarde um projeto que começou tarde por culpa de outro. Com ela, o atraso de acesso deixa de ser invisível e passa a ter dono.

**O critério de saída responde à pergunta que ninguém faz no começo: quando a gente vai embora?** "O squad sai quando o dono treinado opera dois fechamentos seguidos sem chamar a gente." É um critério de comportamento, não de calendário. A semana 14 é a previsão, mas o que encerra o projeto é a pessoa do cliente conseguir operar sozinha.

### Uma referência de prazo

Quanto tempo é razoável? Em junho de 2026, a AWS anunciou uma organização de *forward deployed engineering* com US$ 1 bilhão de investimento inicial, e a cobertura da imprensa especializada descreveu o modelo como **pods de cinco a seis engenheiros trabalhando dentro do cliente em sprints de 45 dias.** A vice-presidente responsável resumiu o ritmo assim: idear em 45 minutos, validar em 45 horas e colocar algo de valor no fluxo de trabalho em 45 dias.

Eu levo isso como **ordem de grandeza, não como promessa.** A própria executiva disse que o prazo varia por caso, e houve análise mais cética sugerindo tratar os 45 dias como meta de marketing até os primeiros pilotos terminarem. Mas o recado de fundo interessa: a referência de mercado para chegar em produção é medida em semanas, não em trimestres. O meu SOW chega ao go-live na semana 9, cerca de 60 dias, com uma fase de shadow mais longa do que a média porque o custo do erro, em conciliação, é alto.

## 6. O que fica quando a gente sai

A sexta seção do SOW merece desenvolvimento próprio, porque ela define se o projeto vira dependência ou autonomia.

O guia da Webfuse tem um teste que eu adotei: **"dá para rodar isso sem o fornecedor?"** Um deployment de verdade responde sim por desenho. Um deployment disfarçado de consultoria precisa de um contrato novo para continuar funcionando.

A lista de artefatos que precisam ficar no cliente, segundo o guia, adaptada ao caso:

- **Infraestrutura como código, na nuvem do cliente.** Não um ambiente configurado na mão. O guia tem uma frase ótima: um ambiente que existe *"só como os cliques de console de um FDE"* é um ambiente que o cliente não consegue reconstruir.
- **Os servidores MCP e os schemas das tools**, com as checagens de permissão no código do servidor, não no prompt. São reaproveitáveis por qualquer cliente compatível: se o cliente trocar de modelo ou de fornecedor amanhã, as tools continuam dele.
- **A suíte de eval e o runner que a executa.** Sem isso, o cliente não consegue saber se o agente continua funcionando depois de uma troca de modelo. Com isso, ele consegue julgar até a próxima proposta de fornecedor.
- **Os runbooks.** O manual de operação: o que fazer num incidente, como trocar a versão do modelo, como mudar uma permissão, quem desliga e com qual gatilho (o Dia 9 definiu: override acima de 15% num dia).
- **Um dono treinado, com nome.** Esse é o artefato que não é arquivo, e é o mais difícil.

O guia chama o fracasso desse item de **"lock-in por acidente"**: sob pressão de prazo, o FDE pula a infraestrutura como código, pula a suíte de eval, ou entrega sem um dono nomeado. Ninguém planejou deixar o cliente dependente. Só ficou mais rápido assim. E seis meses depois, ou o cliente chama o fornecedor para cada mudança, ou o sistema para de funcionar e ninguém sabe por quê.

Na apresentação da AWS apareceu um ponto que eu acho complementar: o ativo mais valioso deixado no cliente seria a **camada semântica**, o mapa das entidades e dos processos do negócio, que a executiva chamou de "ouro". No caso da série, isso é o glossário de exceções do Dia 10, as categorias com nome, volume, dono e desfecho. Ele vale para o próximo agente, para o próximo fornecedor e até para o próximo analista contratado.

Tem uma tensão aqui que eu prefiro nomear em vez de esconder. **Para quem vende, deixar o cliente autônomo parece ruim para o negócio.** Se ele consegue operar sozinho, por que me pagaria de novo? A resposta está na seção 9: porque quem sai bem de um projeto é chamado para o próximo. Dependência gera renovação forçada, e renovação forçada gera um cliente que procura alternativa em silêncio. Autonomia gera confiança, e confiança gera o segundo caso.

## 7. Cinco formas de cobrar, e quem carrega o risco em cada uma

Escopo fechado, a pergunta seguinte é como cobrar. Existem cinco modelos principais, e eu descobri que a forma mais útil de compará-los não é por preço. É por **quem carrega o risco de o agente não funcionar.**

<figure>
  <img src="{{ '/assets/images/desafio-fde/dia-11-quem-carrega-o-risco.svg' | relative_url }}" alt="Uma barra horizontal em degradê vai do cinza, à esquerda, marcado risco com o cliente, ao laranja, à direita, marcado risco com o fornecedor. Abaixo, cinco cartões em ordem. Tempo e material, por hora ou squad-mês: o cliente paga o esforço, dê certo ou não; exige confiança no time; vira serviço e não escala. Fixo por marco, pago a cada entrega: o risco de prazo passa para o fornecedor; exige escopo fechado e o fora escrito; vira projeto previsível. Plataforma mais deploy, implantação mais assinatura, com borda verde destacada: divide o risco, porque a implantação é única e a operação recorre; exige plataforma de verdade, não um repositório; vira receita recorrente. Por tarefa, por conciliação aceita, com borda laranja destacada: o fornecedor só ganha se o agente for usado; exige definição escrita do que conta como tarefa; vira receita que escala com o uso. Por resultado, porcentagem da economia gerada: o fornecedor só ganha se o número do cliente mexer; exige baseline e uma regra de atribuição assinada; vira sociedade no resultado. Uma chave sob os cartões de plataforma e de tarefa indica o que eu proporia na fase dois. Uma faixa escura no pé descreve a proposta: fase um com preço fixo; depois, assinatura de plataforma mais um real e vinte por conciliação aceita, com teto mensal; e a cláusula que fecha o incentivo, em que cada baixa desfeita no mês seguinte desconta dez conciliações, para que agressividade deixe de dar lucro.">
  <figcaption>Ninguém escolhe modelo de preço pelo preço. Escolhe por quanto risco cada lado aceita carregar.</figcaption>
</figure>

**Tempo e material.** Por hora ou por squad-mês. É o mais simples de vender e o mais honesto quando ninguém sabe o tamanho do problema. Mas o risco fica inteiro com o cliente: ele paga o esforço, dê certo ou não. E ele tem um efeito colateral pior, que é o incentivo. Quem cobra por hora não tem motivo econômico para terminar rápido nem para deixar o cliente autônomo. É serviço, e serviço não escala.

**Preço fixo por marco.** Pago a cada entrega, contra critério. O risco de prazo passa para o fornecedor: se o protótipo levar seis semanas em vez de duas, o problema é dele. Funciona muito bem quando o escopo é fechado, e por isso exige o SOW da seção anterior com o "fora" bem escrito. Sem isso, preço fixo vira uma briga permanente sobre o que estava incluído. É o modelo natural para a **fase 1**: o cliente sabe exatamente quanto custa descobrir se a conta fecha.

**Plataforma mais deploy.** Uma taxa de implantação, cobrada uma vez, mais uma assinatura recorrente pela operação. É o modelo que transforma projeto em receita recorrente, e o mais desejado por quem quer construir empresa, não consultoria. A exigência é dura: **precisa existir plataforma de verdade**, ou seja, uma coisa que roda igual para vários clientes, com atualização centralizada. Cobrar assinatura de um repositório customizado para um cliente só é cobrar manutenção com nome bonito, e o cliente percebe no segundo ano.

**Por tarefa ou resolução.** Um valor por conciliação feita, por ticket resolvido, por documento processado. É o modelo que mais cresceu no mercado de agentes, especialmente em atendimento. A Intercom cobra **US$ 0,99 por resolução** no seu agente de atendimento, e a Sierra, empresa do Bret Taylor, cobra quando o agente resolve sozinho e não cobra quando ele transfere para uma pessoa. O argumento do Taylor, repetido em várias entrevistas, é que token é insumo, e o que o cliente quer comprar é o trabalho feito.

**Por resultado.** Uma porcentagem da economia gerada, ou do caixa antecipado. É o mais alinhado de todos no papel: o fornecedor só ganha se o número do cliente mexer. E é o mais difícil de operar, por três motivos que o Dia 10 já anunciou. Exige **baseline** medida antes, senão não há economia a calcular. Exige uma **regra de atribuição** assinada: se o fechamento caiu de D+9 para D+5, quanto disso foi o agente e quanto foi o novo controller que entrou no mesmo mês? E exige que o cliente aceite abrir o número dele para o fornecedor todo mês.

### A definição é o contrato

Tem um detalhe do modelo por tarefa que eu acho a coisa mais instrutiva do dia, e ele está nas regras de cobrança da própria Intercom.

O que conta como resolução? Segundo a documentação deles, conta quando o cliente confirma que o problema foi resolvido, **ou quando ele não pede mais ajuda depois da resposta do agente.** O cliente que simplesmente vai embora conta como uma "resolução presumida", e é cobrado.

Não estou dizendo que isso é errado. Pode ser uma regra perfeitamente razoável para atendimento. O ponto é outro: **num modelo por tarefa, a definição de "tarefa" é o produto.** Quem escreve a definição escreve a fatura. E um cliente que não leu a definição com cuidado pode descobrir, três meses depois, que está pagando por um número que não é o que ele achava.

Agora traga isso para a conciliação, onde o erro é muito mais caro que um cliente frustrado com o chat. Se eu cobro por **baixa proposta**, eu tenho incentivo para propor baixa em tudo, inclusive no que deveria escalar. E propor baixa errada é exatamente o erro caro do Dia 8: dar baixa em quem não pagou. **O modelo de preço errado paga o fornecedor para cometer o erro que o guardrail existe para evitar.**

### O que eu proporia no caso

Juntando tudo, a minha proposta para o caso da série seria um híbrido:

- **Fase 1, preço fixo.** Discovery, protótipo e shadow, com a conta medida no fim. O cliente sabe quanto custa descobrir, e pode parar ali.
- **Fase 2, plataforma mais tarefa.** Uma assinatura fixa pela operação (R$ 6 mil por mês, cobrindo infraestrutura, modelo e manutenção) mais **R$ 1,20 por conciliação aceita**, e não por conciliação proposta. "Aceita" quer dizer que a pessoa aprovou sem editar.
- **Teto mensal** na parte variável, para o CFO saber o pior caso do orçamento.
- **E a cláusula que fecha o incentivo:** cada baixa desfeita no mês seguinte desconta dez conciliações da fatura.

Com 80% de 6.000 recebimentos, são 4.800 conciliações aceitas por mês: R$ 5.760 de variável mais R$ 6 mil de assinatura, perto dos R$ 12 mil de custo recorrente que entraram na conta do Dia 10. A estrutura não mudou o ROI. Mudou quem ganha com o quê.

A cláusula do desconto merece uma conta. Se cada baixa desfeita apaga o ganho de dez boas, o fornecedor deixa de ganhar dinheiro quando uma em cada dez baixas volta, ou seja, quando o acerto cai para perto de 90%. O gate do Dia 8 já exige 99,5%, então a cláusula não pesa no mês normal. Ela pesa no mês ruim, e principalmente na **decisão de calibragem**: com ela, deixar o agente mais agressivo para inflar o volume deixa de ser bom negócio. O incentivo do fornecedor passa a apontar para o mesmo lado do guardrail.

Eu não sei se essa é a estrutura que o mercado usa. Sei que ela responde às três perguntas que eu faria se estivesse do outro lado: quanto eu pago se não funcionar (pouco, e só na fase 1), quanto eu pago no pior mês (o teto) e o que acontece se o agente errar (o fornecedor sente no bolso).

### Uma nota sobre margem

Tem uma tentação, quando se fala de preço, de querer margem de software desde o primeiro contrato. O ensaio da a16z sobre FDE é contra isso, e com números. A ServiceNow abriu capital com margem bruta de 63,2%, e a Workday com 54,1%, muito abaixo dos cerca de 80% considerados ideais em software. Em 2024 estavam em 79% e 75%. A Salesforce teria queimado mais de US$ 52 milhões para gerar US$ 22 milhões de receita antes de montar o ecossistema de parceiros.

A recomendação do texto, entre as boas práticas, é desenhar incentivos que permitam **vender serviço a preço de custo**, porque à medida que o contrato cresce, *"tudo o que importa é sucesso, e retenção"*. Isso conversa diretamente com a próxima seção: a margem do primeiro contrato importa menos do que o que ele abre.

## 8. O usuário que pode matar o agente sem dizer não

Volto ao papel que deixei em aberto na seção 2, porque ele é o que mais me preocupa num projeto como esse.

Em 2024, a Klarna anunciou que o seu assistente de IA estava fazendo o trabalho equivalente a **700 atendentes**, com 2,3 milhões de conversas no primeiro mês. Virou o caso mais citado do ano sobre IA substituindo trabalho humano. Em maio de 2025, o CEO, Sebastian Siemiatkowski, disse publicamente que a empresa tinha ido longe demais: o custo tinha sido um fator de avaliação "predominante demais", e a qualidade do atendimento caiu. A Klarna voltou a contratar atendentes humanos, num modelo híbrido.

Tem duas leituras desse caso, e as duas me interessam. A primeira é técnica: o agente foi medido por custo e não por qualidade, e o Dia 8 inteiro é sobre não fazer isso. A segunda é de gestão: **um projeto de IA anunciado como substituição de pessoas cria um ambiente em que as pessoas que sobram não têm nenhum motivo para ajudá-lo a funcionar.**

E elas têm muitas formas de não ajudar. Em abril de 2026, uma pesquisa da Writer com a Workplace Intelligence, com 2.400 profissionais em cinco países, chegou a um número que circulou bastante: **29% admitiram agir contra a estratégia de IA da própria empresa**, e entre os mais jovens o número chegou a 44%. O que eles chamam de sabotagem é concreto: usar ferramenta não aprovada, colocar informação da empresa em ferramenta pública, recusar a ferramenta oficial. É uma pesquisa encomendada por um fornecedor de IA, então eu trato o número como indício, não como fato. Mas a direção bate com o que eu já vi acontecer em projeto de dado: a adoção não morre com um não. Morre com mil pequenos "agora não".

### Como a sabotagem silenciosa aparece num agente de conciliação

Ela é discreta, e por isso precisa de métrica:

- **A planilha paralela continua viva.** O analista concilia na planilha "só para conferir", e o agente vira um segundo trabalho, não um alívio.
- **Override em tudo.** A pessoa corrige toda proposta, mesmo as certas, porque corrigir é o que mantém o controle com ela.
- **Rótulo apressado.** Os casos de eval do marco 2 são rotulados de qualquer jeito, e a suíte passa a medir uma coisa que não é a realidade.
- **Exceção que contorna o agente.** Os casos difíceis vão direto para a pessoa, por e-mail, sem passar pela fila. O agente fica só com os fáceis, e o número de exceção tratada nunca sobe.

Nenhum desses comportamentos é má-fé declarada, mas todos fazem o projeto parecer pior do que ele de fato é.

### O override tem dois lados ruins

O Dia 9 apresentou a **taxa de override** como o sinal mais honesto da qualidade: a proporção de propostas que a pessoa corrige antes de aprovar. Hoje eu acrescento uma leitura que eu não tinha feito.

Override **alto demais** pode ser agente ruim, ou pode ser desconfiança. Para separar uma coisa da outra, olhe o conteúdo das correções: se a pessoa muda a proposta para outra cobrança, o agente errou; se ela "corrige" para a mesma resposta que o agente deu, ou rejeita e depois aprova igual, é desconfiança.

Override **baixo demais** também é sinal ruim. Se a pessoa aprova 100% das propostas, inclusive as que o agente marcou como incertas, ela parou de revisar. É o viés de automação: o humano no circuito virou um carimbo. E um carimbo humano dá a falsa sensação de controle, que é pior que controle nenhum, porque ninguém está olhando enquanto todo mundo acha que alguém está.

**Zero override é tão alarmante quanto 40%.** O número saudável fica no meio, e muda com o tempo: alto no começo, caindo à medida que o agente melhora e a confiança é construída. Eu passei a acompanhar a curva, não o número.

### A métrica de adoção é tão importante quanto a de precisão

Um agente com 99,5% de precisão que ninguém usa tem impacto zero. Então, na operação assistida, eu acompanharia quatro números junto com a precisão:

| Métrica | O que mostra | Sinal de alerta |
|---|---|---|
| **Cobertura** | % dos recebimentos que passam pelo agente | Cai, ou estaciona abaixo do volume total |
| **Override** | % das propostas corrigidas | Acima de 15% num dia, ou perto de zero por semanas |
| **Tempo até aceitar** | Quanto a pessoa leva para aprovar uma proposta | Tempo igual ao do processo manual: ela está refazendo |
| **Trabalho paralelo** | Se a planilha antiga continua sendo atualizada | Qualquer atualização depois do mês 1 |

A última não aparece em painel nenhum: só se descobre perguntando, ou olhando a data da última modificação da planilha.

### O que funciona

Três coisas, e nenhuma é discurso motivacional.

**Fazer do analista o autor do eval.** No marco 2, os casos da suíte de eval são as exceções que o analista conhece de cabeça. Se ele escreve os casos, o agente passa a ser avaliado pelo critério dele. Isso muda a relação: ele deixa de ser o substituído e vira o professor. E dá crédito público, com nome, a quem ensinou o agente.

**Mostrar o agente errando, e ele corrigindo.** O momento em que um analista vê o agente propor uma baixa errada, e é ele quem pega o erro, é o momento em que ele entende o seu papel novo. A demo perfeita produz o efeito oposto: se o agente nunca erra, para que eu sirvo?

**Responder a pergunta do emprego com honestidade.** No Dia 10, a oitava pergunta do roteiro de discovery era "se isso melhorasse, o que você faria com o tempo que sobrasse?". A resposta dessa pergunta precisa virar compromisso do sponsor antes do go-live: a análise de inadimplência que está na fila há um ano, o atendimento ao cliente que paga errado, o fechamento sem hora extra. Se não houver destino para as horas, eu não vou prometer que haverá. Mas eu também não vou deixar o analista descobrir pelo corredor.

O agente precisa ser visto como **um colega que tira o trabalho chato**, e isso não se consegue dizendo. Se consegue desenhando o projeto para que seja verdade.

## 9. Expansão: a métrica que prova o modelo

Até aqui, tudo foi sobre a primeira entrega. Mas o modelo de FDE só fecha a conta econômica por causa da segunda.

A ideia tem um nome antigo em software, *land and expand*: entrar com um caso pequeno e crescer dentro da conta. No deployment, ela é ainda mais forte, por três motivos.

**O custo de aprendizado já foi pago.** Os conectores do banco e do ERP existem. O glossário do domínio existe. O runner de eval existe. O segundo caso começa a partir de um patamar que o primeiro levou semanas para construir.

**A confiança foi construída.** O dono do dado já disse sim uma vez e viu que o sistema dele não caiu. O blocker já conhece o trace. O sponsor viu o número mexer. Cada uma dessas pessoas é uma conversa que, no segundo caso, começa do sim em vez do zero.

**Os próximos problemas foram descobertos no caminho.** Esse é o motivo mais interessante, e o menos óbvio. Ao conciliar 900 exceções por mês, o agente gera um dado que ninguém tinha: o mapa de onde a cobrança nasce mal. Os 420 recebimentos por mês de pagador diferente do titular dizem algo sobre a régua de cobrança. O próximo caso não foi vendido. Foi encontrado.

<figure>
  <img src="{{ '/assets/images/desafio-fde/dia-11-nrr-e-expansao.svg' | relative_url }}" alt="Painel em duas partes. À esquerda, um gráfico de cascata da receita de uma mesma base de clientes, em reais mil, com números fictícios: receita inicial de mil, em barra cinza escura; expansão de mais seiscentos, em verde; contração de menos cinquenta e churn de menos cinquenta, em laranja; e mil e quinhentos um ano depois, em barra escura. Abaixo, a fórmula: NRR igual a mil mais seiscentos menos cinquenta menos cinquenta, dividido por mil, igual a cento e cinquenta por cento. À direita, quatro caixas em sequência descrevem o land and expand na conta da conciliação. Um, conciliação de recebíveis, o primeiro contrato: nascem aqui os conectores do banco e do ERP, o glossário, o runner de eval e a confiança de quem assina. Dois, régua de cobrança, em que a exceção vira o próximo caso: os quatrocentos e vinte pagadores divergentes por mês mostram onde a cobrança nasce mal identificada, e ninguém sabia antes. Três, contas a pagar, o mesmo trilho na outra direção: reaproveita conectores, graph e guardrails; o dono do dado já disse sim uma vez e o blocker já conhece o trace. Quatro, em laranja, projeção de caixa, o que o CFO queria desde o início: só fica possível agora, porque o histórico conciliado virou dado confiável, e ninguém venderia isso no dia um. Uma faixa escura no pé diz que, se o segundo caso custa o mesmo que o primeiro, não é expansão, é outro projeto vendido para o mesmo cliente.">
  <figcaption>O quarto caso é o que o CFO queria desde o começo. Ele só fica vendável depois que os três primeiros produziram o dado.</figcaption>
</figure>

Repare no quarto passo da figura. A projeção de caixa é, muito provavelmente, o que o CFO gostaria de ter desde o primeiro dia. Mas ninguém conseguiria entregar isso no começo, porque projeção de caixa depende de histórico de recebimento confiável, e histórico confiável é justamente o que a conciliação produz. **O caso mais valioso da conta não pode ser o primeiro. Ele precisa ser construído pelos anteriores.**

### NRR: a métrica que mostra se isso está acontecendo

A forma de medir expansão é a **NRR**, *net revenue retention*, a retenção líquida de receita. Ela olha só para a base de clientes que já existia, e pergunta: um ano depois, quanto essa mesma base está pagando?

```
NRR = (Receita inicial + Expansão − Contração − Churn) ÷ Receita inicial
```

Com números redondos: uma base que pagava R$ 1 milhão, teve R$ 600 mil de expansão, R$ 50 mil de contração (clientes que reduziram o contrato) e R$ 50 mil de churn (clientes que saíram).

```
NRR = (1.000 + 600 − 50 − 50) ÷ 1.000 = 1,5 → 150%
```

**Uma NRR acima de 100% significa que a base cresce sozinha**, mesmo sem nenhum cliente novo. Com 150%, se a empresa parasse de vender amanhã, ainda cresceria 50% no ano seguinte só com quem já é cliente.

Os números públicos de empresas que operam com o modelo de FDE são altos. A Palantir, a empresa que deu origem ao modelo, reportou retenção líquida de **157% no segundo trimestre de 2026**, subindo de 150% no trimestre anterior. A HappyRobot, uma startup de agentes de voz para logística, levantou uma rodada em agosto de 2026 e o fundador disse à Fortune que a retenção líquida da empresa **passou de 150%**, com um cliente de cadeia de suprimentos que multiplicou o contrato por dez em um ano. O segundo número é declarado pela empresa, não auditado, mas mostra a forma do modelo.

### Três ressalvas sobre NRR

**NRR alta pode esconder concentração.** Se um cliente gigante multiplicou o contrato por dez, ele sozinho pode puxar a NRR da base inteira para cima. A pergunta seguinte é sempre: qual a NRR sem os três maiores clientes?

**NRR alta com custo de entrega alto é receita linear.** Esse é o ponto que mais conversa com os próximos dias. Se cada expansão exige um squad novo trabalhando o mesmo número de semanas que o primeiro caso, a receita cresce, mas o custo cresce junto. É uma consultoria crescendo, não uma empresa de produto. **Se o segundo caso custa o mesmo que o primeiro, não é expansão: é outro projeto vendido para o mesmo cliente.** Expansão de verdade é o segundo caso custando metade, porque reaproveita os componentes do primeiro. Como fazer isso é exatamente o assunto do Dia 12, com componentização, e do Dia 13, com produtização.

**Expansão depende de pessoas, não só de entrega.** A champion que foi promovida porque o primeiro caso deu certo vai para uma área nova, e leva o projeto com ela, se ela lembrar de você. É uma das formas mais comuns de expansão em software corporativo, e não aparece em nenhuma planilha: a sua melhor fonte do próximo caso é a carreira da pessoa que te trouxe no primeiro.

## 10. Quando o vocabulário de vendas ajuda, e quando atrapalha

Fechando a parte de conceito, uma ressalva que eu acho importante para quem, como eu, vem de dados e engenharia.

O vocabulário de hoje é de vendas: economic buyer, champion, MEDDICC, land and expand, NRR. Ele ajuda muito a organizar o pensamento. Mas tem um risco de usá-lo do jeito errado: **tratar as pessoas do sponsor map como alvos de convencimento, e não como pessoas com interesses legítimos.**

O blocker não é um obstáculo a contornar. É alguém que tem razão sobre um risco que você ainda não viu. O dono do dado não é alguém a convencer. É alguém cujo sistema você precisa respeitar. O usuário impactado não é alguém a "gerenciar". É alguém cuja vida profissional vai mudar por causa do seu projeto.

A diferença aparece na prática. Quem trata o sponsor map como mapa de alvos faz a reunião com cada um para dizer o que ele quer ouvir. Quem trata como mapa de interesses desenha o projeto para que cada um ganhe de verdade, e aí não precisa convencer ninguém. O primeiro jeito funciona até a primeira contradição aparecer. O segundo funciona até o fim.

Foi a coisa mais importante que eu levei de anos trabalhando entre áreas: **política corporativa não é manipulação. É gestão de interesses legítimos que não coincidem.** O sponsor map é só o jeito de escrever isso.

## Prática: o mapa e o contrato

O exercício de hoje tem duas entregas.

**1. O sponsor map do caso**, com os cinco papéis e as quatro perguntas, usando papéis genéricos e nunca nomes reais. A regra que eu me impus: para cada papel, escrever também **o que ele faria para travar o projeto se fosse ignorado.** É a coluna que mais ensina, porque obriga a pensar como a pessoa e não como o projeto.

| Papel | Ganha | Perde | Precisa ver | Entra | Como travaria se fosse ignorado |
|---|---|---|---|---|---|
| Sponsor | D+5, caixa previsível | Orçamento, credibilidade | A conta da fase 1, em faixa | Semana 0 e marcos | Não renova a fase 2 |
| Champion | Visibilidade, carreira | Capital político | Vitória pequena no mês 1 | Antes de todos | Para de defender nas reuniões em que você não está |
| Usuário impactado | Fim da fila, menos retrabalho | Emprego, saber exclusivo | Ele corrigindo o agente | Discovery e eval | Planilha paralela, override em tudo |
| Blocker | O risco que não aconteceu | Tudo, se der errado | Identidade, menor privilégio, trace | Semana 1 | Veta o go-live na véspera |
| Dono do dado | Quase nada | A estabilidade do sistema | Só leitura, limite, plano de volta | Semana 1 | A credencial "está na fila de mudança" |

**2. O SOW de uma página**, no formato da seção 5, com os seis elementos. A regra para considerar pronto: **critério de aceite com número e critério de saída com comportamento.** Se o aceite tem adjetivo ("boa precisão") ou a saída tem só data ("semana 14"), não está pronto.

E três frases que eu quero ter na ponta da língua na próxima reunião de escopo:

- *"Quem é o dono do dado aqui? Eu queria essa pessoa na primeira semana."*
- *"Sem critério de aceite com número e critério de saída, isso é serviço, não produto."*
- *"Como o time que faz isso hoje ganha com o projeto? Se a gente não souber responder, eles vão responder por nós."*

## TLDR

- **Conta fechada não aprova projeto.** Quem decide, quem paga e quem é impactado quase nunca são a mesma pessoa, e cada um tem um motivo diferente para querer ou temer o agente.
- **A previsão da Gartner de 40% de projetos agênticos cancelados até 2027** aponta custo, valor pouco claro e risco mal controlado. Nenhum dos três é "o modelo errou": são escopo, número combinado e blocker que entrou tarde.
- **Cinco papéis:** sponsor (quer o número), champion (quer a carreira), usuário impactado (teme o emprego), blocker (quer risco zero, e costuma ter razão) e dono do dado (não ganha nada e trava tudo).
- **Para cada papel, quatro perguntas:** o que ganha, o que perde, o que precisa ver e quando entra. A última é a que mais se esquece e a que mais custa.
- **A diferença entre blocker e aliado é quase sempre calendário.** Segurança que entra na semana 1 ajuda a desenhar. Segurança que entra na véspera trava.
- **O dono do dado é o primeiro sim do projeto**, não o último. Sem acesso de leitura na semana 1, não há baseline, e sem baseline não há régua.
- **Projeto de um champion é tão estável quanto o cargo dele.** Tenha mais de um fio desde cedo.
- **MEDDICC é checklist de risco**, não roteiro de venda. Cada letra que você não preenche é uma pergunta para a próxima reunião.
- **Decision process é onde mora o atraso**, e "não fazer nada" é o concorrente que mais ganha.
- **O SOW cabe numa página e tem seis elementos:** número-alvo com fonte do cliente, escopo com o "fora" do mesmo tamanho, aceite numérico assinado antes do shadow, marcos, participação do cliente com cláusula de atraso, e saída por comportamento.
- **Cortar o próprio escopo compra confiança.** Botar o `txid` no SOW reduz a receita do fornecedor e aumenta a credibilidade do resto do documento.
- **A referência de mercado para chegar em produção é medida em semanas.** A AWS fala em pods de cinco a seis engenheiros e sprints de 45 dias: trate como ordem de grandeza, não como promessa.
- **O teste da saída é "dá para rodar sem o fornecedor?".** Ficam IaC, MCP, evals, runbooks e um dono treinado. O contrário é lock-in por acidente.
- **Compare modelos de preço por quem carrega o risco**, não pelo valor. Fixo serve para descobrir; plataforma mais tarefa serve para operar.
- **No preço por tarefa, a definição de tarefa é o produto.** Quem escreve a definição escreve a fatura.
- **Cobrar por baixa proposta paga o fornecedor para cometer o erro que o guardrail evita.** Cobre por conciliação aceita, com desconto para baixa desfeita.
- **O usuário impactado mata o agente sem dizer não**: planilha paralela, override em tudo, rótulo apressado, exceção que contorna a fila.
- **Zero override é tão alarmante quanto 40%.** Acompanhe a curva, não o número.
- **Adoção se mede como precisão**: cobertura, override, tempo até aceitar e trabalho paralelo.
- **Faça do analista o autor do eval.** Ele deixa de ser o substituído e vira o professor.
- **NRR acima de 100% é a base crescendo sozinha.** A Palantir reportou 157% no segundo trimestre de 2026.
- **Se o segundo caso custa o mesmo que o primeiro, não é expansão.** É outro projeto vendido para o mesmo cliente.
- **O caso mais valioso da conta não pode ser o primeiro.** Ele é construído pelos dados que os anteriores produziram.
- **Política corporativa não é manipulação.** É gestão de interesses legítimos que não coincidem.

## Fontes

**Leituras**

- [Forward Deployed Engineering: the FDE guide — Webfuse](https://www.webfuse.com/blog/forward-deployed-engineering-fde-guide) — o *operating loop* em oito passos, com o eval construído antes do agente e a passagem para um dono nomeado; a lista de artefatos que ficam no cliente; o teste "dá para rodar sem o fornecedor?" e a ideia de lock-in por acidente.
- [Trading Margin for Moat: Why the Forward Deployed Engineer Is the Hottest Job in Startups — Joe Schmidt, a16z (jun/2025)](https://a16z.com/services-led-growth/) — as boas práticas (vender pequeno no começo, alinhar incentivos, deixar rastro, estar presente para entender o poder interno) e a resposta aos contra-argumentos, com as margens de ServiceNow, Workday e Salesforce.
- [Why AI companies hire forward deployed engineers: the a16z thesis, 15 months later — A10X](https://a10x.dev/blog/why-ai-companies-hire-forward-deployed-engineers) — o que mudou desde o ensaio da a16z: demanda por vagas, rodadas de investimento e a estrutura das equipes. É o blog de uma empresa que vende treinamento na área: leia com isso em mente.
- [AWS creates forward deployed engineering hub — CIO Dive (jun/2026)](https://ciodive.com/news/aws-creates-forward-deployed-engineering-hub/824109) e [SiliconANGLE](https://siliconangle.com/2026/06/30/aws-launches-forward-deployed-engineering-team-speed-enterprise-agentic-ai-adoption/) — o investimento de US$ 1 bilhão, os pods de cinco a seis engenheiros, os sprints de 45 dias e a camada semântica como ativo deixado no cliente.
- [Gartner prevê cancelamento de mais de 40% dos projetos de IA agêntica até 2027 — cobertura da Reuters (jun/2025)](https://www.thestar.com.my/tech/tech-news/2025/06/25/over-40-of-agentic-ai-projects-will-be-scrapped-by-2027-gartner-says) — custo crescente, valor de negócio pouco claro e controle de risco insuficiente como causas.
- [Who created MEDDIC — MEDDICC](https://meddicc.com/resources/who-created-meddic) — a origem do método na PTC nos anos 1990, com Dick Dunkel, Jack Napoli e John McMahon. É o site de uma empresa que vende treinamento do método: os detalhes históricos variam entre fontes.
- [Fin pricing — Intercom](https://fin.ai/pricing) e [como a Intercom define uma resolução](https://www.intercom.com/help/en/articles/8205718) — o preço por resolução e a regra que conta a resolução presumida. Um bom exemplo de como a definição de tarefa é o contrato.
- [Bret Taylor on AI agents and outcome-based pricing — Cheeky Pint](https://cheekypint.transistor.fm/27) — o fundador da Sierra explicando por que cobrar só quando o agente resolve, e não por uso de token.
- [Klarna is hiring customer service agents after AI couldn't cut it — Entrepreneur (mai/2025)](https://www.entrepreneur.com/business-news/klarna-ceo-reverses-course-by-hiring-more-humans-not-ai/491396) — o recuo da Klarna, com a declaração do CEO de que o custo foi um fator de avaliação predominante demais.
- [Enterprise AI adoption survey 2026 — Writer](https://writer.com/blog/enterprise-ai-adoption-survey-results-press-release/) e [cobertura da Fortune](https://fortune.com/2026/04/08/gen-z-workers-sabotage-ai-rollout-backlash/) — os 29% de profissionais que admitem agir contra a estratégia de IA da empresa. Pesquisa encomendada por um fornecedor de IA: trate como indício.
- [Palantir Q2 2026 — destaques da teleconferência de resultados](https://www.gurufocus.com/news/9000347/palantir-technologies-inc-pltr-q2-2026-earnings-call-highlights-record-93-revenue-growth-and-raised-guidance-signal-unprecedented-ai-demand) — retenção líquida de 157%, subindo de 150% no trimestre anterior.
- [HappyRobot worth $1.2 billion — Fortune (ago/2026)](https://fortune.com/2026/08/04/happyrobot-worth-1-2-billion-founder-says-just-getting-started/) — a retenção líquida acima de 150% e o cliente que multiplicou o contrato por dez, segundo o fundador.

**Nota sobre as fontes.** Todos os valores em reais deste artigo são fictícios e servem para mostrar a estrutura do contrato e do preço, não para servir de referência de mercado. Os números de NRR de empresas são os que elas próprias divulgaram: o da Palantir vem de resultado trimestral de empresa de capital aberto; o da HappyRobot é uma declaração do fundador à imprensa. As pesquisas sobre resistência de funcionários são, em sua maioria, encomendadas por fornecedores de IA, e os percentuais devem ser lidos como direção, não como medida. O sponsor map e o SOW são do caso fictício da série, inspirado num processo financeiro real que eu conheço bem.

**[→ Acompanhe os próximos artigos](/blog/)**
