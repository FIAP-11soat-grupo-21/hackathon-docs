# Hackathon FIAP X - Processamento de Videos

Este repositorio centraliza a documentacao de uma plataforma de processamento de videos orientada a eventos, criada para receber uploads de videos em partes (chunks), extrair frames em paralelo e entregar um arquivo `.zip` final ao usuario. A solucao foi desenhada com microservicos + componentes serverless na AWS, com foco em escalabilidade, resiliencia, seguranca e rastreabilidade do processamento.

## Objetivo do projeto (resumo rapido)

O desafio é evoluir um projeto simples para uma solucao robusta que:
- processe varios videos simultaneamente;
- suporte picos sem perder requisicoes;
- tenha autenticacao por usuario e senha;
- permita acompanhar status dos videos;
- notifique o usuario em caso de sucesso/falha.

Neste contexto, a arquitetura adotada usa:
- API Gateway + Cognito para borda e autenticacao;
- microservicos de dominio (User, Video Solicitation, Video Processor, Notification);
- SNS + SQS + Lambda para orquestracao assincrona (Saga coreografado);
- S3, DynamoDB e RDS para persistencia.

## Como se guiar neste repositorio

Ordem sugerida de leitura para onboarding:

1. [`docs/Arquitetura.md`](docs/Arquitetura.md)
   - Entenda o problema, os servicos e o fluxo ponta a ponta (`chunk-uploaded` -> `chunk-processed` -> `all-chunk-processed` -> `video-processing-completed`).
   - Veja tambem os cenarios de compensacao (falhas e rollback no Saga).

2. [`docs/infraestrutura.md`](docs/infraestrutura.md)
   - Veja como a arquitetura vira infraestrutura AWS (camadas, rede privada, ECS, ALB, VPC Link, observabilidade).
   - Use o diagrama em [`images/Hackathon-infra.jpg`](images/Hackathon-infra.jpg) para visualizar o fluxo rapidamente.

3. [`docs/Implantação.md`](docs/Implantação.md)
   - Siga a sequencia de provisionamento com Terraform/Terragrunt e a ordem de deploy dos repositorios de servico.
   - Confira os ajustes de credenciais (GHCR) e configuracao do SES para notificacoes.

## Mapa rapido de conteudo

- [`docs/Arquitetura.md`](docs/Arquitetura.md): desenho arquitetural, microservicos, eventos, Saga e payloads de eventos.
- [`docs/infraestrutura.md`](docs/infraestrutura.md): topologia AWS, componentes por camada, fluxos tecnicos, seguranca e escalabilidade.
- [`docs/Implantação.md`](docs/Implantação.md): passo a passo de provisioning/deploy.
- [`images/Hackathon-infra.jpg`](images/Hackathon-infra.jpg): diagrama da infraestrutura.

## Repositorios relacionados (implementacao)

A documentacao referencia os seguintes repositorios de implementacao:
- [`hackathon-infra`](https://github.com/FIAP-11soat-grupo-21/hackathon-infra)
- [`hackathon-user-microservice`](https://github.com/FIAP-11soat-grupo-21/hackathon-user-microservice)
- [`hackathon-video-solicitation-microservice`](https://github.com/FIAP-11soat-grupo-21/hackathon-video-solicitation-microservice)
- [`hackathon-video-processor-microservice`](https://github.com/FIAP-11soat-grupo-21/hackathon-video-processor-microservice)
- [`hackathon-notification-microservice`](https://github.com/FIAP-11soat-grupo-21/hackathon-notification-microservice)
- [`hackathon-auth-microservice`](https://github.com/FIAP-11soat-grupo-21/hackathon-auth-microservice)
