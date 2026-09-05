# Documentação Técnica e Arquitetura de MLOps: Calibração Serverless de Redes de Sensores IoT na AWS

**Pipeline de Engenharia de Dados e Calibração Dinâmica**  
*Set-2026*

---

## Resumo

Este documento descreve a arquitetura fim a fim de Engenharia de Dados e MLOps para calibração dinâmica de sensores ambientais de baixo custo em microclima florestal. O sistema integra ingestão contínua de telemetria via AWS IoT Core (MQTT e HTTPS), inferência serverless em tempo real via AWS Lambda conteinerizada (Scikit-Learn), armazenamento no Amazon DynamoDB e rotinas de auditoria de paridade determinística e monitoramento de degradação estatística (*Data & Model Drift*).

---

## 1. Visão Geral do Projeto

Sensores ambientais de baixo custo frequentemente apresentam erros sistemáticos, desvios térmicos e perda gradual de calibração ao longo do tempo. Esta solução resolve o desafio implementando um pipeline modular com quatro propriedades operacionais:

1. **Ingestão Heterogênea:** Suporte simultâneo a MQTT nativo com confirmação de entrega (QoS 1) e lotes via HTTPS REST.
2. **Inferência Determinística em Tempo de Execução:** Predição via modelo Scikit-Learn empacotado em imagem compatível com o padrão *Open Container Initiative* (OCI) no AWS Lambda, aproveitando o cache de camadas do *runtime* e a persistência do *pipeline* em memória global para minimizar a latência de inicialização (*cold start*).
3. **Blindagem Contra Data Leakage:** Particionamento temporal estrito balizado pela cronologia do sensor de bancada de referência (`Clima_01_EW`).
4. **Sustentação e Auditoria Contínua:** Monitoramento de deriva de covariáveis (teste bicaudal de Kolmogorov-Smirnov) e verificação de paridade numérica absoluta entre nuvem e ambiente de bancada ($\vert{}\hat{y}_{\text{cloud}} - \hat{y}_{\text{local}}\vert{} < 10^{-4}$).

---

---

## 2. Topologia da Arquitetura

O fluxo de dados entre os módulos de campo, a infraestrutura serverless e a camada de validação analítica opera da seguinte forma:

```text
[Sensores de Campo / Simulador IoT]
        |
        +-- (MQTT com QoS 1 / WebSockets / SigV4)
        +-- (HTTPS REST Batch / boto3)
        v
[AWS IoT Core] (Broker MQTT / Tópico: floresta/sensores/+)
        |
        v (Regra SQL / Invocação Síncrona)
[AWS Lambda] (Imagem Docker OCI via Amazon ECR)
        |   └── Pipeline: SimpleImputer + DecisionTreeRegressor
        v
[Amazon DynamoDB] (Tabela: LeiturasSensoresCalibrados)
        ^
        | (Auditoria de Paridade & Monitoramento Estatístico)
[Módulo de Auditoria Local / EventBridge Agendado]
```
---

## 3. Procedimento de Configuração na AWS

Para replicar a infraestrutura em nuvem na região `us-east-1`, execute os passos técnicos detalhados a seguir:

### 3.1. Amazon DynamoDB
1. Crie uma nova tabela denominada `LeiturasSensoresCalibrados`.
2. Defina a chave de partição (*Partition Key*): `sensor_name` (Tipo: `String`).
3. Defina a chave de ordenação (*Sort Key*): `timestamp_epoch_ms` (Tipo: `Number`).
4. Configure a capacidade no modo **On-Demand** para eliminar alocação fixa e custos com instâncias ociosas.

### 3.2. AWS Identity and Access Management (IAM)
A função de inferência `FlorestaHumidityCalibrator` depende de uma Role de execução vinculada a dois documentos de política no padrão JSON, ambos versionados no repositório na pasta `utils/`:

1. **Política de Confiança (*Trust Policy* — `utils/trust_policy.json`):** Define a relação de confiança necessária para que o serviço `lambda.amazonaws.com` assuma a identidade operacional da Role.
2. **Política de Permissões de Acesso (*Permissions Policy* — `utils/iam_dynamodb_policy.json`):** Concede as permissões estritas para telemetria no Amazon CloudWatch Logs e escrita na tabela `LeiturasSensoresCalibrados`. A utilização deste arquivo evita o erro de restrição de acesso (`AccessDeniedException`) mantendo o princípio do menor privilégio (*Least Privilege*), sem necessidade de recorrer a políticas administrativas irrestritas como a `AmazonDynamoDBFullAccess_v2`.

**Procedimento de Configuração:**
* **Via Console AWS:** Acesse o serviço IAM, selecione **Roles** -> **Create role**, escolha o caso de uso **Lambda** (a AWS aplica a política de confiança automaticamente). Em seguida, acesse a aba **Permissions**, selecione **Add permissions** -> **Create inline policy**, mude para a aba **JSON** e cole o conteúdo de `utils/iam_dynamodb_policy.json`, substituindo `<ACCOUNT_ID>` pelo identificador numérico da sua conta.
* **Via AWS CLI (Automação):** A criação da Role e a aplicação das permissões podem ser executadas diretamente pelo terminal:
  ```bash
  # 1. Cria a Role associando a Trust Policy
  aws iam create-role \
    --role-name FlorestaHumidityCalibrator-Role \
    --assume-role-policy-document file://utils/trust_policy.json

  # 2. Anexa a política de permissões com menor privilégio
  aws iam put-role-policy \
    --role-name FlorestaHumidityCalibrator-Role \
    --policy-name FlorestaDynamoCloudWatchPolicy \
    --policy-document file://utils/iam_dynamodb_policy.json
    
### 3.3. Amazon ECR e AWS Lambda (Container OCI)
Para empacotar as bibliotecas de Machine Learning sem esbarrar nos limites de tamanho das camadas *zip*, utilize a imagem Docker:

```bash
# 1. Autenticar no Amazon ECR
aws ecr get-login-password --region us-east-1 | docker login \
  --username AWS --password-stdin <ACCOUNT_ID>.dkr.ecr.us-east-1.amazonaws.com

# 2. Compilação e tagueamento
docker build -t floresta-calibrator:latest 3-servico_serverless_lambda/
docker tag floresta-calibrator:latest \
  <ACCOUNT_ID>[.dkr.ecr.us-east-1.amazonaws.com/floresta-calibrator:latest](https://.dkr.ecr.us-east-1.amazonaws.com/floresta-calibrator:latest)

# 3. Publicação
docker push <ACCOUNT_ID>[.dkr.ecr.us-east-1.amazonaws.com/floresta-calibrator:latest](https://.dkr.ecr.us-east-1.amazonaws.com/floresta-calibrator:latest)
```
No console do AWS Lambda, crie uma função selecionando a opção *Container image*, aponte para o repositório ECR criado e ajuste a memória para *512 MB* com timeout de *15 segundos*.

### 3.4. AWS IoT Core (Motor de Regras)
Crie uma regra de roteamento no AWS IoT Core (*Message Routing*) para direcionar a telemetria à Lambda:
* **Consulta SQL:** `SELECT * FROM 'floresta/sensores/+'`
* **Ação:** Invocação da função Lambda provisionada.

## 4. Estrutura de Diretórios do Repositório

O projeto adota uma arquitetura em quatro pastas isoladas por domínio de responsabilidade:

```text
calibracao-sensores-aws-iot/
|
+-- requirements.txt                     # Dependências globais de desenvolvimento
+-- README.md                            # Documentação técnica do repositório
|
+-- utils/
|   +-- trust_policy.json                # Relação de confiança (STS:AssumeRole para Lambda)
|   +-- iam_dynamodb_policy.json         # Permissões estritas de escrita e telemetria
|
+-- 1-ingestao_simulador/
|   +-- 1.1-data_generator.py            # Extração remota e criação das camadas SQLite
|   +-- 1.2-publisher_iot_https.py       # Publicador em lote via HTTPS (boto3)
|   +-- 1.3-publisher_iot_mqtt.py        # Publicador de telemetria via MQTT (awsiotsdk)
|   +-- tabela_prata_dados_sensores.db   # Base de dados local com target de bancada
|
+-- 2-treinamento_modelo/
|   +-- 2.1-train_calibrator.py          # Treinamento com corte temporal não-vazante
|   +-- best_model.joblib                # Artefato Scikit-Learn serializado
|
+-- 3-servico_serverless_lambda/
|   +-- Dockerfile                       # Imagem OCI para runtime Python 3.11
|   +-- lambda_function.py               # Handler de inferência e persistência
|   +-- best_model.joblib                # Artefato acoplado ao container
|   +-- requirements.txt                 # Dependências mínimas de produção
|
+-- 4-validacao_auditoria/
    +-- 4.1-auditoria_pipeline.py        # Auditoria de paridade (DynamoDB vs Local)
    +-- 4.2-drift_monitoring.py          # Avaliação de Data Drift e Model Drift
    +-- 4.3-analise_graficos.ipynb       # Notebook de figuras para o artigo
```

## 5. Detalhamento Funcional dos Módulos

* **`utils/`:** Contém os manifestos declarativos de segurança em nuvem: a relação de confiança do serviço (`trust_policy.json`) e a política de privilégios mínimos estritos (`iam_dynamodb_policy.json`) para telemetria no CloudWatch e persistência no DynamoDB.
* **`1.1-data_generator.py`:** Realiza conexão com a base corporativa por meio de túnel SSH seguro, isola o intervalo das leituras, efetua a interpolação temporal por janela do sensor de bancada (`Clima_01_EW`) e constrói as camadas locais Bronze e Prata em bancos SQLite. Possui rotinas para fallback interativo no terminal caso as credenciais não estejam definidas no ambiente.
* **`1.2-publisher_iot_https.py`:** Realiza leitura na camada Prata, seleciona os nós experimentais (`Floresta_08` a `Floresta_14`) e dispara cargas em lote via chamadas REST para o endpoint ATS da AWS.
* **`1.3-publisher_iot_mqtt.py`:** Simula a transmissão física dos dispositivos de borda via protocolo MQTT nativo sobre WebSockets com autenticação SigV4 e garantia de entrega (**QoS 1**).
* **`2.1-train_calibrator.py`:** Implementa a partição contínua 70% Treino / 20% Validação / 10% Teste baseando-se estritamente na linha do tempo da bancada de referência. Treina o pipeline de regressão (`SimpleImputer` + `DecisionTreeRegressor`), extrai importâncias físicas e gera o artefato `best_model.joblib`.
* **`3-servico_serverless_lambda/Dockerfile`:** Constrói o container da função serverless a partir da imagem base da AWS, copiando apenas as dependências mínimas necessárias para a inferência.
* **`3-servico_serverless_lambda/lambda_function.py`:** Código executado na nuvem. Mantém o modelo Scikit-Learn instanciado em memória global (redução drástica de latência), converte floats para o padrão `Decimal` exigido pelo DynamoDB e persiste a leitura bruta paralelamente à calibrada.
* **`4.1-auditoria_pipeline.py`:** Faz varredura na tabela do DynamoDB e reproduz a predição localmente a partir das mesmas entradas brutas, comprovando que a execução serverless é determinística ($|\text{divergência}| < 10^{-4}$).
* **`4.2-drift_monitoring.py`:** Avalia deriva de covariáveis (*Data Drift*) através do teste estatístico de Kolmogorov-Smirnov sobre umidade, temperatura e $CO_2$, e analisa a perda de calibração (*Model Drift*) monitorando as variações de MAE e $R^2$.
* **`4.3-analise_graficos.ipynb`:** Notebook padronizado para geração de figuras com qualidade de publicação científica (300 DPI), abrangendo séries temporais comparativas, dispersão de identidade 1:1, funções de densidade de resíduos (KDE) e matrizes de correlação cruzada.
    
## 6. Guia de Execução Sequencial

### 6.1. Ambiente Virtual e Dependências
```bash
git clone [https://github.com/seu-usuario/calibracao-sensores-aws-iot.git](https://github.com/seu-usuario/calibracao-sensores-aws-iot.git)
cd calibracao-sensores-aws-iot
python3 -m venv venv
source venv/bin/activate   # Windows: venv\Scripts\activate
pip install -r requirements.txt
```

### 6.2. Ciclo Operacional do Pipeline
1. **Geração das Camadas Locais (Bronze e Prata):**
   ```bash
   python 1-ingestao_simulador/1.1-data_generator.py
   ```

2. Treinamento do Calibrador sem Vazamento Temporal:

```Bash
python 2-treinamento_modelo/2.1-train_calibrator.py
```

3. Disparo de Telemetria (Simulação de Nós de Borda):

```Bash
# Via chamada HTTPS em lote:
python 1-ingestao_simulador/1.2-publisher_iot_https.py

# Ou via protocolo MQTT com QoS 1:
python 1-ingestao_simulador/1.3-publisher_iot_mqtt.py
```

4. Auditoria de Paridade Nuvem vs. Local:

```Bash
python 4-validacao_auditoria/4.1-auditoria_pipeline.py
```

5. Monitoramento de Deriva Estatística (Drift):

```Bash
python 4-validacao_auditoria/4.2-drift_monitoring.py
```

6. Renderização dos Gráficos:

```Bash
python 4-validacao_auditoria/4.3-analise_graficos.py
```
## 7. Tabela de Variáveis de Ambiente

Para automação de execuções em servidores ou rotinas de CI/CD, configure as seguintes variáveis no sistema operacional:

| Variável de Ambiente | Finalidade Operacional | Padrão |
| :--- | :--- | :--- |
| `AWS_ACCESS_KEY_ID` | Chave pública de autenticação IAM | *Prompt no terminal* |
| `AWS_SECRET_ACCESS_KEY` | Chave secreta de autenticação IAM | *Prompt mascarado* |
| `AWS_DEFAULT_REGION` | Região de execução dos serviços AWS | `us-east-1` |
| `AWS_IOT_ENDPOINT` | Endpoint ATS do broker AWS IoT Core | *Prompt no terminal* |
| `DYNAMO_TABLE_NAME` | Tabela de armazenamento das leituras | `LeiturasSensoresCalibrados` |
| `SQLITE_PRATA_PATH` | Caminho local do banco da camada Prata | `tabela_prata_dados_sensores.db` |
