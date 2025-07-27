# **Parte 1**

## Introdução

O processo de integração contínua e implantação contínua (CI/CD) é essencial para a entrega simplificada e consistente de atualizações de software. Este projeto prático fornecerá instruções para automatizar a construção e implantação da aplicação HumanGov SaaS em um cluster Kubernetes usando o AWS CodeCommit, CodeBuild e CodePipeline.

## Pré-requisitos

- Um cluster Kubernetes chamado `humangov-cluster` com pelo menos um estado da aplicação HumanGov implantado em execução no Amazon EKS. Se você tiver destruído seu cluster, siga as instruções do Módulo 6 para provisioná-lo novamente. Não se esqueça de destruí-lo quando terminar este projeto prático.
- Um repositório AWS CodeCommit pré-existente chamado `human-gov-application`que contém o código-fonte da aplicação HumanGov SaaS.

## Instruções Detalhadas

## Parte 1: Configuração do Pipeline CI/CD

### Comece fazendo o push das alterações no código-fonte da aplicação HumanGov para o AWS CodeCommit

Habilite Credenciais Gerenciadas

```
cd ~/environment/human-gov-application/src
git status
git add -A
git commit -m "added k8s manifests"
git push
```

### **Configurar o AWS CodePipeline**

1. **Criar um Novo Pipeline:**
    - Acesse o AWS CodePipeline.
    - Inicie o processo de 'Criar pipeline'.
    - Nome: `human-gov-cicd-pipeline`
    - Use o repositório do CodeCommit `human-gov-application` como a fonte.
    - Adicione o projeto 'HumanGovBuild' como estágio de construção.
    - Adicione o projeto 'HumanGovDeploy' como estágio de implantação.

### Configurar **o AWS CodeBuild para Construir a Imagem Docker**

1. **Criar um Projeto de Construção:**
    - Dê um nome ao projeto (por exemplo, **`HumanGovBuild`**).
    - Conecte-o ao seu repositório CodeCommit existente (**`human-gov-application`**).
    - Configure o ambiente para dar suporte às construções Docker.
    - Para a especificação de construção, utilize o seguinte **`buildspec.yml`**:
    - Adicione a variável de ambiente ECR_REPO com a URI do repositório ECR.

```yaml
version: 0.2
phases:
  install:
    runtime-versions:
      docker: 20
  pre_build:
    commands:
      - echo Logging in to Amazon ECR...
      - aws --version
      - REPOSITORY_URI=$ECR_REPO
      **- aws ecr-public get-login-password --region $AWS_DEFAULT_REGION | docker login --username AWS --password-stdin public.ecr.aws/l4c0j8h9**  
  build:
    commands:
      - echo Build started on `date`
      - echo Building the Docker image...
      - cd src
      - docker build -t $REPOSITORY_URI:latest .
      - docker tag $REPOSITORY_URI:latest $REPOSITORY_URI:$CODEBUILD_RESOLVED_SOURCE_VERSION
  post_build:
    commands:
      - echo Build completed on `date`  
      - echo Pushing the Docker image...
      - docker push $REPOSITORY_URI:latest
      - docker push $REPOSITORY_URI:$CODEBUILD_RESOLVED_SOURCE_VERSION
      - export imageTag=$CODEBUILD_RESOLVED_SOURCE_VERSION
      - printf '[{\"name\":\"humangov-app\",\"imageUri\":\"%s\"}]' $REPOSITORY_URI:$imageTag > imagedefinitions.json
      - cat imagedefinitions.json
      - ls -l

env:
  exported-variables: ["imageTag"]
  
artifacts:
  files: 
    - src/imagedefinitions.json
    - src/humangov-sao-paulo.yaml
    - src/humangov-rio-de-janeiro.yaml
```

1. **Adicione a permissão AmazonElasticContainerRegistryPublicFullAccess ao ECR na role de serviço** 
    - Acesse o console do IAM > Funções (Roles).
    - Procure pela função criada "HumanGovBuild" para o CodeBuild.
    - Adicione a permissão **AmazonElasticContainerRegistryPublicFullAccess**.

### Configurar o AWS CodeBuild para Implantação da Aplicação

1. **Criar um Projeto de Implantação:**
    - Repita o processo de criação de projetos no CodeBuild.
    - Dê a este projeto um nome diferente (por exemplo, **`HumanGovDeployToProduction`**).
    - Configure as variáveis de ambiente AWS_ACCESS_KEY_ID e AWS_SECRET_ACCESS_KEY para as credenciais do usuário **`eks-user`** no Cloud Build, para que ele possa autenticar-se no cluster Kubernetes.
    
    *Observe: em um ambiente de produção do mundo real, é recomendável usar uma função do IAM para essa finalidade. Neste exercício prático, estamos usando diretamente as credenciais do usuário **`eks-user`** para facilitar o processo, já que nosso foco é na CI/CD e não na autenticação do usuário neste momento. A configuração desse processo no EKS é mais extensa. Consulte a seção de Referência e consulte "Habilitando o acesso de princípio do IAM ao seu cluster"*
    
    - Para a especificação de implantação, utilize o seguinte **`buildspec.yml`**:

```yaml
version: 0.2

phases:
  install:
    runtime-versions:
      docker: 20
    commands:
      - curl -o kubectl https://amazon-eks.s3.us-west-2.amazonaws.com/1.18.9/2020-11-02/bin/linux/amd64/kubectl
      - chmod +x ./kubectl
      - mv ./kubectl /usr/local/bin
      - kubectl version --short --client
  post_build:
    commands:
      - aws eks update-kubeconfig --region $AWS_DEFAULT_REGION --name humangov-cluster
      - kubectl get nodes
      - ls
      - cd src
      - IMAGE_URI=$(jq -r '.[0].imageUri' imagedefinitions.json)
      - echo $IMAGE_URI
      - sed -i "s|CONTAINER_IMAGE|$IMAGE_URI|g" humangov-california.yaml
      - sed -i "s|CONTAINER_IMAGE|$IMAGE_URI|g" humangov-florida.yaml
      - kubectl apply -f humangov-california.yaml
      - kubectl apply -f humangov-florida.yaml
```

- Substitua a URI da imagem na linha 18 dos arquivos **`humangov-california.yaml`** e **`humangov-florida.yaml`** por CONTAINER_IMAGE.
- Faça um commit e envie as alterações.

```bash
git add -A
git commit -m "replaced image uri with CONTAINER_IMAGE"
git push
```

### **Teste sua Pipeline de CI/CD**

1. **Faça uma Alteração no CodeCommit:**
    - Atualize o código da aplicação no repositório **`human-gov-application`**.
    - Faça um commit e envie as alterações.
2. **Observe a Execução da Pipeline:**
    - Observe como o CodePipeline aciona automaticamente a compilação.
    - Após a compilação, a fase de implantação deve começar.
3. **Verifique a Implantação:**
    - Verifique o Kubernetes usando os comandos **`kubectl`** para confirmar a atualização da aplicação.



# **Parte 2**

## Introdução

O processo de integração contínua e implantação contínua (CI/CD) é essencial para a entrega simplificada e consistente de atualizações de software. Este projeto prático fornecerá instruções para automatizar a construção e implantação da aplicação HumanGov SaaS em um cluster Kubernetes usando o AWS CodeCommit, CodeBuild e CodePipeline.

## Pré-requisitos

- Um cluster Kubernetes chamado `humangov-cluster` com pelo menos um estado da aplicação HumanGov implantado em execução no Amazon EKS. Se você tiver destruído seu cluster, siga as instruções do Módulo 6 para provisioná-lo novamente. Não se esqueça de destruí-lo quando terminar este projeto prático.
- Um repositório AWS CodeCommit pré-existente chamado `human-gov-application`que contém o código-fonte da aplicação HumanGov SaaS.

## Instruções Detalhadas

## **Parte 2: Adicionando o Estágio de Teste ao Pipeline de CI/CD do HumanGov**

1. **Adicionar código-fonte para simulação de testes**
OBS: Lembre-se de que esta é apenas uma simulação para entender onde colocar os testes em um pipeline de CI/CD e a importância deles. No mundo real, os testes adequados de unidade e outros testes devem ser escritos pelos desenvolvedores de acordo com a lógica de negócios e as regras esperadas para a aplicação.
    - Crie uma nova pasta chamada **`tests`** no diretório **`human-gov-application/src`**
    - Crie os arquivos **`[app.py](http://tests.py)`** e **`test_app.py`** na pasta **`tests`** e adicione o código a seguir a eles
    **`app.py`**:

## Parte 2: **Adicionando o Estágio de Teste ao Pipeline de CI/CD do HumanGov**

1. **Adicionar código-fonte para simulação de testes**
OBS: Lembre-se de que esta é apenas uma simulação para entender onde colocar os testes em um pipeline de CI/CD e a importância deles. No mundo real, os testes adequados de unidade e outros testes devem ser escritos pelos desenvolvedores de acordo com a lógica de negócios e as regras esperadas para a aplicação.
    - Crie uma nova pasta chamada **`tests`** no diretório **`human-gov-application/src`**
    - Crie os arquivos [`app.py`](http://tests.py) e `test_app.py` na pasta `tests` e adicione o código a seguir a eles`app.py`:
        
        ```python
        def home_page(state):
            return 'Human Resources Management System - State of ' + state
        ```
        
        `test_app.py`:
        
        ```python
        import pytest
        from app import home_page
        
        def test_home_page():
            assert home_page('California') == 'Human Resources Management System - State of California'
        ```
        
    - Adicione a biblioteca `pytest` ao `requirements.txt` para que a biblioteca possa ser instalada na imagem Docker

1. **Adicionar a Etapa de Testes ao Pipeline CI/CD**
    - No **`humangov-cicd-pipeline`**, adicione uma nova etapa e ação chamada **`HumanGovTest`** logo após o **`HumanGovBuild`**.
    - Para a especificação de testes, use o seguinte **`buildspec.yml`**:
    - **Atualize a linha em vermelho** com a sua linha de autenticação ECR.
    
    ```yaml
    version: 0.2
    
    phases:
      build:
        commands:
          - # Get Image URI
          - cd src
          - IMAGE_URI=$(jq -r '.[0].imageUri' imagedefinitions.json)
          - echo $IMAGE_URI
          # Log in to Amazon ECR.
          - echo Logging in to Amazon ECR...
          - aws --version
          **- aws ecr-public get-login-password --region $AWS_DEFAULT_REGION | docker login --username AWS --password-stdin public.ecr.aws/l4c0j8h9**  
          # Pull the Docker image from ECR.
          - docker pull $IMAGE_URI
          # Run pytest in the Docker container.
          - docker run --rm $IMAGE_URI pytest tests/
    ```
    
    - **Adicione a permissão AmazonElasticContainerRegistryPublicFullAccess ao ECR na função de serviço codebuild-HumanGovTest-service-role**
    - Faça commit e push das alterações para executar o pipeline e observe a etapa **`HumanGovTest`** sendo executada imediatamente após a etapa **`HumanGovBuild`**.
    
    ```bash
    git add -A
    git commit -m "added tests"
    git push
    ```
    
2. **Alterar os scripts de teste para que eles possam falhar e interromper o processo de CI.**
    - Altere a linha 2 no arquivo `tests/app.py` conforme abaixo. Remova o "H" de HumanGov para forçar o processo de teste a falhar, simulando um erro do desenvolvedor que resultará em um bug de software que deve ser detectado pelo processo de teste. Faça um commit e faça o push das alterações.
    
    ```python
    def home_page(state):
        return **'uman** Resources Management System - State of ' + state
    ```
    
    ```bash
    git add -A
    git commit -m "simulating a dev mistake"
    git push
    ```
    
    - Observe que o teste falhará e não implantará o código na implantação do Kubernetes.
    - Reverta as alterações no arquivo **`tests/tests.py`** de acordo com o abaixo. Faça um commit e faça o push novamente.
    
    ```python
    def home_page(state):
        return 'Human Resources Management System - State of ' + state
    ```
    
    ```bash
    git add -A
    git commit -m "fixed dev mistake"
    git push
    ```
    
    
# **Parte 3**

## Introdução

O processo de integração contínua e implantação contínua (CI/CD) é essencial para a entrega simplificada e consistente de atualizações de software. Este projeto prático fornecerá instruções para automatizar a construção e implantação da aplicação HumanGov SaaS em um cluster Kubernetes usando o AWS CodeCommit, CodeBuild e CodePipeline.

## Pré-requisitos

- Um cluster Kubernetes chamado `humangov-cluster` com pelo menos um estado da aplicação HumanGov implantado em execução no Amazon EKS. Se você tiver destruído seu cluster, siga as instruções do Módulo 6 para provisioná-lo novamente. Não se esqueça de destruí-lo quando terminar este projeto prático.
- Um repositório AWS CodeCommit pré-existente chamado `human-gov-application`que contém o código-fonte da aplicação HumanGov SaaS.

## Instruções Detalhadas

## Parte 3: **Simular Entrega Contínua adicionando uma etapa de Aprovação Manual antes de ir para Produção em vez de Implantação Contínua**

*“Entrega Contínua vs Implantação Contínua: Principais Diferenças. Simplificando, a Entrega Contínua concentra-se em garantir que o software esteja sempre pronto para ser lançado com aprovação manual, enquanto a Implantação Contínua automatiza o processo de lançamento, implantando automaticamente as mudanças na produção uma vez que os testes são aprovados.”*

<img width="1165" height="626" alt="image" src="https://github.com/user-attachments/assets/778d58e7-0645-4a22-8f99-9261b28afef2" />


---

<img width="1152" height="642" alt="image" src="https://github.com/user-attachments/assets/7634d5e8-2b75-45a3-992d-b8c22ce20f8c" />


---

1. Adicione o novo tenant `staging` a aplicação HumanGov para ser o nosso ambiente “staging” 
    1. Provisionar o DynamoDB e o bucket S3 para a aplicação de encenação com o Terraform e anotar os nomes dos recursos. Abra o arquivo Terraform **`human-gov-infrastructure/terraform/variables.tf`** no Editor do Cloud9 e adicione "staging" à lista de estados.
    
    ```bash
    variable "states" {
      description = "The list of state names"
      default     = ["sao-paulo","rio-de-janeiro", "staging"]
    }
    ```
    
    Aplique a configuração Terraform
    
    ```bash
    cd /home/ec2-user/environment/human-gov-infrastructure/terraform
    terraform plan
    terraform apply
    ```
    
    b. Duplique o arquivo de implantação Kubernetes
    
    ```bash
    cd /home/ec2-user/environment/human-gov-application/src
    cp humangov-sao-paulo.yaml humangov-staging.yaml
    ```
    
    c. Abra o arquivo **`humangov-staging.yaml`** e substitua todas as entradas de **`sao-paulo`** por **`staging`** faça o replace com um editor.
    
    d. Atualize o nome do AWS_BUCKET para o nome do bucket de Staging no arquivo **`humangov-staging.yaml`** E substitua o placeholder **`CONTAINER_IMAGE`** pelo URI da imagem do ECR. Salve o arquivo.
    
    e. Implante a aplicação HumanGov para o ambiente de Staging
    
    ```bash
    kubectl apply -f humangov-staging.yaml
    ```
    
    f. Abra o arquivo `humangov-ingress-all.yaml`no Cloud9 e adicione a regra de Ingress para o ambiente de Staging. **Certifique-se de substituir o ARN do certificado em vermelho pelo ARN do seu certificado**.
    
    ```yaml
    apiVersion: networking.k8s.io/v1
    kind: Ingress
    metadata:
      name: humangov-python-app-ingress
      annotations:
        alb.ingress.kubernetes.io/scheme: internet-facing
        alb.ingress.kubernetes.io/target-type: ip
        alb.ingress.kubernetes.io/group.name: frontend
        **alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:us-east-1:xxxxxxxx:certificate/xxxxxxxxxxx**
        alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}, {"HTTPS":443}]'
        alb.ingress.kubernetes.io/ssl-redirect: '443'
      labels:
        app: humangov-python-app-ingress
    spec:
      ingressClassName: alb
      rules:
        - host: sao-paulo.humangov.click
          http:
            paths:
            - path: /
              pathType: Prefix
              backend:
                service:
                  name: humangov-nginx-service-sao-paulo
                  port:
                    number: 80
        - host: florida.humangov.click
          http:
            paths:
            - path: /
              pathType: Prefix
              backend:
                service:
                  name: humangov-nginx-service-rio-de-janeiro
                  port:
                    number: 80
        **- host: staging.humangov.click
          http:
            paths:
            - path: /
              pathType: Prefix
              backend:
                service:
                  name: humangov-nginx-service-staging
                  port:
                    number: 80**
    ```
    
    g. Implante as alterações de Ingress
    
    ```bash
    kubectl apply -f humangov-ingress-all.yaml
    ```
    
    h. Adicione a entrada DNS do Route 53 para o ambiente de Staging.
    
<img width="1107" height="623" alt="image" src="https://github.com/user-attachments/assets/9d7fb946-ddb8-416b-81dc-bd01eae44ed9" />

    
    i. Teste a aplicação (por exemplo `staging.humangov.click`- **substitua `humangov.click`pelo seu domínio**). 
    
2.  Incorpore o ambiente `staging` em nosso pipeline de CI/CD
    1. **Crie um projeto de implantação:**
        - No Code Pipeline, edite o **`human-gov-cicd-pipeline`** e adicione um novo estágio e grupo de ações logo após o estágio **`HumanGovTest`**.
        - Selecione o CodeBuild.
        - Dê a esse projeto um nome diferente (por exemplo, **`HumanGovDeployToStaging`**).
        - Configure as variáveis de ambiente AWS_ACCESS_KEY_ID e AWS_SECRET_ACCESS_KEY para as credenciais do usuário **`eks-user`** no Cloud Build, para que ele possa autenticar no cluster Kubernetes.
            
            *Observe: em um ambiente de produção do mundo real, use uma função IAM para essa finalidade. Neste projeto prático, estamos usando as credenciais **`eks-user`** diretamente para facilitar o processo, já que nosso foco é o CI/CD e não a autenticação do usuário neste momento. Esse processo de configuração no EKS é um pouco extenso. Consulte a seção de Referência e consulte "Enabling IAM principal access to your cluster".*
            
        - Para a especificação de implantação, use o seguinte **`buildspec.yml`**:
    
    ```yaml
    version: 0.2
    
    phases:
      install:
        runtime-versions:
          docker: 20
        commands:
          - curl -o kubectl https://amazon-eks.s3.us-west-2.amazonaws.com/1.18.9/2020-11-02/bin/linux/amd64/kubectl
          - chmod +x ./kubectl
          - mv ./kubectl /usr/local/bin
          - kubectl version --short --client
      post_build:
        commands:
          - aws eks update-kubeconfig --region $AWS_DEFAULT_REGION --name humangov-cluster
          - kubectl get nodes
          - ls
          - cd src
          - IMAGE_URI=$(jq -r '.[0].imageUri' imagedefinitions.json)
          - echo $IMAGE_URI
          - sed -i "s|CONTAINER_IMAGE|$IMAGE_URI|g" humangov-staging.yaml
          - kubectl apply -f humangov-staging.yaml
    ```
    
     
    
3. Adicione uma aprovação manual após o estágio **`HumanGovDeployToStaging`** para converter o pipeline de CI/CD de **Implantação Contínua** para **Entrega Contínua**.
4. Atualize o arquivo **`humangov-staging.yaml`**, substituindo a URI da imagem, na linha 18, pelo espaço reservado **`CONTAINER_IMAGE`**.
5. Faça uma alteração no código para testar o processo. Faça um commit e envie as alterações no código para acionar um novo processo de lançamento.

```bash
git add -A
git commit -m "changed home page"
git push
```

1. Corrija o BuildSpec do projeto HumanGovBuild. Adicione **`staging src/humangov-staging.yaml`** na parte inferior do arquivo BuildSpec. Tenha cuidado com a formatação.
    
    ```yaml
    ...
    artifacts:
      files: 
        - src/imagedefinitions.json
        - src/humangov-sao-paulo.yaml
        - src/humangov-rio-de-janeiro.yaml
        **- src/humangov-staging.yaml**
    ```
    
2. Abra o pipeline e clique em Release Change para executá-lo novamente

## **Capturando as Evidências do Projeto Prático**

Acesse o console do CodePipeline.

Abra o **`human-gov-cicd-pipeline`** e tire uma captura de tela de todo o seu pipeline mostrando todas as etapas criadas (Build, Test, DeployToStaging, ManualApproval, DeployToProduction) concluídas com sucesso.

# Resultado

<img width="1917" height="929" alt="image" src="https://github.com/user-attachments/assets/e4b802ff-c98e-4506-89c5-298520b23f48" />

<img width="1917" height="921" alt="image" src="https://github.com/user-attachments/assets/0fb9d366-ead2-4d7c-a575-a607ceab0227" />

<img width="1905" height="927" alt="image" src="https://github.com/user-attachments/assets/c3cc80dd-54cd-422e-928e-b48866a2773b" />


## **Destruindo o Ambiente**

1. Exclua o CodePipeline
2. Exclua todos os Build projects
3. Exclua o Kubernetes Ingress
    
    ```bash
    kubectl delete -f humangov-ingress-all.yaml
    ```
    
4. Exclua os recursos restantes da aplicação no Kubernetes
    
    ```bash
    kubectl delete -f humangov-sao-paulo.yaml
    kubectl delete -f humangov-rio-de-janeiro.yaml
    kubectl delete -f humangov-staging.yaml
    ```
    
5. Exclua o cluster EKS usando a CLI
    
    ```bash
    eksctl delete cluster --name humangov-cluster --region us-east-1
    
    ```
    
6. Aguarde até que o cluster EKS seja excluído. Verifique novamente no Console da AWS.
