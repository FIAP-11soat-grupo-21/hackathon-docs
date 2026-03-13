# Visão Geral de implantação da aplicação

Este documento fornece uma visão geral da implantação da aplicação, incluindo os passos necessários para configurar o ambiente de produção e garantir que a aplicação esteja funcionando corretamente.

*Atenção: Todo esse processo possui seu respectivo pipeline no repositório, deve-se seguir a mesma ordem de execução para garantir o funcionamento* 

## Passos para Implantação
- ATENÇÃO - Utilizamos a AWS para implantação da aplicação, portanto, é necessário ter uma conta na AWS e acesso ao console de gerenciamento.

1. Toda a infraestrutura da aplicação é provisionada utilizando o Terraform + Terragrunt. Certifique-se de ter o ambos instalados e configurado em seu ambiente local. 
- [terraform](https://developer.hashicorp.com/terraform/tutorials/aws-get-started/install-cli)
- [terragrunt](https://terragrunt.gruntwork.io/docs/getting-started/install/)

2. Realize o clone do repositório [hackathon-infra](https://github.com/FIAP-11soat-grupo-21/hackathon-infra) localmente em sua máquina.
3. Navegue até o diretório `src/` dentro do repositório clonado.
4. Atualize o arquivo src/Secrets/GHCR/terragrunt.hcl com as credenciais do GHCR (GitHub Container Registry) para permitir que o Terraform possa acessar as imagens de contêiner necessárias para a aplicação. As credenciais devem incluir o nome de usuário e a senha do GHCR, que podem ser obtidos na seção de configurações do GitHub.
5. Execute o comando `terragrunt apply` para iniciar o processo de implantação. Este comando irá provisionar toda a infraestrutura necessária para a aplicação, incluindo servidores, bancos de dados, e outros recursos na AWS.
6. Após a implantação ser concluída, já iremos possuir toda a base para poder adicionar as aplicações e serviços. Dito isso, o próximo passo é clonar cada um dos repositórios de aplicação para realizar a implantação de cada um dos serviços. Os repositórios são:

- [Video Solicitation](https://github.com/FIAP-11soat-grupo-21/hackathon-video-solicitation-microservice)
- [Video Processor](https://github.com/FIAP-11soat-grupo-21/hackathon-video-processor-microservice)
- [Notification](https://github.com/FIAP-11soat-grupo-21/hackathon-notification-microservice)
- [User](https://github.com/FIAP-11soat-grupo-21/hackathon-user-microservice)
- [Auth](https://github.com/FIAP-11soat-grupo-21/hackathon-auth-microservice)

Em cada um dos repositórios, navegue para o diretório `infra/`, ajuste as configurações das variáveis em `variables.tfvars` e execute o comando `terragrunt apply` para provisionar os recursos específicos para cada serviço.

7. Feito isso, crie entidades de email no Amazon SES para cada um dos serviços que necessitam enviar notificações por email. Isso pode ser feito através do console de gerenciamento da AWS, navegando até o serviço Amazon SES e seguindo as instruções para criar identidades de email. Atualize a variável SENDER_EMAIL na function do notification-service com o email criado para garantir que as notificações sejam enviadas corretamente.

8. Após esses passos é possível realizar o processo de testes para garantir que a aplicação esteja funcionando corretamente.