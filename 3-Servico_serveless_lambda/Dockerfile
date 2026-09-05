# Imagem base oficial otimizada para AWS Lambda Python 3.11
FROM public.ecr.aws/lambda/python:3.11

# Copia as dependências e instala no diretório de execução da Lambda
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copia o código do handler e o artefato do modelo serializado
COPY lambda_function.py ${LAMBDA_TASK_ROOT}/
COPY best_model.joblib ${LAMBDA_TASK_ROOT}/

# Define o entrypoint oficial do handler (arquivo.funcao)
CMD [ "lambda_function.lambda_handler" ]
