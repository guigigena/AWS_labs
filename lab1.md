-----

# Guia de Laboratório: Construção de uma Aplicação Web na AWS

Este guia prático e passo a passo o levará pela construção de uma arquitetura segura e escalável utilizando serviços da AWS como **VPC, Fargate, Cognito, API Gateway, S3** e **CloudFront**. O objetivo é que você entenda a interconexão de cada componente e como o fluxo de dados se move do frontend para o backend de forma segura, utilizando uma função **Lambda como autorizador**.

-----

## Diagramas de Referência

Para uma melhor visualização dos componentes e seus fluxos, consulte os diagramas a seguir.

### Diagrama de Arquitetura de Alto Nível

```
+--------------+                                                                                              +---------------+
| Usuário      |                                                                                              | Serviços AWS  |
+------+-------+                                                                                              +---------------+
       |                                                                                                             |
       |  (1) Acesso ao Frontend                                                                                     |
       v                                                                                                             |
+----------------+          (3) Autenticação (JWT)  +----------------+                                              |
| CloudFront/S3  | <------------------------------+ | Amazon Cognito |                                              |
| (Frontend)     |                                  +----------------+                                              |
+----------------+                                                                                                   |
       |                                                                                                             |
       |  (2) Chamada de API Protegida                                                                               |
       v                                                                                                             |
+----------------+           (4) Invoca Lambda           +------------------+                                      |
| API Gateway    |------------------------------------> | Lambda           |                                       |
+----------------+           (Validação JWT)              | Authorizer       |                                       |
       |                      (5) Retorna Permissão      +------------------+                                       |
       |                                                                                                             |
       | (6) Integração com o Backend          +------------------+     +-----------------+     +----------------+     |
       +-------------------------------------> | Application Load |     | VPC Link        |     | AWS Fargate    |     |
                                                | Balancer (ALB)   | <-> | (Comunicação    | <-> | (Backend API)  |     |
                                                +------------------+     | Privada)        |     |                |     |
                                                                         +-----------------+     +----------------+     |
```

-----

## Pré-requisitos

Para seguir este guia, você precisará de:

  * Uma conta da AWS.
  * O **AWS CLI** configurado localmente.
  * O **Docker** instalado e funcionando.
  * Um editor de código (VS Code, por exemplo).
  * Um navegador web para acessar o Console da AWS.

-----

## Passo 1: Configuração da Infraestrutura de Rede (VPC)

A **VPC (Virtual Private Cloud)** é a base da nossa arquitetura. Ela garante que nossos recursos estejam isolados e seguros.

  * **Crie a VPC:** No console da AWS, navegue até o serviço **VPC**. Crie uma nova VPC com o nome `lab-arch-vpc` e o bloco CIDR `10.0.0.0/16`.
  * **Crie as Sub-redes:** Crie duas sub-redes públicas (`lab-public-subnet-1` e `lab-public-subnet-2`) e duas sub-redes privadas (`lab-private-subnet-1` e `lab-private-subnet-2`). Associe as públicas a um **Internet Gateway** e as privadas a um **NAT Gateway**.

-----

## Passo 2: O Backend com Fargate

Aqui, você irá preparar o código do backend e implantá-lo em um contêiner no AWS Fargate.

1.  **Crie o Código da Aplicação de Teste:** Crie uma pasta `backend` com os arquivos `package.json`, `server.js` e `Dockerfile`. O código do `server.js` pode ser mantido simples, pois a validação do token será feita pela função Lambda, e não pela aplicação.
2.  **Construa e Envie a Imagem para o ECR:** No console, crie um repositório ECR chamado `lab-backend-repo`. Use o AWS CLI para fazer o login no Docker e, em seguida, construir e enviar a sua imagem para o repositório.
3.  **Crie um Cluster ECS e o Serviço Fargate:**
      * Navegue até o serviço **ECS**.
      * Crie um "Cluster" do tipo **Fargate**.
      * Crie uma "Task Definition" para o seu contêiner, usando a imagem do ECR.
      * Crie um "Service" baseado na sua "Task Definition".
      * **Importante:** Vincule este serviço a um **Application Load Balancer (ALB)**. O ALB deve estar nas suas **sub-redes públicas**, e o Fargate nas suas **sub-redes privadas**.

-----

## Passo 3: A Camada de API com Lambda Authorizer (NOVO)

Esta é a etapa mais crítica, onde você irá criar a função Lambda para validar o token e configurar o API Gateway para usá-la.

1.  **Crie a Função Lambda (Authorizer):**

      * No serviço **Lambda**, crie uma nova função (`lab-authorizer-lambda`) com o runtime **Node.js 18.x**.
      * Cole o código abaixo no arquivo `index.js`. **Não se esqueça de substituir os placeholders com seus IDs do Cognito\!**

    <!-- end list -->

    ```javascript
    const jwt = require('jsonwebtoken');
    const jwkToPem = require('jwk-to-pem');
    const axios = require('axios');

    // SUBSTITUA COM SEUS VALORES
    const userPoolId = 'SEU_USER_POOL_ID';
    const appClientId = 'SEU_APP_CLIENT_ID';
    const region = 'SUA_REGIAO'; // ex: 'us-east-1'

    const iss = `https://cognito-idp.${region}.amazonaws.com/${userPoolId}`;
    let pems;

    const validateToken = async (token) => {
        if (!pems) {
            try {
                const response = await axios.get(`${iss}/.well-known/jwks.json`);
                pems = response.data.keys.reduce((acc, key) => {
                    acc[key.kid] = jwkToPem({
                        kty: key.kty,
                        n: key.n,
                        e: key.e
                    });
                    return acc;
                }, {});
            } catch (error) {
                console.error("Erro ao obter JWKS", error);
                throw new Error('Unauthorized');
            }
        }

        const decodedJwt = jwt.decode(token, { complete: true });
        if (!decodedJwt) {
            throw new Error('Unauthorized');
        }

        const kid = decodedJwt.header.kid;
        const pem = pems[kid];
        if (!pem) {
            throw new Error('Unauthorized');
        }

        const verified = jwt.verify(token, pem, { issuer: iss, audience: appClientId });
        if (!verified) {
            throw new Error('Unauthorized');
        }
    };

    exports.handler = async (event) => {
        const token = event.headers.authorization;

        if (!token || !token.startsWith('Bearer ')) {
            return {
                isAuthorized: false,
                context: {
                    message: 'Token não fornecido ou inválido'
                }
            };
        }

        const jwtToken = token.split(' ')[1];

        try {
            await validateToken(jwtToken);
            return { isAuthorized: true, context: { message: 'Acesso autorizado' } };
        } catch (error) {
            console.error("Erro de validação:", error.message);
            return { isAuthorized: false, context: { message: 'Acesso negado' } };
        }
    };
    ```

      * Instale as dependências (`jsonwebtoken`, `jwk-to-pem`, `axios`) usando o `package.json` na sua função Lambda.

2.  **Crie a API Gateway (do tipo correto):**

      * Vá para o serviço **API Gateway** e clique em **"Create API"**.
      * Escolha o tipo **"HTTP API"** e clique em "Build". **(Não escolha REST API\!)**

3.  **Configure o Autorizador Lambda:**

      * No menu lateral da sua nova API, vá para **"Authorizers"**.
      * Clique em **"Create"** (ou "Create Authorizer") e configure um novo autorizador do tipo **"Lambda"**.
      * Selecione a função Lambda (`lab-authorizer-lambda`) que você acabou de criar.
      * **Identity token source:** `Authorization`.

4.  **Crie as Rotas e Integrações:**

      * No menu lateral, vá para **"Routes"**.
      * Crie a rota `GET /protected`.
      * Clique na rota e anexe o `Lambda Authorizer` que você criou.
      * Depois, crie a integração da rota `GET /protected` com o seu **Application Load Balancer (ALB)**, usando um **VPC Link**.

-----

## Passo 4: Autenticação com Amazon Cognito

Aqui você criará o diretório de usuários, mas sem a configuração de autorizador no API Gateway, pois isso é responsabilidade da sua função Lambda.

1.  **Crie um Pool de Usuários (User Pool):**
      * No serviço **Cognito**, crie um novo "User Pool".
      * Defina **"Email"** como método de login e configure a política de senhas.
2.  **Crie um App Client:**
      * Crie um "App client" no seu User Pool.
      * **Anote o App client ID e o User Pool ID.** Estes valores são essenciais para o código da sua função Lambda e para o arquivo `app.js` do frontend.

-----

## Passo 5 (Alternativo): O Frontend com AWS Amplify Hosting

Esta etapa substitui a configuração manual de S3 e CloudFront, automatizando a hospedagem e a implantação contínua (CI/CD) de forma muito mais simples.

1.  **Prepare e Envie o Código para um Repositório Git:**
      * Crie um novo repositório Git (no **GitHub** ou **GitLab**) para o seu projeto frontend.
      * Envie seus arquivos (`index.html`, `app.js`, `style.css`) para o novo repositório.
2.  **Crie a Aplicação no AWS Amplify:**
      * No console da AWS, navegue até o serviço **AWS Amplify**.
      * Clique em **"New app"** e selecione **"Host web app"**.
      * Escolha seu provedor Git e conecte sua conta.
      * Selecione o repositório e a branch (`main`, por exemplo).
3.  **Obtenha a URL do Site:**
      * Após a implantação ser concluída (status "Deployed"), a URL do seu site estará disponível na página principal da aplicação. Copie esta URL para testar o seu laboratório.

-----

## Passo 6: Teste e Validação

Agora é a hora de verificar se tudo está funcionando.

  * Acesse a **URL do seu site no AWS Amplify**.
  * No console do Cognito, crie um novo usuário de teste.
  * Use as credenciais desse usuário para fazer o login na aplicação.
  * Clique no botão **"Chamar API Protegida"**.

Se tudo estiver correto, a mensagem "This is a protected endpoint\! Only authenticated users can see this." deve aparecer, indicando que o fluxo de autenticação e a chamada da API funcionaram corretamente.
