# Sistema Bancário com Transações Concorrentes

Aplicação acadêmica em Java 17 com interface desktop Swing para testar contas,
depósitos, saques e transferências. O domínio protege os saldos contra acessos
concorrentes; a persistência JDBC guarda contas e movimentações entre execuções.

## Requisitos para executar

- JDK 17 ou superior;
- Maven 3.9 ou superior;
- ambiente desktop com suporte a Swing.

As dependências FlatLaf, FlatLaf Extras, MigLayout, H2 e JUnit são resolvidas pelo Maven.

## Compilar e executar

No PowerShell, rode o comando abaixo. A opção `-f` aponta diretamente para o
`pom.xml`, então não importa se o terminal abriu em `Downloads` ou dentro da
pasta do projeto. O caminho funciona sem configurar a variável `$mvn`:

```powershell
& "$env:LOCALAPPDATA\Programs\Apache\Maven\apache-maven-3.9.11\bin\mvn.cmd" -f "$env:USERPROFILE\Downloads\Sistema_Banc-rio-main\Sistema_Banc-rio-main\pom.xml" clean package
```

Espere o comando terminar. Durante a execução, o Maven lista cada teste. No
final, `BUILD SUCCESS` significa que compilou e passou; `BUILD FAILURE` indica
que houve um erro e as mensagens acima mostram onde investigar. Neste projeto,
o pacote completo tem 25 testes. Os relatórios detalhados ficam em
`target/surefire-reports`.

Somente depois de ver `BUILD SUCCESS`, execute este comando para abrir a janela
Swing:

```powershell
& "$env:LOCALAPPDATA\Programs\Apache\Maven\apache-maven-3.9.11\bin\mvn.cmd" -f "$env:USERPROFILE\Downloads\Sistema_Banc-rio-main\Sistema_Banc-rio-main\pom.xml" exec:java
```

Esse último comando permanece ativo enquanto a aplicação estiver aberta. Para
encerrá-lo, feche a janela do sistema ou pressione `Ctrl+C` no terminal. Também
é possível abrir o projeto no IntelliJ/Eclipse e executar
`br.com.sistemabancario.app.Main`.

Na primeira execução são criadas duas contas de demonstração. A base H2 fica em
`data/sistema-bancario.mv.db`; as contas e transações permanecem salvas quando
a aplicação é fechada. Para iniciar uma base H2 em memória para uma execução
manual, passe outra URL JDBC ao Maven:

```powershell
& "$env:LOCALAPPDATA\Programs\Apache\Maven\apache-maven-3.9.11\bin\mvn.cmd" -f "$env:USERPROFILE\Downloads\Sistema_Banc-rio-main\Sistema_Banc-rio-main\pom.xml" "-Dsistema.banco.url=jdbc:h2:mem:banco;DB_CLOSE_DELAY=-1" exec:java
```

## Executar os testes

```powershell
& "$env:LOCALAPPDATA\Programs\Apache\Maven\apache-maven-3.9.11\bin\mvn.cmd" -f "$env:USERPROFILE\Downloads\Sistema_Banc-rio-main\Sistema_Banc-rio-main\pom.xml" test
```

Uma classe de teste:

```powershell
& "$env:LOCALAPPDATA\Programs\Apache\Maven\apache-maven-3.9.11\bin\mvn.cmd" -f "$env:USERPROFILE\Downloads\Sistema_Banc-rio-main\Sistema_Banc-rio-main\pom.xml" "-Dtest=ContaTest" test
```

Um método específico:

```powershell
& "$env:LOCALAPPDATA\Programs\Apache\Maven\apache-maven-3.9.11\bin\mvn.cmd" -f "$env:USERPROFILE\Downloads\Sistema_Banc-rio-main\Sistema_Banc-rio-main\pom.xml" "-Dtest=ContaTest#depositarAtualizaSaldoERegistraTransacao" test
```

Os testes de JDBC usam bases H2 isoladas em memória. Os testes de exportação
usam diretórios temporários JUnit.

## Usar a aplicação

Ao iniciar, a aplicação mostra uma tela de acesso demonstrativa. Use o nome do
titular como usuário, em minúsculas e com ponto entre nomes, e a senha fictícia
`123456`. Exemplos das contas iniciais: `ana.silva` e `bruno.costa`. O botão
**Sair** encerra a sessão e volta para o acesso.

Esse login é apenas uma simulação local para apresentação: não há cadastro nem
armazenamento de senhas, e ele não deve ser usado com credenciais reais.

O cabeçalho permite selecionar uma conta e alternar entre os temas claro e
escuro. A navegação lateral abre as telas **Início**, **Transferir**,
**Depositar**, **Sacar**, **Extrato**, **Minhas contas**, **Simulação** e
**Race condition**.

- Em **Início**, consulte o saldo, o titular, as cinco movimentações recentes e
  atalhos para operações e exportação. O ícone de olho oculta ou revela o saldo.
- Em **Transferir**, selecione origem e destino, informe o valor e revise a
  confirmação. Depósitos e saques seguem o mesmo fluxo de revisão e exibem um
  comprovante ao concluir.
- Valores aceitam formatos como `25,50` e `25.50`; são processados com
  `BigDecimal` e exibidos em moeda brasileira.
- Em **Extrato**, filtre por entradas/saídas e período; exporte o arquivo `.txt`
  em UTF-8 pelo botão de exportação.
- Em **Minhas contas**, consulte saldos ou abra uma nova conta.
- Em **Simulação**, rode operações concorrentes em quatro threads e compare os
  saldos esperados e obtidos, além das operações realizadas e recusadas.
- Em **Race condition**, compare visualmente o saldo inseguro com o saldo da
  classe `Conta` sincronizada.

Operações bancárias, consultas ao serviço, simulações e escrita do extrato
executam em segundo plano para manter a janela responsiva.

## Arquitetura

- `domain`: `Cliente`, `Conta`, `Transacao`, tipos e validações de negócio.
  `Conta` é o único objeto que modifica o saldo; ela protege saldo e extrato
  com o mesmo monitor.
- `persistence`: contratos `ContaRepository` e `TransacaoRepository`, além
  das implementações em memória e JDBC.
- `persistence.jdbc`: `ConnectionFactory`, `JdbcContaRepository` e
  `JdbcTransacaoRepository`. Cada chamada abre e fecha sua própria conexão.
  A gravação de uma movimentação e a atualização atômica do saldo ocorrem em
  transação JDBC; os dois registros de uma transferência são gravados juntos.
- `simulacao`: operações de teste concorrente, `OperacaoBancariaThread`,
  `SimulacaoRunner` e relatório de saldos esperados e obtidos.
- `simulacao.demo`: conta deliberadamente insegura, isolada da conta real,
  utilizada somente para mostrar a perda de atualização.
- `service`: `BancoService` coordena domínio e repositórios. Locks de serviço,
  ordenados por ID em transferências, mantêm a ordem entre a mudança no domínio
  e a persistência sem manter monitores de `Conta` durante JDBC.
- `io`: `ExtratoExporter` grava o extrato recebido em UTF-8 sem acessar banco.
- `ui`: interface Swing; coleta entradas e delega operações ao serviço.
- `app`: `Main` compõe persistência, serviço e janela Swing.
- `src/main/resources/schema.sql`: tabelas de contas/transações e índice de
  consulta de extrato.

## Requisitos entregues

O projeto inclui operações monetárias com `BigDecimal`, validações de saldo e
valor, registros imutáveis, simulação multithread, contagem de saques recusados,
conferência de saldos, proteção contra deadlock, persistência JDBC, exportação
de extrato e testes JUnit de domínio, serviço, concorrência, JDBC e I/O.

O arquivo `RELATORIO-RACE-CONDITION.md` registra a comparação “antes e depois”;
`docs/extrato-exemplo.txt` é um exemplo de extrato exportado.

## Extras de interface (fora dos requisitos da disciplina)

- Interface Swing redesenhada como app de banco digital fictício, com cabeçalho,
  menu lateral, navegação por telas, tema claro/escuro, cartões e ícones SVG
  próprios. Não usa marca nem imagens do Itaú.
- Fluxos de operação com confirmação, comprovante e mensagens inline; extrato
  filtrável; visão de contas; painéis de simulação e demonstração da corrida.
- FlatLaf Extras fornece o suporte aos ícones SVG; MigLayout fica disponível
  como dependência visual da interface.

Esses itens são conveniências de apresentação e não substituem nem alteram os
requisitos acadêmicos do domínio, persistência ou concorrência.

## Outros recursos adicionais

- Banco H2 em arquivo local, sem precisar instalar ou configurar um servidor
  externo. A conexão pode ser trocada por outra URL JDBC.
- Repositórios em memória para testes rápidos sem banco real.
- Demonstração determinística da race condition: uma barreira força as duas
  threads inseguras a lerem o mesmo saldo antes de gravar, expondo a atualização
  perdida sem depender de `Thread.sleep()`.
- Simulação e demonstração podem ser iniciadas pela própria interface Swing.

## Arquivos importantes

- `AGENT.md`: especificação detalhada e regras arquiteturais do projeto.
- `Especificacao_6_Sistema_Bancario_Concorrente.pdf`: enunciado da atividade.
- `GUIA-APRESENTACAO.md`: roteiro de demonstração, arquitetura e respostas para
  dúvidas comuns.
- `RELATORIO-RACE-CONDITION.md`: evidência da corrida e da correção.
- `docs/extrato-exemplo.txt`: exemplo de extrato no formato exportado.
