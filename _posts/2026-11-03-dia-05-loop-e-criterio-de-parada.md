---
title: "Dia 5 — Harness II: o loop e o que separa demo de produção"
date: 2026-11-03
categories: [desafio-fde]
tags: [fde, loop, harness, critério-de-parada, graph, multi-agente]
excerpt: "Demo é um agente que começa. Produção é um agente que termina. O ciclo agêntico, os quatro critérios de parada com números, possuir o próprio fluxo de controle, desenhar o graph e quando multi-agente compensa."
---

Faça um agente rodar na frente de uma plateia e ele impressiona. Deixe o mesmo agente rodando sozinho numa sexta à noite e você descobre uma coisa que nenhuma demo mostra: **todo agente sabe começar, poucos sabem terminar.**

Demo é um agente que começa. Produção é um agente que para na hora certa, pelo motivo certo, e deixa rastro de por que parou.

No Dia 4 eu falei da primeira metade do harness: o que o modelo vê. Hoje é a segunda metade, e é a que decide se aquilo vira sistema ou fica em vídeo de LinkedIn. **O loop, o critério de parada e o graph.**

## 1. O loop, em cinco verbos

Todo agente, de qualquer framework, roda o mesmo ciclo:

> **perceber → decidir → agir → observar → reavaliar**

Traduzindo para o que acontece na máquina: o harness monta a janela e chama o modelo; o modelo responde pedindo uma ferramenta (ou entregando a resposta final); o harness executa a ferramenta; o resultado volta para a janela; o harness chama o modelo de novo, agora com uma informação a mais.

E repete. Até alguém mandar parar.

Repare onde está o verbo no comando. **O modelo não roda o loop. Quem roda é o harness.** Essa é a mesma frase do Dia 3 aparecendo de outro ângulo: o modelo não decide continuar nem decide parar. Ele responde uma vez, com um pedido ou com um texto. Quem interpreta aquilo e decide dar mais uma volta é o seu código.

Isso tem uma consequência prática que muita gente só descobre depois de levar um susto: **cada volta do loop é uma cobrança nova da janela inteira.** Não é o delta. É tudo de novo: system prompt, tools, histórico, resultados das ferramentas anteriores. Dez voltas com uma janela que cresce a cada passo não custam dez vezes. Custam bem mais.

Agente que não para não "trava". Ele continua. Educadamente. Queimando dinheiro.

<figure>
  <img src="{{ '/assets/images/desafio-fde/dia-05-o-loop-em-cinco-verbos.svg' | relative_url }}" alt="Infográfico em duas partes. A primeira mostra o ciclo em cinco cards: perceber (harness monta a janela), decidir (modelo responde uma vez, pedindo tool ou entregando texto final), agir (harness executa a ferramenta), observar (harness devolve o resultado para a janela) e reavaliar (harness dá outra volta ou para por um critério explícito), com seta de retorno indicando que a volta recomeça com a janela maior do que antes. A segunda parte mostra seis barras horizontais empilhadas, onde a parte cinza é a porção fixa (prompt e tools) e a laranja é o histórico acumulado que cresce a cada volta. Na volta 1 o custo é 1,0 vezes; na volta 6 chega a 4,0 vezes. Um selo destaca que as seis voltas somam cerca de 14 vezes a primeira chamada. Uma legenda indica que as proporções são ilustrativas mas o formato da curva não é.">
  <figcaption>Quatro dos cinco passos são o seu código. O modelo aparece uma vez, responde e sai. E a janela — e o custo — cresce a cada volta.</figcaption>
</figure>

## 2. "Parar quando necessário" não é um critério

Essa frase aparece em quase todo prompt que eu já li de agente que deu errado, incluindo os meus.

O problema é que ela parece uma instrução e não é. É um adjetivo. "Necessário" para quem? Com que número? O modelo vai preencher o vazio com o que for plausível naquele passo, e o plausível muda a cada volta. Num dia ele para cedo demais e devolve o lote pela metade. No outro, fica tentando resolver um lançamento impossível por quarenta passos.

Critério de parada é **condição verificável por código**. Não precisa de bom senso para ser avaliada. Ou é verdadeira, ou não é.

São quatro. Toda vez que eu olho um agente que alguém quer colocar em produção, eu procuro as quatro. Quando falta uma, normalmente é a mesma.

### Os quatro critérios

| Critério | O que é | Com número |
|---|---|---|
| **Objetivo atingido e verificado** | Terminou **e** uma checagem independente confirmou | Todos os lançamentos do lote têm status, e a soma conciliada bate com o extrato |
| **Orçamento esgotado** | Limite de passos, tokens, tempo ou dinheiro | Máximo de 8 passos por lançamento, 3 min por lote, R$ 0,40 por lançamento |
| **Pedido de ajuda humana** | O agente reconhece que não deve decidir | Divergência acima de R$ 500, cliente sem cadastro, confiança baixa |
| **Loop improdutivo** | Repetição sem progresso | O mesmo lançamento tocado 3 vezes, ou a mesma tool chamada 2 vezes com os mesmos parâmetros |

Duas palavras merecem destaque na primeira linha: **e verificado**. Um agente que declara "terminei" não terminou. Ele afirmou que terminou. A diferença entre as duas coisas é um dos buracos mais caros em produção, e é o motivo de existir o item 5 deste artigo.

O quarto critério é o que quase todo mundo esquece, e é o que mais queima dinheiro. O agente não está travado: ele está tentando. Chama a mesma ferramenta, recebe o mesmo resultado vazio, raciocina de novo sobre o mesmo resultado vazio, tenta outra vez. Do lado de fora parece trabalho. É loop.

Detectar isso é simples e é código, não prompt: guarde um hash de (nome da tool + parâmetros) a cada chamada. Repetiu, conte. Chegou a 2 ou 3, aborte. Guarde também um contador por item: se o mesmo lançamento foi tocado três vezes sem mudar de estado, ele não vai fechar nesta rodada.

### Para cada parada, um destino

Critério sem destino é meio critério. Para cada uma das quatro saídas, defina o que acontece em seguida:

| Parada | Destino |
|---|---|
| Objetivo atingido | Registrar o resultado, gravar o rastro, notificar quem precisa saber |
| Orçamento esgotado | Parar, **salvar o progresso parcial**, alertar o responsável. Nunca descartar o que já foi feito |
| Pedido de ajuda | Escalar com o contexto junto, e ficar pausado esperando a resposta |
| Loop improdutivo | Abortar aquele item, mandar para a fila de exceções com o rastro das tentativas |

Esse "salvar o progresso parcial" parece detalhe de implementação e é decisão de produto. Um agente que processou 180 de 200 lançamentos e morreu no orçamento entregou 90% do trabalho. Se ele descarta tudo, entregou zero e você ainda pagou pelos 180.

<figure>
  <img src="{{ '/assets/images/desafio-fde/dia-05-o-loop-e-as-quatro-saidas.svg' | relative_url }}" alt="Diagrama do ciclo agêntico em cinco caixas escuras — perceber, decidir, agir, observar, reavaliar — com uma seta de retorno indicando que o resultado volta para o modelo na janela. Abaixo, quatro caixas laranja mostram as saídas: objetivo atingido e verificado, com destino fechar o lote e registrar; orçamento esgotado, com oito passos por item, três minutos por lote e R$ 0,40 por item, destino parar, salvar progresso e alertar; pedido de ajuda, com divergência acima de R$ 500 ou cliente sem cadastro, destino escalar e retomar depois; loop improdutivo, com o mesmo item tocado três vezes ou a mesma tool com os mesmos parâmetros duas vezes, destino abortar e mandar para a fila de exceções.">
  <figcaption>As quatro saídas. Sem a 2 e a 4, o agente não trava — ele continua rodando, gastando e errando em silêncio.</figcaption>
</figure>

### O limite do framework não é o seu critério

Os frameworks já trazem uma rede de proteção. No LangGraph, por exemplo, existe um `recursion_limit` que por padrão vale 25 passos; estourou, ele interrompe com a mensagem *"Recursion limit of 25 reached without hitting a stop condition"*.

Leia a mensagem com atenção, porque ela diz exatamente o que aconteceu: o grafo rodou até o teto **sem encontrar uma condição de parada**. Isso não é o agente parando. É o cinto de segurança funcionando depois da batida.

Rede de proteção é obrigatória: sempre configure um teto absoluto. Mas ela é a última linha, não o desenho. Se os seus agentes param no teto do framework com alguma frequência, você não tem critério de parada: você tem um disjuntor.

## 3. Possua o seu fluxo de controle

Aqui entra o guia que mais mudou a forma como eu desenho isso. O **12-Factor Agents**, do Dex Horthy, pega a ideia dos doze fatores de aplicações em nuvem e escreve o equivalente para agentes. São doze princípios sobre o que você deve controlar em vez de terceirizar.

Cinco deles são do assunto de hoje. Vale conhecer pelo nome, porque aparecem em discussão técnica. Vou apresentar na ordem que faz sentido para entender o conjunto:

**Fator 6 — Launch/Pause/Resume (lançar, pausar e retomar).** Esse é o fator que separa brinquedo de sistema. Um agente de produção precisa parar esperando um humano e continuar depois — três horas depois, ou três dias depois, sem o processo ficar aberto na memória de um servidor. Na prática, exige que o estado do agente seja persistido fora do processo. É o que o LangGraph faz com `interrupt()` mais um *checkpointer* durável, e o que o Microsoft Agent Framework faz com `RequestInfoExecutor` e checkpoints no fim de cada etapa.

**Fator 7 — Contact humans with tool calls (contatar humanos é uma tool).** Isso é mais bonito do que parece. Pedir aprovação não é uma exceção, um erro ou um `if` no fim do código: é uma ação como qualquer outra, com nome (`escalar_excecao`), parâmetros e registro. O agente "chama o humano" do mesmo jeito que chamaria o banco de dados. O ganho é prático: pedido de ajuda vira algo que você mede, versiona e melhora, em vez de algo que simplesmente acontece.

Repare como os fatores 6 e 7 se completam. O 7 diz que pedir ajuda é uma ação; o 6 diz que o sistema precisa sobreviver à espera. Juntos, eles transformam "o agente não consegue decidir isso" de falha em comportamento previsto.

**Fator 8 — Own your control flow (seja dono do seu fluxo de controle).** O coração do guia. A decisão de quando parar, pausar, repetir ou escalar é regra de negócio, e regra de negócio é sua. Quando ela está escondida no `while` interno de um framework, você não consegue nem explicar o comportamento do sistema numa reunião, muito menos mudá-lo. O loop tem umas dez linhas de código: escreva as dez linhas. Os fatores 6 e 7 só funcionam porque o 8 está em vigor: você não pode pausar com propósito nem escalar com registro se a decisão de quando fazer isso está buried dentro de uma abstração que você não controla.

**Fator 9 — Compact errors into the context window (compacte erros na janela).** Quando uma ferramenta falha, não jogue o stack trace de 300 linhas na janela. Devolva um erro curto e acionável: *"CNPJ não encontrado na base de clientes; tente buscar pelo nome"*. Duas razões. A primeira é token. A segunda é melhor: erro bem escrito é instrução. Ele diz ao modelo o que tentar em seguida, e o agente se recupera sozinho. Só cuide para que falha repetida tenha teto — senão, vira o loop improdutivo do critério 4.

**Fator 10 — Small, focused agents (agentes pequenos e focados).** Três agentes com cinco ferramentas cada são melhores do que um agente com quinze. O motivo é o do Dia 4: cada tool a mais é texto na janela e uma decisão a mais a tomar. Agente pequeno você consegue testar, entender quando erra e trocar sem mexer no resto.

## 4. O graph: desenhar quem decide o quê

"Graph" é só o processo desenhado como **nós** (etapas) e **arestas** (transições). O que torna o desenho útil não é a caixinha: é marcar, em cada nó, **quem decide**. Código, LLM ou humano.

Essa é a pergunta de arquitetura do dia: *quais etapas são fixas e em quais o modelo decide?*

O desenho da conciliação que eu venho usando desde o Dia 3 fica assim:

```text
[ingestão do extrato] → [normalização (código)] → [roteador LLM: tipo]
        ├─ PIX simples ──→ [match determinístico] ──→ [registrar]
        ├─ cartão/agrupado → [agente conciliador] ─┬→ [validador] → [registrar]
        │                                          └→ [humano: revisar]
        └─ desconhecido ──→ [fila de exceções / humano]
```

<figure>
  <img src="{{ '/assets/images/desafio-fde/dia-05-graph-da-conciliacao.svg' | relative_url }}" alt="Diagrama do graph da conciliação em nós coloridos por quem decide: caixas claras para código, escuras para LLM e laranjas para humano. A espinha tem ingestão do extrato e cobranças e normalização, ambas em código, e um roteador em LLM que classifica o item. Dele saem três ramos: match direto por valor, CNPJ e data, em código, com cerca de 75% do volume, que vai para registrar; agente conciliador para casos parciais e agrupados, com cerca de 20%, que passa por um validador em código que confere soma e regra, e segue para registrar se passou ou para um humano se reprovou; e fila de exceções, com cerca de 5%, para casos sem cadastro ou acima de R$ 500, que vai direto para decisão humana.">
  <figcaption>O caminho feliz nunca toca o modelo. Os percentuais são ilustrativos — medir essa divisão é a primeira coisa a fazer num processo real.</figcaption>
</figure>

Olhe a distribuição antes de olhar a tecnologia. O caso simples, que costuma ser a maior fatia do volume, **é código puro**. Sem modelo, sem token, sem variação entre uma execução e outra. Determinístico, barato, auditável e rápido.

Não é economia de pão-duro. É arquitetura. Passar um PIX trivial pelo LLM significa pagar token, introduzir variação onde não existia, criar uma superfície nova de erro e perder a capacidade de dizer "este lançamento foi conciliado porque valor, CNPJ e data batem" com certeza absoluta. Você troca uma regra determinística por uma probabilidade. Por nada.

O modelo entra onde a regra não fecha: pagamento parcial, recebimento agrupado, nome que não bate com o cadastro, antecipação de cartão. Casos que são poucos em volume e que consomem a maior parte do tempo de quem faz o trabalho hoje. **A IA entra nos 20% de exceções que comem 80% do esforço.** Foi assim que eu aprendi a olhar processo, e é a lente que eu uso agora em tudo que me pedem para automatizar.

Uma observação sobre percentuais: os números do desenho são chute até você medir. Medir essa divisão — quanto do volume cai em cada ramo — costuma ser mais valioso do que qualquer escolha de framework, porque é ela que diz se o projeto tem ROI. Volta no Dia 10.

E tem um benefício do graph que não é técnico. **Esse desenho é a melhor ferramenta de conversa que existe com quem opera o processo.** Você leva num A4, mostra para a pessoa do financeiro e pergunta: "é isso?". Em quinze minutos você descobre dois ramos que faltavam e uma regra que ninguém documentou. Nenhum documento de requisitos faz isso tão rápido.

## 5. Quem gera não deveria ser quem avalia

Pedir para o modelo revisar o próprio trabalho é tentador e funciona menos do que parece.

Ele tende a ser condescendente consigo mesmo. A Anthropic encontrou isso de forma bem direta nos próprios experimentos com agentes de longa duração: os agentes avaliavam o próprio trabalho de maneira generosa demais, e separar quem faz de quem avalia — com um avaliador deliberadamente cético — deu resultado melhor do que pedir autocrítica.

Faz sentido, e dá para explicar sem antropomorfizar: o raciocínio que gerou a resposta está inteiro na janela. O modelo está relendo a própria argumentação, que por construção é a cadeia mais plausível que ele conseguiu montar. É juiz lendo a defesa que ele mesmo escreveu.

O avaliador separado pode ser três coisas, em ordem crescente de custo:

1. **Uma regra determinística.** A soma bate? O status é válido? A data está no período? Sempre que der para checar com código, cheque com código.
2. **Outro prompt, com janela limpa**, que vê só o resultado e os critérios — não o caminho até ele.
3. **Outro modelo**, quando o risco justifica a conta.

Na conciliação: o agente **propõe**, o validador independente confere soma e regra, e só então grava. Note que `propor_conciliacao` nunca efetiva nada — isso é do Dia 4, e agora dá para ver por quê. A separação entre propor e efetivar é o que torna possível colocar um avaliador no meio.

Esse é também o primeiro critério de parada ganhando dente. "Objetivo atingido **e verificado**" só existe se alguém que não é o agente fizer a verificação. Isso é meio caminho para evals, que é o Dia 8.

## 6. Multi-agente: quando compensa

Multi-agente é o assunto mais em alta e o mais mal usado.

Vale olhar o número que a Anthropic publicou sobre o próprio sistema de pesquisa multi-agente. O ganho foi grande: o sistema com um agente líder coordenando sub-agentes superou o agente único em **90,2%** na avaliação interna de pesquisa. E o custo também: agentes consomem cerca de **4 vezes mais tokens que um chat, e sistemas multi-agente cerca de 15 vezes mais**.

Quinze vezes. Esse número precisa estar na cabeça de quem decide arquitetura, porque ele define o filtro: **multi-agente só se paga em tarefa de alto valor, ampla e paralelizável.** Pesquisa aberta é o exemplo perfeito — cada sub-agente explora uma direção diferente ao mesmo tempo, com janela própria, e a amplitude é o próprio valor.

Conciliação não é isso. É uma tarefa estreita, com passos conhecidos e ambiguidade concentrada num ponto. Jogar um orquestrador com cinco sub-agentes nisso é pagar 15 vezes para resolver um problema que um nó determinístico mais um agente focado resolvem melhor — e de quebra ficam auditáveis.

| Situação | Escolha |
|---|---|
| Passos conhecidos, regras claras | **Workflow** (código + LLM pontual) |
| Passos variáveis, poucas tools, ambiguidade localizada | **Agente único focado** |
| Tarefa ampla, paralelizável, de alto valor | **Orquestrador + sub-agentes** |

A outra lição daquele relato é desconfortável para quem gosta de desenhar organograma de agentes: o que mais melhorou o sistema não foi a topologia, foi **descrição de tarefa**. Sub-agente com instrução vaga duplica trabalho. Cada um precisa de objetivo, formato de saída, orientação de ferramenta e fronteira clara. Delegar para agente tem o mesmo problema de delegar para gente: instrução ruim volta em retrabalho.

Para processo financeiro repetitivo, a resposta quase sempre é a primeira linha da tabela. Workflow determinístico, com LLM só no ponto de ambiguidade. Pouco glamour, muito resultado.

## 7. O loop que eu descobri em mim

Essa parte não estava no plano do dia, mas foi o que mais me pegou.

Escrevendo os quatro critérios para o agente, eu percebi que **a maior parte do meu trabalho roda sem nenhum deles.**

Objetivo atingido e verificado? Quase nunca definido antes de começar. Eu começo a tarefa e descubro no meio o que seria "pronto". Orçamento? Também não: eu entro numa análise sem decidir quanto tempo ela merece, e saio três horas depois com algo 15% melhor do que já estava bom às duas horas. Pedido de ajuda? Esse eu uso tarde demais, sempre. Loop improdutivo? Esse eu reconheço bem: é a quarta versão do mesmo slide, mexendo em coisas que ninguém vai notar.

Coincidência nenhuma: são exatamente os mesmos quatro, com os mesmos sintomas. Agente sem critério de parada gasta token; pessoa sem critério de parada gasta sexta à noite.

Então eu comecei a escrever os critérios antes das tarefas que costumam me engolir. Em uma linha, com número: *pronto é X; no máximo 90 minutos; se travar em Y, pergunto para o Fulano; se eu estiver na terceira versão da mesma coisa, acabou.*

É estranho de tão simples, e funciona pelo mesmo motivo que funciona no agente: **tira a decisão de parar de dentro do loop.** Quem está no meio da tarefa é a pior pessoa para julgar se ela já acabou — vale para o modelo e vale para mim.

## Prática de hoje

Pegue um processo que você quer automatizar e faça duas coisas, no papel mesmo, em meia hora:

**1. Escreva os quatro critérios de parada, com números.** Se algum deles saiu sem número, não é critério — reescreva. E ao lado de cada um, o destino: registrar, escalar, fila de exceções, tentar depois.

**2. Desenhe o graph e marque cada nó** como código, LLM ou humano. Depois estime a porcentagem do volume que passa por cada ramo. Se mais da metade do volume estiver passando por um nó de LLM, olhe de novo: provavelmente tem regra determinística disfarçada de ambiguidade ali.

## TLDR

- **Demo é um agente que começa; produção é um agente que termina.** O loop é a segunda metade do harness.
- **O ciclo em cinco verbos:** perceber, decidir, agir, observar, reavaliar. Quem roda o loop é o harness, não o modelo.
- **Cada volta recobra a janela inteira.** Loop longo não custa proporcional, custa mais.
- **"Parar quando necessário" é adjetivo, não critério.** Critério é condição verificável por código.
- **Os quatro:** objetivo atingido **e verificado**, orçamento esgotado, pedido de ajuda humana, loop improdutivo. O quarto é o mais esquecido e o que mais queima dinheiro.
- **Cada parada precisa de destino** — e orçamento esgotado tem que salvar o progresso parcial.
- **O teto do framework é disjuntor, não desenho.** Parar no `recursion_limit` significa que faltou condição de parada.
- **12-Factor Agents:** lançar/pausar/retomar (6), contatar humano é uma tool (7), seja dono do fluxo de controle (8), compacte erros na janela (9), agentes pequenos e focados (10).
- **Graph é marcar quem decide em cada nó:** código, LLM ou humano. É também a melhor ferramenta de conversa com quem opera o processo.
- **O caminho feliz é código.** O LLM entra nos 20% de exceção que consomem 80% do esforço.
- **Quem gera não avalia.** Modelo é condescendente consigo mesmo; use regra determinística, prompt separado ou outro modelo.
- **Multi-agente custa ~15× um chat.** Só para tarefa ampla, paralelizável e de alto valor. Processo repetitivo pede workflow com agente focado no ponto de ambiguidade.
- **Os quatro critérios também servem para você.** Quem está dentro do loop é a pior pessoa para decidir se ele já acabou.

## Fontes

- [12-Factor Agents — Dex Horthy / HumanLayer](https://github.com/humanlayer/12-factor-agents) — os doze fatores; para hoje, o 6 (lançar, pausar, retomar), o 7 (contatar humanos com tool calls), o 8 (seja dono do fluxo de controle), o 9 (compactar erros na janela) e o 10 (agentes pequenos e focados).
- [How we built our multi-agent research system — Anthropic](https://www.anthropic.com/engineering/multi-agent-research-system) — a arquitetura orquestrador-workers, o ganho de 90,2% na avaliação interna, o consumo de ~15× tokens de um chat e as lições sobre descrição de tarefa para sub-agentes.
- [Effective harnesses for long-running agents — Anthropic](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents) — o avaliador separado e por que a autoavaliação do agente é generosa demais.
- [Human-in-the-loop e interrupts — LangGraph (LangChain docs)](https://docs.langchain.com/oss/python/langgraph/interrupts) — pausar com `interrupt()`, retomar com `Command`, e por que isso exige um checkpointer durável.
- [Workflows — Microsoft Agent Framework](https://learn.microsoft.com/en-us/agent-framework/concepts/workflows/) — executores e arestas tipadas, checkpoint no fim de cada etapa e `RequestInfoExecutor` para o nó humano.
- [Building effective agents — Anthropic](https://www.anthropic.com/engineering/building-effective-agents) — a distinção entre workflow e agente que sustenta a tabela de decisão.

**[→ Acompanhe os próximos artigos](/blog/)**
