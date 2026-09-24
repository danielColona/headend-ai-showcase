# HeadendAI

**Plataforma de consulta em linguagem natural para infraestrutura de Headend de TV Digital.**
100% on-premise, sem nuvem, rodando em um servidor de 8 GB sem GPU.

O operador pergunta *"quais canais da TS 12 não têm legenda?"* e recebe a resposta
em tabela, com resumo falado — sem abrir a interface do equipamento, sem escrever SQL,
sem depender de nenhum serviço externo.

> Este repositório é a **documentação pública** do projeto. O código e os dados
> operacionais são privados: o sistema consulta a infraestrutura real de um
> operador de TV por assinatura.

---

## Em números

| | |
|---|---|
| **67** intenções de consulta | catálogo declarativo, roteamento determinístico |
| **1.060** testes automatizados | suíte roda em ~86 s, sem tocar em equipamento real |
| **405** perguntas certificadas | validadas contra o sistema real, viram regressão |
| **5** integrações ao vivo | consulta o equipamento no momento da pergunta |
| **~26.000** itens monitorados | via API do Zabbix |
| **~1.700** serviços / **~9.400** PIDs | inventário varrido e correlacionado |
| **70–220 ms** | latência de roteamento de uma pergunta |
| **0 GPU** | LLM local em CPU, e opcional |

---

## Stack

`Python` · `FastAPI` · `SQLite` · `Docker` · `Docker Compose` · `Ollama` ·
`MCP (Model Context Protocol)` · `Svelte` · `Playwright` · `pytest` ·
`GitHub Actions` · `Zabbix API` · `Ubuntu Server`

**Domínio:** MPEG-TS · ISDB-T · multicast · SID/PID/TS-ID · NIT · statmux · QAM · SNMP

---

## A decisão central: o LLM não decide nada

A abordagem óbvia para "pergunta em linguagem natural → consulta" é dar ferramentas
ao LLM e deixá-lo escolher. **Isso foi testado e descartado, com evidência.**

O modelo local disponível para o hardware do projeto (1,7 B de parâmetros, CPU-only,
~4 tokens/s) não produz *function calling* confiável: inventa parâmetros e não emite
chamadas estruturadas de forma consistente. Num contexto de operação de rede, uma
resposta plausível e errada é pior que nenhuma resposta.

A arquitetura inverte a responsabilidade:

```
pergunta do operador
      │
      ▼
┌─────────────────────────────────────────────┐
│  MOTOR DETERMINÍSTICO                       │
│                                             │
│  normalização → sinônimos                   │
│  camada 0.5   → gazetteer de entidades      │
│  roteamento   → regex por intenção          │
│  despacho     → SQL parametrizada           │
│  formatação   → tabela + resumo             │
└─────────────────────────────────────────────┘
      │
      ▼  (opcional, desligável)
   LLM local — só reescreve a resposta pronta
```

**O LLM nunca escolhe a consulta, nunca monta SQL, nunca toca no dado.** Ele recebe
um resultado já correto e, se ligado, o reescreve como frase natural. Desligá-lo
não degrada a resposta — remove apenas o floreio.

Consequência mensurável: o resumo em linguagem natural é montado deterministicamente
em **0,03–0,18 ms**, contra segundos de geração por LLM. É o mesmo texto que a
síntese de voz lê.

---

## Tolerância à fala real do operador

Operador não digita como o equipamento nomeia. Ele escreve *"sportv hd"*, e o
serviço se chama outra coisa, com prefixo e sufixo de origem. Quatro camadas
resolvem isso antes de qualquer regex rodar:

| Camada | O que faz |
|---|---|
| **0 — normalização** | acentos, caixa, 53 grupos de sinônimos |
| **0.5 — gazetteer** | reconhece nome de canal real na frase e o substitui por um marcador, antes do roteamento |
| **1–2 — fuzzy** | similaridade de nome e de frase, com margem mínima de desempate |
| **3 — semântica** | embeddings locais, offline, como último recurso |

A camada 0.5 é a que mais economiza regra: sem ela, cada intenção precisaria
antecipar todas as formas de escrever cada nome.

**O contraponto que isso exige:** um gazetteer agressivo gera falso positivo. Três
guardas gerais foram necessárias — janela contendo palavra de equipamento não vira
nome de canal; nome resolvido não pode ser muito menor que a janela que casou;
janela feita só de vocabulário técnico de stream não vira canal. Cada uma nasceu de
um caso real, e todas são regras gerais verificadas contra o banco inteiro — não
remendos caso a caso.

---

## Três problemas reais, e o que ensinaram

### O resumo que mentia

O motor cortava listas em um teto de segurança; a camada de apresentação tinha um
teto próprio, **menor**. Resultado: a apresentação descartava linhas que o motor
havia entregue — 7% das respostas reais — e o resumo afirmava extremos calculados
sobre o recorte. Chegou a dizer "18,2 Mbps" onde o valor real era 40.

**Aprendizado aplicado:** dois tetos que precisam ser iguais não podem ser duas
constantes — há um teste que reprova se divergirem. E resultado truncado é marcado
como tal, porque quem consome não pode afirmar contagem nem extremo sobre um
recorte.

### O servidor derrubado pela própria medição

Uma varredura de 400 perguntas disparada contra a API de produção por HTTP esgotou
a memória do host, que precisou ser religado. O endpoint era síncrono e nada
cancelava o trabalho de um cliente que já havia desistido: *timeout* do cliente +
próxima pergunta = concorrência sem limite.

**Aprendizado aplicado:** medição em massa roda em processo contra base de teste
versionada, nunca por HTTP contra produção. A API ganhou teto de concorrência, e
todo container ganhou limite de memória — não havia nenhum antes.

### O relatório que chamava de defeito o que era configuração correta

Comparando dois identificadores de transporte entre subsistemas, o sistema acusou
756 divergências. A análise estava aritmeticamente correta e conceitualmente errada:
a norma permite numeração própria por região, e a comparação precisa acontecer
*dentro* de cada contexto, não entre eles.

**Aprendizado aplicado:** a regra de negócio veio do operador, não do dado. Dado
sozinho não diz qual diferença é defeito e qual é intenção — e um relatório confiante
sobre a premissa errada custa mais caro que não ter relatório.

---

## Engenharia de qualidade

**Nenhum teste fala com equipamento real.** Uma fixture desliga acesso à rede por
padrão; base de teste versionada e materializada a cada sessão. Testes que exigem
equipamento são marcados e ficam fora da suíte normal.

**Duas regressões de roteamento, não uma.** Os exemplos do próprio catálogo têm
ponto cego — são escritos por quem acabou de escrever a regra. As 405 perguntas
certificadas são o gabarito duro: já passaram por teste real, com a intenção
registrada. Foi essa segunda regressão que, no dia em que nasceu, detectou uma
intenção nova roubando perguntas de outras duas.

**Teste que não falha não conta.** Uma versão anterior da regressão imprimia
"2 divergências" e saía com código 0. Se um teste não consegue *reprovar*, é script
de inspeção, não teste.

---

## Superfícies

O mesmo motor responde por quatro caminhos, sem duplicação de lógica:

- **API REST** (FastAPI) — o caminho principal
- **Servidor MCP** — integração com clientes que falam Model Context Protocol
- **Interface de chat** — fork de um projeto open source, com autocomplete de
  perguntas certificadas e tabela interativa (ordenar, filtrar, copiar)
- **CLI** — iteração rápida em desenvolvimento

O fork da interface é buildado em CI (o build de ~7 GB derrubava o servidor de
produção) e publicado como imagem de container privada.

---

## Consulta ao vivo: cinco exceções deliberadas

A maior parte das respostas vem de inventário varrido periodicamente. Cinco
intenções consultam o equipamento **no momento da pergunta** — alarmes ativos,
taxa real por porta, status de receptores e moduladores.

Não é uma mudança de arquitetura: o roteamento continua 100% determinístico, só a
busca do dado é ao vivo. Cada exceção foi uma decisão explícita, registrada, com
justificativa — porque generalizar "consulta ao vivo" transformaria cada pergunta
numa dependência de rede.

Duas dessas integrações exigiram **engenharia reversa de protocolo proprietário**,
a partir de captura de tráfego real do software de gerência do fabricante. Durante
esse trabalho ficou evidente que o mesmo endpoint servia leitura e escrita de
configuração, distinguidos por um único campo — uma escrita acidental alteraria
parâmetros de RF ao vivo. O módulo usa exclusivamente as operações confirmadas como
leitura pura, e a regra está documentada para quem mexer depois.

---

## Status

13 sprints concluídas. Sistema em produção, usado por operadores reais, com
varredura mensal automatizada e histórico de decisões documentado sprint a sprint.

---

## Sobre esta documentação

Escrita a partir de um projeto privado real. Números, incidentes e decisões são
verdadeiros; identificadores de rede, nomes de equipamento, inventário e código
operacional foram deliberadamente omitidos.
