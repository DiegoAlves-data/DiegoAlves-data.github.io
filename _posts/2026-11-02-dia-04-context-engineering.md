---
title: "Dia 4 — Harness I: Context Engineering"
date: 2026-11-02
categories: [desafio-fde]
tags: [fde, context-engineering, harness]
excerpt: "Quando um agente erra, quase nunca é o modelo. É o que estava na janela quando ele decidiu. Contexto como orçamento finito, altitude de prompt, design de tool, just-in-time retrieval, compaction, sub-agentes e o harness da conciliação desenhado do zero."
---

Quando um agente erra, a primeira reação de quase todo mundo é trocar de modelo.

Quase sempre é a reação errada.

A documentação da LangChain diz isso sem rodeio: quando um agente falha, o motivo mais comum é que ele não recebeu o contexto certo, e não que o modelo seja fraco. O modelo decidiu mal porque viu mal. Trocar o cérebro não resolve um problema de visão.

No Dia 3 eu deixei uma frase pendurada: o modelo não executa nada, quem executa é o harness. Hoje é a primeira metade do harness. A metade que decide **o que o modelo vê**.

## De prompt engineering para context engineering

Em 2023, a conversa era sobre escrever o prompt certo. A palavra mágica, a ordem das instruções, o "pense passo a passo".

Isso fazia sentido quando o uso era uma pergunta e uma resposta. Deixou de fazer quando os agentes passaram a rodar por dezenas de passos. A cada volta do loop entra coisa nova na janela: resultado de ferramenta, histórico, trecho de documento, erro de API. O prompt que você escreveu com carinho vira uma fração pequena do que o modelo está lendo no passo 14.

A pergunta mudou. Saiu "qual é a melhor frase?" e entrou **"qual é o melhor conjunto de informações na janela, neste passo?"**

Andrej Karpathy deu a definição que pegou: context engineering é a arte e a ciência de encher a janela de contexto com exatamente a informação certa para o próximo passo. Repara no "próximo passo". Não é o contexto da tarefa inteira. É o do passo seguinte, que muda a cada volta.

## O que é o harness

Harness é tudo o que envolve o modelo e decide o que ele vê e o que ele pode fazer. Seis peças:

1. **System prompt.** Papel, objetivo, regras, formato de saída.
2. **Tools.** O que ele pode consultar e executar.
3. **Gestão de contexto.** O que entra na janela, o que sai, o que é resumido.
4. **Memória.** Notas persistentes que vivem fora da janela.
5. **Loop e regras de parada.** Quantas voltas, quando parar. É o Dia 5.
6. **Guardrails, permissões e logs.** O que ele não pode fazer e o registro do que fez. É o Dia 9.

Hoje o foco são as quatro primeiras. Juntas, elas respondem a uma pergunta só: o que está na mesa quando o modelo decide?

## Contexto é orçamento, não depósito

A Anthropic trata a janela de contexto como um **orçamento de atenção**. E a razão é mecânica, não metafórica.

A arquitetura transformer faz cada token se relacionar com todos os outros. Com n tokens, são n² relações. Quando a janela cresce, a atenção do modelo se espalha por mais pares, e a capacidade de lembrar e usar a informação certa cai. Esse efeito tem nome: *context rot*. O contexto apodrece à medida que cresce.

A consequência contraria a intuição de quem vem de dados. Em análise, mais dado costuma ser melhor. Na janela de um modelo, **mais contexto não é contexto melhor.**

O objetivo, nas palavras da própria Anthropic, é encontrar o menor conjunto possível de tokens de alto sinal que maximiza a chance do resultado desejado.

Na conciliação de PIX do Dia 3, isso significa não colar 12 meses de extrato no prompt. São milhares de linhas, a maioria irrelevante para o lançamento que o agente está olhando agora. Você paga por cada uma delas e ainda piora a decisão.

## Altitude do prompt

O system prompt erra de dois jeitos opostos.

O primeiro é o **if-else disfarçado de texto**. Quarenta regras numeradas, cada uma tentando prever um caso. Fica frágil: o primeiro caso que ninguém previu cai num buraco entre a regra 23 e a 24. E fica caro de manter, porque cada exceção nova vira mais uma regra.

O segundo é o **prompt vago**. "Você é um assistente financeiro, concilie os lançamentos com cuidado." O modelo não sabe o que é "com cuidado" no seu negócio. Ele vai preencher com o plausível, e o plausível nem sempre é o certo.

A altitude certa fica no meio: **heurísticas, não regras.** Específico o bastante para guiar o comportamento, flexível o bastante para o modelo raciocinar no caso novo.

Na conciliação:

> Concilie usando valor, data com até 2 dias úteis de diferença e CNPJ. Na dúvida, escale.

Uma linha que cobre centenas de casos e diz o que fazer quando nenhum deles se aplica.

## Design de tool

A tentação é dar ao agente uma ferramenta para cada coisa. `buscar_pix`, `buscar_pix_por_valor`, `buscar_pix_por_data`, `buscar_pix_por_cnpj`, `buscar_ted`, `buscar_recebimento`...

Cada tool a mais é texto a mais na janela (nome, descrição, parâmetros) e uma decisão a mais para o modelo. Pior: tools que se sobrepõem criam ambiguidade. Se o modelo pode buscar por três caminhos parecidos, ele vai escolher errado uma parte das vezes.

O teste que a Anthropic propõe é simples e eu uso direto: **se um engenheiro humano não consegue dizer com certeza qual tool usar numa situação, o agente também não vai conseguir.**

Para a conciliação, três bastam:

| Tool | Propósito |
|---|---|
| `buscar_lancamentos` | Traz lançamentos filtrados por período, conta e tipo |
| `propor_conciliacao` | Registra o casamento proposto entre recebimento e cobrança, sem efetivar |
| `escalar_excecao` | Manda o caso para um humano, com o motivo |

Poucas, sem sobreposição, cada uma com um propósito que cabe numa frase. E o resultado de cada tool também é contexto: `buscar_lancamentos` deve devolver só os campos que importam para a decisão, não as 40 colunas do extrato.

## Few-shot curado

Exemplos funcionam muito bem. Por isso mesmo, viram vício.

O erro comum é empilhar no prompt todo caso extremo que já deu problema. Vinte exemplos, cada um uma exceção diferente. O modelo passa a tratar a exceção como padrão e começa a enxergar caso estranho onde não tem.

O melhor caminho é **poucos exemplos canônicos**, que mostrem o comportamento esperado na maioria das situações. Na conciliação, cinco conciliações exemplares e duas difíceis, sendo uma delas um escalonamento feito do jeito certo. Para o modelo, um exemplo bem escolhido diz mais do que três parágrafos de instrução.

## Just-in-time retrieval e progressive disclosure

Aqui mora o conceito que eu mais gosto. E o que eu mais usava sem saber o nome.

A estratégia tradicional é carregar tudo antes: buscar os documentos relevantes, colar na janela e só então chamar o modelo. A alternativa é **dar ao agente referências leves** (caminhos de arquivo, nomes de tabela, links) e deixar que ele carregue o conteúdo quando precisar, usando as ferramentas.

Na conciliação: em vez de colar o extrato, você dá o caminho do extrato. O agente abre o que precisa, quando precisa.

E a estrutura dessas referências já é informação. Uma pasta `extratos/2026-10/` com arquivos `banco-a-conta-1234.csv` e `adquirente-cartao.csv` diz ao agente de onde vem cada dado antes de ele abrir qualquer coisa. Nome de arquivo, hierarquia de pasta e data de modificação são metadados que o modelo usa para se orientar. O agente vai descobrindo o contexto em camadas, explorando. Isso é **progressive disclosure**.

Agora o "eu já fazia isso". O sistema que eu uso para organizar meu trabalho com IA tem um arquivo de instrução na raiz, que o agente lê em toda sessão. Ele não contém o conteúdo dos projetos. Contém o mapa: onde fica cada coisa, que skill existe para cada tarefa, que arquivo ler antes de abrir os outros. As skills ficam em pastas próprias e só entram na janela quando a tarefa pede. Cada projeto tem um `README.md` que o agente lê antes de abrir qualquer arquivo solto.

Tem até uma regra que eu escrevi depois de levar susto com a conta de tokens: o índice das minhas transcrições de reunião passou de 49 KB, então o arquivo de instrução diz explicitamente para **não ler o índice** e buscar por nome de arquivo com `grep`. Na época eu chamava isso de economia. É just-in-time retrieval.

O próprio Claude Code funciona num modelo híbrido parecido: um arquivo de instrução entra inteiro no começo, e o resto o agente busca com `glob` e `grep` conforme precisa. O ganho é óbvio. O custo também: explorar é mais lento do que receber tudo pronto. A escolha depende da tarefa. Conteúdo que quase nunca muda e é usado sempre entra antes. O resto, sob demanda.

## Quando a tarefa não cabe numa janela

Conciliar um mês cabe. Conciliar o fechamento do ano inteiro, provavelmente não. Três técnicas para isso.

**Compaction.** Perto do limite da janela, o harness resume a conversa até ali e recomeça uma janela nova com esse resumo. A arte está no que preservar: decisões tomadas, pendências, exceções em aberto. E no que jogar fora: resultados brutos de tool que já foram usados. Resultado de ferramenta antigo é o candidato mais seguro ao corte.

**Notas estruturadas.** O agente escreve fora da janela, num arquivo de progresso, e relê quando precisa. A memória deixa de depender do histórico da conversa.

**Sub-agentes.** Um agente principal coordena e delega tarefas focadas para sub-agentes, cada um com uma janela limpa. O sub-agente pode gastar dezenas de milhares de tokens explorando e devolver só a síntese. Na conciliação, um sub-agente "auditor de exceções" revisa os casos estranhos e devolve só a lista final. O agente principal nunca vê a bagunça.

A Anthropic publicou uma imagem para tarefas longas que eu achei precisa: **um profissional trabalhando em turnos**, em que cada pessoa que chega não tem memória do turno anterior. A solução é a mesma de qualquer operação humana de turno. Cada sessão começa lendo o arquivo de progresso (o que foi feito, o que falta, que decisões foram tomadas), faz um avanço incremental, registra tudo e passa o turno. No fechamento anual, uma sessão por mês, com o resumo do mês anterior como ponto de partida.

Dois achados desses artigos valem atenção.

O primeiro: em tarefas longas, alguns modelos começam a **encerrar o trabalho antes da hora** quando percebem que estão chegando ao limite da janela. Os autores chamam isso de *context anxiety*. O agente "fecha" para caber, não porque terminou. Recomeçar a janela do zero, com um handoff bem feito, resolve melhor do que compactar, porque compactar mantém o agente na mesma sessão. Modelos mais novos já mostram menos esse comportamento, mas vale testar no seu.

O segundo: agentes tendem a **avaliar o próprio trabalho de forma generosa demais**. Separar quem faz de quem avalia, com um avaliador ajustado para ser cético, funcionou melhor do que pedir ao mesmo agente para criticar a própria saída. Isso volta no Dia 8, quando o assunto for evals.

## As quatro estratégias

A LangChain organiza tudo isso em quatro verbos. Eu uso como checklist quando olho um harness:

| Estratégia | O que faz | Na conciliação |
|---|---|---|
| **Write** (escrever fora) | Salva notas, plano e progresso fora da janela | Arquivo de progresso do fechamento anual |
| **Select** (selecionar) | Traz para a janela só o que importa agora | `buscar_lancamentos` filtrado pelo período e pela conta |
| **Compress** (comprimir) | Resume histórico, corta resultado longo de tool | Resumo entre os meses; tool devolve só os campos da decisão |
| **Isolate** (isolar) | Divide o trabalho em janelas separadas | Sub-agente auditor de exceções |

Se um agente está caro ou errando, uma dessas quatro está faltando. Quase sempre.

<figure>
  <img src="{{ '/assets/images/desafio-fde/dia-04-o-que-entra-na-janela.svg' | relative_url }}" alt="Diagrama em três colunas. Fora da janela: extrato de 12 meses, histórico do loop, exceções estranhas e progresso do fechamento. O harness decide, com quatro estratégias: select traz só o período e a conta; compress resume e corta resultado antigo; isolate usa sub-agente em janela limpa; write grava fora e relê sob demanda. A janela do próximo passo tem uma parte fixa (system prompt com heurísticas, 3 tools sem sobreposição, 5 exemplos canônicos e 2 difíceis) e uma parte que muda (lançamentos do mês, resumo das decisões, lista final de exceções do auditor, caminho do arquivo de progresso). Embaixo, duas barras comparam a janela com tudo colado, que erra, custa e demora mais, com a janela de alto sinal.">
  <figcaption>O harness como filtro. A parte fixa da janela é desenho de prompt; a parte que muda é context engineering acontecendo a cada volta do loop.</figcaption>
</figure>

## Um system prompt, antes e depois

Este é o tipo de prompt que eu vejo com frequência (e que já escrevi):

```text
Você é um assistente financeiro especialista em conciliação bancária.
Concilie os recebimentos com as cobranças com muito cuidado e atenção.

Regras:
1. Se o valor for igual, concilie.
2. Se o valor for diferente em até R$ 0,10, concilie.
3. Se o valor for diferente em mais de R$ 0,10, não concilie.
4. Se a data for igual, concilie.
5. Se a data for diferente em 1 dia, concilie.
6. Se a data for diferente em 2 dias e for segunda-feira, concilie.
[... mais 30 regras ...]
37. Se o CNPJ não bater mas o nome for parecido, use seu julgamento.

Segue o extrato completo do ano:
[12 meses de extrato colados aqui]
```

Os problemas aparecem todos de uma vez. "Com muito cuidado e atenção" não diz nada. As regras são código disfarçado, e a regra 37 contradiz a lógica das outras 36. As tools nem aparecem. E o extrato inteiro na janela garante context rot desde a primeira chamada.

Reescrito pelo método:

```markdown
# Papel e objetivo
Você é o conciliador de recebimentos da empresa. Seu objetivo é casar
cada recebimento (PIX ou cartão) com a cobrança correspondente.

# Contexto do negócio
- Tipos de cobrança: mensalidade recorrente, cobrança avulsa e parcelada.
- Fontes: extrato bancário (fonte da verdade do caixa), planilha de
  cobranças e notas fiscais emitidas.

# Como decidir
- Casa quando: valor igual, CNPJ/CPF igual, data em até 2 dias úteis.
- Pagamento parcial ou agrupado: proponha, mas marque para revisão.
- Divergência acima de R$ 500 ou cliente sem cadastro: escale.

# Ferramentas
- buscar_lancamentos: use primeiro, sempre filtrando por período.
- propor_conciliacao: nunca marque como conciliado sem passar por ela.
- escalar_excecao: use quando estiver em dúvida. Escalar é barato;
  errar é caro.

# Formato de saída
JSON conforme o schema ConciliacaoResultado.

# Exemplos
<3 a 5 exemplos canônicos, incluindo 1 escalonamento correto>
```

Quatro diferenças importam. Os blocos têm cabeçalho claro, então o modelo (e quem mantém o prompt) sabe onde cada coisa está. As regras viraram heurísticas. Cada tool vem com orientação de quando usar. E o custo do erro está escrito: **escalar é barato, errar é caro.** Essa última frase muda o comportamento do agente mais do que qualquer regra numerada, porque diz para que lado ele deve errar quando não tiver certeza.

E o extrato sumiu do prompt. Ele entra pela tool, filtrado, quando o agente pedir.

## O harness da conciliação, desenhado

Juntando as peças, o desenho tem três partes.

**Context: o que entra na janela.** System prompt com heurísticas, três tools, cinco exemplos canônicos e dois difíceis. Do extrato, só a referência. A cada passo, o resultado da última busca, já enxuto.

**Loop: como o ciclo roda.** O agente busca os lançamentos do período, propõe conciliações uma a uma e escala o que não fecha. Para quando não houver mais recebimento pendente no período, ou quando bater um limite de voltas. As regras de parada são o Dia 5.

**Graph: os nós da operação.** Um nó de ingestão (código determinístico, sem modelo) normaliza o extrato e a planilha. Um nó de conciliação com o agente. Um nó de auditoria com o sub-agente cético, que revisa só as exceções. E um nó humano, que recebe o que foi escalado. O modelo entra só onde há ambiguidade de verdade, que era a conclusão do Dia 3.

## Por que isso é assunto de negócio

Contexto ruim custa de três formas.

**Erro.** O agente decide mal porque viu mal. Em conciliação, erro vira lançamento errado no fechamento, e o retrabalho de alguém do financeiro.

**Token.** O mesmo trabalho fica três, quatro vezes mais caro, porque cada volta do loop reenvia uma janela inchada.

**Latência.** Janela maior demora mais para processar. O fechamento que devia sair em uma hora sai no fim do dia.

Por isso context engineering não é detalhe técnico para a equipe resolver depois. É uma alavanca de ROI. Quando alguém pergunta numa reunião por que o piloto ficou caro ou por que errou em 8% dos casos, a resposta raramente é "precisamos de um modelo melhor". Normalmente é: olha o que estava na janela.

Um exercício para hoje: pegue o prompt mais longo que você usa no trabalho. Conte quantas linhas são regra, quantas são heurística e quantas são dado colado que poderia vir por ferramenta. Depois corte metade.

## TLDR

- **Quando o agente erra, olhe a janela antes de trocar o modelo.** O motivo mais comum de falha é contexto errado, não modelo fraco.
- **Context engineering é encher a janela com a informação certa para o próximo passo.** A pergunta deixou de ser a frase do prompt.
- **Harness é tudo em volta do modelo:** system prompt, tools, gestão de contexto, memória, loop e guardrails.
- **Contexto é orçamento finito.** Cada token disputa atenção; janela cheia sofre *context rot*. Busque o menor conjunto de tokens de alto sinal.
- **Altitude do prompt:** heurísticas, não 40 regras. Nem if-else disfarçado, nem vago.
- **Poucas tools, sem sobreposição.** Se um humano não sabe qual usar, o agente também não sabe.
- **Few-shot curado:** poucos exemplos canônicos, não um catálogo de exceções.
- **Just-in-time e progressive disclosure:** dê o caminho do extrato, não o extrato. Nome de pasta e de arquivo também é contexto.
- **Tarefa longa:** compaction, notas fora da janela, sub-agentes com janela limpa e trabalho em turnos com arquivo de progresso.
- **Write, select, compress, isolate.** Se o agente está caro ou errando, uma das quatro está faltando.
- **Escalar é barato; errar é caro.** Escreva o custo do erro no prompt.
- **Contexto ruim custa em erro, token e latência.** Context engineering é alavanca de ROI.

## Fontes

**Leituras**

- [Effective context engineering for AI agents — Anthropic](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) — a leitura mais importante do tema: orçamento de atenção, altitude do prompt, design de tool, just-in-time retrieval, compaction, notas e sub-agentes.
- [Context engineering for agents — LangChain](https://blog.langchain.com/context-engineering-for-agents/) — as quatro estratégias: write, select, compress, isolate.
- [Effective harnesses for long-running agents — Anthropic](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents) — a analogia dos turnos e o arquivo de progresso.
- [Harness design for long-running application development — Anthropic](https://www.anthropic.com/engineering/harness-design-long-running-apps) — *context anxiety*, reset x compaction e o avaliador separado.
- [AI Agents for Beginners, Lição 12: Context Engineering — Microsoft](https://github.com/microsoft/ai-agents-for-beginners/tree/main/12-context-engineering) — revisão didática, com tradução para português do Brasil no repositório.

**Vídeo**

- [Intro to Large Language Models — Andrej Karpathy](https://www.youtube.com/watch?v=zjkBMFhNj_g) — o trecho "LLM OS", a partir de uns 41 minutos: a janela de contexto como memória RAM.

## Material de apoio

**O playbook de context engineering**: 12 páginas com o essencial do dia: context rot, as peças do harness, as quatro estratégias, os oito conceitos com exemplo, o trabalho em turnos, a anatomia do system prompt e a conta de negócio.

<figure>
  <a href="{{ '/assets/docs/dia-04-context-engineering-playbook.pdf' | relative_url }}" target="_blank" rel="noopener">
    <img src="{{ '/assets/images/desafio-fde/dia-04-playbook-capa.jpg' | relative_url }}" alt="Capa do PDF O Playbook do Arquiteto: Context Engineering, sobre como construir agentes autônomos dominando o dado e o harness.">
  </a>
  <figcaption>Clique na capa para abrir o PDF completo.</figcaption>
</figure>

**[↓ Baixar o PDF (2,5 MB)]({{ '/assets/docs/dia-04-context-engineering-playbook.pdf' | relative_url }})**

**[→ Acompanhe os próximos artigos](/blog/)**
