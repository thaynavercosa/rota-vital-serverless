# Rota Vital — Código Vermelho | Serverless

## 1. Sobre o projeto

Atividade individual da disciplina de Infraestrutura de Software, do curso de Análise e Desenvolvimento de Sistemas da CESAR School — 2026.2.

O objetivo desta atividade é implementar uma função serverless responsável por gerar guias de transporte de bolsas de sangue a partir das requisições do sistema Código Vermelho (Rota Vital).

A função recebe uma requisição, processa os dados e retorna uma guia de transporte no formato JSON.

## 2. Tecnologias utilizadas

- AWS Lambda
- AWS SAM CLI 1.174.1
- Python 3.12
- Docker
- GitHub Codespaces
- API Gateway (emulado)
- curl

## 3. Caminho escolhido

Foi escolhido o Caminho A, utilizando o AWS SAM CLI como emulador.

Toda a atividade foi desenvolvida no GitHub Codespaces, sem necessidade de instalar ferramentas no computador físico ou publicar a função na AWS.

## 4. Como executar

Com o Docker e o AWS SAM CLI disponíveis, acessar a pasta do projeto:

```bash
cd rota-vital
```

Construir o projeto:

```bash
sam build --use-container
```

Invocar a função utilizando um evento:

```bash
sam local invoke -e events/pedido.json
```

Iniciar a API sem reaproveitamento de contêineres:

```bash
sam local start-api
```

Em outro terminal, realizar uma requisição HTTP:

```bash
curl -X POST http://127.0.0.1:3000/guias \
  -H "Content-Type: application/json" \
  -d '{"requisicaoId":"R-002","hospital":"H-Caruaru"}'
```

Para testar o reaproveitamento, parar o servidor e executar:

```bash
sam local start-api --warm-containers LAZY
```

## 5. Resultados dos experimentos

### Invocações por evento

| Informação | Primeira invocação | Segunda invocação |
|---|---:|---:|
| Duration | 46,51 ms | 47,72 ms |
| Init Duration | 0,04 ms | 0,02 ms |
| Billed Duration | 47 ms | 48 ms |
| Memória | 128 MB | 128 MB |
| Contador | 1 | 1 |
| Status HTTP | 201 | 201 |

### Reaproveitamento do ambiente

| Chamada | Sem reaproveitamento | EAGER | LAZY |
|---|---:|---:|---:|
| Primeira | 1 | 1 | 1 |
| Segunda | 1 | 2 | 2 |
| Terceira | 1 | 3 | 3 |

A configuração inicial do arquivo samconfig.toml utilizava o modo EAGER. Essa configuração foi removida da seção local_start_api para permitir o teste sem reaproveitamento.

### Tempos de execução no modo LAZY

| Chamada | Duration |
|---|---:|
| Primeira | 68,93 ms |
| Segunda | 1,46 ms |
| Terceira | 1,17 ms |

### Teste de timeout

Foi enviada a seguinte requisição:

```bash
curl -i -X POST http://127.0.0.1:3000/guias \
  -H "Content-Type: application/json" \
  -d '{"requisicaoId":"R-003","demorar":true}'
```

| Limite | Espera programada | Resultado |
|---|---|---|
| 3 segundos | 5 segundos | HTTP 502 Bad Gateway |
| 10 segundos | 5 segundos | HTTP 201 Created |

Com o limite de 3 segundos, foi registrada a mensagem:

```text
Function 'HelloWorldFunction' timed out after 3 seconds
```

Após alterar o Timeout no template.yaml para 10 segundos, reconstruir o projeto e reiniciar o servidor, a mesma requisição foi executada com sucesso.

## 6. Respostas das perguntas

### 1. Por que a primeira invocação demorou mais que a segunda? O que acontece na fase de Init?

Na primeira execução de uma função serverless, o ambiente precisa ser preparado. Esse processo é chamado de Cold Start. Na fase de Init, o ambiente de execução é inicializado, as bibliotecas são carregadas e as variáveis globais são definidas.

Nas duas invocações por evento, a primeira apresentou um tempo de inicialização maior, mas a segunda teve um tempo total de execução ligeiramente maior.

Já no modo LAZY, a diferença ficou mais evidente: a primeira chamada levou 68,93 ms, enquanto a segunda levou apenas 1,46 ms, pois o ambiente já estava preparado e foi reaproveitado.

### 2. O que o campo invocacaoNesteAmbiente mostrou com e sem warm-containers? O que isso diz sobre guardar estado numa variável global da função?

Sem o reaproveitamento dos contêineres, o contador retornou 1 nas três chamadas.

Com os modos EAGER e LAZY, o contador retornou 1, 2 e 3, mostrando que a variável global manteve seu valor enquanto o mesmo ambiente foi reutilizado.

Isso mostra que não podemos depender de variáveis globais para armazenar informações importantes, pois o ambiente pode ser encerrado ou substituído. Para guardar informações permanentemente, devemos utilizar um banco de dados ou outro armazenamento externo.

### 3. O que aconteceu com demorar = true? Qual é o tempo máximo de uma execução no Lambda, e o que fazer com um trabalho mais longo?

Quando demorar recebeu o valor true, a função tentou esperar 5 segundos. Como o limite estava configurado para 3 segundos, a execução foi interrompida e a API retornou o erro HTTP 502.

Depois de aumentar o limite para 10 segundos, a função foi executada normalmente e retornou HTTP 201.

O tempo máximo de execução de uma função AWS Lambda é de 15 minutos, ou 900 segundos.

Para trabalhos mais longos, podemos dividir o processamento em tarefas menores, utilizar AWS Step Functions ou executar o processamento em serviços como AWS ECS e AWS Batch.

### 4. Quanto custa um milhão de execuções com 128 MB e 100 ms?

Dados fornecidos na atividade:

- Requisições: US$ 0,20 por milhão.
- Processamento: US$ 0,0000166667 por GB-s.
- Memória: 128 MB = 0,125 GB.
- Duração: 100 ms = 0,1 segundo.

Cálculo:

```text
Consumo por execução:
0,125 × 0,1 = 0,0125 GB-s

Consumo de um milhão de execuções:
0,0125 × 1.000.000 = 12.500 GB-s

Custo de processamento:
12.500 × 0,0000166667 = US$ 0,20833375

Custo total:
0,20833375 + 0,20 = US$ 0,40833375
```

Portanto, o custo aproximado seria de US$ 0,41 por milhão de execuções, sem considerar a camada gratuita ou custos de outros serviços.

### 5. Em que parte do Rota Vital uma função assim faria sentido, e em qual não faria?

Uma função serverless faria sentido na geração de guias de transporte, no envio de notificações, na emissão de relatórios e na criação de alertas relacionados aos estoques de sangue.

Essas atividades podem ser executadas quando uma requisição ou evento acontece, sem precisar manter um servidor dedicado funcionando o tempo inteiro.

Não seria adequado utilizar uma única função Lambda para manter conexões permanentes ou executar processos contínuos que ultrapassem seu limite de tempo.

Também não seria adequado utilizar variáveis globais da função para armazenar permanentemente os estoques de sangue ou as requisições das instituições.

## 7. Evidências

Os registros dos testes realizados estão organizados na pasta `evidencias/`.

## 8. Uso de IA

Foi utilizada a ferramenta ChatGPT como apoio na resolução de erros, interpretação, suporte com o codigo
