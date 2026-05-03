# Central Inteligente de Chamados com Databricks + MongoDB

Projeto educacional de integração entre **Databricks**, **MongoDB Atlas**, **Python**, **Pandas** e **Jobs**, simulando uma central inteligente de chamados de suporte.

O objetivo é demonstrar um fluxo completo de dados:

```text
Excel fictício
   ↓
Databricks
   ↓
MongoDB: chamados_raw
   ↓
Databricks
   ↓
MongoDB: chamados_insights
   ↓
Databricks
   ↓
MongoDB: chamados_dashboard
```

---

## 1. Objetivo do projeto

Este projeto simula uma central de chamados onde os dados nascem em uma planilha Excel, são enviados para o MongoDB em formato documental e depois são processados pelo Databricks para gerar insights e métricas consolidadas.

O pipeline realiza:

1. Leitura de uma planilha Excel com chamados fictícios.
2. Conversão de cada linha em documento JSON/BSON.
3. Envio dos chamados brutos para o MongoDB Atlas.
4. Leitura dos chamados no MongoDB pelo Databricks.
5. Cálculo de risco de SLA e ação recomendada.
6. Escrita dos insights em uma segunda collection.
7. Geração de um documento agregado para dashboard.
8. Orquestração das etapas usando Databricks Job.

---

## 2. Tecnologias utilizadas

- Databricks
- Databricks Notebooks
- Databricks Jobs
- Databricks Secrets
- Databricks Volumes
- MongoDB Atlas
- MongoDB Compass
- Python
- Pandas
- PyMongo
- Certifi
- OpenPyXL

---

## 3. Estrutura recomendada do repositório

```text
central-inteligente-chamados-mongodb/
│
├── README.md
│
├── notebooks/
│   └── central-inteligente-chamados-mongodb/
│       ├── 00_setup_mongodb.ipynb
│       ├── 01_excel_para_mongodb_raw.ipynb
│       ├── 02_mongodb_raw_para_insights.ipynb
│       └── 03_mongodb_insights_para_dashboard.ipynb
│
├── samples/
│   └── chamados_suporte_ficticios.xlsx
│
└── docs/
    └── imagens/
```

---

## 4. Notebooks do projeto

### `00_setup_mongodb.ipynb`

Notebook auxiliar responsável por centralizar:

- imports comuns;
- conexão com MongoDB Atlas;
- uso de Databricks Secrets;
- criação do `client` MongoDB;
- funções reutilizáveis.

Este notebook é chamado pelos demais notebooks usando:

```python
%run ./00_setup_mongodb
```

---

### `01_excel_para_mongodb_raw.ipynb`

Responsável por ler a planilha Excel de chamados e enviar os dados brutos para a collection:

```text
chamados_raw
```

Fluxo:

```text
Excel
   ↓
Pandas DataFrame
   ↓
Documentos JSON/BSON
   ↓
MongoDB chamados_raw
```

Cada linha da planilha vira um documento MongoDB com estrutura semelhante a:

```json
{
  "_id": "CH-0001",
  "id_chamado": "CH-0001",
  "cliente": {
    "nome": "Conecta Telecom",
    "segmento": "Telecom",
    "cidade": "Goiânia",
    "uf": "GO"
  },
  "categoria": "Sistema fora do ar",
  "prioridade": "Baixa",
  "status": "Em atendimento",
  "data_abertura": "2026-04-08T10:00:00",
  "data_fechamento": null,
  "canal_origem": "Portal",
  "atendente": "Fernanda Alves",
  "descricao": "Cliente informa indisponibilidade total do sistema.",
  "qtd_interacoes": 11,
  "satisfacao": null,
  "sla_horas": 48,
  "eventos": [
    {
      "tipo": "abertura",
      "data": "2026-04-08T10:00:00"
    }
  ],
  "origem": {
    "sistema": "databricks"
  }
}
```

---

### `02_mongodb_raw_para_insights.ipynb`

Responsável por ler os documentos da collection `chamados_raw`, calcular métricas por chamado e salvar o resultado na collection:

```text
chamados_insights
```

Principais transformações:

- cálculo de tempo aberto;
- cálculo de tempo de resolução;
- classificação de risco de SLA;
- geração de ação recomendada;
- enriquecimento do documento original.

Exemplo de campos gerados:

```text
risco_sla
acao_recomendada
tempo_aberto_horas
tempo_resolucao_horas
origem.processado_por
origem.data_processamento
```

---

### `03_mongodb_insights_para_dashboard.ipynb`

Responsável por ler a collection `chamados_insights`, gerar métricas agregadas e salvar os resultados na collection:

```text
chamados_dashboard
```

Essa collection recebe dois tipos de documentos:

```text
snapshot_atual
snapshot_YYYYMMDD_HHMMSS
```

O documento `snapshot_atual` representa o estado mais recente do dashboard. Os snapshots históricos preservam execuções anteriores.

---

## 5. Collections MongoDB

O projeto utiliza três collections principais.

### `chamados_raw`

Armazena os chamados brutos vindos da planilha Excel.

### `chamados_insights`

Armazena os chamados processados pelo Databricks, com risco de SLA e ação recomendada.

### `chamados_dashboard`

Armazena métricas consolidadas para consumo por dashboard, API ou aplicação.

Exemplo de estrutura da collection `chamados_dashboard`:

```json
{
  "_id": "snapshot_atual",
  "tipo": "dashboard_chamados",
  "total_chamados": 500,
  "indicadores": {
    "total_risco_alto": 208,
    "total_risco_medio": 0,
    "total_risco_baixo": 2,
    "total_fora_sla": 194,
    "total_dentro_sla": 96,
    "percentual_risco_alto": 41.6,
    "percentual_fora_sla": 38.8,
    "percentual_dentro_sla": 19.2
  },
  "distribuicoes": {
    "por_risco_sla": {},
    "por_status": {},
    "por_prioridade": {},
    "por_categoria": {},
    "por_segmento": {}
  },
  "rankings": {
    "top_clientes_criticos": [],
    "top_categorias_criticas": []
  }
}
```

---

## 6. Regras de classificação de SLA

O projeto usa regras simples para classificar os chamados.

Status considerados abertos:

```text
Aberto
Em atendimento
Pendente
```

Regra de risco:

```text
Se o chamado está aberto e tempo_aberto_horas > sla_horas:
    risco_sla = Alto

Se o chamado está aberto e tempo_aberto_horas >= 70% do SLA:
    risco_sla = Médio

Se o chamado está aberto e ainda está confortável:
    risco_sla = Baixo

Se o chamado está encerrado e passou do SLA:
    risco_sla = Encerrado fora do SLA

Se o chamado está encerrado dentro do prazo:
    risco_sla = Encerrado dentro do SLA
```

---

## 7. Databricks Secret

A connection string do MongoDB **não deve ser escrita diretamente nos notebooks**.

O projeto usa Databricks Secrets:

```python
mongo_uri = dbutils.secrets.get(
    scope="mongodb",
    key="uri"
)
```

O secret esperado é:

```text
scope: mongodb
key: uri
```

O valor do secret deve conter a connection string do MongoDB Atlas, por exemplo:

```text
mongodb+srv://<usuario>:<senha>@<cluster>.mongodb.net/?retryWrites=true&w=majority
```

> Nunca envie a connection string real para o GitHub.

---

## 8. Dependências

As dependências usadas nos notebooks são:

```python
%pip install openpyxl "pymongo[srv]" certifi
```

Para notebooks que não leem Excel, `openpyxl` não é obrigatório. Porém, o primeiro notebook do pipeline precisa dele para abrir arquivos `.xlsx`.

---

## 9. Arquivo de entrada

O projeto utiliza um arquivo Excel fictício:

```text
chamados_suporte_ficticios.xlsx
```

Caminho usado no Databricks Volume durante os testes:

```text
/Volumes/workspace/default/chamados-suporte/chamados_suporte_ficticios.xlsx
```

Esse caminho pode ser adaptado conforme o ambiente.

---

## 10. Execução manual

A execução manual deve seguir esta ordem:

```text
01_excel_para_mongodb_raw
        ↓
02_mongodb_raw_para_insights
        ↓
03_mongodb_insights_para_dashboard
```

O notebook `00_setup_mongodb` não precisa ser executado diretamente, pois é chamado pelos outros notebooks com `%run`.

---

## 11. Execução por Databricks Job

O projeto pode ser executado como um Job com três tasks dependentes:

```text
01_excel_para_mongodb_raw
        ↓
02_mongodb_raw_para_insights
        ↓
03_mongodb_insights_para_dashboard
```

Nome sugerido para o Job:

```text
job_central_inteligente_chamados_mongodb
```

Cada task deve usar o tipo:

```text
Notebook
```

A dependência deve ser configurada assim:

```text
Task 2 depende da Task 1
Task 3 depende da Task 2
```

Resultado esperado:

```text
01_excel_para_mongodb_raw              Succeeded
02_mongodb_raw_para_insights           Succeeded
03_mongodb_insights_para_dashboard     Succeeded
```

---

## 12. Validação no MongoDB Compass

Após a execução do Job, valide no MongoDB Compass:

```text
Database:
databricks_learning
```

Collections esperadas:

```text
chamados_raw
chamados_insights
chamados_dashboard
```

Quantidade esperada no cenário de teste:

```text
chamados_raw        → 500 documentos
chamados_insights   → 500 documentos
chamados_dashboard  → pelo menos 2 documentos
```

Na collection `chamados_dashboard`, valide:

```text
snapshot_atual
snapshot_YYYYMMDD_HHMMSS
```

---

## 13. Exemplo de resultado agregado

Exemplo de saída gerada na collection `chamados_dashboard`:

```text
Total de chamados: 500
Risco alto: 208
Fora do SLA: 194
Dentro do SLA: 96
Risco baixo: 2
```

Distribuições geradas:

```text
por_risco_sla
por_status
por_prioridade
por_categoria
por_segmento
```

Rankings gerados:

```text
top_clientes_criticos
top_categorias_criticas
```

---

## 14. Segurança

Antes de subir para o GitHub, confirme:

```text
[ ] Não existe mongo_uri hardcoded
[ ] Não existe senha nos notebooks
[ ] Não existe notebook temporário de criação de secret
[ ] Outputs dos notebooks foram limpos
[ ] A planilha usada é fictícia
[ ] Não há dados reais de clientes
[ ] Não há tokens ou credenciais
```

---

## 15. Arquivos que não devem ser enviados ao GitHub

Não envie:

```text
.env
credentials.*
secrets.*
*.pem
*.key
temp_criar_secret_mongodb.ipynb
arquivos com connection string real
prints do MongoDB Atlas com senha
exports de dados reais
logs com dados sensíveis
```

---

## 16. Sugestão de `.gitignore`

```gitignore
# Credenciais
.env
*.pem
*.key
credentials.*
secrets.*

# Notebooks temporários
temp_*.ipynb
*_temp.ipynb

# Dados reais ou sensíveis
data/
real_data/
producao/
clientes/
private/

# Saídas e logs
*.log
*.tmp
erro_*.txt
saida_*.json
snapshot_*.json

# Python
__pycache__/
*.pyc

# Jupyter / Databricks
.ipynb_checkpoints/
*.dbc
```

---

## 17. Possíveis evoluções

Próximas melhorias possíveis:

1. Adicionar logs técnicos em tabela Delta.
2. Criar dashboard visual no Databricks.
3. Agendar o Job para execução automática.
4. Adicionar alertas em caso de falha.
5. Criar testes automatizados para as regras de SLA.
6. Transformar funções comuns em um pacote Python.
7. Separar ambientes de desenvolvimento e produção.
8. Criar um endpoint/API para consumir `chamados_dashboard`.
9. Adicionar camada Bronze, Silver e Gold no Databricks.
10. Criar métricas históricas por dia, semana e mês.

---

## 18. Resumo final

Este projeto demonstra uma integração completa entre Databricks e MongoDB Atlas.

O Databricks atua como motor de processamento e orquestração, enquanto o MongoDB Atlas atua como banco documental para armazenar dados brutos, insights e métricas agregadas.

A arquitetura final é:

```text
Excel fictício
   ↓
Databricks Notebook 01
   ↓
MongoDB chamados_raw
   ↓
Databricks Notebook 02
   ↓
MongoDB chamados_insights
   ↓
Databricks Notebook 03
   ↓
MongoDB chamados_dashboard
   ↓
Databricks Job
```

O projeto é uma base prática para estudos de engenharia de dados, integração com banco NoSQL, pipelines em nuvem e automação com Databricks Jobs.
