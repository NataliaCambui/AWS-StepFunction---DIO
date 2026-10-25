# AWS-StepFunction---DIO
Projeto AWS StepFunction - orquestração de fluxo de trabalho
# Mini Projeto — Automação de Validação de Notas Fiscais com AWS Step Functions

## 1. Introdução

Neste projeto, desenvolvi uma solução de automação para o processamento e a validação de notas fiscais utilizando serviços da Amazon Web Services (AWS).

O objetivo foi aplicar os conceitos de orquestração de workflows, funções serverless, armazenamento de arquivos, persistência de dados e mensageria.

A solução foi construída utilizando o AWS Step Functions para controlar a sequência de execução, o Amazon S3 para armazenar os arquivos, o AWS Lambda para validar os dados, o Amazon DynamoDB para registrar os resultados e o Amazon SNS para enviar notificações.

O projeto foi desenvolvido como um protótipo didático, utilizando notas fiscais fictícias em formato JSON.

## 2. Objetivo

O objetivo principal foi automatizar o fluxo de validação de notas fiscais, reduzindo a necessidade de verificações manuais e demonstrando como diferentes serviços da AWS podem trabalhar de maneira integrada.

Os objetivos específicos foram:

- Armazenar arquivos de notas fiscais no Amazon S3.
- Criar uma função Lambda para validar os dados dos arquivos.
- Utilizar o AWS Step Functions para coordenar o processamento.
- Direcionar o fluxo de acordo com o resultado da validação.
- Registrar os resultados no DynamoDB.
- Enviar notificações por meio do SNS.
- Monitorar as execuções e tratar possíveis falhas.

## 3. Tecnologias utilizadas

| Tecnologia | Finalidade |
|---|---|
| AWS Step Functions | Orquestração do workflow |
| Amazon S3 | Armazenamento dos arquivos |
| AWS Lambda | Execução da validação em Python |
| Amazon DynamoDB | Registro dos resultados |
| Amazon SNS | Envio de notificações |
| Amazon CloudWatch | Monitoramento e logs |
| AWS IAM | Controle de permissões |

## 4. Funcionamento da solução

O processo foi planejado para funcionar da seguinte maneira:

1. O arquivo JSON da nota fiscal é enviado para um bucket do Amazon S3.
2. A execução do Step Functions é iniciada com o nome do bucket e a chave do arquivo.
3. O Step Functions aciona a função Lambda responsável pela validação.
4. A Lambda lê o arquivo e verifica os campos obrigatórios, o CNPJ e o valor da nota.
5. O resultado é encaminhado para um estado de decisão.
6. Se a nota estiver válida, ela será classificada como aprovada. Caso contrário, será classificada como rejeitada.
7. O resultado e o motivo da validação serão registrados no DynamoDB.
8. Uma notificação será enviada pelo SNS.
9. A execução será finalizada e poderá ser consultada no Step Functions e no CloudWatch.

## 5. Implementação passo a passo

### Etapa 1 — Criar o bucket no Amazon S3

A primeira etapa foi criar um bucket para armazenar os arquivos das notas fiscais.

No console da AWS, acessei o Amazon S3, selecionei a opção para criar um bucket e defini um nome exclusivo, como `mini-projeto-notas-fiscais`.

Depois, criei uma pasta lógica chamada `entrada` para organizar os arquivos recebidos.

Para o projeto, utilizei um arquivo fictício com os seguintes dados:

```json
{
  "numeroNota": "NF-1001",
  "cnpj": "12345678000190",
  "valor": 1500.00
}
```

Salvei esse conteúdo como `nota-1001.json` e enviei o arquivo para o bucket.

O Amazon S3 foi utilizado como ponto de armazenamento dos arquivos. Nesta primeira versão, a execução do workflow é iniciada manualmente pelo console. Assim, consigo testar cada etapa antes de implementar um disparo automático pelo envio de arquivos.

### Etapa 2 — Criar a função AWS Lambda

A segunda etapa foi criar uma função Lambda chamada `ValidarNotaFiscal`.

No console da AWS, acessei o serviço Lambda, criei uma função com o runtime Python disponível e configurei as permissões necessárias para ler os objetos do bucket utilizado no projeto.

A função recebe o nome do bucket e a chave do arquivo, lê o JSON armazenado no S3 e verifica os dados da nota.

O código utilizado foi:

```python
import json
import boto3
from decimal import Decimal

s3 = boto3.client("s3")


def lambda_handler(event, context):
    bucket = event["bucket"]
    chave = event["chave"]

    try:
        objeto = s3.get_object(
            Bucket=bucket,
            Key=chave
        )

        nota = json.loads(
            objeto["Body"].read().decode("utf-8"),
            parse_float=Decimal
        )

        numero = nota.get("numeroNota")
        cnpj = nota.get("cnpj")
        valor = nota.get("valor")

        erros = []

        if not numero:
            erros.append("Número da nota obrigatório")

        if not isinstance(cnpj, str) or (
            len(cnpj) != 14 or not cnpj.isdigit()
        ):
            erros.append("CNPJ deve conter 14 dígitos")

        if not isinstance(valor, (int, float, Decimal)) or (
            isinstance(valor, bool) or valor <= 0
        ):
            erros.append("Valor deve ser maior que zero")

        status = "APROVADA" if not erros else "REJEITADA"

        return {
            "numeroNota": str(numero or "NAO_INFORMADO"),
            "cnpj": cnpj or "",
            "valor": str(valor) if valor is not None else "0",
            "status": status,
            "erros": erros
        }

    except Exception as erro:
        raise RuntimeError(
            "Falha ao ler ou processar o arquivo"
        ) from erro
```

O código retorna o resultado da validação em um objeto JSON.

Caso os dados estejam corretos, o status será `APROVADA`. Caso algum dos campos não atenda às regras implementadas, o status será `REJEITADA`, acompanhado da lista de erros.

Falhas técnicas, como um arquivo inexistente ou um JSON inválido, são tratadas como exceções para que o Step Functions possa aplicar suas políticas de repetição e tratamento de erros.

**Observação:** essa validação é apenas estrutural. Em um sistema real, seria necessário validar o CNPJ pelo cálculo dos dígitos verificadores, aplicar regras fiscais e conferir os valores da nota com o contrato correspondente.

### Etapa 3 — Criar a tabela no Amazon DynamoDB

Para armazenar os resultados, criei uma tabela no Amazon DynamoDB chamada `ResultadosNotas`.

Configurei `numeroNota` como chave de partição, permitindo identificar cada nota registrada.

A tabela foi utilizada para guardar informações como:

- Número da nota fiscal.
- CNPJ do emissor.
- Valor da nota.
- Status da validação.
- Motivos da rejeição, quando existentes.

Essa etapa permite consultar posteriormente quais notas foram aprovadas e quais precisam de correção.

Em uma aplicação real, também seria necessário definir uma estratégia de idempotência, por exemplo, utilizando uma chave composta pela empresa, número da nota e série, evitando duplicidades.

### Etapa 4 — Criar um tópico no Amazon SNS

Em seguida, criei um tópico no Amazon SNS chamado `NotificacoesNotas`.

Configurei uma assinatura de e-mail para receber os avisos e confirmei a assinatura por meio da mensagem enviada pelo serviço.

O objetivo foi notificar os responsáveis quando uma nota fosse aprovada ou rejeitada.

As mensagens foram planejadas para conter o número da nota, o resultado da validação e os motivos da rejeição, quando aplicável.

Para o funcionamento correto, configurei as permissões para que o Step Functions pudesse publicar mensagens no tópico.

### Etapa 5 — Criar a máquina de estados no AWS Step Functions

A etapa principal do projeto foi criar a máquina de estados responsável por coordenar todo o processamento.

No console do Step Functions, criei uma State Machine do tipo Standard e utilizei a linguagem Amazon States Language (ASL) para definir os estados e as transições.

A máquina de estados foi organizada da seguinte maneira:

- `ValidarNota`: executa a função Lambda.
- `DecidirResultado`: avalia o status retornado pela função.
- `RegistrarResultado`: grava os dados no DynamoDB.
- `NotificarResultado`: envia a mensagem pelo SNS.
- `Sucesso`: encerra o processamento.

Também configurei uma política de repetição para falhas transitórias na execução da Lambda e um estado de tratamento de falha técnica.

A definição completa da máquina de estados está apresentada no Apêndice A.

### Etapa 6 — Configurar o workflow

Nesta etapa, configurei a máquina de estados do AWS Step Functions utilizando a linguagem Amazon States Language (ASL).

A definição abaixo representa o fluxo de processamento das notas fiscais. Antes de utilizá-la, é necessário substituir os valores indicados pelos ARNs e nomes reais dos recursos criados na AWS.

```json
{
  "Comment": "Workflow de validacao de notas fiscais",
  "StartAt": "ValidarNota",
  "States": {
    "ValidarNota": {
      "Type": "Task",
      "Resource": "arn:aws:states:::lambda:invoke",
      "Parameters": {
        "FunctionName": "<ARN_DA_LAMBDA>",
        "Payload.$": "$"
      },
      "ResultSelector": {
        "resultado.$": "$.Payload"
      },
      "ResultPath": "$.validacao",
      "Retry": [
        {
          "ErrorEquals": [
            "Lambda.ServiceException",
            "Lambda.AWSLambdaException",
            "Lambda.SdkClientException"
          ],
          "IntervalSeconds": 2,
          "MaxAttempts": 3,
          "BackoffRate": 2
        }
      ],
      "Catch": [
        {
          "ErrorEquals": ["States.ALL"],
          "ResultPath": "$.erroTecnico",
          "Next": "FalhaTecnica"
        }
      ],
      "Next": "DecidirResultado"
    },
    "DecidirResultado": {
      "Type": "Choice",
      "Choices": [
        {
          "Variable": "$.validacao.resultado.status",
          "StringEquals": "APROVADA",
          "Next": "RegistrarResultado"
        },
        {
          "Variable": "$.validacao.resultado.status",
          "StringEquals": "REJEITADA",
          "Next": "RegistrarResultado"
        }
      ],
      "Default": "FalhaTecnica"
    },
    "RegistrarResultado": {
      "Type": "Task",
      "Resource": "arn:aws:states:::dynamodb:putItem",
      "Parameters": {
        "TableName": "<NOME_DA_TABELA>",
        "Item": {
          "numeroNota": {
            "S.$": "$.validacao.resultado.numeroNota"
          },
          "cnpj": {
            "S.$": "$.validacao.resultado.cnpj"
          },
          "valor": {
            "S.$": "$.validacao.resultado.valor"
          },
          "status": {
            "S.$": "$.validacao.resultado.status"
          },
          "erros": {
            "S.$": "States.JsonToString($.validacao.resultado.erros)"
          }
        }
      },
      "Next": "NotificarResultado"
    },
    "NotificarResultado": {
      "Type": "Task",
      "Resource": "arn:aws:states:::sns:publish",
      "Parameters": {
        "TopicArn": "<ARN_DO_TOPICO_SNS>",
        "Subject": "Resultado da validacao fiscal",
        "Message.$": "States.JsonToString($.validacao.resultado)"
      },
      "Next": "Sucesso"
    },
    "Sucesso": {
      "Type": "Succeed"
    },
    "FalhaTecnica": {
      "Type": "Fail",
      "Error": "FalhaNoProcessamento",
      "Cause": "Nao foi possivel concluir a validacao"
    }
  }
}
```

#### Como funciona

1. `ValidarNota`: executa a função Lambda e recebe o resultado da validação.
2. `DecidirResultado`: verifica se a nota foi aprovada ou rejeitada.
3. `RegistrarResultado`: armazena os dados no DynamoDB.
4. `NotificarResultado`: publica uma mensagem no SNS.
5. `Sucesso`: finaliza o workflow.
6. `FalhaTecnica`: encerra a execução quando ocorre uma falha técnica não recuperada.

**Observação:** essa definição é destinada à implantação na AWS. Como o projeto está sendo desenvolvido inicialmente sem acesso à AWS, o código deverá ser adaptado para a versão local simulada antes de ser executado no computador.


Como essa definição funciona?
1. O estado ValidarNota chama a Lambda e guarda seu retorno em validacao.resultado.
2. O estado DecidirResultado verifica o status retornado.
3. O estado RegistrarResultado salva os dados no DynamoDB.
4. O estado NotificarResultado publica a mensagem no SNS.
5. O estado Sucesso encerra a execução.
6. Se ocorrer uma falha técnica durante a validação, a máquina tenta novamente nas condições configuradas e, se o erro persistir, direciona a execução para FalhaTecnica.
Atenção: o Retry apresentado cobre erros transitórios específicos da integração Lambda. Ajustes adicionais podem ser necessários para outros erros e para as etapas do DynamoDB e SNS. O Catch não significa que toda falha será recuperada; ele apenas direciona a execução para um estado de falha controlada.

Etapa 7 — Executar o workflow
Para iniciar uma execução, abri a máquina de estados no console, selecionei a opção de iniciar execução e informei o JSON de entrada.
{
  "bucket": "mini-projeto-notas-fiscais",
  "chave": "entrada/nota-1001.json"
}


O Step Functions recebe o nome do bucket e a chave do arquivo, e encaminha essas informações para a Lambda.

A partir desse momento, o workflow coordena as etapas seguintes sem que seja necessário executar cada serviço manualmente.
6. Testes realizados
Planejei os testes com diferentes entradas para verificar o comportamento da solução.
Cenário	Entrada	Resultado esperado
Nota válida	CNPJ com 14 dígitos e valor positivo	APROVADA
CNPJ inválido	CNPJ com tamanho incorreto	REJEITADA
Valor inválido	Valor igual a zero	REJEITADA
Número ausente	Campo numeroNota vazio	REJEITADA
Arquivo inexistente	Chave do objeto incorreta	Falha técnica após as tentativas

Nos testes de validação, o objetivo foi confirmar se o resultado retornado pela Lambda era interpretado corretamente pelo estado Choice.

No cenário de falha técnica, o objetivo foi verificar se o Step Functions aplicava as tentativas configuradas e encaminhava a execução para o estado de falha.

Importante: esses são os cenários e resultados esperados do projeto. Para afirmar que os testes foram executados com sucesso, é necessário implantá-lo na AWS e conferir as execuções reais.
7. Monitoramento das execuções
Para acompanhar o funcionamento, utilizei o diagrama visual de execução do Step Functions como principal recurso de observabilidade.

Nesse diagrama, é possível verificar:
- Quais estados foram executados.
- O resultado retornado pela Lambda.
- Qual caminho foi escolhido pelo estado Choice.
- Em qual etapa ocorreu uma falha.
- Quanto tempo cada etapa levou.
Também utilizei o Amazon CloudWatch como referência para consultar os logs da Lambda e identificar possíveis erros durante a leitura dos arquivos.

O monitoramento é importante para diagnosticar falhas e melhorar a confiabilidade do processo automatizado.
8. Boas práticas aplicadas
Durante o desenvolvimento, considerei algumas boas práticas de arquitetura em nuvem:

Separação de responsabilidades: a Lambda executa a validação, enquanto o Step Functions controla o fluxo.

Permissões mínimas: cada recurso deve receber apenas as permissões necessárias para executar sua função.

Tratamento de erros: o workflow utiliza Retry e Catch para lidar com falhas técnicas na validação.

Idempotência: em uma versão de produção, o registro precisa impedir duplicidades e evitar que uma mesma nota seja encaminhada mais de uma vez para pagamento.

Proteção de dados: informações fiscais devem ser tratadas com cuidado, evitando dados desnecessários nos logs e restringindo o acesso aos recursos.

Controle de custos: a utilização de serviços serverless permite começar com uma infraestrutura pequena, mas os custos dependem do número de execuções, armazenamento, chamadas e notificações.

9. Melhorias futuras
Como evolução do projeto, pretendo implementar as seguintes funcionalidades:
- Iniciar automaticamente o workflow quando um arquivo for enviado ao S3, utilizando Amazon EventBridge.
- Adicionar uma classificação de notas aprovadas com alertas, sem bloquear indevidamente o pagamento.
- Integrar o Amazon Textract para extrair dados de documentos PDF.
- Comparar os dados extraídos com informações contratuais.
- Utilizar Amazon SQS para desacoplar etapas e absorver picos de processamento.
- Criar mecanismos de idempotência e controle de duplicidade.
- Integrar o fluxo com uma aplicação externa responsável pelo processo de pagamento.
10. Conclusão
Com este mini projeto, apliquei os conceitos de orquestração de workflows utilizando o AWS Step Functions e integrei diferentes serviços da AWS em um processo automatizado.

O projeto demonstra como arquivos podem ser processados, validados, classificados, registrados e notificados por meio de um fluxo controlado e monitorável.

Além de reduzir a necessidade de execução manual das etapas, essa arquitetura permite evoluir gradualmente para cenários mais complexos, como auditoria fiscal, validação contratual e integração com sistemas legados.

Concluo que o AWS Step Functions é uma ferramenta adequada para coordenar processos com múltiplas etapas, especialmente quando existe a necessidade de visualizar o fluxo, controlar decisões e tratar falhas de forma estruturada.

O resultado esperado é uma base funcional para estudos de automação em nuvem e para futuras aplicações em processos empresariais.
