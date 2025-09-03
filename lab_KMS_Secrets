Aqui está o conteúdo do arquivo Markdown que descreve o laboratório real na AWS, pronto para ser copiado e usado.

-----

### Caso de Uso: Laboratório Real AWS Secrets Manager & KMS

Este guia prático e passo a passo simula uma aplicação que precisa de uma senha de banco de dados de forma segura, sem armazená-la diretamente no código.

-----

### Pré-requisitos

  - Uma conta AWS.
  - Credenciais de usuário IAM com permissões de administrador ou permissões específicas para **IAM**, **KMS** e **Secrets Manager**.
  - O AWS CLI configurado em sua máquina local para testar a aplicação de exemplo.

-----

### Passo 1: Criar uma Chave de Criptografia (KMS Key)

Esta chave será usada pelo Secrets Manager para criptografar seu segredo.

1.  **Navegue até o KMS:** Abra o console da AWS e pesquise por "KMS".
2.  **Crie a Chave:** No painel de navegação esquerdo, selecione **"Customer managed keys"** e clique em **"Create key"**.
3.  **Configurações:**
      - Escolha **"Symmetric"** e **"Encrypt and decrypt"**.
      - Dê um apelido (alias) para a chave, por exemplo: `meu-lab-chave-kms`.
      - Defina as permissões de uso e administração da chave para o seu usuário IAM.
4.  **Finalize a Criação:** Revise e finalize a criação da chave. O alias dela será algo como `alias/meu-lab-chave-kms`.

-----

### Passo 2: Armazenar o Segredo no Secrets Manager

Agora, você armazenará a senha do banco de dados de teste. O Secrets Manager irá criptografá-la automaticamente com a chave KMS que você acabou de criar.

1.  **Navegue até o Secrets Manager:** Abra o console da AWS e pesquise por "Secrets Manager".
2.  **Armazene um novo segredo:** Clique em **"Store a new secret"**.
3.  **Configurações do segredo:**
      - Em **"Secret type"**, selecione **"Credentials for Amazon RDS database"**.
      - Preencha os campos `Username` e `Password` com valores de teste (ex: `user_lab` e `P@ssw0rd!123`).
      - Dê um nome para o segredo, por exemplo: `meu-lab-senha-db`.
4.  **Escolha a chave KMS:** Na seção **"Encryption key"**, selecione a chave KMS que você criou no Passo 1 (`alias/meu-lab-chave-kms`).
5.  **Finalize:** Siga os passos e finalize a criação. Você verá seu novo segredo listado.

-----

### Passo 3: Criar um Perfil IAM para a Aplicação

Para seguir o princípio do menor privilégio, sua aplicação deve ter apenas permissão para ler o segredo.

1.  **Navegue até o IAM:** Abra o console da AWS e pesquise por "IAM".
2.  **Crie uma nova política:**
      - No painel de navegação, selecione **"Policies"** e clique em **"Create policy"**.
      - Vá para a aba **"JSON"** e insira o seguinte código. **Lembre-se de substituir** o `<seu-secret-arn>` pelo ARN do seu segredo e o `<sua-chave-kms-arn>` pelo ARN da sua chave KMS.
    <!-- end list -->
    ```json
    {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Effect": "Allow",
                "Action": "secretsmanager:GetSecretValue",
                "Resource": "<seu-secret-arn>"
            },
            {
                "Effect": "Allow",
                "Action": "kms:Decrypt",
                "Resource": "<sua-chave-kms-arn>"
            }
        ]
    }
    ```
      - Dê um nome para a política (ex: `meu-lab-policy-secrets`) e salve.
3.  **Crie um usuário IAM para a aplicação:**
      - No IAM, selecione **"Users"** e clique em **"Create user"**.
      - Dê um nome ao usuário (ex: `app_secrets_user`) e anexe a política que você criou a este usuário.

-----

### Passo 4: Testar a Aplicação de Exemplo com AWS CLI

Você simulará o comportamento de sua aplicação usando a AWS CLI para se comunicar com o Secrets Manager.

1.  **Configure o Ambiente:**
      - Abra seu terminal e instale a biblioteca `boto3` para Python: `pip install boto3`.
      - Configure o AWS CLI com as credenciais do usuário IAM que você criou no Passo 3. Execute o comando `aws configure`.
2.  **Execute o Teste:**
      - No seu terminal, execute o seguinte comando, **substituindo o nome do segredo** pelo que você definiu:
    <!-- end list -->
    ```bash
    aws secretsmanager get-secret-value --secret-id meu-lab-senha-db
    ```
3.  **Verifique o Resultado:**
      - Se tudo estiver configurado corretamente, a saída do comando será a senha decriptografada. Isso prova que sua aplicação consegue acessar o segredo de forma segura, sem ter que gerenciar a criptografia por conta própria.

-----

### Passo 5: Limpar os Recursos

Para evitar cobranças, é fundamental remover os recursos que você criou.

1.  **Delete o Segredo:** No console do Secrets Manager, selecione seu segredo e escolha **"Delete secret"**.
2.  **Desative a Chave KMS:** No console do KMS, selecione sua chave e, em **"Key actions"**, escolha **"Schedule key deletion"**.
3.  **Delete a Política e o Usuário IAM:** No console do IAM, exclua a política e o usuário que você criou para este laboratório.
