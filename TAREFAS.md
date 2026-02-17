# Tarefas do Analisador LEAF

Ordem sugerida e razão de cada passo. **Não inclui código** — só o que fazer e porquê.

---

## Porque foi esta a abordagem escolhida

- **Responsabilidade única por ficheiro:** Cada módulo faz uma coisa bem definida. Assim, quando algo falha ou quando queres alterar comportamento (ex.: mudar o limiar de ERROR, ou acrescentar export CSV), sabes onde mexer. O enunciado pede modularidade e manutenibilidade — esta divisão responde a isso.

- **Contratos claros entre módulos:** O reader devolve “linhas”; o parser recebe linhas e devolve “entradas de log ou inválido”; o reporter recebe “entradas + estatísticas” e imprime. Ninguém mistura leitura de disco com regex nem com impressão. Isso facilita testes (podes testar o parser com strings sem abrir ficheiros) e futuras extensões (um novo exportador usa as mesmas entradas que o reporter).

- **Eficiência desde o início:** O enunciado diz para pensar em ficheiros de 1GB. Por isso a escolha é: o reader não carrega o ficheiro todo; entrega uma linha de cada vez. O main (ou quem orquestra) processa linha a linha e só guarda o estritamente necessário para o relatório final (ex.: contagens e talvez a lista de entradas, ou só as contagens se o relatório for só números). Assim, a abordagem é “stream” em vez de “load all”.

- **Resiliência no sítio certo:** O parser é o único que sabe o que é uma linha LEAF válida. Tudo o que não encaixa no padrão é tratado como inválido — sem exceções não tratadas, sem crash. O reader só trata erros de I/O (ficheiro não existe, sem permissão). Separação clara: I/O no reader, formato no parser.

- **Escalabilidade para export:** O relatório em texto é “uma forma de ver” os dados. Se amanhã precisares de JSON ou CSV, não queres duplicar parsing nem leitura; queres reutilizar o mesmo parser e a mesma estrutura de dados e só mudar “quem escreve o resultado”. Por isso os modelos de dados ficam num módulo à parte e o reporter é “apenas” um consumidor desses dados — outros consumidores podem ser adicionados sem refazer o resto.

- **Testabilidade:** Parser e (em parte) reporter podem ser testados sem ficheiros reais: passas strings ao parser; passas listas de entradas ao reporter. O reader pode ser testado com um ficheiro temporário pequeno. O main orquestra e tem pouca lógica própria, o que reduz a superfície de testes na parte mais difícil de isolar (CLI).

---

## Que tipo de código cada ficheiro deve ter (e porquê)

### `config`

**Tipo de código:** Apenas constantes (valores numéricos, strings fixas) definidas ao nível do módulo. Nada de funções que façam cálculos ou I/O. Opcionalmente, uma única estrutura ou objeto que agrupe todas as opções, para um único ponto de configuração.

**Porquê:** Este ficheiro existe para centralizar o que é “configurável”. O limiar de alerta de ERROR, por exemplo, deve ser um nome legível (ex.: LIMIAR_ALERTA_ERROR) e um valor. Qualquer módulo que precise desse valor importa daqui. Assim, não há números ou strings mágicas espalhadas no parser ou no reporter; quando o enunciado diz “alerta se ERROR > X”, o X vive aqui e o resto do código referencia o nome, não o número.

---

### `models`

**Tipo de código:** Definições de tipos de dados: uma estrutura que representa uma entrada de log (com campos para timestamp, level, source, mensagem) e, se fizer sentido na linguagem que usas, uma enumeração ou conjunto fixo de valores para o nível (INFO, WARN, ERROR, DEBUG). Pode incluir uma forma de criar essa estrutura a partir de valores individuais. Nada de lógica de negócio (sem parsing, sem impressão, sem leitura de ficheiros).

**Porquê:** O parser produz “uma entrada de log”; o reporter (e futuros exportadores) consomem “entradas de log”. Se esse conceito for um tipo explícito, o contrato fica claro e a linguagem pode ajudar a evitar erros (ex.: não passar uma string onde se espera uma entrada). Toda a lógica de “o que é uma linha válida” fica no parser; aqui fica só “como representamos essa linha em memória”. Isso facilita export para JSON/CSV porque já tens uma estrutura bem definida para serializar.

---

### `parser`

**Tipo de código:** Funções puras (ou quase puras) que recebem uma string (uma linha) e devolvem ou uma entrada de log (tipo definido em `models`) ou um indicador de “linha inválida”. A extração dos campos deve usar expressões regulares para identificar o padrão LEAF (timestamp entre parêntesis rectos, LEVEL:, SRC:, MSG:). Pode haver uma função principal “parse_line” e, se quiseres, funções auxiliares para validar ou normalizar a data. Nenhuma leitura de ficheiros nem impressão para o terminal; no máximo logging interno de debug, se necessário.

**Porquê:** O parser é o único que sabe o formato LEAF. Concentrar aqui toda a lógica de regex e validação evita duplicação e garante que “lixo” (comentários, linhas vazias, malformadas) é tratado num único sítio: devolves “inválido” e quem chama conta. O critério de avaliação “domínio de regex” aplica-se aqui. Manter o parser sem I/O permite testá-lo facilmente com listas de strings, incluindo casos de borda (linha vazia, linha sem data, etc.).

---

### `reader`

**Tipo de código:** Funções que recebem um caminho de ficheiro e: (1) verificam se o ficheiro existe; (2) verificam se há permissão de leitura (conforme a linguagem); (3) abrem o ficheiro e produzem as linhas uma a uma (iterador/gerador/stream), sem carregar o ficheiro inteiro na memória. Tratamento de erros de I/O com mensagens claras (ficheiro não encontrado, permissão negada). Este módulo não deve conhecer o formato LEAF nem fazer parsing; só devolve linhas de texto.

**Porquê:** O requisito de eficiência (ficheiros grandes) exige leitura por linha. O reader é o único que toca no sistema de ficheiros; assim, falhas de I/O são tratadas aqui e o parser não precisa de try/catch de disco. Separar “como leio” de “como interpreto” permite trocar a fonte no futuro (ex.: ler de stdin ou de um URL) sem mexer no parser. O enunciado pede explicitamente validação de existência e permissões — este é o sítio certo.

---

### `reporter`

**Tipo de código:** Funções que recebem dados já processados: total de linhas lidas, número de linhas inválidas, contagens por nível (e, se necessário, a lista de entradas ou apenas as contagens). A partir disso, imprimem no terminal o relatório final: totais, contagem por nível (ex.: “3 Errors, 5 Infos…”), e o alerta se o número de ERROR for superior ao limiar definido em `config`. Nada de abrir ficheiros nem de fazer regex; apenas formatação de texto e impressão. A decisão “devo mostrar alerta?” usa o limiar importado de `config`.

**Porquê:** A “apresentação” do resultado fica isolada. Se amanhã quiseres outro formato de relatório (ex.: mais compacto ou em outro idioma), alteras só aqui. Se quiseres export para JSON/CSV, crias outro módulo que recebe os mesmos dados e escreve noutro sítio — o reporter continua a ser só “saída em texto no terminal”. Usar o limiar de `config` mantém a regra “alerta se ERROR > X” num único sítio (config) e a decisão de “mostrar ou não” no reporter, que é quem conhece a contagem.

---

### `main`

**Tipo de código:** Ponto de entrada do programa: leitura dos argumentos da linha de comandos (caminho do ficheiro), validação mínima (ex.: argumento presente). Depois, orquestração em sequência: chamar o reader para obter as linhas; para cada linha, chamar o parser; acumular entradas válidas e contar linhas inválidas e totais; no fim, chamar o reporter com esses dados. Tratamento de erros que venham do reader (ficheiro inexistente ou sem permissão): sair com mensagem clara, sem crash. Nada de regex, nem de fórmulas de contagem complexas, nem de formatação do relatório — só fluxo e chamadas entre módulos.

**Porquê:** O main existe para “ligar” os outros módulos. Se contiver pouca lógica própria, é fácil de ler e de alterar (ex.: no futuro acrescentar um argumento para “modo quiet” ou “output em JSON”). A validação de argumentos e o tratamento de erros de I/O aqui ou no reader garantem que o utilizador recebe feedback claro quando algo falha. O enunciado exige que o caminho do ficheiro seja passado como argumento — isso é feito aqui ao ler a linha de comandos.

**Resumo por ficheiro:**

| Ficheiro   | Tipo de código                                      | Não deve ter                          |
|------------|-----------------------------------------------------|---------------------------------------|
| config     | Constantes (números, strings)                       | Funções, I/O, parsing                 |
| models     | Estruturas de dados, enumerações de níveis          | Lógica de negócio, regex, I/O          |
| parser     | Funções puras + regex que devolvem entrada ou inválido | Leitura de ficheiros, impressão   |
| reader     | Validação de ficheiro + iteração linha a linha      | Conhecimento do formato LEAF, parsing |
| reporter   | Formatação e impressão do relatório no terminal     | Leitura de ficheiros, regex           |
| main       | Argumentos CLI + orquestração (chamar reader→parser→reporter) | Regex, contagens complexas, formatação |

---

## 1. Configuração (`config`)

**O que fazer:** Definir constantes que o resto do projeto usa (ex.: limiar X para alerta de ERROR, extensão esperada `.leaf` se quiseres validar).

**Raciocínio:** Evita “números mágicos” espalhados no código. Quando o avaliador disser “alerta se ERROR > X”, o X deve vir daqui. Assim, amanhã podes mudar só neste sítio.

---

## 2. Modelos de dados (`models`)

**O que fazer:** Definir a estrutura de uma “entrada de log” válida (timestamp, level, source/componente, mensagem) e, se fizer sentido, os níveis possíveis (INFO, WARN, ERROR, DEBUG).

**Raciocínio:** O parser produz “algo” e o reporter consome “algo”. Se esse “algo” for um tipo bem definido (estrutura/classe), o contrato fica claro: o parser devolve uma lista (ou iterador) desses objetos; o reporter recebe essa lista e as estatísticas. Assim, se amanhã quiseres exportar para JSON/CSV, já tens a estrutura de dados pronta.

---

## 3. Parser (`parser`)

**O que fazer:** Receber uma linha de texto e decidir se é uma linha LEAF válida. Se for, extrair timestamp, level, source e mensagem e devolver um objeto do tipo definido em `models`; se não for, devolver nada (ou sinalizar que a linha é inválida). Usar regex para capturar o padrão. Tratar os dois formatos de data que aparecem no ficheiro de teste (YYYY-MM-DD e DD/MM/YYYY) se o enunciado o permitir.

**Raciocínio:** O parsing é o coração do projeto e não depende de ficheiros nem de impressão. Podes testar só com strings. Tudo o que é “lixo” (comentários, linhas vazias, linhas malformadas) deve ser tratado aqui: não crashar, apenas considerar a linha inválida e deixar quem chama contar quantas dessas houve. O critério “domínio de regex” é avaliado aqui.

---

## 4. Leitor de ficheiro (`reader`)

**O que fazer:** Receber um caminho de ficheiro. Validar se o ficheiro existe e se tens permissão de leitura; em caso contrário, sair com mensagem clara (sem crash). Ler o ficheiro **linha a linha** (stream/iteração), não carregar o ficheiro todo na memória.

**Raciocínio:** O requisito de eficiência (pensar em ficheiros de 1GB) implica não fazer “ler tudo para uma lista de linhas”. Cada linha deve ser lida, passada ao parser e descartada da memória antes da próxima. A validação de existência e permissões cumpre o requisito de tratamento de erros.

---

## 5. Reporter (`reporter`)

**O que fazer:** Receber o total de linhas lidas, o número de linhas inválidas, as entradas de log válidas (ou apenas as contagens por nível) e produzir o relatório no terminal: total de linhas, contagem por nível (ex.: 3 Errors, 5 Infos…), e alerta se o número de ERROR for maior que o limiar X (usar o valor definido em `config`).

**Raciocínio:** Separa “o que calcular” da “lógica de parsing” e da “leitura”. Assim, amanhã podes acrescentar outro “reporter” (ex.: escrever JSON/CSV) sem mexer no parser nem no reader. A lógica de contagem pode ficar aqui ou num sítio pequeno partilhado; o importante é que a impressão do relatório final fique concentrada neste módulo.

---

## 6. Ponto de entrada CLI (`main`)

**O que fazer:** Ler o argumento da linha de comandos (caminho do ficheiro `.leaf`). Orquestrar: chamar o reader para obter as linhas, para cada linha chamar o parser, acumular resultados válidos e contar linhas inválidas; no fim, chamar o reporter com totais e contagens. Tratar erros de ficheiro (não existe, sem permissão) antes de começar o processamento.

**Raciocínio:** O `main` não deve conter regex nem fórmulas de contagem; só fluxo e chamadas aos outros módulos. Assim, o projeto fica modular e fácil de manter. O requisito “receber o caminho como argumento” é cumprido aqui.

---

## 7. Documentação e entrega

**O que fazer:** Preencher o README com o comando exato para correr a aplicação e um exemplo usando um ficheiro em `data/` (ex.: `data/system_audit.log`). Fazer commits com mensagens claras (por funcionalidade ou por módulo) e usar branches se o enunciado o exigir.

**Raciocínio:** O enunciado pede README a explicar como correr e uso correto de Git. A entrega é avaliada pelo estado do repositório na data/hora indicada.

---

## Ordem resumida

| # | Módulo   | Porquê nesta ordem |
|---|----------|--------------------|
| 1 | config   | Nada depende de I/O; outros módulos usam constantes daqui. |
| 2 | models   | Parser e reporter precisam da mesma “forma” de dados. |
| 3 | parser   | Lógica pura; testável com strings; não depende de ficheiros. |
| 4 | reader   | Fornece linhas ao parser; desacoplado do formato LEAF. |
| 5 | reporter | Usa resultados do parser + config; não lê disco. |
| 6 | main     | Junta tudo; depende de reader, parser, reporter e config. |
| 7 | README/Git | Entrega e avaliação. |

---

## Dica de escalabilidade

Quando implementares, pensa: “Se amanhã me pedirem export para JSON ou CSV, onde é que mexo?”. Se a resposta for “só crio um novo módulo que recebe a mesma lista de entradas e escreve noutro formato”, a estrutura está boa. O `reporter` atual só “reporta” em texto no terminal; um `exporter_json` ou `exporter_csv` seria outro consumidor dos mesmos dados — daí a importância de `models` e de o parser devolver estruturas bem definidas.

---

## Divisão por tarefas (tickets / PRs no Git)

Cada item abaixo é um **ticket** (issue) e um **PR** (branch → implementação → merge). A ordem respeita dependências entre módulos.

---

### Ticket 1 — Config: constantes e limiar de alerta

**Branch sugerida:** `feat/config`  
**Ficheiros:** `src/config.py`

**Conteúdo:** Implementar o módulo de configuração (limiar X para alerta de ERROR, e outras constantes que uses).

**Porquê um PR só para isto:** O config não depende de mais nada e é usado por reporter (e opcionalmente main). É uma alteração pequena e isolada; o revisor vê só “quais são os valores oficiais do projeto”. Merge primeiro para os próximos PRs poderem importar o limiar sem conflitos.

---

### Ticket 2 — Models: estrutura de entrada de log e níveis

**Branch sugerida:** `feat/models`  
**Ficheiros:** `src/models.py`

**Conteúdo:** Definir a estrutura que representa uma linha de log válida (timestamp, level, source, message) e o conjunto de níveis (INFO, WARN, ERROR, DEBUG).

**Porquê um PR só para isto:** O parser e o reporter dependem deste contrato. Se fizeres o models num PR separado, o revisor pode validar a estrutura de dados antes de ver regex ou relatórios. Qualquer PR seguinte que use “entrada de log” baseia-se neste merge. Evita misturar “definição de dados” com “lógica de parsing” no mesmo diff.

---

### Ticket 3 — Parser: validação e extração de linhas LEAF

**Branch sugerida:** `feat/parser`  
**Ficheiros:** `src/parser.py`  
**Base:** branch com `models` já merged (ex.: `main`).

**Conteúdo:** Implementar a função (ou funções) que recebem uma linha, aplicam regex ao formato LEAF, e devolvem uma entrada do tipo definido em `models` ou “inválido”. Cobrir os formatos de data do enunciado e tratar linhas vazias/comentários/malformadas sem crash.

**Porquê um PR só para isto:** O parser é o núcleo da lógica de negócio e o foco do critério “domínio de regex”. Vale a pena rever isoladamente: só ficheiro de parsing, sem I/O nem impressão. Podes (e deves) ter testes unitários neste PR; o revisor concentra-se em regex e casos de borda. Depende de `models`, por isso o PR abre depois do Ticket 2 merged.

---

### Ticket 4 — Reader: validação de ficheiro e leitura linha a linha

**Branch sugerida:** `feat/reader`  
**Ficheiros:** `src/reader.py`

**Conteúdo:** Implementar a validação de existência e permissões do ficheiro e a leitura em stream (uma linha de cada vez), sem carregar o ficheiro todo na memória.

**Porquê um PR só para isto:** Não depende de `models` nem de `parser`; só de I/O. Pode ser desenvolvido em paralelo ao Ticket 3 (ou depois). Um PR só para o reader mantém o diff focado em “como lemos o ficheiro” e em tratamento de erros de disco, o que facilita a revisão e cumpre o requisito de eficiência de forma explícita no histórico do Git.

---

### Ticket 5 — Reporter: relatório final e alerta de ERROR

**Branch sugerida:** `feat/reporter`  
**Ficheiros:** `src/reporter.py`  
**Base:** `config` (e opcionalmente `models` se passares entradas para formatação) já merged.

**Conteúdo:** Implementar a geração do relatório no terminal: total de linhas, linhas inválidas, contagem por nível, e alerta quando o número de ERROR supera o limiar definido em `config`.

**Porquê um PR só para isto:** Separa “como apresentamos o resultado” do “como lemos” e “como parseamos”. O revisor vê apenas formatação e uso do limiar; não mistura com regex nem com abertura de ficheiros. Depende de `config` (e possivelmente da forma dos dados em `models`), por isso depois dos tickets 1 e 2.

---

### Ticket 6 — Main: CLI e orquestração

**Branch sugerida:** `feat/main-cli`  
**Ficheiros:** `src/main.py`  
**Base:** todas as branches anteriores merged (config, models, parser, reader, reporter).

**Conteúdo:** Implementar o ponto de entrada: ler o argumento (caminho do ficheiro), orquestrar reader → parser → reporter, e tratar erros de I/O com mensagens claras.

**Porquê um PR só para isto:** É o único PR que “liga” todos os módulos. Se o main for pequeno (só fluxo e chamadas), o diff é fácil de rever: confirma-se que não há lógica de negócio duplicada e que os erros de ficheiro são tratados. Deixar para o fim evita conflitos de merge com os outros módulos e permite que cada um seja revisto e testado antes de integrar.

---

### Ticket 7 — Documentação: README e como correr

**Branch sugerida:** `docs/readme`  
**Ficheiros:** `README.md`

**Conteúdo:** Preencher a secção “Como correr” com o comando exato e um exemplo usando um ficheiro em `data/` (ex.: `data/system_audit.log`).

**Porquê um PR só para isto:** A documentação pode ser a última coisa a fechar, quando o comando e o fluxo estão estáveis. Um PR só de docs mantém o histórico limpo ( “adiciona instruções de execução” ) e cumpre o requisito do enunciado sem misturar com código. O revisor pode validar se um utilizador novo consegue correr a aplicação só com o README.

---

### Resumo da ordem e dependências

| Ticket | Branch      | Depende de     | Justificação do PR isolado |
|--------|-------------|----------------|----------------------------|
| 1      | feat/config | —              | Base do projeto; nada depende de I/O. |
| 2      | feat/models | —              | Contrato de dados; parser e reporter dependem disto. |
| 3      | feat/parser | 2 (models)     | Lógica de regex e validação; revisão focada. |
| 4      | feat/reader | —              | I/O e eficiência; independente do formato LEAF. |
| 5      | feat/reporter | 1 (config)    | Apresentação; usa limiar, não parsing. |
| 6      | feat/main-cli | 1,2,3,4,5     | Orquestração; junta tudo sem duplicar lógica. |
| 7      | docs/readme | 6 (opcional)   | Documentação final; requisito explícito do enunciado. |

**Nota:** Os tickets 3 e 4 não dependem um do outro; podes desenvolver parser e reader em paralelo (em branches separadas a partir de `main` depois de 1 e 2 merged) e fazer dois PRs independentes.
