Repositório com comandos básicos via cli da Amazon DynamoDB para entrega de projeto bootcamp DIO.
Esse repositório teve como base o repositório https://github.com/cassianobrexbit/dio-live-dynamodb

### Recursos necessários
  - Acesso ao Amazon DynamoDB
  - Amazon CLI para execução em linha de comando
  - Baixar arquivo zip anexado nesse repositório e executar os comandos no mesmo diretório que contém os arquivo .json que serão usados em alguns comandos de execução em lote

### Acesso via AWS CLI:
- Você pode usar o AWS CloudShell, que é web;
- Baixar o awscliv2.zip e instalá-lo na sua distribuição Linux - https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html;
- Criar a credencial via Identity and Access Management (IAM) que irá gerar o "AWS Access Key ID" e o "AWS Secret Access Key". Esse foi o método de autenticação que eu utilizei, mas existem outros;
- Se autenticar usando o comando "aws configure" que irá solicitar o "AWS Access Key ID", "AWS Secret Access Key" e o "Default region name", por exemplo, us-east-2.

---

### Criando tabela Movies no DynamoDB:

Cuidado ao criar a tabela, se algum atributo estiver com nome errado você terá que criar uma nova, copiar os dados para a nova, deletar a tabela antiga e renomear a nova com o nome da antiga.

Essa tabela será criada com 3 índices globais secundários:
- IndexName: MoviesTitleIndex, KeySchema: Title (HASH), ReleaseDate (RANGE), ProjectionType: ALL
- IndexName: MoviesDirectorIndex, KeySchema: Director (HASH), ReleaseDate (RANGE), ProjectionType: ALL
- IndexName: MoviesProducerIndex, KeySchema: Producer (HASH), ReleaseDate (RANGE), ProjectionType: ALL

Os índices globais secundários serão criados com a opção "ProjectionType: ALL", o que significa que todos os atributos da tabela serão incluídos no índice, e usarão o atributo ReleaseDate como chave de classificação (RANGE), não o atributo Year.

```
aws dynamodb create-table \
    --table-name Movies \
    --attribute-definitions \
        AttributeName=Title,AttributeType=S \
        AttributeName=ReleaseDate,AttributeType=N \
        AttributeName=Director,AttributeType=S \
        AttributeName=Producer,AttributeType=S \
    --key-schema \
        AttributeName=Title,KeyType=HASH \
    --global-secondary-indexes \
        IndexName=MoviesTitleIndex,KeySchema=["{AttributeName=Title,KeyType=HASH}","{AttributeName=ReleaseDate,KeyType=RANGE}"],Projection="{ProjectionType=ALL}",ProvisionedThroughput="{ReadCapacityUnits=10,WriteCapacityUnits=5}" \
        IndexName=MoviesDirectorIndex,KeySchema=["{AttributeName=Director,KeyType=HASH}","{AttributeName=ReleaseDate,KeyType=RANGE}"],Projection="{ProjectionType=ALL}",ProvisionedThroughput="{ReadCapacityUnits=10,WriteCapacityUnits=5}" \
        IndexName=MoviesProducerIndex,KeySchema=["{AttributeName=Producer,KeyType=HASH}","{AttributeName=ReleaseDate,KeyType=RANGE}"],Projection="{ProjectionType=ALL}",ProvisionedThroughput="{ReadCapacityUnits=10,WriteCapacityUnits=5}" \
    --provisioned-throughput \
        ReadCapacityUnits=10,WriteCapacityUnits=5 --no-cli-pager
```

O processo de criação de um índice pode demorar alguns minutos. Para verificar o andamento dessa tarefa, execute esse comando. No entanto, visualizar via interface web é muito mais fácil de entender o andamento da tarefa:

```
aws dynamodb describe-table --table-name Movies --no-cli-pager
```


### Inserindo itens na tabela:

Para inserir um item na tabela com os campos "Director" e "Producer", você precisa alterar o arquivo JSON que terá esse formato e conteúdo. O arquivo deve estar no mesmo diretório em que estiver executando o comando aws:

```
{
  "Title": {"S": "Interstellar"},
  "ReleaseDate": {"N": "2014"},
  "Director": {"S": "Christopher Nolan"},
  "Producer": {"S": "Paramount Pictures"}
}
```

-  E então, execute o comando abaixo para inserir o item na tabela Movies:

```
aws dynamodb put-item --table-name Movies --item file://itemmovie.json
```

-   Para inserir vários itens, você precisa criar um arquivo JSON com os novos campos em cada item (arquivo disponível neste repositório) e, em seguida, executar o comando abaixo:

```
aws dynamodb batch-write-item --request-items file://batchmovies.json
```

**Obs:** Se todos os itens forem importados com sucesso, a mensagem abaixo será exibida informando que não há nenhum item não processado:

```
{
    "UnprocessedItems": {}
}
```

### Criando um índice global secundário:

-   Para criar um índice global secundário baseado no campo "ReleaseDate" do filme, você pode executar o seguinte comando:

```
aws dynamodb update-table \
  --table-name Movies \
  --attribute-definitions AttributeName=ReleaseDate,AttributeType=N \
  --global-secondary-index-updates \
  "[{
    \"Create\": {
      \"IndexName\": \"MoviesReleaseDateIndex\",
      \"KeySchema\": [{
        \"AttributeName\": \"ReleaseDate\",
        \"KeyType\": \"HASH\"
      }],
      \"Projection\": {
        \"ProjectionType\": \"ALL\"
      },
      \"ProvisionedThroughput\": {
        \"ReadCapacityUnits\": 10,
        \"WriteCapacityUnits\": 5
      }
    }
  }]" --no-cli-pager

```

**  O processo de criação de um índice pode levar alguns minutos. **

### Realizar buscas diversas:

-   Para pesquisar um item na tabela por título do filme:

```
aws dynamodb query \
    --table-name Movies \
    --key-condition-expression "Title = :title" \
    --expression-attribute-values  '{":title":{"S":"The Matrix"}}'
```

-   Para pesquisar um item por título do filme e ano de lançamento, passando os critérios diretamente na query:

```
aws dynamodb query \
    --table-name Movies \
    --index-name MoviesTitleIndex \
    --key-condition-expression "Title = :title and ReleaseDate = :ReleaseDate" \
    --expression-attribute-values '{":title":{"S":"Interstellar"}, ":ReleaseDate":{"N":"2014"}}'
```

-   Para pesquisar um item por título do filme e ano de lançamento, passando um arquivo .json com os critérios da query:
    
```
aws dynamodb query \
    --table-name Movies \
    --index-name MoviesTitleIndex \
    --key-condition-expression "Title = :title and ReleaseDate = :ReleaseDate" \
    --expression-attribute-values file://keyconditions_movie.json   
```

### Recriando índice:

-   Pesquisar item por título do filme e diretor:

*  Exclua o índice MoviesDirectorIndex, que foi criado no momento da criação da tabela usando o atributo ReleaseDate como chave de ordenação e torna obrigatório informar o ano de lançamento do filme. Para remover essa restrição, exclua esse índice e recrie-o:
    
```
aws dynamodb update-table \
    --table-name Movies \
    --global-secondary-index-updates '[        {"Delete":{"IndexName":"MoviesDirectorIndex"}}    ]'  --no-cli-pager
```

*  Recrie o índice MoviesDirectorIndex para não precisar informar o ano do filme. Será criado um índice secundário com o nome MoviesDirectorIndex que terá Director como chave de ordenação e Title

    ** O processo de criação de um indíce pode demorar alguns minutos **

```
aws dynamodb update-table \
    --table-name Movies \
    --attribute-definitions AttributeName=Director,AttributeType=S AttributeName=Title,AttributeType=S AttributeName=ReleaseDate,AttributeType=N \
    --global-secondary-index-updates '[{"Create":{"IndexName":"MoviesDirectorIndex","KeySchema":[{"AttributeName":"Director","KeyType":"HASH"},{"AttributeName":"Title","KeyType":"RANGE"}],"Projection":{"ProjectionType":"INCLUDE","NonKeyAttributes":["ReleaseDate"]}}}]' \
    --billing-mode PAY_PER_REQUEST  --no-cli-pager
```

**Obs:** 
    Para fins de estudos, adicionei a opção "--billing-mode PAY_PER_REQUEST" que altera o modo de cobrança da tabela do DynamoDB para PAY_PER_REQUEST, que permite pagar apenas pelos recursos que foram utilizados. É importante lembrar que essa opção requer uma análise cuidadosa das necessidades de recursos e um monitoramento constante para evitar surpresas com cobranças inesperadas.


*  Após a exclusão do índice do diretor e recriarmos corretamente, a busca irá funcionar sem precisar informar o ano:

```
aws dynamodb query \
    --table-name Movies \
    --index-name MoviesDirectorIndex \
    --key-condition-expression "Director = :director and Title = :title" \
    --expression-attribute-values '{":director":{"S":"Christopher Nolan"},":title":{"S":"Inception"}}'
```

-   Pesquisar item por título do filme e produtora cujo data de lançamento (ReleaseDate) tiver ocorrido entre 2008 a 2009. Esse comando irá pesquisar itens em um índice secundário que foi criado com a chave de partição sendo o título do filme e a chave de ordenação sendo a data de lançamento. O comando utiliza a expressão "BETWEEN" para buscar todos os itens que tiveram a data de lançamento entre os anos de 2008 e 2009. Além disso, também utiliza a expressão "FilterExpression" para filtrar os resultados pela produtora do filme.

```
aws dynamodb query \
    --table-name Movies \
    --index-name MoviesTitleIndex \
    --key-condition-expression "Title = :title AND ReleaseDate BETWEEN :start_year AND :end_year" \
    --filter-expression "Producer = :producer" \
    --expression-attribute-values '{":title":{"S":"The Dark Knight"}, ":producer":{"S":"Warner Bros. Pictures"}, ":start_year":{"N":"2008"}, ":end_year":{"N":"2009"}}'
```

Espero que essas explicações adicionais possam ajudar a entender melhor o uso desses comandos e recursos no DynamoDB.
