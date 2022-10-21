# sre-core-banking-templates


# Criando compartilhamento de pipelines com o GitHub Actions

O GitHub Actions é uma ferramenta de fluxo de trabalho que permite a automação de fluxo de trabalho que também é conhecido por pipeline. 

Nesta documentação será apresentado um repositório que armazenará templates de pipelines compartilhadas (**`sre-core-banking-templates`**)  que serão utilizadas/invocadas por outras pipelines em diversos repositórios diferentes de dentro da organização **`andersonsilvadocktech`**, que por sua vez poderá compartilhar esses fluxos de trabalho entre os diferentes repositórios da sua organização.
<br>
## Contexto
Imagina dentro de um projeto quando se cria um novo microserviço, no repositório é criado um pipeline de deploy e ajustado os parâmetros pertinentes ao projeto. 
Esse procedimento funciona muito bem, mas há pontos negativos, como a manutenção, que acaba se tornando cada vez mais trabalhosa com o aumento dos reposditórios e seus pipelines. 

Assim nessa documentação será apresentado uma forma de centralizar os templates de pipelines, e os invocar em projetos que eles se fizerem necessários.
</br>
## Cenário utilizado
Foram criados dois repositórios na organização **andersonsilvadocktech** que são:
- **sre-core-banking-templates** - Repositório onde será armazenado o pipeline ***`build-and-push-ecr.yml`*** (Esse pipeline faz o build de um projeto em python armazenado no repositório que a invocará e instala os módulos listados no arquivo `requirements.txt` do mesmo e publica a imagem no AWS ECR informado) a ser invocado pela outra pipeline.
```
name: 'Pipeline build and push in ECR'

on:
  workflow_call:
    inputs:
      TAG_IMAGE:
        required: true
        type: string
      ECR_REGISTRY:
        required: true
        type: string
      ECR_REPOSITORY:
        required: true
        type: string

jobs:
  build-push-ecr-develop:
    name: sre-core-banking-templates build and push in ECR
    runs-on: dev
    # env:
    #   TAG_IMAGE: $(echo $GITHUB_SHA | head -c 7)
    #   ECR_REGISTRY: <AccountID>.dkr.ecr.<region>.amazonaws.com
    #   ECR_REPOSITORY: <org-github>/<repository-name>
    steps:
    - name: check out code
      uses: actions/checkout@v2

    - name: aws login ecr 
      run: |
        aws ecr get-login-password --region sa-east-1 | docker login --username AWS --password-stdin ${{inputs.ECR_REGISTRY}}

    - name: Build container image
      run: |
        docker build -t ${{inputs.ECR_REPOSITORY}} .
        docker tag ${{inputs.ECR_REPOSITORY}}:latest ${{inputs.ECR_REGISTRY}}/${{inputs.ECR_REPOSITORY}}:${{inputs.TAG_IMAGE}}

    - name: Push image to ECR
      run: | 
        docker push ${{inputs.ECR_REGISTRY}}/${{inputs.ECR_REPOSITORY}}:${{inputs.TAG_IMAGE}}

```

***Para que o pipeline acima possa invocado por outros pipelines, a chave `workflow_call` deve ser adionada ao bloco de valores de `on` (linhas 3 a 4):***

```
on:
  workflow_call:
  ...
```

</br></br></br>

---
</br></br></br>


- **sre-core-banking-poc-invoke** - Repositório que armazena o projeto python e também a pipeline ***`pipeline-main.yml`*** que invoca a pipeline ***`build-and-push-ecr.yml`***.
```
name: 'Pipeline reusable/invoke workflows'

on:
  workflow_dispatch:
  push:
    branches: [ "main" ]
  pull_request:
    branches: [ "main" ]
    
jobs:
  build-and-push:
    uses: andersonsilvadocktech/sre-core-banking-templates/.github/workflows/build-and-push-ecr.yml@main
    with:
      TAG_IMAGE: $(echo $GITHUB_SHA | head -c 7)
      ECR_REGISTRY: <AccountID>.dkr.ecr.<region>.amazonaws.com
      ECR_REPOSITORY: <ECR>/<name-repo>


```
***Para invocar o fluxo desejado, utiliza-se na pipeline de chamada o bloco `uses:` com a seguinte instrução `<ORG-GITHUB>/<REPO>/.github/workflows/<NOME_PIPE>.yml` (linha 12).***

</br></br></br>
## Referência
- Para um entendimento mais detalhado basta consultar https://docs.github.com/pt/actions/using-workflows.