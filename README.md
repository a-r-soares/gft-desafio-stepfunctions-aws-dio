## DIO - Bootcamp GFT - Fundamentos de Cloud com AWS

<br>

## Desafio - Explorando Workflows Automatizados com AWS Step Functions

Este documento descreve a arquitetura e os aprendizados obtidos durante a modelagem de uma Máquina de Estados (State Machine) no AWS Step Functions, simulando um fluxo de processamento e validação de pedidos de e-commerce.

<br>

## 📸 Arquitetura do Fluxo

<img width="1352" height="692" alt="image" src="https://github.com/user-attachments/assets/16d3f9f4-db8a-42cb-b4d7-6cf8627ecec0" />


---

<br>

## 🛠️ Descrição do Fluxo Lógico

O fluxo foi desenhado utilizando o mecanismo de consulta moderno **JSONata**, estruturando-se nas seguintes etapas:

1. **Validar pedido (AWS Lambda):** Recebe o payload inicial com os dados e o ID do pedido. Realiza a validação cadastral e financeira do evento.
2. **Pagamento aprovado? (Choice State):** Uma tomada de decisão baseada na regra condicional JSONata `{% $states.input.status == 'APROVADO' %}`.
   - **Caminho Feliz (Sim):** Direciona para o processamento de logística.
   - **Caminho Padrão / Default (Não):** Direciona para o fluxo de contingência e tratamento de erro de pagamento.
3. **Processar envio (AWS Lambda):** Ativado apenas em caso de sucesso no pagamento. Recebe as propriedades originais do pedido para gerar o despacho físico.
4. **Notificar cliente (Amazon SNS):** Publica uma mensagem em um tópico Pub/Sub avisando o comprador que o produto foi postado.
5. **Notificar comercial (Amazon SNS):** Rota de falha que dispara um alerta para a equipe interna de vendas atuar na tentativa de recuperação do carrinho/venda.
6. **Pagamento recusado (Fail State):** Finaliza a execução da máquina de estados explicitando o status de falha para fins de monitoramento e métricas de negócios.

---

<br>

## 🧠 Principais Aprendizados e Conceitos Técnicos

### 1. Resolução do Erro de Validação de Sintaxe (Rule #1)
Durante a configuração inicial, ocorreu o erro `Rule #1 : Must be a boolean or a valid JSONata expression`. A falha foi gerada porque expressões avaliadas do JSONata (como `{% $states.input %}`) não devem ser tratadas de forma literal pelo validador do painel de código puro. A correção consistiu em envolver as tags de expressão do JSONata entre aspas duplas (`"{% ... %}"`) dentro dos blocos JSON para que o motor do Step Functions realize o parse dinâmico em tempo de execução sem quebrar a gramática estática do painel visual.

### 2. Gerenciamento de Estado com Variáveis (Assign)
No Step Functions, a saída de um estado substitui a entrada do estado seguinte, o que pode causar a perda de dados importantes ao longo do caminho. Para solucionar isso, foi utilizada a funcionalidade **Atribuir variáveis (Assign)** no bloco inicial. Foram salvas duas variáveis na memória global da execução:
* `$idSalvo`: Guardando o ID do pedido para uso posterior nas mensagens de texto do SNS.
* `$dadosDoPedido`: Guardando o payload completo recebido no início para ser consumido pela Lambda de envio na rota do "Sim".

### 3. Anatomia de um ARN (Amazon Resource Name)
Foi compreendida a função do ARN como o identificador único universal de recursos dentro do ecossistema AWS (como Lambdas e Tópicos SNS). A estrutura divide-se em tipo de serviço, região geográfica do data center, identificação numérica da conta proprietária e nome do recurso, agindo essencialmente como o "endereço postal" necessário para a comunicação entre microsserviços.

### 4. Tratamento Semântico de Erros (Fail State)
Ficou claro que fluxos encerrados por insucesso de regras de negócio (como um pagamento recusado) não devem simplesmente apontar para um encerramento padrão de sucesso. A utilização do componente **Fail State** garante a correta categorização da execução no histórico do serviço, viabilizando a extração de indicadores de desempenho (KPIs) confiáveis.

## 🚀 Considerações Avançadas de Arquitetura

### 5. Isolamento de Execuções e Escala Elástica
Ficou compreendido que o Step Functions gerencia execuções de forma totalmente isolada e assíncrona. Cada pedido gera uma instância independente da máquina de estados. O processamento concorrente de múltiplos pedidos ocorre naturalmente através da escalabilidade nativa da AWS, o que difere do conceito de "Parallel State" (onde tarefas distintas de um mesmo pedido são executadas simultaneamente).

### 6. Complexidade e Ramificações do Fluxo
A arquitetura baseada em Step Functions permite evolução contínua. É perfeitamente possível encadear múltiplos blocos `Choice` ao longo do fluxo para criar regras de negócio mais complexas (como ramificar entregas entre produtos físicos e digitais). Consequentemente, um único fluxo pode conter múltiplos estados terminais de Sucesso (`Succeed`) ou de Falha (`Fail`), categorizando com precisão cada desfecho do processo.

---

## 💡 Curiosidade de Arquitetura: Do Mainframe para a Nuvem (A Base Nunca Muda)

Para profissionais que estão vivenciando uma transição de carreira ou de ambiente tecnológico — especialmente aqueles vindos do ecossistema de **Mainframe** —, o AWS Step Functions pode parecer, à primeira vista, um conceito de "outro planeta". No entanto, uma análise estrutural revela que os princípios fundamentais de engenharia de software permanecem exatamente os mesmos, tendo sido apenas modernizados em suas técnicas e nomenclaturas.

Existe um "De/Para" perfeito entre o processamento tradicional e a orquestração Serverless moderna:

*   **A Máquina de Estados (State Machine) é o seu JCL (Job Control Language):** Assim como o JCL amarra a ordem de execução de um Job, o arquivo JSONata do Step Functions dita o fluxo lógico de como o processo deve navegar.
*   **Os Estados (States/Tasks) são os seus Steps:** Cada função Lambda ou chamada de SNS atua exatamente como um *Step* de processamento do JCL executando um programa específico.
*   **O Estado 'Succeed' é o seu RC=0 (Return Code):** Quando o caminho feliz do fluxo é concluído com sucesso (como no envio do pedido e notificação do cliente), a AWS encerra a execução com sucesso, equivalente ao clássico `Return Code 0`.
*   **O Estado 'Fail' é o seu ABEND (Abnormal End):** A caixinha vermelha de falha configurada na rota de pagamento recusado funciona exatamente como um *Abend* controlado. Ela interrompe o fluxo de forma limpa, gerando um código de erro (`Error`) e uma causa (`Cause`) para auditoria.

### 🚀 Um Incentivo para a Transição de Carreira

A grande evolução conceitual reside no modelo de ativação: enquanto o Mainframe brilha no processamento assíncrono em lote (*Batch*) processando arquivos sequenciais massivos na madrugada, o Step Functions aplica os mesmos conceitos em tempo real, de forma distribuída, elástica e totalmente orientada a eventos disparados pelas ações instantâneas dos usuários.

O conhecimento acumulado em plataformas legadas não é descartado; ele é uma **superpotência**. Entender de controle de fluxo, tratamento de erros e integridade de dados confere uma vantagem competitiva imensa na Nuvem. Mudar de ambiente não significa recomeçar do zero, significa apenas atualizar o dicionário e as ferramentas para colonizar esse novo território com uma bagagem que já se provou sólida há décadas.


---

<img width="20" height="20" alt="image" src="https://github.com/user-attachments/assets/3f883ffa-dbfb-43d7-ae40-bd9456d6440f" /> ### Agradecimentos

[Equipe GFT](https://www.gft.com/br/pt)

[Equipe DIO](https://www.dio.me)

Utilização de IA principalmente para tornar a documentação mais clara, objetiva e fluída.

Brazil, Julho de 2026
