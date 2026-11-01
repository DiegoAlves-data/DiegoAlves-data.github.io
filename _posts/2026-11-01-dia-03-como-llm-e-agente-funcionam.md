---
title: "Dia 3 — Como um LLM e um agente funcionam"
date: 2026-11-01
categories: [desafio-fde]
tags: [fde, llm, agentes, tool-calling, rag, glossario]
excerpt: "O modelo não executa nada. Ele só pede. Quem executa é o programa em volta dele. Token, janela de contexto, tool calling, RAG, workflow x agente e o vocabulário de engenharia que aparece em toda conversa de deployment."
---

Repita comigo: **o modelo não executa nada.**

Ele não consulta o banco. Não manda e-mail. Não abre a planilha. Ele só pede. Quem executa é o programa em volta dele.

Parece detalhe. Não é. Quando essa frase entra na cabeça, metade do vocabulário de agentes de IA passa a fazer sentido sozinho: harness, tool calling, MCP, guardrail, eval. Quando não entra, você passa a conversa inteira concordando com a cabeça e anotando palavra para pesquisar depois.

O Dia 3 é sobre isso. Fundamentos. O modelo mental de ponta a ponta do que acontece quando um agente executa uma tarefa, e o vocabulário básico de engenharia de software que aparece em toda reunião técnica. Nada de matemática, nada de código obrigatório.

Para não ficar abstrato, vou usar o mesmo exemplo do começo ao fim: **conciliar os recebimentos PIX de um mês com as notas fiscais emitidas.** Um processo financeiro chato, repetitivo, cheio de exceção. O tipo de coisa que todo mundo quer entregar para um agente.

## 1. O que é um LLM

Um modelo de linguagem grande é, na essência, **uma função que prevê o próximo pedaço de texto.**

Você entrega um texto. Ele calcula a probabilidade de cada próximo pedaço possível. Escolhe um. Acrescenta ao texto. Repete.

Só isso.

Um chatbot é essa função com um roteiro: "isto é uma conversa entre um usuário e um assistente útil", seguido da sua pergunta. O modelo vai completando a fala do assistente, pedaço por pedaço.

Andrej Karpathy, na palestra mais didática que existe sobre o tema, brinca que um LLM é literalmente **dois arquivos**: um com os pesos e outro com o código que os executa. Os pesos são bilhões de números ajustados durante o treino. É o conhecimento do modelo, comprimido.

Esse conhecimento vem em duas etapas:

- **Pré-treino.** O modelo lê uma quantidade absurda de texto e aprende a prever a próxima palavra. Custa milhões de dólares e acontece raramente. O resultado é um "completador de documentos" que sabe muita coisa, mas não sabe conversar.
- **Pós-treino.** O modelo é ajustado com exemplos de boas respostas e com avaliação humana (o famoso RLHF). É isso que transforma o completador em assistente.

Duas consequências práticas saem daqui.

**Alucinação.** O modelo gera o texto *plausível*, não o *verificado*. Se o plausível estiver errado, ele erra com total confiança. Não é defeito de caráter, é a natureza da função. A defesa em produção é sempre a mesma: colocar o fato certo na frente do modelo (via busca ou ferramenta) e verificar a saída.

**Raciocínio.** Os modelos mais recentes "pensam" antes de responder: geram passos intermediários antes da resposta final. Ganham qualidade em problema difícil. Pagam em tokens e em tempo. Não é de graça, e nem toda tarefa precisa.

## 2. Tokens e janela de contexto

O modelo não lê palavras. Lê **tokens**, pedaços de palavra.

Em inglês, um token dá em média uns três quartos de palavra. Em português, o mesmo texto costuma gastar um pouco mais. Tudo é cobrado em token: os de entrada (o que você manda) e os de saída (o que ele gera). E os de saída custam mais.

A **janela de contexto** é tudo o que o modelo enxerga numa chamada: instruções, histórico da conversa, documentos, resultados de ferramentas. Tudo junto, na mesma mesa.

A analogia de Karpathy é a mais útil que eu conheço: **o LLM é o processador, a janela de contexto é a memória RAM.**

E como toda RAM:

- **É limitada.** Cabe muito, mas não cabe tudo.
- **Lotada, piora.** Quanto mais coisa na janela, pior o modelo presta atenção no que importa. A Anthropic chama isso de *context rot*: o contexto "apodrece" à medida que cresce.
- **Apaga entre uma chamada e outra.** O modelo não tem memória. Cada chamada começa do zero.

Esse último ponto merece uma pausa. Quando um chat "lembra" do que você disse cinco mensagens atrás, não é o modelo lembrando. É o programa em volta reenviando o histórico inteiro a cada chamada. Memória, em agente, é sempre um trabalho do programa: decidir o que reinserir na janela e o que deixar de fora.

Na conciliação, isso significa uma coisa bem concreta: se o agente processa 600 lançamentos e você joga todos na janela de uma vez, você paga caro e ainda piora a qualidade. O trabalho de engenharia é decidir o que entra, quando e em que formato. Isso tem nome, *context engineering*, e é o tema do Dia 4.

## 3. Não determinismo e temperatura

O modelo sorteia entre os próximos pedaços prováveis. Por isso a mesma pergunta pode gerar respostas diferentes.

A **temperatura** regula esse sorteio. Baixa: mais previsível. Alta: mais criativa. Para conciliar extrato, você quer baixa. Para brainstorm de nome de campanha, talvez alta.

A consequência para produção é a que mais muda a cabeça de quem vem de software tradicional:

> Não dá para testar agente como se testa software comum. "Rodou uma vez, funciona" não vale.

Você precisa rodar muitas vezes e medir taxa. Acertou 97 de 100 conciliações? Em quais errou? O erro é aceitável ou é o tipo de erro que vira multa? Isso é avaliação (*evals*), e tem dia próprio: o Dia 8.

## 4. A chamada de API

Em produção, um programa conversa com o modelo por **API**: manda uma mensagem estruturada pela internet para o servidor do provedor (Anthropic, OpenAI, Google, ou as versões dentro de Azure e AWS) e recebe a resposta.

Essa mensagem tem quatro partes:

| Parte | O que é | Na conciliação |
|---|---|---|
| **System prompt** | As instruções permanentes | "Você é um conciliador financeiro. Nunca marque um lançamento como conciliado sem nota correspondente." |
| **Mensagens** | O histórico: usuário, assistente, resultados de ferramentas | O pedido do analista e tudo o que já aconteceu |
| **Tools** | A lista de ferramentas disponíveis, cada uma com nome, descrição e parâmetros | `buscar_pix`, `buscar_notas`, `marcar_conciliado` |
| **Parâmetros** | Modelo, temperatura, limite de tokens | Temperatura baixa, limite de saída curto |

Tudo trafega em **JSON**, um formato de texto com campos e valores:

```json
{"conta": "1234", "valor": 150.00, "data": "2026-10-14"}
```

Um **schema** é o contrato que diz quais campos existem e de que tipo. E **saída estruturada** é quando você obriga o modelo a responder num JSON que segue esse schema. Parece burocracia. É o que permite que outro programa leia a resposta sem adivinhar.

## 5. Tool calling

Volte para a frase do começo. O modelo não executa nada.

Então como um agente "consulta o extrato"?

Quando o modelo precisa de algo do mundo, ele devolve **um pedido estruturado**. Algo assim:

```json
{
  "tipo": "tool_use",
  "nome": "buscar_pix",
  "parametros": {"conta": "1234", "periodo": "2026-10"}
}
```

Quem recebe esse pedido é o programa em volta, o **harness**. O harness:

1. lê o pedido;
2. chama o sistema real (a API do banco);
3. pega o resultado;
4. coloca o resultado na janela de contexto;
5. chama o modelo de novo.

O modelo lê o resultado e decide o próximo passo: pedir outra coisa ou responder. E o ciclo se repete até ele parar de pedir.

<figure>
  <img src="{{ '/assets/images/desafio-fde/dia-03-um-pedido-de-ponta-a-ponta.svg' | relative_url }}" alt="Diagrama de sequência: usuário pede para conciliar os PIX de outubro; o harness envia contexto ao modelo; o modelo devolve pedidos de ferramenta; o harness executa nos sistemas e devolve o resultado ao modelo, em loop, até a resposta final. Selos marcam onde os tokens são cobrados.">
  <figcaption>Um pedido, de ponta a ponta. Repare onde está o custo: toda volta do loop reenvia a janela inteira, e ela só cresce.</figcaption>
</figure>

A imagem que eu guardo:

**o modelo é o cérebro que só fala. O harness são as mãos, os olhos e as regras.**

Quando alguém numa conversa técnica disser "o gargalo é o harness, não o modelo", é disso que está falando. O modelo vem pronto, por API, igual para todo mundo. O que diferencia um agente que funciona de um que não funciona quase sempre está em volta dele: que contexto entra, que ferramentas existem, quando parar, o que fica registrado.

E repara no diagrama de novo: **cada volta do loop é uma chamada nova, cobrada de novo, com a janela maior.** Um agente que dá 15 voltas para conciliar um mês custa bem mais do que 15 vezes uma pergunta simples. É por isso que "deixa o agente se virar" sai caro rápido.

## 6. RAG, embeddings e fine-tuning

Pergunta que aparece em toda reunião: como colocar na cabeça do modelo um conhecimento que ele não tem? As regras internas do cliente, os contratos, a política de exceção.

Três caminhos, três usos:

**RAG (Retrieval-Augmented Generation).** Antes de perguntar ao modelo, um sistema de busca encontra os trechos relevantes e coloca na janela. A busca costuma usar **embeddings**: cada trecho de texto vira um vetor de números que representa o significado, e textos com sentido parecido ficam "perto" um do outro. Resultado: você busca por significado, não só por palavra exata. "Estorno" encontra "devolução".

**Tools.** Para ação e para dado vivo. Saldo de hoje, lançamento de ontem, marcar algo como conciliado. RAG traz texto; tool traz o mundo.

**Fine-tuning.** Treinar o modelo de novo com seus exemplos. Raramente é a resposta. Faz sentido para estilo ou para tarefa muito específica e muito repetitiva, não para "ensinar o modelo sobre a empresa".

| Preciso de... | Use | Exemplo na conciliação |
|---|---|---|
| Conhecimento que muda pouco | RAG | A política de quando um PIX pode ser casado com nota de valor diferente |
| Dado vivo ou ação no sistema | Tool | Puxar os PIX do mês, marcar como conciliado |
| Comportamento muito específico, repetido milhares de vezes | Fine-tuning (raramente) | Um formato de saída que nenhum prompt consegue estabilizar |

## 7. Workflow x agente

Aqui mora a distinção que eu mais vou repetir daqui pra frente. Ela vem do texto mais citado do setor sobre o tema, *Building effective agents*, da Anthropic:

- **Workflow:** LLMs e ferramentas orquestrados por **caminhos predefinidos em código**. Quem decide a sequência é o programador. Previsível, barato, testável.
- **Agente:** o **LLM decide dinamicamente** os próximos passos e quais ferramentas usar. Flexível. Mais caro, mais lento, mais difícil de testar.

E a regra de ouro: **comece pela solução mais simples.** Muitas vezes uma única chamada, bem feita, com bom contexto, resolve. Só suba de complexidade quando houver evidência de ganho.

O guia *12-Factor Agents*, da HumanLayer, chega ao mesmo lugar por outro caminho, com uma observação de campo que eu acho libertadora: os bons "agentes" em produção são, na maior parte, **software determinístico com chamadas de LLM nos pontos certos.** Não é um modelo solto decidindo tudo. É engenharia normal com inteligência onde ela compensa.

Para a conciliação, isso quer dizer: provavelmente você não precisa de um agente autônomo. Precisa de um workflow bem desenhado, com o modelo entrando só onde há ambiguidade de verdade.

<figure>
  <img src="{{ '/assets/images/desafio-fde/dia-03-escada-de-complexidade.svg' | relative_url }}" alt="Escada com cinco degraus: uma chamada bem feita; LLM aumentado com busca, tools e memória; workflow fixo com prompt chaining, routing e paralelização; workflow com LLMs coordenando, orquestrador-workers e avaliador-otimizador; e agente, onde o LLM decide os passos. Da esquerda para a direita, aumenta flexibilidade, custo e dificuldade de testar.">
  <figcaption>A escada da complexidade. O degrau certo é o mais baixo que resolve.</figcaption>
</figure>

## 8. Os seis padrões de composição

O mesmo texto da Anthropic organiza os sistemas com LLM em seis padrões de composição. Vale decorar, porque quase todo diagrama de agente que você vai ver é uma combinação deles:

| Padrão | Como funciona | Na conciliação |
|---|---|---|
| **LLM aumentado** | Um LLM com busca, ferramentas e memória. É o tijolo de tudo | Uma chamada que classifica um lançamento consultando a política interna |
| **Prompt chaining** | Etapas fixas; a saída de uma alimenta a próxima | Extrair dados da nota → validar → casar com o PIX |
| **Routing** | Classifica a entrada e manda para o tratamento certo | PIX, cartão e boleto vão para fluxos diferentes |
| **Paralelização** | Várias chamadas ao mesmo tempo, dividindo o trabalho ou votando | Conciliar 12 meses em paralelo, um por mês |
| **Orquestrador–workers** | Um LLM divide o trabalho e delega para sub-agentes | Um orquestrador distribui contas para sub-agentes |
| **Avaliador–otimizador** | Um gera, outro avalia, em ciclo | Um propõe a conciliação, outro audita as exceções |

Repara: o primeiro é o tijolo, e os outros cinco ainda são **workflows**. O código manda na sequência. O agente de verdade, aquele que decide sozinho, é o degrau de cima. E na maioria dos casos você nem chega lá.

## 9. A pilha em três camadas

Se eu tivesse que reduzir o dia a um desenho, seria este. Ele vai ser o esqueleto de todos os diagramas daqui pra frente:

<figure>
  <img src="{{ '/assets/images/desafio-fde/dia-03-pilha-em-tres-camadas.svg' | relative_url }}" alt="Pilha em três camadas: no topo o modelo, que raciocina e pede; no meio o harness, que monta o contexto, roda o loop, executa tools, aplica regras de parada, guardrails, permissões, memória e logs; embaixo os dados e sistemas do cliente, como ERP, banco, planilha, adquirente e e-mail. Harness e dados estão marcados como onde mora o trabalho de deployment.">
  <figcaption>Modelo, harness, dados. A camada de cima vem pronta. As duas de baixo, não.</figcaption>
</figure>

E aqui o Dia 3 conversa com o Dia 2. Lá eu escrevi que o modelo é 5% do trabalho e o resto é o ambiente do cliente. O desenho mostra onde estão os outros 95%: no harness e na camada de dados. ERP sem documentação, permissão amarrada, regra que mora na cabeça de três pessoas.

**A camada de cima todo mundo compra igual. As duas de baixo são onde o deployment ganha ou perde.**

Para ver as peças juntas, este mapa resume o território: fundamentos de LLM, RAG e bancos vetoriais, orquestração, MCP e engenharia de prompt. Alguns desses temas (MCP, orquestração) ganham dia próprio mais à frente.

<figure>
  <a href="{{ '/assets/images/desafio-fde/dia-03-arquitetura-de-ia-mapa.jpg' | relative_url }}" target="_blank" rel="noopener">
    <img src="{{ '/assets/images/desafio-fde/dia-03-arquitetura-de-ia-mapa.jpg' | relative_url }}" alt="Mapa da arquitetura de IA em cinco pilares: fundamentos de LLMs (janela de contexto, embeddings, busca por significado); RAG e bancos de vetores (fluxo RAG, chunking, bases vetoriais); orquestração com LangChain e LangGraph; protocolo MCP; e engenharia de prompt (zero e few-shot, chain of thought).">
  </a>
  <figcaption>Material de apoio: o mapa dos cinco pilares de uma aplicação com IA. Clique para ampliar.</figcaption>
</figure>

## 10. Glossário de engenharia de software

Esse é o vocabulário de engenharia de software que aparece o tempo todo, e que ninguém para pra explicar. Uma frase cada:

| Termo | Em uma frase |
|---|---|
| **Repositório (repo) / Git** | A pasta versionada onde fica o código. Cada mudança registrada é um *commit* |
| **Pull request (PR)** | O pedido para incorporar uma mudança, revisada por outra pessoa |
| **Ambientes (dev / staging / prod)** | Desenvolvimento, homologação e produção. "Subir para prod" é colocar no ar de verdade |
| **Deploy** | O ato de publicar uma versão num ambiente |
| **CI/CD** | Automação que testa e publica o código a cada mudança |
| **Endpoint** | O "endereço" de uma API que executa uma operação específica |
| **SDK / framework** | Bibliotecas que facilitam o uso de uma API (ex.: LangGraph, Claude Agent SDK, Microsoft Agent Framework) |
| **Container (Docker)** | Um "pacote" que roda igual em qualquer máquina |
| **Cloud (Azure, AWS, GCP)** | Computadores alugados. "Rodar na cloud do cliente" é rodar dentro da conta dele, com as regras dele |
| **Webhook** | Um sistema avisa o outro quando algo acontece ("PIX recebido") |
| **Cron / job agendado** | Tarefa que roda num horário fixo |
| **Latência / throughput** | Quanto tempo uma tarefa leva / quantas tarefas por período o sistema aguenta |
| **Autenticação / OAuth / secrets** | Como o sistema prova quem é. Senha e chave ficam num cofre (*vault*), nunca no código |
| **Logs** | O diário do que o sistema fez |

Um conselho de quem dá aula de comunicação: não decore a tabela. **Explique cada termo em voz alta, com um exemplo seu.** Se travar, o termo é frágil. Volte nele.

## TLDR

- **O modelo não executa nada. Ele só pede.** Quem executa é o harness.
- **LLM é uma função que prevê o próximo pedaço de texto.** Pré-treino dá conhecimento; pós-treino dá comportamento de assistente.
- **Alucinação é da natureza da função.** Defesa: fato certo na janela + verificação da saída.
- **Janela de contexto é a RAM.** Limitada, piora quando lota e apaga entre chamadas. Memória é trabalho do harness.
- **Não determinismo muda o teste.** Rodar uma vez não prova nada; é preciso medir taxa (evals).
- **Tool calling:** o modelo devolve um pedido estruturado; o harness executa, devolve o resultado e chama o modelo de novo. Cada volta custa, e a janela só cresce.
- **RAG para conhecimento, tool para ação, fine-tuning raramente.**
- **Workflow antes de agente.** O degrau certo da escada é o mais baixo que resolve.
- **Modelo, harness, dados.** A camada de cima vem pronta. O deployment ganha ou perde nas duas de baixo.

## Fontes

**Leituras**

- [Building effective agents — Anthropic (2024)](https://www.anthropic.com/engineering/building-effective-agents) — a distinção workflow x agente e os seis padrões de composição.
- [Effective context engineering for AI agents — Anthropic](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) — janela de contexto como recurso finito e o conceito de *context rot*.
- [A practical guide to building agents — OpenAI (PDF)](https://cdn.openai.com/business-guides-and-resources/a-practical-guide-to-building-agents.pdf) — seções "When should you build an agent?" e "Agent design foundations".
- [12-Factor Agents — HumanLayer](https://github.com/humanlayer/12-factor-agents) — a observação de campo de que bons agentes são, em sua maioria, software determinístico.
- [AI Agents for Beginners — Microsoft](https://github.com/microsoft/ai-agents-for-beginners) — Lição 1; o repositório tem tradução para português do Brasil.

**Vídeos**

- [Large Language Models explained briefly — 3Blue1Brown (8 min)](https://www.youtube.com/watch?v=LPZh9BOjkQs) — a melhor introdução visual que existe.
- [Intro to Large Language Models — Andrej Karpathy (1h)](https://www.youtube.com/watch?v=zjkBMFhNj_g) — os "dois arquivos", a analogia processador/RAM. É de 2023: trate a parte de futuro como previsão.
- [Transformers, the tech behind LLMs — 3Blue1Brown (27 min)](https://www.youtube.com/watch?v=wjZofJX0v4M) — opcional, para quem quer ver a engrenagem por dentro.

## Material de apoio

**A anatomia de um agente de IA** — 14 páginas, do zero à orquestração: janela de contexto, embeddings, pipeline de RAG, chunking, LangChain e LangGraph, MCP e a arquitetura completa de um agente.

<object data="{{ '/assets/docs/dia-03-anatomia-de-um-agente-de-ia.pdf' | relative_url }}" type="application/pdf" width="100%" height="560" style="border:1px solid #e5e7eb;border-radius:6px;">
  <p>Seu navegador não exibe PDF aqui. <a href="{{ '/assets/docs/dia-03-anatomia-de-um-agente-de-ia.pdf' | relative_url }}">Baixe o PDF</a>.</p>
</object>

**[↓ Baixar o PDF (3,7 MB)]({{ '/assets/docs/dia-03-anatomia-de-um-agente-de-ia.pdf' | relative_url }})**

**[→ Acompanhe os próximos artigos](/blog/)**
