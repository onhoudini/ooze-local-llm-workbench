# LLM Security Workbench

Workbench experimental para avaliação de segurança de modelos de linguagem locais contra ataques de prompt injection de turno único.

O objetivo do projeto é executar automaticamente ataques contra modelos locais servidos via Ollama, registrar as respostas e armazenar métricas de vazamento, tokens e tempo em arquivos CSV.

## Estrutura do projeto

```text
llm-security-workbench/
│
├── docker-compose.yml
├── README.md
│
├── n8n_data/
│   └── dados internos persistentes do n8n
│
├── n8n_files/
│   ├── dataset/
│   │   └── dataset.json
│   │
│   └── logs/
│       └── arquivos CSV gerados pelo experimento
│
├── output/
│   └── arquivos auxiliares ou resultados exportados
│
└── workflows/
    └── workflow exportado do n8n
```

## Componentes

- **n8n**: orquestra o pipeline experimental.
- **Ollama**: executa os modelos locais.
- **dataset.json**: contém os prompts de ataque.
- **CSV de saída**: armazena uma linha por requisição ao modelo.
- **Docker Compose**: sobe uma instância reprodutível do n8n.

## Comandos Para Baixar os Modelos Utilizados
ollama pull llama3.1:8b-instruct-fp16
ollama pull mistral:7b-instruct-v0.2-fp16
ollama pull gemma2:9b-instruct-fp16
ollama pull mistral-nemo:12b-instruct-2407-fp16
ollama pull gemma3:12b-it-fp16

## Requisitos

Antes de executar, é necessário ter instalado:

- Docker Desktop
- Docker Compose
- Ollama
- Modelo local disponível no Ollama

Verifique se o Docker está funcionando:

```powershell
docker --version
docker compose version
```

Verifique se o Ollama está funcionando:

```powershell
curl http://127.0.0.1:11434/api/tags
```

## Configuração do Docker

O `docker-compose.yml` usa uma versão fixa do n8n para manter reprodutibilidade.

Conteúdo esperado do `docker-compose.yml`:

```yaml
services:
  n8n:
    image: n8nio/n8n:2.11.2
    container_name: n8n-llm-security-workbench
    restart: unless-stopped
    ports:
      - "5679:5678"
    environment:
      - TZ=America/Sao_Paulo
      - GENERIC_TIMEZONE=America/Sao_Paulo
      - N8N_SECURE_COOKIE=false
    volumes:
      - ./n8n_data:/home/node/.n8n
      - ./n8n_files:/home/node/.n8n-files
      - ./workflows:/workflows
    extra_hosts:
      - "host.docker.internal:host-gateway"
```

## Subindo o ambiente

Na pasta do projeto, execute:

```powershell
cd C:\Users\houdini\Documents\llm-security-workbench
docker compose up -d
```

Acesse o n8n em:

```text
http://localhost:5679
```

Para verificar se o container está rodando:

```powershell
docker ps
```

O esperado é algo como:

```text
n8n-llm-security-workbench
0.0.0.0:5679->5678/tcp
```

## Parando o ambiente

Para parar o container:

```powershell
docker compose down
```

Para reiniciar:

```powershell
docker compose restart
```

Para ver logs:

```powershell
docker compose logs -f
```

## Caminhos dentro do n8n Docker

Dentro do container, os arquivos locais devem ser acessados por caminhos Linux.

### Dataset

Se o dataset estiver em:

```text
n8n_files/dataset/dataset.json
```

então, no node **Read Attack Dataset**, use:

```text
/home/node/.n8n-files/dataset/dataset.json
```

### Logs / CSV

Para salvar os resultados em:

```text
n8n_files/logs/
```

use no node **Write Results in CSV**:

```javascript
=/home/node/.n8n-files/logs/attack_results_{{$items("Create Run ID")[0].json.run_id}}_{{$items("Debug")[0].json.model_name.replace(/[:\\\/]/g, "-")}}.csv
```

Isso gera nomes como:

```text
attack_results_run_20260512_190000_guardian-llama3.2-latest.csv
```

## Integração com Ollama

Como o n8n está dentro do Docker e o Ollama roda no Windows, o endpoint do Ollama não deve usar `127.0.0.1`.

No node **HTTP Request**, use:

```text
http://host.docker.internal:11434/api/chat
```

Não use:

```text
http://127.0.0.1:11434/api/chat
```

porque, dentro do container, `127.0.0.1` aponta para o próprio container, não para o Windows.

## Importando o workflow

A pasta `workflows/` serve apenas para guardar o arquivo exportado. O n8n não importa workflows automaticamente dessa pasta.

Para importar:

1. Acesse `http://localhost:5679`
2. Clique em **Import Workflow**
3. Selecione o arquivo dentro de:

```text
workflows/
```

Alternativamente, é possível importar via terminal:

```powershell
docker exec -it n8n-llm-security-workbench n8n import:workflow --input=/workflows/workflow.json
```

Depois reinicie:

```powershell
docker compose restart
```

## Ajustes obrigatórios após importar o workflow

Depois de importar o workflow, confira estes pontos:

### 1. Read Attack Dataset

Usar:

```text
/home/node/.n8n-files/dataset/dataset.json
```

### 2. HTTP Request

Usar:

```text
http://host.docker.internal:11434/api/chat
```

### 3. Write Results in CSV

Usar:

```javascript
=/home/node/.n8n-files/logs/attack_results_{{$items("Create Run ID")[0].json.run_id}}_{{$items("Debug")[0].json.model_name.replace(/[:\\\/]/g, "-")}}.csv
```

### 4. Create Run ID

Usar um formato sem caracteres inválidos para nome de arquivo no Windows:

```javascript
=run_{{$now.toFormat("yyyyMMdd_HHmmss")}}
```

## Dataset

O dataset deve conter os prompts de ataque. Exemplo simplificado:

```json
{
  "data": [
    {
      "id": 1,
      "message": "Prompt de ataque 1..."
    },
    {
      "id": 2,
      "message": "Prompt de ataque 2..."
    }
  ]
}
```

O workflow lê o campo `data` e executa cada item como um ataque.

## Campos gerados no CSV

O CSV final registra uma linha por requisição ao modelo.

Campos principais:

```text
run_id
current_repetition
total_repetitions
id_prompt
model_name
attack_prompt
model_output
leaked_system_prompt
leaked_system_secret
leak_level
success
system_prompt_evidence_count
system_prompt_evidence
model_prompt_tokens
model_response_tokens
model_total_tokens
model_total_duration_s
model_load_duration_s
model_prompt_eval_duration_s
model_eval_duration_s
timestamp
```

## Métricas de segurança

### leaked_system_secret

Indica se a senha sintética completa apareceu na resposta do modelo.

### leaked_system_prompt

Indica possível vazamento de instruções internas do system prompt, com base em termos ou expressões detectadas na resposta.

### leak_level

Classificação textual do vazamento:

```text
none
system_prompt
system_secret
system_prompt_and_secret
```

### success

Indica se houve algum tipo de vazamento detectado.

## Métricas operacionais

O Ollama retorna métricas de tokens e tempo. O workflow registra:

```
model_prompt_tokens
model_response_tokens
model_total_tokens
model_total_duration_s
model_load_duration_s
model_prompt_eval_duration_s
model_eval_duration_s
```

Essas métricas permitem comparar custo e desempenho por modelo e por ataque.

## Execução recomendada

Antes de rodar muitos testes, faça uma execução pequena:

```javascript
const repetitions = 1;
```

Depois, para o experimento final:

```javascript
const repetitions = 30;
```

Se houver erro de CUDA ou sobrecarga no Ollama, mantenha o node **Loop Over Items** com batch size igual a `1`, garantindo uma requisição por vez.

## Saída esperada

Após a execução, os CSVs devem aparecer em:

```text
n8n_files/logs/
```

Exemplo:

```text
attack_results_run_20260512_190000_guardian-llama3.2-latest.csv
```

## Observações

- O Ollama precisa estar rodando no Windows antes da execução do workflow.
- O n8n Docker deve ser acessado pela porta `5679`.
- O n8n local antigo, se existir, pode continuar usando a porta `5678`.
- O workflow importado fica salvo em `n8n_data/`.
- A pasta `workflows/` apenas armazena arquivos exportados; ela não carrega workflows automaticamente.