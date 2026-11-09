---
title: "Dia 10 — Discovery, leitura de processo e ROI: a conta que o CFO aprova"
date: 2026-11-08
categories: [desafio-fde]
tags: [fde, discovery, processo, roi, payback, financeiro, produto]
excerpt: "A série vira para o lado do negócio. As três regras do Mom Test e por que elas são mais difíceis para quem conhece o domínio, a leitura de processo em sete passos com volume e tempo em cada etapa, a matriz de volume × custo do erro, a diferença entre valor defensável e valor vendável, o modelo de ROI com a premissa que ninguém mostra, e o glossário financeiro que destrava a conversa com quem assina."
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
| 10 | Hoje: por que alguém pagaria por ele |

O caso é inspirado num processo financeiro real que eu conheço bem, mas tudo aqui é anônimo e os números dos exemplos são fictícios.

---

Ontem eu fechei a parte técnica. Tenho um agente que lê contexto, decide, para na hora certa, fala com os sistemas, sobrevive a falhas, tem suíte de eval, guardrail em código e trace auditável.

Ele não vale nada.

Não por defeito técnico. Porque ninguém comprou. Entre um agente que funciona e um projeto que existe tem uma distância que nenhum commit atravessa, e é dela que esta segunda metade da série trata. Hoje é o dia em que eu troco de lado da mesa: saio do harness e vou para a sala onde se decide se o harness vai ser construído.

E tem uma honestidade que eu preciso registrar antes de começar. Aqui eu não sou aprendiz. Levantar processo, achar gargalo, montar conta de retorno e defender número na frente de quem assina é o que eu faço há anos. O problema é exatamente esse: **o que eu faço por intuição não é método.** Intuição não se delega, não se ensina ao time e não se audita quando o cliente discorda. Então o trabalho de hoje não é aprender discovery. É transformar em passo escrito uma coisa que hoje mora no meu reflexo.

Quem é forte numa área tem uma tentação específica: achar que, porque acerta, não precisa de processo. Já me pegou antes.

## 1. A metade que derruba mais projeto do que o modelo

Em agosto de 2025 saiu o relatório que mais circulou em grupo de executivo no ano, o *GenAI Divide*, da iniciativa NANDA do MIT. A manchete que pegou foi uma só: **cerca de 95% dos pilotos de IA generativa não produziram retorno financeiro mensurável.**

O número virou meme e merece as ressalvas de sempre: a base é pequena para uma afirmação tão larga — 150 entrevistas, 350 respostas de funcionários e a análise de 300 deployments públicos —, e "piloto sem retorno mensurável" não é a mesma coisa que "projeto fracassado". Boa parte do mercado citou o 95% como se fosse censo. Não é.

Mas o diagnóstico por trás do número é o que interessa, e esse eu acho difícil de contestar: a falha não estava na qualidade do modelo. Estava na **integração com o processo** e na ausência de aprendizado organizacional em volta. E tem um detalhe no relatório que eu acho o mais revelador de todos: mais da metade do orçamento de IA generativa foi para ferramentas de vendas e marketing, enquanto o retorno maior apareceu na **automação de back-office**.

Traduzindo: o dinheiro foi para onde a IA é visível, e o retorno estava onde ela é chata.

Conciliação é chata. Fechamento é chato. Contas a receber é chato. É exatamente por isso que esses processos pagam: são repetitivos, medidos, com custo conhecido e dono identificável. Quem faz deployment aprende a gostar do chato.

A consequência prática para o meu trabalho é que a pergunta mais importante do projeto não é técnica, e é feita no primeiro dia: **qual processo, de quem, medido por qual número?** Errar essa pergunta não dá erro de compilação. Dá seis meses de trabalho bonito que ninguém renova.

## 2. Discovery sem viés: as três regras, e por que elas são mais difíceis para mim

O Mom Test, do Rob Fitzpatrick, é um livro curto sobre uma coisa só: como conversar com alguém sem receber mentira educada. O nome vem do teste que dá título ao livro — uma conversa é boa quando nem a sua mãe conseguiria mentir nela, porque você não falou da sua ideia.

Três regras:

**1. Fale da vida dele, não da sua ideia.** No momento em que você descreve a solução, a conversa acabou. A partir dali o outro responde sobre a sua proposta, não sobre a rotina dele — e, se gostar de você, responde bem.

**2. Pergunte sobre fatos específicos do passado, não opinião sobre o futuro.** "Vocês usariam um agente de conciliação?" não vale nada. "Me conta o último fechamento que atrasou: o que aconteceu, quem ficou até mais tarde, o que travou?" vale ouro. O passado aconteceu. O futuro é cortesia.

**3. Fale menos, ouça mais.** Se você ocupou mais de um terço do tempo, você vendeu em vez de aprender.

E o livro nomeia os três tipos de **dado ruim** que a gente coleciona achando que é validação: elogio, *fluff* (o genérico, o hipotético, o "normalmente a gente faz assim") e pedido de funcionalidade. O terceiro é o mais perigoso, porque parece o mais concreto dos três. Não é: é a solução que o cliente imaginou, e ela chega sem o problema que a gerou.

<figure>
  <img src="{{ '/assets/images/desafio-fde/dia-10-discovery-sinais.svg' | relative_url }}" alt="Diagrama em duas colunas comparando o que não ensina nada e o que vale ouro numa conversa de discovery. A coluna da esquerda, em laranja, tem o título dado ruim, parece validação e não é, com três caixas. Primeira caixa, elogio: que ideia bacana, isso aí é o futuro; custo de dizer isso é zero e o valor informativo é zero. Segunda caixa, fluff, que reúne futuro, genérico e hipotético: a gente normalmente faz assim, eu usaria sim; é resposta sobre um mundo que não aconteceu. Terceira caixa, pedido de funcionalidade: se tivesse um botão de exportar eu comprava; é a solução dele, não o problema dele, então pergunte por quê. A coluna da direita, em verde, tem o título sinal de ouro, alguém já pagou por isso, também com três caixas. Primeira, workaround: uma planilha paralela, uma macro, um estagiário, um freelancer; ninguém mantém gambiarra por problema que não dói. Segunda, dinheiro já gasto no problema: licença, hora extra, consultoria, projeto que não saiu; orçamento que já existe é a prova de que a dor foi aprovada. Terceira, um número que alguém já acompanha: o dia do fechamento, o prazo médio de recebimento, a inadimplência; é a régua do seu ROI, e ela não foi inventada por você. Uma faixa no pé traz a regra do um terço: se você falou mais de um terço do tempo, você vendeu, não aprendeu, e o cliente foi educado com você; troque toda pergunta sobre o futuro por uma sobre o passado, não você usaria, mas me conta a última vez que isso atrasou.">
  <figcaption>Os três sinais de ouro têm uma coisa em comum: todos custaram dinheiro a alguém antes de você chegar.</figcaption>
</figure>

Os **sinais de ouro** são o outro lado da moeda, e eu reorganizei em três:

- **Workaround.** Uma planilha paralela, uma macro, um estagiário que "cuida daquilo", um freelancer contratado por fora. Gambiarra é caríssima de manter. Ninguém mantém gambiarra por problema que não dói.
- **Dinheiro já gasto no problema.** Licença de ferramenta, hora extra recorrente, consultoria, um projeto que tentou resolver e não saiu. Orçamento que já existe é prova de que a dor foi aprovada em algum comitê.
- **Um número que alguém já acompanha.** Esse é o mais valioso dos três, e eu volto nele na seção 7: é a régua do seu ROI, e a grande vantagem é que você não a inventou.

Agora a parte que eu precisei admitir sobre mim. O Mom Test foi escrito para fundador que está validando ideia. **No deployment, o viés é outro e é pior.** Eu não chego com uma ideia para validar: eu chego conhecendo o domínio. E quem conhece o domínio completa a frase do cliente.

É um movimento quase involuntário. A pessoa do financeiro diz "aí o problema é quando o pagador é diferente do titular" e eu já estou respondendo "sei, quando é pagamento de terceiro e o nome não casa". Me sinto útil. Construí rapport. E acabei de contaminar o dado mais importante da reunião, porque a partir dali eu vou ouvir a minha versão do problema sendo confirmada, não a dele sendo descrita.

A regra que eu escrevi para mim, e que é mais difícil de seguir do que as três do livro: **quando eu souber a resposta, eu pergunto mais uma vez.** "Me mostra na tela como você resolveu o último caso desses." Expertise é ótima para interpretar o que você ouviu. É péssima no momento de coletar.

E uma correção de rota sobre *em que* eu estou fazendo discovery. Em venda de produto, você descobre se o problema existe. Em deployment, o problema existe — ele tem planilha, dono e hora extra. O que eu descubro é outra coisa: **o preço do problema e a forma dele.** Discovery aqui não é validação de hipótese. É levantamento de processo com propósito de orçamento.

## 3. Leitura de processo: sete passos, com número em cada caixa

Esse é o método que eu rodava de cabeça e agora está escrito. Sete passos, na ordem.

**1. Desenhe o fluxo como ele é, não como deveria ser.** O *as-is*. Isso parece óbvio e quase nunca acontece, porque a primeira descrição que você recebe é sempre o processo oficial — o que está no manual, o que o gestor acredita que acontece. O processo real tem três atalhos que ninguém documentou e um passo que existe só porque uma pessoa específica confere uma coisa específica desde 2019.

A única forma de chegar no *as-is* é a pergunta mais barata do discovery: **"me mostra."** Peça para a pessoa fazer na frente de você, com um caso real, enquanto narra. Em vinte minutos de tela compartilhada você aprende mais do que em duas reuniões de apresentação.

**2. Meça volume e tempo de cada etapa.** Não "é bem demorado". Quantos casos por mês e quantos minutos cada um. Se a resposta vier em adjetivo, você ainda não tem dado. E aceite estimativa com desconto: a pessoa que faz o trabalho sabe o tempo do caso típico e subestima o tempo do caso ruim, sempre.

**3. Ache o gargalo.** Onde a fila acumula. Não é onde o trabalho é mais difícil: é onde o trabalho **espera**. Em processo financeiro, o gargalo quase sempre é uma espera por informação de outra pessoa — um e-mail para o comercial perguntando de quem é aquele depósito.

**4. Ache o retrabalho.** O que volta. Baixa errada que é desfeita, cobrança indevida que gera reclamação, lançamento que o contador devolve no fechamento. Retrabalho é o custo mais invisível do processo, porque ele acontece num mês diferente do erro e normalmente numa planilha de outra pessoa.

**5. Ache as exceções caras.** É aqui que a IA ganha, e é a parte mais contraintuitiva do método.

**6. Pergunte "e se errar?"** O custo do erro, em reais e em consequência. Quem descobre, quando descobre, quem paga.

**7. Identifique o número que o cliente já persegue.** Antes de você existir no projeto.

Aplicado ao caso da série, com volume e tempo em cada etapa, o mapa fica assim:

<figure>
  <img src="{{ '/assets/images/desafio-fde/dia-10-mapa-de-processo.svg' | relative_url }}" alt="Mapa de processo em cinco etapas horizontais, com volume e tempo em cada caixa, para a conciliação de seis mil recebimentos por mês, totalizando cerca de quatrocentas horas de trabalho manual. Etapa 1, baixar o extrato: uma vez por dia, no banco, e colar na planilha; cerca de cinco horas por mês. Etapa 2, achar a cobrança: seis mil buscas no ERP por valor, data e nome; dois minutos no caso fácil. Etapa 3, casar e dar baixa, destacada em verde: cinco mil e cem casos fáceis, oitenta e cinco por cento do volume, cerca de cento e setenta horas por mês. Etapa 4, tratar a exceção, em caixa escura e apontada por uma seta laranja com a legenda a fila dorme aqui: novecentos casos, quinze por cento do volume, quinze minutos cada, resolvidos por e-mail, telefone e deixa pra depois, somando cerca de duzentas e vinte e cinco horas por mês. Etapa 5, fechar o mês: o número sai em D mais nove, com o que sobrou na fila, enquanto a meta do CFO é D mais cinco. Uma seta tracejada volta da etapa cinco para a etapa três com a legenda retrabalho: cerca de dois por cento das baixas voltam no mês seguinte, cerca de cento e vinte casos. Abaixo, uma faixa laranja nomeia as novecentas exceções uma por uma, porque é isto que a demo não mostra: pagador diferente do titular da cobrança, quatrocentos e vinte por mês; pagamento parcial, com juros ou desconto, duzentos e sessenta por mês; um recebimento para várias cobranças ou o contrário, cento e quarenta por mês; duplicidade e devolução, oitenta por mês. Uma faixa escura no pé conclui que quinze por cento do volume consome cinquenta e seis por cento das horas, e que é ali que o agente se paga, não no caso fácil, que já é rápido; e que o número que o cliente já persegue existe antes de você chegar, fechar em D mais cinco em vez de D mais nove, régua que já está pendurada na parede dele e não é o seu KPI de projeto.">
  <figcaption>O mesmo processo, lido com número: 15% do volume consome 56% das horas.</figcaption>
</figure>

Olhe a etapa 4. Essa conta é a razão de existir do projeto.

São 900 casos por mês, 15% do volume, consumindo 225 das 400 horas. **Quinze por cento do volume come 56% do esforço.** É a mesma distribuição torta que aparece em quase todo processo operacional que eu já levantei, e ela tem uma consequência estratégica que muita gente inverte.

A automação tradicional sempre atacou o caso fácil, porque o caso fácil é regra: se o valor e a data batem, dá baixa. Dá para fazer com um `VLOOKUP`, e muita empresa já fez. O resultado é que o caso fácil **já está barato** — 5.100 casos por 170 horas, dois minutos cada. Sobrou o resíduo, que é onde está o dinheiro: cada exceção custa sete vezes mais tempo que um caso fácil, e exceção é precisamente o que regra não resolve, porque ela exige ler um texto livre, inferir intenção e julgar sob ambiguidade.

Isso é a descrição do que um LLM faz. **A IA não entra para fazer o que a regra já fazia. Ela entra onde a regra parou.**

E aqui vai o alerta que eu gostaria de ter escutado antes, e que vem da literatura de process mining: automatizar o caminho felizinho sem tratar a exceção não melhora o processo — **reproduz o defeito mais rápido.** Pior: a exceção que não foi desenhada vira uma fila sem dono, sem código de motivo e sem caminho de resolução. O trabalho manual não desaparece; ele muda de lugar e fica menos visível, porque agora está escondido atrás de um sistema que "automatizou a conciliação".

Daí o teste que eu passei a aplicar em todo mapa de processo: **cada exceção tem nome, volume, dono e desfecho?** Se o cliente não consegue nomear as categorias de exceção, o projeto ainda não está pronto para escopo. E note, na figura, que as quatro categorias nomeadas ali são as mesmas que, no Dia 8, viraram casos da suíte de eval, e no Dia 9, condições de escalonamento. O mapa de processo é a fonte do eval. Essa ligação eu só vi montando os dois.

## 4. Priorização: volume × custo do erro, e o esforço só desempata

Com o mapa na mão, a pergunta é por onde começar. Dois eixos resolvem 90% da decisão.

<figure>
  <img src="{{ '/assets/images/desafio-fde/dia-10-matriz-de-priorizacao.svg' | relative_url }}" alt="Matriz de quatro quadrantes com volume no eixo horizontal, de baixo para alto, e custo do erro no eixo vertical, de baixo para alto. Quadrante superior esquerdo, volume baixo e custo do erro alto, em cinza: caso a caso, e provavelmente não; erro caro mas que acontece pouco, em que uma regra em código, um checklist ou uma pessoa boa resolvem mais barato; agente aqui é vaidade de engenharia e só compensa quando um único erro paga o projeto inteiro; exemplo, liberar um pagamento de sete dígitos. Quadrante superior direito, volume alto e custo do erro alto, em laranja: comece com humano no circuito; volume que justifica e erro que dói, em que o agente propõe, a pessoa aprova e você mede o override; automatize depois de provar com número, não com demonstração, seguindo a escada do Dia 7; um ponto escuro marca o caso desta série, a conciliação. Quadrante inferior esquerdo, volume baixo e custo baixo, tracejado: não faça; pouco volume e erro barato, em que o deploy custa mais que o processo inteiro e ninguém sente a diferença; é o quadrante que mais aparece nas listas de ideias de IA que saem de workshop; exemplo, aquele relatório que uma pessoa faz uma vez por mês. Quadrante inferior direito, volume alto e custo baixo, em verde: automatize primeiro; volume alto, erro barato e fácil de corrigir, a vitória rápida que compra credibilidade e orçamento; entregue isso no mês um e o resto do roadmap deixa de ser promessa; exemplo, classificar e rotear a exceção sem decidir nada. Uma faixa no pé explica que o terceiro eixo é o esforço e que ele só desempata: entre dois casos do mesmo quadrante, ganha o que tem dado acessível e dono presente; e que custo do erro alto não é veto, é ordem de execução, porque não diz não automatize, diz prove antes.">
  <figcaption>O quadrante não escolhe o caso. Ele define quanto humano fica no caminho e em que ordem as coisas entram.</figcaption>
</figure>

A leitura rápida:

- **Volume alto + custo do erro baixo** → automatize primeiro. É a vitória rápida, e vitória rápida não é vaidade: é o que compra orçamento para a fase 2.
- **Volume alto + custo do erro alto** → comece com humano no circuito e automatize depois de provar. É onde a conciliação mora, e é por isso que, desde o Dia 5, o agente propõe e uma pessoa efetiva.
- **Volume baixo + custo alto** → raramente vale um agente. Vale uma regra, um checklist, um segundo par de olhos. Só compensa quando um único erro paga o projeto.
- **Volume baixo + custo baixo** → não faça. E esse é o quadrante que mais aparece nas listas de "oportunidades de IA" que saem de workshop, porque ele é tecnicamente divertido e politicamente inofensivo.

O **esforço** é o terceiro eixo, e o meu erro antigo era usá-lo cedo demais. Esforço não escolhe o caso: ele desempata dois casos do mesmo quadrante. E, na prática, esforço quase nunca é sobre o modelo. É sobre acesso: o dado existe em formato utilizável? Tem API ou vai ser raspagem de relatório? Tem alguém do lado do cliente com tempo para responder dúvida de domínio? Projeto empaca por acesso, não por dificuldade de modelagem.

A frase que eu levo dessa matriz: **custo do erro alto não é veto, é ordem de execução.** Ele não diz "não automatize". Diz "prove antes". Quem trata custo de erro como impedimento perde o caso que mais paga; quem ignora o custo de erro perde a conta.

## 5. Dois tipos de valor: o defensável e o vendável

Aqui está a distinção que mais mudou a forma como eu escrevo proposta, e ela é simples o bastante para caber numa frase: **tem valor que eu consigo provar e tem valor que eu consigo vender, e eles raramente são o mesmo.**

**Economia de custo — defensável.** Horas liberadas, erros evitados, multa evitada, hora extra que deixa de existir. É chato, é pequeno e ninguém faz manchete com isso. Mas é auditável: sai de volume × tempo × custo/hora, e o controller consegue conferir cada parcela.

**Ganho de receita ou velocidade — vendável.** Caixa antecipado, inadimplência menor, fechamento mais rápido, decisão tomada com três dias de antecedência. É o que empolga a diretoria e o que cabe no slide. E é mais difícil de provar, porque depende de atribuição: se a inadimplência caiu 1,2 ponto, quanto disso foi o agente e quanto foi a economia, o mix de clientes ou a mudança na régua de cobrança?

A regra que eu adotei: **comece pela economia de custo para financiar o ganho de receita.** A conta defensável aprova o orçamento; a tese de caixa renova o contrato. Inverter a ordem é o erro clássico da proposta de IA — abre com transformação e fecha com o CFO pedindo "me mostra de onde sai esse número", que é a pergunta que mata a reunião.

E tem uma armadilha dentro do lado defensável que eu já paguei para aprender. **Hora liberada não é dinheiro.** O CFO sabe disso melhor que ninguém. Se as 320 horas por mês não viram uma dessas três coisas, elas não entram na conta dele:

1. **Contratação que não acontece.** O time ia crescer com o volume e não cresce. Essa é a mais aceita.
2. **Capacidade realocada para algo que o negócio já queria.** As mesmas pessoas passam a fazer análise de inadimplência, que estava na fila há um ano. Vale, mas precisa de nome e dono.
3. **Hora extra ou terceiro que deixa de ser pago.** Essa é a única que sai direto do caixa, e é a mais forte das três.

Se você não consegue encaixar a hora liberada em uma dessas, apresente como **capacidade**, não como economia. É honesto e sobrevive à pergunta seguinte. Dizer "libera 320 horas" e deixar a plateia completar a conta como "demite quatro pessoas" é um risco que ninguém controla — e normalmente não é o que o cliente quer fazer.

## 6. O modelo de ROI, com a parcela que quase ninguém mostra

A estrutura é esta, e ela caberia num guardanapo:

```
Valor anual = (volume × tempo poupado × custo/hora)
            + (erros evitados × custo do erro)
            + ganho de caixa

ROI     = (Valor anual − Custo total) ÷ Custo total
Payback = Investimento inicial ÷ Valor mensal líquido
```

O que separa uma conta que passa de uma que não passa não é a fórmula. São três coisas: a origem de cada premissa, a presença do custo recorrente e a honestidade sobre a faixa.

<figure>
  <img src="{{ '/assets/images/desafio-fde/dia-10-roi-e-payback.svg' | relative_url }}" alt="Painel em duas partes sobre o retorno do projeto, com números fictícios para seis mil recebimentos por mês, quatrocentas horas manuais e custo por hora carregado de oitenta reais. À esquerda, de onde vem o valor por mês, em cinco barras horizontais: horas poupadas, vinte e cinco mil e seiscentos reais, correspondendo a trezentas e vinte horas por mês vezes oitenta reais, com oitenta por cento do volume sem toque humano; erros evitados, vinte e cinco mil e duzentos reais, de trinta erros por mês a mil e duzentos reais cada, dos quais setenta por cento são evitados; valor bruto, cinquenta mil e oitocentos reais, em barra escura; menos o custo recorrente de doze mil reais, em barra laranja, que cobre plataforma, modelo, revisão humana e manutenção; e valor líquido de trinta e oito mil e oitocentos reais, em barra verde. À direita, um gráfico de quando a conta vira positiva, com o eixo vertical em reais mil acumulados, de zero a quatrocentos e cinquenta, e o eixo horizontal em meses, de zero a doze. Uma linha verde sobe do zero e cruza a linha tracejada laranja do investimento inicial de trezentos mil reais num ponto marcado como payback de aproximadamente oito meses. Uma nota informa que no ano um o ROI é de cerca de trinta e sete por cento e que, no regime, com o investimento já pago, passa de trezentos por cento. Abaixo, três caixas de cenário: pessimista, com cinquenta por cento automático e quarenta por cento dos erros evitados, levando a payback de cerca de dezesseis meses; base, com oitenta por cento automático e setenta por cento dos erros, com payback de cerca de oito meses; e o que você apresenta, que é a faixa e qual premissa mexe mais no resultado. Uma faixa escura no pé diz que a frase final tem três números e nenhum adjetivo: de zero para oitenta por cento em noventa dias, trezentas e vinte horas por mês, payback em oito meses.">
  <figcaption>Duas parcelas de valor, um custo recorrente, e uma faixa em vez de um número.</figcaption>
</figure>

A conta da figura, item por item:

| Item | Premissa | Valor |
|---|---|---|
| Volume | 6.000 recebimentos/mês | — |
| Tempo manual | 4 min em média → 400 h/mês | — |
| Custo/hora carregado | R$ 80 | R$ 32 mil/mês |
| Automação | 80% sem toque humano | R$ 25,6 mil/mês poupados |
| Erros evitados | 0,5% de erro × R$ 1.200, 70% evitados | ≈ R$ 25,2 mil/mês |
| **Valor bruto** | | **≈ R$ 50,8 mil/mês ≈ R$ 610 mil/ano** |
| Investimento inicial | squad por 2 meses | R$ 300 mil, uma vez |
| Custo recorrente | plataforma, modelo, revisão, manutenção | R$ 12 mil/mês |
| **Valor líquido** | | **≈ R$ 38,8 mil/mês** |
| **Payback** | 300 ÷ 38,8 | **≈ 8 meses** |

Quatro observações sobre essa tabela, e elas valem mais do que ela.

**O custo/hora é carregado, não é o salário.** R$ 80 por hora não é o que a pessoa recebe: é o que a pessoa custa. Salário mais encargos, benefícios, equipamento, software e a parcela de estrutura. A conta usual divide o custo anual total pelas horas efetivamente trabalhadas, e o multiplicador sobre o salário varia muito por país e por regime — o Brasil é dos mais pesados. Usar salário bruto em vez de custo carregado subestima o valor do projeto, e o controller vai corrigir você para cima, o que é a única correção simpática que existe numa reunião dessas.

**O custo recorrente é a parcela que desaparece das propostas.** Plataforma, tokens, infraestrutura, a pessoa que revisa a amostra do Dia 8, a manutenção do prompt quando o banco muda o layout do extrato. Omitir isso faz o payback parecer 6 meses em vez de 8 — e cria a conversa mais desconfortável possível no mês 7, quando aparece uma linha de custo que não estava no slide. Apresentar o recorrente, de propósito, é o que separa uma proposta de uma peça de marketing.

**O ROI do ano 1 não é o ROI do regime.** No ano 1, com os R$ 300 mil de investimento e R$ 144 mil de recorrente contra R$ 610 mil de valor, o ROI é de ≈ 37%. No ano 2, com o investimento pago, passa de 300%. Os dois números são verdade; apresentar só o segundo é desonesto e apresentar só o primeiro é burro. Eu mostro os dois, sempre nessa ordem.

**O número que você apresenta é uma faixa.** Se a automação chegar a 50% em vez de 80% e a prevenção de erro a 40% em vez de 70%, o líquido cai para ≈ R$ 18 mil/mês e o payback dobra: 16 meses. Mesma estrutura, mesma planilha, premissa diferente. Então eu levo os dois cenários e digo qual premissa mexe mais no resultado — no caso, a taxa de automação, de longe.

Isso tem um efeito secundário que eu só percebi depois de fazer algumas vezes: **mostrar o cenário pessimista aumenta a credibilidade da conta mais do que qualquer argumento sobre o cenário base.** Quem apresenta um número único está pedindo fé. Quem apresenta uma faixa com a premissa exposta está convidando para a auditoria — e quem convida para a auditoria normalmente não precisa dela.

## 7. A premissa é o produto: de onde vêm esses números

Uma planilha de ROI vale pela origem das premissas, e nada mais. Com premissa inventada, ela é um exercício de Excel com aparência de rigor — e é exatamente isso que o CFO está farejando quando pede "me explica essa linha".

As quatro origens que eu aceito, em ordem de força:

1. **Medido no processo do cliente, em amostra real.** Cronometrar 30 casos, contar a fila de um mês, exportar o log do ERP.
2. **Número que o cliente já acompanha.** Está no relatório dele, com histórico. Não tem discussão possível sobre a régua.
3. **Benchmark público, declarado como benchmark.** Serve para contexto, nunca como premissa central.
4. **Estimativa da pessoa que faz o trabalho, marcada como estimativa.** Aceitável, com desconto, e sempre a ser confirmada depois.

E aqui a série fecha um ciclo que eu não tinha planejado. No Dia 8 eu descrevi o **shadow mode**: o agente roda em paralelo, decide, não executa nada, e você compara a decisão dele com a da pessoa. Eu apresentei aquilo como instrumento de qualidade — a forma de medir se o agente acerta antes de deixar ele tocar em qualquer coisa.

Ele é mais que isso. **Shadow mode é o instrumento de medição do ROI.** Rodando em sombra, eu meço de uma vez: o tempo real que a pessoa leva por caso, a taxa de acerto do agente, a taxa real de exceção, o volume verdadeiro e quantas vezes a pessoa discorda da proposta. Essas são, uma a uma, as premissas da tabela da seção anterior. O teste técnico e a prova de valor são o mesmo experimento.

Daí a forma de escopo que eu passei a defender: **a fase 1 do projeto não entrega o agente em produção. Entrega a conta.** Quatro a seis semanas em sombra, num processo só, com um número no fim. Se a conta não fechar, o cliente gastou pouco e descobriu cedo — e esse é um resultado honesto de vender, não um fracasso a esconder. Se fechar, a fase 2 não começa com promessa: começa com medição.

Isso também resolve um problema de negociação. Pedir R$ 300 mil para um projeto inteiro no primeiro contato é pedir um ato de fé. Pedir uma fase curta que termina em uma conta auditada é pedir um experimento. O segundo pedido é muito mais fácil de aprovar, e ele dá a você o ativo mais valioso da relação: o dado do processo do cliente antes de qualquer compromisso de performance.

## 8. A frase que o CFO aprova

Todo esse trabalho precisa caber em uma linha, porque é nesse formato que ele vai ser repetido por alguém que não estava na reunião.

> "Este deploy leva a conciliação automática de 0% para 80% em 90 dias, libera 320 horas por mês e evita cerca de R$ 300 mil por ano em erros, com payback em 8 meses."

A anatomia dela:

- **Um número que já existia** (% conciliado automaticamente), com o ponto de partida e o de chegada.
- **Um prazo com marco** (90 dias), não "no médio prazo".
- **Duas parcelas de valor**, uma defensável (horas, erros) e uma verificável depois (o payback).
- **Nenhum adjetivo.** Nada de "significativo", "robusto", "transformacional". Adjetivo em frase de ROI é sinal de que falta número.

E as três formas de essa frase morrer, que eu já vi todas:

**O número não é dele.** Você inventou uma métrica nova — "taxa de automação cognitiva" — que ninguém acompanha e, portanto, ninguém sente falta. Use a régua que já está na parede, mesmo que ela seja tosca.

**As horas não viram dinheiro.** É a armadilha da seção 5. Sem contratação evitada, realocação nomeada ou custo variável que cai, hora liberada é capacidade, não economia. Apresente como capacidade.

**O prazo não tem marco intermediário.** "Em 90 dias" sem nada verificável no dia 30 é uma promessa com três meses de escuro no meio. Eu quebro em dois: no dia 30, o agente roda em sombra com a conta medida; no dia 90, 80% sem toque humano. O primeiro marco é o que compra paciência para o segundo.

## 9. Falar a língua de quem assina

Vale um aviso: essa é a seção menos glamourosa do artigo e a que mais muda o resultado de uma reunião. Não existe credibilidade técnica que compense você perguntar o que é DSO no meio de uma conversa com o time financeiro.

O domínio que eu estou estudando é o de uma **intermediadora de pagamentos** — o tipo de empresa que fica entre quem cobra e quem paga, processando cartão, boleto e PIX, cuidando da liquidação e devolvendo o dinheiro para o lojista. É um ótimo domínio para agente por três razões: volume gigante, dado estruturado e um custo de erro conhecido até o centavo.

O vocabulário mínimo, e eu organizei por onde cada termo aparece no fluxo do dinheiro:

| Termo | O que é |
|---|---|
| **Conciliação bancária** | Casar o que entrou e saiu no banco com o que estava previsto: cobranças emitidas e contas a pagar |
| **Contas a receber / a pagar (AR/AP)** | O que os clientes devem à empresa / o que a empresa deve aos fornecedores |
| **Fechamento contábil (D+N)** | Quantos dias, depois do fim do mês, o financeiro leva para fechar os números |
| **DSO** (*days sales outstanding*) | O prazo médio de recebimento. Quanto menor, melhor para o caixa |
| **Régua de cobrança** | A sequência de lembretes e ações para cobrar quem atrasou |
| **Inadimplência** | A proporção de valores vencidos e não pagos |
| **Adquirência / adquirente** | Quem processa o pagamento com cartão para o lojista e responde pela liquidação |
| **Sub-adquirente / facilitador** | Quem fica entre o adquirente e o lojista, simplificando o acesso e assumindo parte do risco |
| **MDR** (*merchant discount rate*) | A taxa cobrada sobre cada venda no cartão |
| **Antecipação de recebíveis** | Receber hoje a venda a prazo, com desconto. É crédito disfarçado de operação |
| **Chargeback** | O estorno de uma compra no cartão, contestada por quem pagou |
| **Split de pagamento** | Dividir automaticamente um pagamento entre vários recebedores |
| **PIX, PIX Cobrança, boleto** | Os meios de pagamento. PIX com `txid` é o que faz a conciliação virar quase trivial |
| **NF-e / NFS-e** | A nota fiscal eletrônica de produto / de serviço |
| **ERP** | O sistema de gestão onde mora a contabilidade. É o sistema de registro, e quem manda na verdade |
| **DRE × fluxo de caixa** | O resultado contábil × o dinheiro que de fato entrou e saiu. Duas verdades diferentes sobre a mesma empresa |
| **BaaS** (*banking as a service*) | Oferecer serviço financeiro regulado usando a licença de um parceiro |

Três desses termos merecem mais que uma linha, porque são onde o agente encosta.

**`txid` e a conciliação que não precisa de IA.** O PIX tem um campo de identificação da transação que, quando o cobrador preenche com o identificador da cobrança, faz o casamento virar uma consulta de banco de dados. Nenhum modelo necessário. Essa é a resposta honesta que eu daria numa primeira reunião: antes de automatizar a conciliação com agente, veja quantos dos seus recebimentos podem simplesmente chegar identificados. É possível que metade do problema seja de desenho de cobrança, não de IA. Dizer isso custa escopo e compra confiança — e, no Dia 11, quando eu falar de SOW, esse é o tipo de concessão que muda o tom de toda a negociação.

**A diferença entre DRE e caixa.** O DRE diz que a empresa teve resultado; o extrato diz quanto dinheiro existe. A distância entre os dois é feita de prazo, de recebível em trânsito e de coisa mal conciliada — e é a pergunta que mais aparece em dor de dono de negócio: "se deu lucro, onde está o dinheiro?". Investigar essa diferença é um dos trabalhos em que agente rende muito, justamente porque a resposta exige cruzar fontes e ler texto livre.

**O custo do erro em cada direção.** Em pagamentos, errar para um lado e para o outro custa coisas bem diferentes: deixar de dar baixa em quem pagou gera cobrança indevida, reclamação e risco regulatório; dar baixa em quem não pagou gera perda direta e some do radar por semanas. É a matriz assimétrica do Dia 8, agora com nome de negócio.

E três números públicos que servem de contexto — benchmark, nunca premissa:

- **Fechamento mensal.** Nos levantamentos da APQC, os melhores fecham em cerca de 4,8 dias ou menos, a mediana fica em torno de 6,4 dias e o quartil de baixo precisa de 10 dias ou mais. Os dados são antigos, de antes desta geração de automação, mas a ordem de grandeza ainda orienta a conversa: fechar em D+9 é ruim, e o cliente já sabe disso.
- **Prazo médio de recebimento.** Também pela APQC, o quartil superior recebe em 30 dias ou menos e o inferior leva 46 dias ou mais. Varia muito por setor, então serve para enquadrar, não para comparar.
- **Meio de pagamento no Brasil.** Pelos dados do Banco Central referentes ao primeiro semestre de 2026, o PIX concentrou 55,2% da quantidade de transações de pagamento no país, contra 30,8% de cartões somados. Isso tem consequência direta de engenharia: o maior volume do seu problema de conciliação chega por um trilho com identificador próprio e texto livre no meio.

Nenhum desses três números entra numa planilha de ROI. Todos os três entram na primeira reunião, porque mostram que você conhece a régua antes de pedir para medir.

## 10. O que disso vira produto

Essa seção é a que eu mais queria escrever, porque ela conecta o dia de hoje com o que vem no fim da série.

Tudo que eu descrevi até aqui parece serviço: uma pessoa boa, numa sala, levantando processo e montando conta. E é — na primeira vez. O que define se o negócio escala é o que sobra depois da primeira vez.

Em junho de 2025 o Joe Schmidt, da a16z, escreveu um ensaio que virou referência sobre isso, com um título que resume a tese: *Trading Margin for Moat* — trocando margem por fosso. O argumento central é que os produtos de IA complexos exigem implementação intensiva, e que fundador deveria aceitar margem baixa no começo em troca de fosso depois. Tem uma analogia nele que eu uso em conversa até hoje: *"empresas comprando IA são como a sua avó comprando um iPhone: elas querem usar, mas precisam que você configure."* E a sentença que eu considero a mais importante do texto, porque contraria o instinto de SaaS: *"na corrida para ser dono da camada de ingestão de dados, é míope otimizar para 80% de margem bruta."*

Ele lembra que Salesforce, ServiceNow e Workday todas começaram com margem baixa e serviço pesado, e ganharam poder de precificação depois que o fluxo de trabalho do cliente virou o delas. A tese da peça não é "faça serviço": é que serviço é **estágio**, não modelo. E o que transforma estágio em produto é o que você extrai de cada implementação — biblioteca compartilhada, API clara, documentação e ferramenta de automação para as partes repetitivas, incluindo a leitura de processo.

Olhando o meu dia de hoje com essa lente, eu tenho quatro coisas na mão, e nenhuma delas é o agente:

1. **O roteiro de discovery.** Oito perguntas que funcionam em qualquer processo financeiro. Na primeira vez, é intuição. Na terceira, é um documento que outra pessoa do time consegue conduzir sem mim — e é aí que ele vale algo.
2. **O template de mapa de processo.** Etapa, volume, tempo, gargalo, retrabalho, categorias de exceção com volume, número-alvo. Sempre as mesmas colunas.
3. **A planilha de ROI.** Com a estrutura de parcelas, o custo recorrente obrigatório e os três cenários. Essa é a mais reaproveitável de todas, porque o formato não muda de cliente para cliente: mudam os números.
4. **O glossário do domínio.** Que parece a menos nobre das quatro e é a que mais acelera gente nova no time.

Note o que esses quatro artefatos são: **a camada de produto do trabalho de discovery.** Eles não aparecem na arquitetura do agente, não têm commit, não entram na demo. E são eles que decidem se a segunda implementação custa metade da primeira ou o mesmo.

Tem também uma relação com dado que eu só enxerguei agora. Cada mapa de processo que eu levanto é uma amostra do mesmo problema em empresas diferentes. Depois de cinco conciliações levantadas, eu não tenho cinco projetos: tenho uma **distribuição** — as categorias de exceção que se repetem, os volumes típicos, o tempo por caso, as premissas que normalmente se confirmam e as que normalmente decepcionam. Isso é ativo, e é a diferença entre chegar na sexta reunião perguntando tudo de novo ou chegar dizendo: "em empresas do seu porte, a exceção de pagador divergente costuma ser 40% do volume de exceção; vamos confirmar se aqui é parecido?"

É esse caminho — de muitos projetos parecidos para uma coisa só — que o **Dia 12** trata, no assunto de domain-driven design e componentização, e o **Dia 13** fecha, no N→1 e na produtização. Hoje é o dia em que a matéria-prima é coletada.

Uma última provocação, que eu deixo em aberto porque não tenho resposta fechada. A peça da a16z, de junho de 2025, assume que a implementação é feita por pessoas, e que o custo dela é o que você troca por fosso. Em 2026, boa parte dessa implementação — ler código legado, mapear integração, gerar o conector, escrever o primeiro eval — já é feita com agente. Isso muda a conta do fosso: se o serviço fica mais barato para todo mundo ao mesmo tempo, ele deixa de ser barreira de entrada. Sobra o que o agente não consegue levantar sozinho: o acesso, a confiança de quem assina e o conhecimento das exceções que ninguém documentou. Convenientemente, as três coisas que o dia de hoje ensina a coletar.

## Prática: três artefatos e oito perguntas

O exercício de hoje tem três entregas, e a primeira é a que dá mais trabalho.

**1. O mapa de processo**, no formato da figura da seção 3: cinco etapas, volume e tempo em cada uma, gargalo marcado, retrabalho quantificado, categorias de exceção nomeadas com volume e o número que o cliente já persegue. Regra que eu me impus: nenhuma caixa sem número. Onde eu não tinha número, escrevi "a confirmar", que é melhor do que um adjetivo.

**2. A planilha de ROI**, com as duas parcelas de valor, o custo recorrente explícito, três cenários e a origem declarada de cada premissa. Ela termina obrigatoriamente numa frase do formato: *"Este deploy move [número que o cliente já acompanha] de [X] para [Y] em [prazo], ao custo de [Z], com payback de [N] meses."*

**3. O roteiro de discovery**, oito perguntas em estilo Mom Test. Este é o meu, e ele é para ser lido em voz alta, não para ser enviado por e-mail:

1. **"Me mostra como vocês fazem isso hoje, passo a passo — com um caso real, na sua tela."** A pergunta que mais ensina, e a única da lista em que eu só olho.
2. **"Quantos casos desses passaram na semana passada, e quanto tempo isso tomou de você?"** Volume e tempo, no passado concreto. "Semana passada" é mais confiável que "por mês".
3. **"Me conta a última vez que isso deu errado de verdade. O que aconteceu depois?"** O custo do erro, narrado em vez de estimado. O "depois" é onde está o retrabalho.
4. **"Quanto custou aquele erro, e quem acabou pagando a conta?"** Valor e dono. Quem paga a conta é quase sempre o sponsor do projeto.
5. **"Quais são os casos que vocês deixam para o fim do dia?"** A minha pergunta favorita para achar exceção, porque ninguém se lembra de exceção quando você pergunta "quais são as exceções?". Todo mundo lembra do que empurra para depois.
6. **"Tem alguma planilha, macro ou controle paralelo nisso?"** Caça ao workaround. Se a resposta demorar, pergunte de novo de outro jeito: ninguém admite gambiarra de primeira.
7. **"Que número o seu chefe te cobra todo mês, e como ele está hoje?"** A régua, e o valor atual dela. Sem isso não existe ROI.
8. **"Se isso melhorasse, o que você faria com o tempo que sobrasse?"** A única pergunta sobre o futuro da lista, de propósito. Ela testa se a hora liberada tem destino — e, portanto, se vira dinheiro na conta do CFO.

E a disciplina que vale mais que as oito perguntas: **cronometre a sua própria fala.** Se você passou de um terço, volte na semana seguinte e repita a reunião.

## TLDR

- **A segunda metade do deployment é de negócio, e é ela que derruba mais projeto.** O relatório do MIT que falou de 95% de pilotos sem retorno aponta integração e processo, não qualidade de modelo — e achou o retorno maior no back-office chato, não na vitrine.
- **Discovery é levantamento de processo com propósito de orçamento**, não validação de ideia. O problema já existe: você está descobrindo o preço e a forma dele.
- **As três regras do Mom Test:** fale da vida dele, pergunte sobre fato passado, fale menos de um terço do tempo.
- **Dado ruim tem três formas:** elogio, fluff e pedido de funcionalidade. A terceira é a mais perigosa porque parece concreta.
- **Quem conhece o domínio tem um viés pior que o do fundador:** completa a frase do cliente. Quando eu sei a resposta, eu pergunto mais uma vez.
- **Sinal de ouro é o que já custou dinheiro a alguém:** workaround, orçamento já gasto, um número que já é acompanhado.
- **"Me mostra" vale mais que qualquer pergunta.** Processo oficial é ficção; o as-is tem três atalhos que ninguém documentou.
- **Nenhuma caixa do mapa sem número.** Volume e tempo por etapa, ou é desenho bonito.
- **A exceção é onde a IA ganha.** No caso da série, 15% do volume consome 56% das horas — e o caso fácil já está barato, porque a regra resolveu há anos.
- **Automatizar o caminho felizinho sem desenhar a exceção reproduz o defeito mais rápido**, e empurra o trabalho manual para uma fila sem dono.
- **Cada exceção precisa de nome, volume, dono e desfecho.** Sem isso não há escopo — e são essas categorias que viram caso de eval e condição de escalonamento.
- **Volume × custo do erro decide a ordem, não o caso.** Custo de erro alto não é veto: é "prove antes". Esforço só desempata, e quase sempre é sobre acesso ao dado, não sobre modelo.
- **Separe valor defensável de valor vendável.** Economia de custo aprova o orçamento; tese de caixa renova o contrato. Nessa ordem.
- **Hora liberada não é dinheiro** até virar contratação evitada, capacidade realocada com nome ou custo variável que cai. Se não virar, apresente como capacidade.
- **Mostre o custo recorrente de propósito.** É a parcela que desaparece das propostas e a que cria a reunião ruim do mês 7.
- **Apresente faixa, não número.** O cenário pessimista dobrou o payback do nosso caso, de 8 para 16 meses — e mostrar isso aumenta a credibilidade da conta inteira.
- **ROI do ano 1 e do regime são números diferentes.** 37% e mais de 300%, no mesmo projeto. Mostre os dois.
- **Shadow mode não é só teste de qualidade: é o instrumento de medição do ROI.** Fase 1 não entrega o agente, entrega a conta.
- **A frase final tem três números, um prazo com marco e nenhum adjetivo.**
- **Aprenda o vocabulário de quem assina.** Não existe credibilidade técnica que sobreviva a você perguntar o que é DSO no meio da reunião.
- **Às vezes a resposta honesta não é IA.** Cobrança que chega identificada resolve de graça metade do problema de conciliação. Dizer isso custa escopo e compra o resto da relação.
- **O produto não é o agente: é o roteiro, o template de processo, a planilha e o glossário.** Serviço é estágio, não modelo — e o que sobra de cada implementação é o que faz a segunda custar metade.

## Fontes

**Leituras**

- [The Mom Test — Rob Fitzpatrick](https://www.momtestbook.com/) — as três regras, os três tipos de dado ruim e o teste que dá nome ao livro. Curto, com edição em português. O [guia prático em PDF](https://theforgeventure.eu/wp-content/uploads/2026/04/mom_test.pdf) resume o essencial em 25 minutos.
- [The GenAI Divide: State of AI in Business 2025 — iniciativa NANDA, MIT](https://fortune.com/2025/08/18/mit-report-95-percent-generative-ai-pilots-at-companies-failing-cfo/) — o relatório dos 95% de pilotos sem retorno mensurável, com a atribuição da falha à integração com o processo e o achado de que o retorno maior está no back-office. A base é pequena (150 entrevistas, 350 respostas e 300 deployments públicos): trate como indício, não como censo.
- [Trading Margin for Moat: Why the Forward Deployed Engineer Is the Hottest Job in Startups — Joe Schmidt, a16z (jun/2025)](https://a16z.com/services-led-growth/) — por que aceitar margem baixa no começo, a analogia da avó com o iPhone, e o ponto de que serviço é estágio: o que escala é a biblioteca, a API e a automação que sobram de cada implementação.
- [Days Sales Outstanding — APQC](https://www.apqc.org/resources/blog/what-dso-finance) — os quartis de prazo médio de recebimento: 30 dias ou menos no quartil superior, 46 dias ou mais no inferior, com a ressalva de que a comparação só vale contra pares do mesmo setor.
- [Metric of the Month: Cycle Time for Monthly Close — CFO.com, com dados da APQC](https://www.cfo.com/news/metric-of-the-month-cycle-time-for-monthly-close/659297/) — o fechamento mensal em 4,8 dias no quartil superior, 6,4 na mediana e 10 ou mais no inferior. Dados anteriores a esta geração de automação: servem como ordem de grandeza.
- [Estatísticas do Banco Central do Brasil — pagamentos de varejo e de cartões](https://www.bcb.gov.br/estatisticas) — a participação do PIX e dos cartões na quantidade de transações. A edição divulgada em outubro de 2026, referente ao primeiro semestre, traz PIX em 55,2% e cartões somados em 30,8% ([cobertura do Poder360](https://www.poder360.com.br/poder-economia/pix-concentra-552-das-transacoes-no-1o-semestre-diz-bc/)).
- [Fully burdened labor rate](https://www.hibob.com/financial-metrics/fully-burdened-labor-rate/) — como se monta o custo/hora carregado e por que o multiplicador sobre o salário varia tanto por país e por regime de contratação.
- [What is process mining? — Google Cloud](https://cloud.google.com/discover/what-is-process-mining) — a ideia de reconstruir o as-is a partir do dado de evento em vez de entrevista, e a separação entre caminho principal e tratamento de exceção.

**Blog de domínio financeiro** (conteúdo público de uma intermediadora de pagamentos brasileira, útil para o vocabulário e para os casos de uso de agente no financeiro)

- [Fluxo de caixa: o gap que o DRE esconde e a IA aprendeu a achar](https://www.barte.com/blog-posts/fluxo-de-caixa-com-ia) — a distância entre resultado contábil e caixa, e por que investigar essa diferença depende de dado de recebimento bem estruturado.
- [No financeiro, a IA é 5% do trabalho. O dado é 95%](https://www.barte.com/blog-posts/ia-no-financeiro) — o argumento de que o retorno depende da qualidade do dado, não da escolha do modelo. É a mesma conclusão do Dia 4, dita do lado do negócio.
- [Finanças agênticas: o fim do financeiro que só digita](https://www.barte.com/blog-posts/financas-agenticas) e [Pagamentos agênticos já estão em produção](https://www.barte.com/blog-posts/pagamentos-agenticos) — conciliação, fechamento e projeção de caixa feitos por agente, e o que muda na cobrança B2B.
- [Banking as a Service no Brasil](https://www.barte.com/blog-posts/banking-as-a-service-brasil) — o que é BaaS, como funciona a licença de parceiro e os critérios para avaliar.

**Nota sobre as fontes.** Todos os valores em reais deste artigo são fictícios e servem para mostrar a estrutura da conta, não para servir de referência de preço. Os benchmarks da APQC são de levantamentos antigos e aparecem aqui como ordem de grandeza. O relatório do MIT é citado pela cobertura de imprensa, com as ressalvas de método no próprio texto. E o mapa de processo da seção 3 é inspirado num processo financeiro real que eu conheço bem, com volumes e tempos alterados.

**[→ Acompanhe os próximos artigos](/blog/)**
