---
title: "Dia 6 — Tools e MCP: o agente conectado aos sistemas de verdade"
date: 2026-11-04
categories: [desafio-fde]
tags: [fde, mcp, tools, idempotência, integração]
excerpt: "Até aqui o agente da conciliação pensava, mas não encostava em nada. Como especificar uma tool que o modelo usa direito, o que é idempotência e por que ela importa tanto em produtos financeiros, o que o MCP resolve, quando eu não usaria, e as três tools do caso escritas no formato do protocolo."
---

## O que estamos construindo

Desde o Dia 3 eu uso um único caso para não deixar nada no abstrato: um **agente de conciliação financeira**.

Conciliar é casar duas listas que deveriam bater e quase nunca batem de primeira. De um lado, o que entrou no banco: PIX, boleto, TED, recebimento de cartão. Do outro, o que a empresa cobrou: as cobranças e notas fiscais emitidas no ERP. Para cada recebimento, alguém precisa dizer a qual cobrança ele corresponde, dar baixa e seguir para o próximo.

Na maioria das empresas isso ainda é feito à mão, numa planilha, por uma pessoa do financeiro que conhece os clientes pelo nome. Quando valor, CNPJ e data batem, é rápido. O tempo vai embora no resto: o cliente que pagou duas notas num PIX só, o pagamento parcial, o recebimento feito pelo CPF do sócio em vez do CNPJ da empresa, a antecipação de cartão que chega com desconto. É esse resto que o agente vem resolver.

O trabalho dele cabe em quatro passos: buscar os recebimentos no extrato, procurar a cobrança candidata no ERP, propor o casamento com uma justificativa e, quando não tiver segurança, escalar para uma pessoa. Ele nunca efetiva uma baixa sozinho. Propõe, um validador confere, e acima de um valor um humano aprova.

O que a série já construiu até aqui:

| Dia | O que entrou no agente |
|---|---|
| 3 | A anatomia: o modelo só fala, quem executa é o harness |
| 4 | O que ele vê: system prompt, poucos exemplos e três tools sem sobreposição |
| 5 | Quando ele para: os quatro critérios de parada e o graph que separa código, LLM e humano |
| 6 | Hoje: como ele fala com o banco, o ERP e o time financeiro |

O caso é inspirado num processo financeiro real que eu conheço bem, mas tudo aqui é anônimo e os dados dos exemplos são fictícios.

---

Nos últimos três dias o agente da conciliação ganhou janela de contexto, loop, critério de parada e um graph. Teve uma coisa que eu fui empurrando: até agora ele não encostou em nada de verdade. Ele nunca abriu um extrato de banco nem gravou uma linha no ERP.

Tudo isso passa pelas tools.

Todo desenho de arquitetura de agente tem uma seta que sai do agente e vai para "os sistemas do cliente". Costuma ser a seta mais fina do slide. Na prática, é onde está boa parte do trabalho: o ERP de quinze anos, a API que ninguém documentou, a planilha que só uma pessoa do financeiro entende, o banco cuja autenticação expira de hora em hora.

Hoje o assunto é essa seta. Como se escreve uma tool que o modelo usa direito, e o que o MCP (Model Context Protocol) muda nessa história.

## 1. Uma tool é um contrato com alguém imprevisível

No Dia 3 eu mostrei que o modelo não executa nada. Ele devolve um pedido estruturado ("quero chamar `buscar_pix` com estes parâmetros") e o harness faz a chamada de verdade. A tool é o que o modelo enxerga desse pedido: um nome, uma descrição, um esquema de parâmetros e o formato do que volta.

A Anthropic tem uma definição que eu acho a melhor de todas: uma tool é **um contrato entre um sistema determinístico e um agente não determinístico**.

Uma função comum, `getWeather("SP")`, faz sempre a mesma coisa quando é chamada. O agente, diante da mesma pergunta, pode chamar a tool, pode responder de memória, pode pedir esclarecimento, pode chamar a tool com o parâmetro errado. E faz isso de um jeito diferente em execuções diferentes.

Por isso tool para agente se escreve de outro jeito que API para programador. O programador abre a documentação, testa, lê o código-fonte se precisar. O modelo tem umas poucas linhas de texto na janela e precisa decidir, ali, naquele passo, se chama, como chama e o que fazer com o resultado.

A comparação que me ajuda: API é manual técnico. Tool é o bilhete que você deixa para alguém que começa amanhã no seu lugar e não vai conseguir te ligar.

## 2. Como uma boa tool é escrita

A Anthropic publicou um guia sobre isso, *Writing effective tools for agents*, e boa parte do que eu faço hoje vem de lá. Junto com o que aprendi errando, ficou assim:

**Não embrulhe cada endpoint da API.** O ERP de um cliente pode ter quinze endpoints de cobrança. O agente não precisa de quinze tools. Precisa de três ou quatro no nível da tarefa. O exemplo do guia é bom: em vez de `list_users`, `list_events` e `create_event`, uma tool `schedule_event` que encontra o horário livre e marca. Na conciliação, em vez de `listar_clientes`, `listar_cobrancas` e `filtrar_por_valor`, uma `erp_buscar_cobrancas` que já filtra por cliente, valor e vencimento. Cada tool a mais é texto na janela e uma decisão a mais para o modelo. Isso é do Dia 4 e continua valendo.

**Dê sobrenome para as tools.** Quando o agente se conecta a mais de um sistema, aparecem duas `buscar`. A saída é prefixar pelo sistema: `banco_buscar_lancamentos`, `erp_buscar_cobrancas`. A própria especificação do MCP avisa que nomes iguais em servidores diferentes vão colidir e recomenda esse prefixo.

**Devolva o que ajuda a decidir.** Um ID como `a3f9-77c2-...` não ajuda o modelo em nada. O nome do pagador ajuda. A Anthropic relata que trocar identificadores crípticos por nomes legíveis reduziu alucinação nas tarefas de busca. Eles também testaram um parâmetro `response_format` com as opções `concise` e `detailed`: no exemplo do Slack, a resposta completa gastou 206 tokens e a concisa, 72.

**Pague só pelo que vai usar.** Paginação, filtro e limite, sempre com um cursor para continuar. O Claude Code corta qualquer retorno de tool em 25 mil tokens por padrão, e isso existe por um motivo. Lembra do Dia 5: tudo que a tool devolve fica na janela e é cobrado de novo em cada volta do loop. Um extrato de 2 mil linhas devolvido inteiro custa uma vez para entrar e mais uma vez a cada passo seguinte.

**A descrição é um prompt.** Escreva como explicaria para um colega novo: o que a tool faz, quando usar e, principalmente, quando **não** usar. Parâmetro com nome sem ambiguidade (`cliente_cnpj`, não `cliente`). Formato de data explícito. Parece detalhe, mas o guia conta que ajustes finos nas descrições das tools foram parte do que levou o Claude Sonnet 3.5 ao estado da arte no SWE-bench Verified na época.

**Escreva erros que ensinam.** Compare:

```text
400 Bad Request
```

com

```text
PERIODO_INVALIDO: o intervalo pedido tem 92 dias e o máximo é 31.
Divida em consultas mensais, por exemplo de 2026-08-01 a 2026-08-31.
```

O primeiro faz o agente chutar. O segundo diz o que tentar em seguida, e ele se corrige sozinho. É o fator 9 do 12-Factor Agents que apareceu no Dia 5, agora do lado de quem escreve a tool.

## 3. Ler, escrever, escalar

Antes de escrever qualquer tool, eu classifico. A pergunta é simples: o que acontece se ela for chamada duas vezes, ou chamada errado?

| Tipo | Exemplo na conciliação | Se der errado | O que exige |
|---|---|---|---|
| **Leitura** | Buscar recebimentos no extrato | Gasta token, nada muda no mundo | Filtro, paginação, retorno enxuto |
| **Escrita** | Registrar uma proposta de conciliação | Registro duplicado, baixa em dobro | Idempotência, validação, aprovação acima de um valor |
| **Escalonamento** | Abrir exceção para o time financeiro | Alguém recebe um chamado a mais | Destino claro, contexto junto, prazo |

Leitura é tranquila. Pode repetir à vontade. Escalonamento é a válvula de segurança que o Dia 5 transformou em tool. A escrita é onde mora o risco, e o nome do risco é idempotência.

### Idempotência, com um exemplo que acontece toda semana

Uma operação é idempotente quando fazer duas vezes dá o mesmo resultado que fazer uma.

O cenário clássico: o agente chama `conciliacao_propor`. O ERP recebe, grava, responde. A resposta se perde no caminho, por timeout ou por instabilidade de rede. Do ponto de vista do agente, a chamada falhou. O harness faz o que todo harness bem escrito faz: tenta de novo.

Sem proteção, agora existem duas propostas para o mesmo recebimento. Em conciliação isso vira baixa em dobro. Em pagamento, cobrança em dobro.

A técnica que resolve é a **chave de idempotência**. Cada pedido de escrita leva um identificador. O sistema guarda o resultado da primeira vez que viu aquela chave e, quando ela aparece de novo, devolve o mesmo resultado sem executar outra vez. A Stripe documenta isso muito bem: guarda o status e o corpo da primeira resposta, compara os parâmetros dos pedidos repetidos e recusa se eles mudaram, e só descarta a chave depois de pelo menos 24 horas.

<figure>
  <img src="{{ '/assets/images/desafio-fde/dia-06-retry-e-idempotencia.svg' | relative_url }}" alt="Diagrama em duas linhas com cinco passos cada. Na primeira, sem chave: o agente chama conciliacao_propor, o ERP grava a proposta P-101, ocorre timeout e a resposta não volta, o harness faz retry com a mesma chamada, e o ERP grava P-102, gerando dois registros para o mesmo recebimento. Na segunda, com a chave lanc_8812 mais cob_0457, que vem do negócio e não é sorteada: o agente chama, o ERP grava P-101 sob a chave, ocorre timeout, o retry leva a mesma chave, e o ERP reconhece e devolve P-101 de novo sem gravar nada. Uma nota explica que, se a chave fosse gerada pelo modelo a cada chamada, o retry levaria uma chave nova e o resultado seria o da primeira linha.">
  <figcaption>O timeout é o mesmo nas duas linhas. Só a chave muda o desfecho.</figcaption>
</figure>

Tem um detalhe que eu só entendi direito montando o caso, e que muda o desenho com agente.

A Stripe recomenda gerar a chave como um UUID aleatório. Faz sentido para um sistema tradicional, porque quem repete a chamada é o mesmo código, que guardou a chave numa variável. Com agente, a repetição pode vir de dois lugares: do harness, que repete a mesma requisição, ou do próprio modelo, que numa volta seguinte do loop decide chamar a tool de novo. No segundo caso, se a chave for sorteada a cada chamada, ela chega diferente e a proteção não funciona.

Por isso, em tool de escrita para agente, eu prefiro **chave derivada do negócio**: `id_lancamento + id_cobranca`. O mesmo casamento sempre gera a mesma chave, venha a repetição de onde vier. E melhor ainda se quem monta a chave for o servidor, a partir dos parâmetros, e não o modelo. Uma decisão a menos para ele errar.

## 4. O problema que o MCP resolve

Até aqui, nada disso depende de MCP. Dá para escrever tools ótimas direto no código do seu agente. O problema aparece quando você tem mais de um agente, ou mais de um cliente.

Sem um padrão, cada par agente–sistema é uma integração própria. Três agentes falando com quatro sistemas são doze integrações para escrever, testar e manter. E cada sistema autentica e devolve erro do seu próprio jeito.

O MCP é um protocolo aberto que padroniza essa conversa. Quem constrói uma aplicação de IA implementa o lado client uma vez. Quem expõe um sistema implementa o lado server uma vez. A partir daí, qualquer client conversa com qualquer server. As doze integrações viram sete implementações, e essa diferença cresce rápido conforme entram agentes e sistemas.

<figure>
  <img src="{{ '/assets/images/desafio-fde/dia-06-sem-padrao-e-com-mcp.svg' | relative_url }}" alt="Diagrama em dois painéis. À esquerda, sem padrão: três agentes (conciliação, cobrança e assistente do time) ligados por doze linhas a quatro sistemas (banco, ERP, planilha de regras e CRM), com o rótulo 3 agentes vezes 4 sistemas igual a 12 integrações. À direita, com MCP: os mesmos três agentes, cada um como client MCP, ligados a uma barra vertical laranja chamada protocolo MCP, que se liga aos quatro sistemas, cada um como server MCP, com o rótulo 3 clients mais 4 servers igual a 7 implementações.">
  <figcaption>Multiplicação vira soma. Com dez agentes e vinte sistemas, a diferença é entre 200 integrações e 30 implementações.</figcaption>
</figure>

A própria documentação oficial usa a comparação com o USB-C: antes cada aparelho tinha o seu cabo, agora existe um conector.

A história é curta. A Anthropic abriu o protocolo em novembro de 2024. Em março de 2025 a OpenAI anunciou suporte nos produtos dela, e foi aí que deixou de ser "o padrão de uma empresa". Em dezembro de 2025 a Anthropic doou o MCP para a Agentic AI Foundation, um fundo da Linux Foundation criado junto com OpenAI e Block. Hoje Claude, ChatGPT, VS Code, Cursor e muitas outras ferramentas falam MCP.

Eu uso isso todo dia sem pensar. No assistente que organiza a minha semana, agenda e Drive entram por conectores MCP. Eu não escrevi uma linha de integração com a API do Google. Alguém escreveu o server uma vez, e o meu assistente é só mais um client.

## 5. Por dentro: host, client e server

Três papéis:

- **Host** é a aplicação de IA. Pode ser o Claude Desktop, o VS Code ou o agente de conciliação que você está construindo.
- **Client** é o componente, dentro do host, que mantém a conexão com um server. Um client por server.
- **Server** é o programa que expõe as capacidades de um sistema. "O MCP do ERP" é um server.

O server pode rodar **local**, na máquina do usuário, conversando por stdio (entrada e saída padrão do processo). Ou **remoto**, na internet, por HTTP, atendendo muitos clients ao mesmo tempo. Para um sistema corporativo, quase sempre é remoto.

Um server expõe três tipos de coisa, e o que diferencia cada uma é **quem decide usar**:

| Primitiva | O que é | Quem decide usar | Exemplo na conciliação |
|---|---|---|---|
| **Tools** | Ações | O modelo | `conciliacao_propor` |
| **Resources** | Dados para leitura | A aplicação | O contrato do cliente, com as regras de repasse |
| **Prompts** | Instruções prontas | O usuário | "Fechamento mensal", escolhido num menu |

Essa terceira coluna é a que eu demorei para valorizar. Ela responde uma pergunta de desenho: o modelo deve poder decidir sozinho buscar isso, ou é a aplicação que sempre anexa? O contrato do cliente, por exemplo, não precisa ser uma tool. Ele é contexto fixo de qualquer conciliação daquele cliente, então a aplicação anexa como resource e o modelo não gasta um passo do loop pedindo.

<figure>
  <img src="{{ '/assets/images/desafio-fde/dia-06-arquitetura-mcp.svg' | relative_url }}" alt="Diagrama de arquitetura. À esquerda, o host, que é o agente de conciliação, contém o modelo, que escolhe qual tool chamar, e três clients: conexão com o banco, com o ERP e com o time. Cada client se liga a um server à direita. O server do banco, remoto por HTTP, expõe a tool banco_buscar_lancamentos e o resource extrato do mês, sem prompts. O server do ERP, remoto por HTTP, expõe as tools erp_buscar_cobrancas e conciliacao_propor, o resource contrato e regras, e o prompt fechamento mensal. O server da fila de exceções, local por stdio, expõe a tool humano_escalar_excecao e o resource fila em aberto, sem prompts. A legenda indica que tools são decididas pelo modelo, resources pela aplicação e prompts pelo usuário.">
  <figcaption>O caso da conciliação ligado por MCP. Repare que nem tudo virou tool.</figcaption>
</figure>

A conversa entre client e server segue um roteiro fixo. O client pergunta o que existe (`tools/list`). O server devolve a lista com nome, descrição e esquema de cada tool. Essa lista entra na janela do modelo. Quando o modelo decide usar uma, o client manda `tools/call` com os argumentos e devolve o resultado para o modelo.

Dois detalhes da especificação que valem para qualquer tool, com MCP ou sem:

**Existem dois tipos de erro.** Erro de protocolo (tool que não existe, pedido malformado) volta como erro técnico, e o modelo dificilmente resolve. Erro de execução (data no formato errado, valor fora do limite, regra de negócio violada) volta como resultado normal marcado com `isError: true`, e a especificação diz que ele deve ser mostrado ao modelo justamente para que ele se corrija. É o "erro que ensina" da seção 2, escrito em norma.

**As tools podem declarar como se comportam.** São as *annotations*: `readOnlyHint` (só lê), `destructiveHint` (pode apagar ou sobrescrever), `idempotentHint` (repetir não muda nada) e `openWorldHint` (fala com o mundo externo). O host pode usar isso para pedir confirmação antes de uma escrita, ou liberar leituras sem perguntar. Mas o nome diz tudo: são dicas. A especificação manda o client tratar essas anotações como não confiáveis, a menos que o server seja confiável. Um server mal-intencionado pode marcar como "só leitura" uma tool que apaga dados.

## 6. O conector que vai para o próximo cliente

No Dia 1 eu falei do *gravel road → paved highway*: o primeiro deployment é estrada de terra, aberta na mão, e o que funciona vira estrada asfaltada para os próximos. O MCP é esse mecanismo na camada de integração.

Um server MCP para um ERP específico, construído no primeiro cliente, serve para os próximos clientes que usam o mesmo ERP. O trabalho mais chato de qualquer projeto, que é entender a autenticação, a paginação e os campos estranhos daquele sistema, fica feito.

Mas só fica reutilizável se você separar bem o que é do sistema e o que é do cliente. Pela minha experiência até aqui, a regra é esta:

- **No server:** tudo que é do sistema. Como autenticar, como paginar, como traduzir os campos do ERP para nomes legíveis, como devolver erro.
- **Em resources e configuração:** tudo que é do cliente. As regras de repasse, a tolerância de divergência, o valor acima do qual precisa de aprovação humana.

Se a regra "acima de R$ 500 vai para humano" estiver escrita dentro da tool, o server nasce com cara de um cliente só. Se ela vier de um resource, o mesmo server atende o próximo cliente com outra regra.

## 7. Quando eu não usaria MCP

MCP não é obrigatório, e às vezes atrapalha.

| Faz sentido usar MCP quando | Faz sentido usar API direta quando |
|---|---|
| A mesma integração vai servir a vários agentes, ferramentas ou clientes | É um passo fixo de um workflow determinístico |
| Você quer que o modelo descubra as capacidades e escolha qual usar | Desempenho e controle total pesam mais que flexibilidade |
| Há chance real de reaproveitar entre contas | É uma integração única, sem perspectiva de reúso |

Volte no graph do Dia 5. O nó de match determinístico, que concilia o PIX simples por valor, CNPJ e data, não tem modelo nenhum. É código chamando a API do banco. Colocar MCP ali é acrescentar uma camada para nada. O MCP entra onde o modelo escolhe o que chamar: no agente conciliador, no ramo de exceções.

E existe um custo que aparece quando o número de tools cresce. Cada tool conectada é uma descrição na janela, em toda chamada. A Anthropic escreveu sobre isso em *Code execution with MCP*: com muitos servers conectados, só as definições das tools podem ocupar centenas de milhares de tokens antes de o agente ler o pedido. A alternativa que eles testaram foi apresentar os servers como código que o agente importa sob demanda, e no exemplo do artigo o consumo caiu de 150 mil para 2 mil tokens. Tem contrapartida: executar código gerado pelo agente exige sandbox, limite de recurso e monitoramento.

Para um processo estreito como conciliação, com três ou quatro tools, não chega a ser problema. Mas é bom saber que o custo existe antes de sair conectando vinte servers num agente só.

## 8. A parte de segurança que quase ninguém olha

A especificação é bem direta: deve sempre existir um humano capaz de negar uma chamada de tool. O host deve deixar claro quais tools estão expostas ao modelo e pedir confirmação antes de uma operação sensível. Do lado do server, é obrigatório validar toda entrada, controlar acesso, limitar a taxa de chamadas e sanitizar o que sai.

O conceito que mais me ajudou a enxergar o risco é a *lethal trifecta*, do Simon Willison. O perigo aparece quando um mesmo agente tem acesso a dados privados, lê conteúdo que veio de fora e consegue mandar informação para fora. Com as três coisas juntas, um texto malicioso escondido no conteúdo externo pode instruir o agente a vazar os dados. E o Willison chama atenção para o ponto que pega o MCP em cheio: o risco cresce quando a gente mistura servers à vontade, porque cada um traz um pedaço do trio.

Na conciliação isso é concreto. O campo de descrição de um PIX é escrito por quem pagou. Qualquer pessoa que transfere um centavo para a conta do cliente consegue colocar texto dentro da janela do agente. Se esse mesmo agente pudesse enviar e-mail, o trio estaria completo.

O desenho que eu sigo:

- O agente de conciliação não tem nenhuma tool que mande informação para fora. A tool de escalonamento só grava na fila interna.
- Tool de escrita propõe e não efetiva. Efetivar passa pelo validador do Dia 5 e, acima do limite, por uma pessoa.
- As permissões do server espelham as do usuário no sistema de origem. O agente não enxerga nada que quem o opera não enxergaria.

## Prática: as três tools da conciliação, no formato do protocolo

No Dia 4 eu listei três tools para o agente de conciliação. Hoje elas viram contrato. Para cada uma, primeiro a ficha que eu entregaria para um dev, depois como ela aparece no MCP.

### Tool 1 · Leitura: `banco_buscar_lancamentos`

A ficha:

```text
nome:        banco_buscar_lancamentos
o que faz:   retorna os recebimentos de uma conta num período
quando usar: sempre antes de propor uma conciliação
quando NÃO:  para buscar cobranças emitidas (use erp_buscar_cobrancas)
parâmetros:  conta (texto, obrigatório) · de (data, obrigatório)
             · ate (data, obrigatório, no máximo 31 dias depois de "de")
             · status (pendente | conciliado | todos; padrão: pendente)
             · cursor (texto, opcional)
retorna:     lista de {id, data, valor, pagador_nome, pagador_doc, descricao, status},
             no máximo 100 itens, com cursor para a próxima página
erros:       CONTA_INEXISTENTE · PERIODO_INVALIDO · LIMITE_EXCEDIDO
tipo:        leitura · idempotente: sim
```

Como o server a declara, na resposta a `tools/list`:

```json
{
  "name": "banco_buscar_lancamentos",
  "title": "Buscar recebimentos no extrato",
  "description": "Retorna os recebimentos de uma conta bancária num período de até 31 dias. Use sempre antes de propor uma conciliação. Não use para buscar cobranças emitidas; para isso existe erp_buscar_cobrancas. Devolve no máximo 100 itens; se houver mais, chame de novo com o cursor recebido.",
  "inputSchema": {
    "type": "object",
    "properties": {
      "conta":  { "type": "string", "description": "Apelido da conta, por exemplo 'recebimentos-principal'" },
      "de":     { "type": "string", "format": "date", "description": "Início do período, AAAA-MM-DD" },
      "ate":    { "type": "string", "format": "date", "description": "Fim do período, AAAA-MM-DD, no máximo 31 dias depois de 'de'" },
      "status": { "type": "string", "enum": ["pendente", "conciliado", "todos"], "default": "pendente" },
      "cursor": { "type": "string", "description": "Cursor devolvido pela chamada anterior" }
    },
    "required": ["conta", "de", "ate"],
    "additionalProperties": false
  },
  "annotations": { "readOnlyHint": true, "openWorldHint": false }
}
```

O pedido que o client manda quando o modelo decide usar:

```json
{
  "jsonrpc": "2.0",
  "id": 7,
  "method": "tools/call",
  "params": {
    "name": "banco_buscar_lancamentos",
    "arguments": { "conta": "recebimentos-principal", "de": "2026-10-01", "ate": "2026-10-31" }
  }
}
```

A resposta, com o resultado estruturado:

```json
{
  "jsonrpc": "2.0",
  "id": 7,
  "result": {
    "content": [{ "type": "text", "text": "2 recebimentos pendentes em out/2026 (mostrando 2 de 2)." }],
    "structuredContent": {
      "itens": [
        { "id": "lanc_8812", "data": "2026-10-14", "valor": 4850.00,
          "pagador_nome": "Comercial Andrade Ltda", "pagador_doc": "12.345.678/0001-90",
          "descricao": "NF 3391 parcela 2/3", "status": "pendente" },
        { "id": "lanc_8817", "data": "2026-10-15", "valor": 1200.00,
          "pagador_nome": "M. Ferreira", "pagador_doc": null,
          "descricao": "PIX", "status": "pendente" }
      ],
      "proximo_cursor": null
    },
    "isError": false
  }
}
```

E o erro, quando o modelo pede um trimestre inteiro:

```json
{
  "jsonrpc": "2.0",
  "id": 8,
  "result": {
    "content": [{ "type": "text", "text": "PERIODO_INVALIDO: o intervalo pedido tem 92 dias e o máximo é 31. Divida em consultas mensais, por exemplo de 2026-08-01 a 2026-08-31." }],
    "isError": true
  }
}
```

Repare que o erro volta dentro de `result`, como texto para o modelo ler. Ele não derruba nada. Na próxima volta do loop o agente lê, divide o período em três e segue.

(Os dados acima são fictícios.)

### Tool 2 · Escrita: `conciliacao_propor`

No Dia 4 ela se chamava `propor_conciliacao`. Com sobrenome, fica no padrão das outras. E continua com a mesma regra: ela **propõe** e não efetiva.

```text
nome:          conciliacao_propor
o que faz:     registra a proposta de casar um recebimento com uma cobrança
quando usar:   depois de buscar o recebimento e a cobrança e conferir valor e pagador
quando NÃO:    para casos com divergência de valor acima da tolerância do cliente
               (use humano_escalar_excecao)
parâmetros:    lancamento_id (obrigatório) · cobranca_id (obrigatório)
               · valor_conciliado (obrigatório) · justificativa (texto curto, obrigatório)
retorna:       {proposta_id, status: "criada" | "ja_existente", requer_aprovacao}
idempotência:  chave montada pelo servidor = lancamento_id + cobranca_id;
               se a chave já existe, devolve a mesma proposta com status "ja_existente"
aprovação:     requer_aprovacao = true quando valor_conciliado passa do limite
               definido no resource de regras do cliente
erros:         LANCAMENTO_JA_CONCILIADO · VALOR_DIVERGENTE · COBRANCA_INEXISTENTE
tipo:          escrita aditiva · idempotente: sim
```

```json
{
  "name": "conciliacao_propor",
  "title": "Propor conciliação",
  "description": "Registra a proposta de casar um recebimento do extrato com uma cobrança do ERP. Não efetiva a baixa: a proposta passa por validação e, acima do limite do cliente, por aprovação humana. Chamar de novo com o mesmo lancamento_id e cobranca_id devolve a proposta já existente, sem duplicar. Se o valor divergir além da tolerância, não use esta tool; escale com humano_escalar_excecao.",
  "inputSchema": {
    "type": "object",
    "properties": {
      "lancamento_id":    { "type": "string", "description": "id vindo de banco_buscar_lancamentos" },
      "cobranca_id":      { "type": "string", "description": "id vindo de erp_buscar_cobrancas" },
      "valor_conciliado": { "type": "number", "description": "Valor em reais, ponto como separador decimal" },
      "justificativa":    { "type": "string", "maxLength": 280, "description": "Por que este casamento: o que bateu e o que não bateu" }
    },
    "required": ["lancamento_id", "cobranca_id", "valor_conciliado", "justificativa"],
    "additionalProperties": false
  },
  "annotations": { "readOnlyHint": false, "destructiveHint": false, "idempotentHint": true, "openWorldHint": false }
}
```

A chave de idempotência não aparece entre os parâmetros, e isso é proposital. O server monta a chave a partir de `lancamento_id` e `cobranca_id`. O modelo não tem como errar uma coisa que não é ele que faz.

O que volta numa repetição:

```json
{
  "content": [{ "type": "text", "text": "Proposta P-101 já existia para lanc_8812 + cob_0457. Nada foi gravado de novo." }],
  "structuredContent": { "proposta_id": "P-101", "status": "ja_existente", "requer_aprovacao": false },
  "isError": false
}
```

### Tool 3 · Escalonamento: `humano_escalar_excecao`

```text
nome:        humano_escalar_excecao
o que faz:   abre um item na fila de exceções do time financeiro
quando usar: cliente sem cadastro, divergência acima da tolerância, pagamento
             agrupado que não fecha, ou loop improdutivo (critério 4 do Dia 5)
para quem:   fila do time financeiro; acima do limite de valor do cliente,
             também notifica a coordenação
com que dados: lancamento_id · motivo (enum) · resumo do caso (até 500 caracteres)
             · o que já foi tentado (lista de tools chamadas e o que voltou)
prazo:       resposta em 1 dia útil; acima do limite, no mesmo dia
retorna:     {excecao_id, posicao_na_fila, prazo}
tipo:        escrita interna · idempotente: sim (um item aberto por lancamento_id + motivo)
```

```json
{
  "name": "humano_escalar_excecao",
  "title": "Escalar para o time financeiro",
  "description": "Abre um item na fila de exceções para uma pessoa decidir. Use quando o caso não pode ser resolvido com segurança: cliente sem cadastro, divergência acima da tolerância, pagamento agrupado que não fecha, ou depois de três tentativas sem progresso no mesmo lançamento. Inclua o que já foi tentado, para quem pegar o caso não começar do zero. Escalar é barato; conciliar errado é caro.",
  "inputSchema": {
    "type": "object",
    "properties": {
      "lancamento_id": { "type": "string" },
      "motivo": { "type": "string", "enum": ["sem_cadastro", "divergencia_valor", "pagamento_agrupado", "sem_progresso", "outro"] },
      "resumo": { "type": "string", "maxLength": 500 },
      "tentativas": {
        "type": "array",
        "items": { "type": "object", "properties": { "tool": { "type": "string" }, "resultado": { "type": "string" } } }
      }
    },
    "required": ["lancamento_id", "motivo", "resumo"],
    "additionalProperties": false
  },
  "annotations": { "readOnlyHint": false, "destructiveHint": false, "idempotentHint": true, "openWorldHint": false }
}
```

A última frase da descrição, "escalar é barato; conciliar errado é caro", é a mesma do system prompt do Dia 4. Ela aparece nos dois lugares de propósito. É a frase que diz ao modelo para que lado errar quando ele não tiver certeza.

### Para fazer com o seu processo

Pegue o processo que você vem usando desde o Dia 3 e escreva três fichas, uma de cada tipo. Para a de escrita, responda duas perguntas antes de qualquer outra: qual é a chave de idempotência, e ela vem do negócio ou é sorteada? Para a de escalonamento, escreva para quem vai, com que dados e em quanto tempo alguém responde. Se você não sabe o prazo, descobriu uma conversa que precisa ter com o cliente antes de escrever código.

Depois, passe as três fichas para alguém que não conhece o caso e peça para essa pessoa dizer, sem te perguntar nada, quando usaria cada uma. Onde ela hesitar, o modelo também vai hesitar.

## TLDR

- **A tool é um contrato entre um sistema determinístico e um agente que não é.** Escreva para quem vai ler poucas linhas de texto e decidir sozinho.
- **Poucas tools no nível da tarefa**, com prefixo por sistema, retorno enxuto e legível, paginação e cursor.
- **A descrição é um prompt.** Diga quando usar e quando não usar.
- **Erro bom ensina o próximo passo.** No MCP, erro de execução volta como resultado com `isError: true` para o modelo se corrigir.
- **Classifique antes de escrever:** leitura, escrita ou escalonamento. O risco está na escrita.
- **Escrita precisa de idempotência.** Com agente, prefira chave derivada do negócio e montada pelo servidor, porque a repetição pode vir do próprio modelo.
- **MCP transforma multiplicação em soma:** cada lado implementa o protocolo uma vez. Aberto em 2024, adotado pela OpenAI em 2025, hoje na Linux Foundation.
- **Host, client, server.** Tools o modelo decide usar, resources a aplicação anexa, prompts o usuário escolhe. Nem tudo precisa ser tool.
- **Annotations são dicas, não garantia.** O client deve desconfiar delas se o server não for confiável.
- **O server MCP é o caminho para reaproveitar no próximo cliente**, desde que as regras do cliente fiquem em resources e configuração, fora da tool.
- **API direta continua certa** para passo fixo de workflow determinístico. E muitas tools conectadas custam token em toda chamada.
- **Cuidado com o trio de dados privados, conteúdo externo e saída para fora.** Na conciliação, a descrição do PIX é conteúdo externo.

## Fontes

**Leituras**

- [Writing effective tools for agents, with agents — Anthropic](https://www.anthropic.com/engineering/writing-tools-for-agents) — a definição de tool como contrato, consolidar em vez de embrulhar endpoints, namespacing, retorno com significado (206 contra 72 tokens no exemplo do Slack), o limite de 25 mil tokens do Claude Code e erros acionáveis.
- [What is the Model Context Protocol? — documentação oficial](https://modelcontextprotocol.io/docs/getting-started/intro) — a comparação com o USB-C e o ecossistema de clients.
- [Architecture overview — documentação oficial do MCP](https://modelcontextprotocol.io/docs/learn/architecture) — host, client, server, transportes local e remoto e as três primitivas.
- [Especificação do MCP: Tools (versão 2026-07-28)](https://modelcontextprotocol.io/specification/latest/server/tools) — `tools/list` e `tools/call`, os dois tipos de erro, anotações como não confiáveis, colisão de nomes e as considerações de segurança.
- [Introducing the Model Context Protocol — Anthropic](https://www.anthropic.com/news/model-context-protocol) — o anúncio de novembro de 2024.
- [OpenAI adopts rival Anthropic's standard for connecting AI models to data — TechCrunch](https://techcrunch.com/2025/03/26/openai-adopts-rival-anthropics-standard-for-connecting-ai-models-to-data/) — a adoção pela OpenAI em março de 2025.
- [Donating the Model Context Protocol and establishing the Agentic AI Foundation — Anthropic](https://www.anthropic.com/news/donating-the-model-context-protocol-and-establishing-of-the-agentic-ai-foundation) — a doação para a Linux Foundation em dezembro de 2025.
- [Code execution with MCP — Anthropic](https://www.anthropic.com/engineering/code-execution-with-mcp) — o custo das definições de tools na janela e a queda de 150 mil para 2 mil tokens no exemplo.
- [Idempotent requests — Stripe API](https://docs.stripe.com/api/idempotent_requests) — como uma API de pagamentos implementa chave de idempotência na prática.
- [The lethal trifecta for AI agents — Simon Willison](https://simonwillison.net/2025/Jun/16/the-lethal-trifecta/) — dados privados, conteúdo não confiável e comunicação externa no mesmo agente.
- [O que é MCP? — IBM, em português](https://www.ibm.com/br-pt/think/topics/model-context-protocol) — uma boa revisão do conceito para quem prefere ler em português.

**Vídeo**

- [What is MCP? Integrate AI Agents with Databases & APIs — IBM Technology](https://www.youtube.com/watch?v=eur8dUO9mvE) — dez minutos, bom para ver a arquitetura desenhada.

**[→ Acompanhe os próximos artigos](/blog/)**
