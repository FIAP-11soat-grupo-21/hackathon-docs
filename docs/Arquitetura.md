# Documentação de Arquitetura: Sistema Distribuído de Processamento de Vídeo

## 1. Visão Geral do Sistema

O sistema é uma plataforma de processamento de vídeo altamente escalável baseada em nuvem (AWS). O objetivo principal é receber uploads de vídeos fragmentados (chunked), extrair os frames de cada fragmento e, ao final, compilar todas as imagens geradas em um arquivo `.zip` para download pelo usuário.

A solução utiliza uma arquitetura de **Microserviços** combinada com computação **Serverless**. A comunicação inter-serviços é estritamente **Orientada a Eventos (Event-Driven)**, utilizando o padrão **Saga Coreografado** para gerenciar o estado e o ciclo de vida do processamento do vídeo sem a necessidade de um orquestrador central, garantindo baixo acoplamento e alta disponibilidade.

## 2. Componentes da Arquitetura (Microserviços)

### 2.1. Auth Service (Edge & Security)

Atua como a camada de proteção e entrada da aplicação.

* **Responsabilidade:** Autenticação e autorização de usuários.
* **Integrações AWS:**
* **Amazon Cognito:** Gerenciamento do pool de usuários, emissão e validação de tokens JWT.
* **Amazon API Gateway:** Exposição dos endpoints. Valida os tokens do Cognito antes de rotear o tráfego para os serviços internos, garantindo que a API seja privada e segura.



### 2.2. User Service

* **Responsabilidade:** Gerenciamento do domínio do usuário.
* **Operações:** CRUD de dados cadastrais e preferências do usuário.

### 2.3. Video Solicitation Service (Core de Domínio)

Gerencia o estado lógico da transação do vídeo.

* **Responsabilidades:**
* Salvar e atualizar os metadados e o status geral do vídeo e de seus respectivos chunks.
* Gerar e retornar URLs pré-assinadas (Pre-signed URLs) para o upload direto e seguro dos chunks de vídeo para o **Amazon S3**, desonerando o backend do tráfego pesado.
* Fornecer endpoints de leitura (listar vídeos do usuário, verificar status, obter URL de download do ZIP final).


* **Integrações AWS:** Publica e consome eventos via **Amazon SNS** e **Amazon SQS** para atualizar o status do processamento (Saga).

### 2.4. Video Processor Service (Workers Serverless)

Responsável pelo trabalho computacional intensivo. Desenvolvido como funções isoladas e efêmeras.

* **Responsabilidades & Componentes:**
* **Worker de Extração (AWS Lambda):** Ativado por uma fila SQS (inscrita em um tópico SNS). Baixa o chunk do S3, processa a extração de frames utilizando **FFmpeg** e faz o upload das imagens geradas de volta para o S3. Emite um evento de sucesso ou falha.
* **Worker de Empacotamento (AWS Lambda):** Ativado por SQS/SNS quando todos os chunks de um vídeo são concluídos. Faz o download das imagens finais do S3, gera o arquivo `.zip` consolidado e faz o upload do artefato final.



### 2.5. Notification Service

* **Responsabilidade:** Comunicação transacional com o cliente.
* **Operações:** Escuta as filas do SQS (via tópicos SNS) aguardando eventos de conclusão (sucesso) ou falha (compensação) do Saga. Envia e-mails de notificação ao usuário contendo, por exemplo, o link para download do ZIP final.

---

## 3. Padrão Arquitetural: Event-Driven & Saga Coreografado

O sistema elimina chamadas síncronas (HTTP REST) entre os microserviços para o fluxo de processamento. Em vez disso, utiliza o padrão **Publisher/Subscriber (Pub/Sub)** através de Tópicos (Amazon SNS) e Filas (Amazon SQS).

Isso cria um **Saga Coreografado**, onde cada serviço reage a eventos e produz novos eventos, impulsionando a máquina de estados para frente ou engatilhando ações de compensação em caso de erro.

### Fluxo de Dados (Happy Path)

1. **Solicitação:** O cliente autenticado solicita o upload de um vídeo ao *Video Solicitation*.
2. **Upload:** O *Video Solicitation* registra o status `PENDING` e retorna as URLs do S3. O cliente faz o upload dos chunks diretamente para o S3.
3. **Evento de Chunk Recebido:** Ao finalizar o upload de um chunk, um evento `chunk-uploaded` é publicado no SNS.
4. **Processamento Paralelo:** O *Video Processor* (Worker de Extração) consome o evento via SQS, extrai as imagens com FFmpeg e publica `chunk-precessed`.
5. **Atualização de Estado:** O *Video Solicitation* escuta `chunk-precessed` e atualiza o status do chunk.
6. **Geração do ZIP:** Quando o *Video Solicitation* identifica que todos os chunks de um vídeo foram processados, ele publica o evento `all-chunk-processed`.
7. **Empacotamento:** O *Video Processor* (Worker de Empacotamento) reage a este evento, gera o `.zip`, salva no S3 e publica `video-processing-completed`.
8. **Finalização e Notificação:**
* O *Video Solicitation* atualiza o status final do vídeo para `COMPLETED`.
* O *Notification Service* reage ao evento `video-processing-completed` e envia um e-mail ao usuário com o link para baixar o arquivo.

---

## 4. Estratégias de Escalabilidade e Resiliência

* **Padrão Fanout (SNS + SQS):** A combinação de SNS com SQS garante que as mensagens não sejam perdidas caso um serviço fique indisponível, além de permitir múltiplos consumidores independentes para o mesmo evento.
* **Processamento Assíncrono e Paralelo:** O uso de chunks permite que a Lambda (Video Processor) seja instanciada dezenas ou centenas de vezes simultaneamente, processando pedaços do mesmo vídeo em paralelo e reduzindo drasticamente o tempo total.
* **Upload Direto ao S3:** Ao utilizar URLs pré-assinadas, o tráfego de rede pesado (I/O de vídeo) não passa pela API, economizando custos de computação e evitando gargalos de rede (bottlenecks).


---

## 5. Fluxos de Compensação (Rollback) no Saga Coreografado

No padrão Saga Coreografado, não há um coordenador central para tratar falhas com blocos `try/catch` globais. Se uma etapa falha, o serviço responsável deve publicar um evento de falha para que os demais serviços reajam, desfazendo ações anteriores (ações de compensação) e garantindo a consistência do sistema.

### Cenário de Falha: Erro no FFmpeg (Video Processor)

**Causa:** Um *chunk* (fragmento) do vídeo está corrompido, em um formato não suportado, ou a Lambda sofreu um *timeout*.

**Fluxo de Compensação:**

1. **Falha de Processamento e Retentativas:** A Lambda do *Video Processor* tenta processar o chunk. O SQS é configurado com uma política de retentativa (ex: 3 tentativas). Se falhar em todas, a mensagem original vai para uma **DLQ (Dead Letter Queue)**.
2. **Emissão do Evento de Falha:** Antes de falhar definitivamente ou através de um alarme na DLQ, o *Video Processor* (ou um serviço de monitoramento da DLQ) publica o evento `video-precessing-error` no SNS.
3. **Reação do Core (Video Solicitation):** * O *Video Solicitation* consome o evento de falha.
* Ele altera o status do chunk para `FAILED`.
* Como o vídeo final depende de todos os chunks, o *Video Solicitation* altera o status geral do vídeo (Video Job) para `FAILED`.
* **Ação de Compensação (Cleanup):** O *Video Solicitation* emite um comando interno ou evento para deletar do S3 os metadados e os frames de outros chunks desse mesmo vídeo que já haviam sido processados, evitando custos desnecessários de armazenamento de um trabalho "quebrado".
4. **Reação do Notification Service:**
* Consome o evento indicando que o vídeo falhou (ex: `video-precessing-error`).
* Envia um e-mail ao usuário informando o erro no processamento do arquivo, solicitando um novo upload.

---

## 6. Contratos de Dados (Payloads dos Eventos)

### 6.1. Evento: `chunk-uploaded`

Emitido quando o upload de um fragmento é concluído com sucesso no S3 (Pode ser gerado via S3 Event Notifications roteado para o SNS).

* **Consumidor:** Video Processor (Lambda de Extração), Video Solicitation (para atualizar status).

```json
{
  "eventId": "uuid-1",
  "eventType": "VideoChunkUploaded",
  "source": "s3-event-bridge",
  "data": {
    "videoId": "vid-98765",
    "chunkId": "chunk-001",
    "userId": "usr-123",
    "s3Bucket": "meu-projeto-raw-videos",
    "s3Key": "uploads/vid-98765/chunk-001.mp4"
  }
}

```

### 6.2. Evento: `chunk-precessed`

Emitido pelo *Video Processor* após o FFmpeg extrair as imagens e salvá-las no S3.

* **Consumidor:** Video Solicitation (Controle de estado).

```json
{
    "video_id": "vid-98765",
    "chunk_part": 1,
    "status": "PROCESSED"
}

```

### 6.3. Evento: `video-processing-error` (Compensação)

Emitido se o processamento do chunk falhar irreversivelmente.

* **Consumidor:** Video Solicitation, Notification Service.

```json
{
    "video_id": "vid-98765",
    "user": {
        "id": "234",
        "name": "João Grande",
        "email": "user@email.co,"
    },
    "status": "ERROR",
    "cause": "Error Message",
    "system_trigger": "chunk_processor"
}
```

### 6.4. Evento: `all-chunk-processed`

Emitido pelo *Video Solicitation* quando a contagem de chunks processados se iguala ao total de chunks declarados no upload.

* **Consumidor:** Video Processor (Lambda de Empacotamento).

```json
{
    "vide_id": "1234",
    "user": {
        "id": "234",
        "name": "João Grande",
        "email": "user@email.co,"
    },
    "image_location": "videos/{video_id}/images",
    "bucket": "fiap-videos"
}

```

### 6.5. Evento: `video-processing-completed`

Emitido após a Lambda de empacotamento agrupar tudo em um arquivo `.zip`.

* **Consumidor:** Video Solicitation (atualiza o status final), Notification Service (envia o link).

```json
{
    "vide_id": "1234",
    "download_url": "https://url.download.com",
    "user": {
        "id": "234",
        "name": "João Grande",
        "email": "user@email.co,"
    },
    "status": "COMPLETED"
}

```
