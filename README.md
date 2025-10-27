# Desafio-de-Projeto-5-Executando-tarefas-automatizadas-com-Fun-o-Lambda-e-S3
# ⚙️ Automatização com AWS Lambda, S3, DynamoDB e API Gateway (LocalStack)

## 🧭 Visão geral
A automatização com **Lambda, S3, DynamoDB e API Gateway** permite que eventos da AWS **disparem funções sem intervenção manual**.  
Usando o **LocalStack**, é possível **simular tudo localmente**, sem precisar acessar a nuvem real.

Fluxo geral:
1. Um arquivo é enviado para o **S3**.  
2. O **S3** aciona uma **função Lambda**.  
3. A **Lambda** processa o arquivo e salva informações no **DynamoDB**.  
4. O **API Gateway** pode expor o processo como uma API HTTP.

---

## 🧩 1. Criar a função Lambda

**Arquivo:** `lambda_function.py`
```python
import json
import boto3

dynamodb = boto3.resource('dynamodb', endpoint_url='http://localhost:4566')
table = dynamodb.Table('ArquivosProcessados')

def lambda_handler(event, context):
    for record in event['Records']:
        bucket = record['s3']['bucket']['name']
        key = record['s3']['object']['key']
        print(f"Novo arquivo detectado: {key} no bucket {bucket}")

        # Inserir item no DynamoDB
        table.put_item(Item={
            'Arquivo': key,
            'Bucket': bucket,
            'Status': 'Processado'
        })

    return {'statusCode': 200, 'body': json.dumps('Processamento concluído.')}
```

Compacte o código:
```bash
zip function.zip lambda_function.py
```

---

## 🗄️ 2. Criar o DynamoDB
Crie a tabela que armazenará os arquivos processados:

```bash
awslocal dynamodb create-table     --table-name ArquivosProcessados     --attribute-definitions AttributeName=Arquivo,AttributeType=S     --key-schema AttributeName=Arquivo,KeyType=HASH     --provisioned-throughput ReadCapacityUnits=5,WriteCapacityUnits=5
```

---

## 📦 3. Criar o Bucket S3
```bash
awslocal s3 mb s3://meu-bucket-local
```

---

## 🔐 4. Conceder permissão do S3 para invocar a Lambda
```bash
awslocal lambda add-permission     --function-name ProcessaArquivosS3     --principal s3.amazonaws.com     --statement-id s3invoke     --action lambda:InvokeFunction     --source-arn arn:aws:s3:::meu-bucket-local
```

---

## ⚙️ 5. Criar a Função Lambda no LocalStack
```bash
awslocal lambda create-function     --function-name ProcessaArquivosS3     --runtime python3.9     --handler lambda_function.lambda_handler     --zip-file fileb://function.zip     --role arn:aws:iam::000000000000:role/lambda-role
```

---

## 🪄 6. Configurar a notificação do Bucket S3

**Arquivo JSON de configuração:** `bucket_notification.json`
```json
{
  "LambdaFunctionConfigurations": [
    {
      "LambdaFunctionArn": "arn:aws:lambda:us-east-1:000000000000:function:ProcessaArquivosS3",
      "Events": ["s3:ObjectCreated:*"]
    }
  ]
}
```

Aplicar configuração:
```bash
awslocal s3api put-bucket-notification-configuration     --bucket meu-bucket-local     --notification-configuration file://bucket_notification.json
```

---

## 🌐 7. (Opcional) Criar API com API Gateway
```bash
awslocal apigateway create-rest-api --name "LambdaAPI"
```
Depois, associe o endpoint à função Lambda para expor via HTTP.

---

## ✅ Resultado
Agora, **ao enviar um arquivo para o bucket S3**, o LocalStack irá:
1. Disparar o evento para a **Lambda**.  
2. Executar o código automaticamente.  
3. Gravar um registro no **DynamoDB**.  
4. (Opcional) Disponibilizar o fluxo via **API Gateway**.  

---

## 📚 Pontos de estudo
- Permissões IAM e **eventos S3 → Lambda**  
- **Estrutura JSON de notificações S3**  
- Operações CRUD no DynamoDB  
- Uso do **LocalStack + awslocal CLI**  
- Logs e testes com **CloudWatch local**
