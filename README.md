# ToggleMaster — Tech Challenge Fase 2

Projeto acadêmico de conteinerização e implantação de cinco microsserviços no Amazon EKS.

## Serviços e arquitetura

| Serviço | Linguagem | Função | Dependências |
| --- | --- | --- | --- |
| auth-service | Go | Criar e validar chaves de API | PostgreSQL |
| flag-service | Python | Gerenciar feature flags | PostgreSQL e auth |
| targeting-service | Python | Gerenciar regras de segmentação | PostgreSQL e auth |
| evaluation-service | Go | Avaliar flags e publicar eventos | Redis, flags, targeting e SQS |
| analytics-service | Python | Consumir eventos e persistir análises | SQS e DynamoDB |

Fluxo na nuvem: cliente → NLB → F5 NGINX Ingress Controller → evaluation → SQS → analytics → DynamoDB. Auth, flags e targeting usam três instâncias RDS PostgreSQL independentes. Redis é usado como cache das avaliações.

## Organização

- As cinco pastas `*-service/` contêm os códigos e Dockerfiles.
- `docker-compose.yml` e `docker/` configuram o ambiente local.
- `k8s/` contém Deployments, Services, ConfigMaps, Ingress, HPAs e Jobs de teste.
- `infra/` contém a configuração EKS, valores Helm e políticas IAM.

## Configurações e credenciais

Esta é uma cópia para entrega. Identificadores da conta, endpoints e IDs de rede foram substituídos por valores `YOUR_*`. Substitua-os pelos recursos do seu ambiente antes da implantação. ConfigMaps e políticas de exemplo não contêm credenciais.

Copie `.env.example` para `.env` e preencha valores locais. Nunca publique `.env`, senhas, chaves AWS, tokens ou kubeconfig. Para as senhas usadas nas URLs PostgreSQL do Compose, utilize caracteres seguros para URL; senhas com caracteres especiais exigem codificação apropriada.

Na nuvem, crie os Secrets `auth-service-secret`, `flag-service-secret`, `targeting-service-secret` e `evaluation-service-secret` diretamente no Kubernetes. Eles não são exportados neste repositório. Os três primeiros precisam de `DATABASE_URL` com `sslmode=require`; auth também precisa de `MASTER_KEY`. Evaluation precisa de `SERVICE_API_KEY`, emitida pelo auth da nuvem, sem prefixo Bearer.

## Execução local

Pré-requisitos: Docker Desktop em execução, Docker Compose e acesso à fila SQS Standard de desenvolvimento. A fila é real, inclusive no ambiente local, e seu uso pode gerar custos AWS.

1. Copie e preencha `.env.example` como `.env`. A chave de serviço será obtida após iniciar auth.
2. Inicie bancos e os serviços de configuração:

   ```powershell
   docker compose up -d --build postgres-auth postgres-apps redis dynamodb-local auth-service flag-service targeting-service
   ```

3. Crie uma chave em `POST http://localhost:8001/admin/keys` usando `Authorization: Bearer SUA_MASTER_KEY` e corpo `{"name":"evaluation-service-local"}`. Coloque o campo `key` retornado em `SERVICE_API_KEY` no `.env`.
4. Crie a tabela `ToggleMasterAnalytics`, chave de partição `event_id` do tipo String, no DynamoDB Local (`http://localhost:8000`), usando a AWS CLI com esse endpoint explícito.
5. Cadastre uma flag e sua regra pelos endpoints descritos nos READMEs dos serviços.
6. Execute `docker compose up -d --build` e confira `docker compose ps`: são nove contêineres.

Localmente, auth utiliza uma instância PostgreSQL, enquanto flags e targeting possuem bancos lógicos separados na segunda instância. Há ainda Redis e DynamoDB Local. O DynamoDB Local está em memória: ao reiniciar, recrie a tabela.

Portas locais: auth 8001, flags 8002, targeting 8003, evaluation 8004, analytics 8005 e DynamoDB Local 8000. As portas publicadas estão restritas a `127.0.0.1`.

## Implantação na AWS

Os arquivos são referências para reprodução; não provisionam automaticamente toda a solução. RDS, ElastiCache, SQS, DynamoDB e regras de rede foram criados separadamente pelo console/CLI.

1. Crie cinco repositórios ECR, compile e publique as imagens com a tag referenciada nos Deployments.
2. Crie EKS com `infra/cluster.yaml`, usando perfil AWS autorizado. O laboratório utiliza dois nós, redes públicas para os nós e privadas para bancos/cache, sem NAT Gateway.
3. Crie três instâncias RDS PostgreSQL privadas e execute os respectivos scripts `db/init.sql`. Crie `auth_db` manualmente se não foi definido na criação do RDS. Inicialize os bancos a partir de um pod cliente dentro da VPC.
4. Crie um ElastiCache Redis 7.1 com um nó micro e TLS, uma fila SQS Standard e a tabela DynamoDB indicada acima.
5. Configure IRSA e as ServiceAccounts `evaluation-service` e `analytics-service` no namespace `togglemaster`, com as políticas de `infra/iam/`. Evaluation envia mensagens; analytics recebe/exclui mensagens e grava itens.
6. Crie Secrets e aplique os ConfigMaps, Deployments e Services individualmente. Os Deployments incluem requests, limits e probes.
7. Instale F5 NGINX Open Source (chart testado: 2.8.0, aplicação 5.7.0) no namespace `nginx-ingress`, usando `infra/nginx-values.yaml`.
8. Para o NLB, configure também AWS Load Balancer Controller (chart/aplicação testados: 3.6.2), ServiceAccount própria e a política correspondente. Os valores NGINX entregues solicitam um NLB público e iniciam cobrança quando aplicados.
9. Aplique `k8s/ingress.yaml` e `k8s/hpa.yaml`. Verifique pods, métricas, roteamento e processamento antes de testes de carga.

Não execute `kubectl apply -f k8s/` indiscriminadamente: a pasta também contém Jobs que geram carga e custos. Aplique os arquivos de teste somente de forma intencional.

No Kubernetes não configure `AWS_ENDPOINT_URL_DYNAMODB`: analytics deve acessar a AWS. Não configure chaves estáticas AWS nos pods; utilize as ServiceAccounts associadas a roles.

## Verificações e testes

```powershell
kubectl get nodes
kubectl get pods,services,ingress,hpa -n togglemaster
kubectl top pods -n togglemaster
```

O Ingress usa `Host: togglemaster.test` e caminhos `/evaluate`, `/flags`, `/rules` e `/validate`. Teste o endpoint NLB com esse cabeçalho; não é necessário comprar domínio. O ambiente acadêmico usa HTTP externo, portanto não exponha credenciais administrativas através dele.

`load-test.yaml` gera até 100 avaliações por segundo durante 120 segundos, passando pelo NGINX internamente. `sqs-load-test.yaml` envia 6.000 mensagens para demonstrar a escala independente do analytics. Ambos os HPAs usam CPU, alvo 70%, mínimo 1 e máximo 4 pods. Eles não configuram autoscaling das máquinas.

Resultados observados durante o desenvolvimento: 12.000 avaliações, 100% de HTTP 200 e média de 4,35 ms no teste interno; evaluation e analytics escalaram até quatro réplicas. No teste independente, o produtor enviou 6.000 mensagens e os contadores aproximados da fila retornaram a zero. Esses resultados são evidências do ambiente testado, sem garantia de desempenho em outros ambientes.

Pare o analytics local antes dos testes na nuvem: os ambientes compartilham a fila e podem disputar mensagens.

## Escolha do NGINX

Foi escolhido F5 NGINX Ingress Controller com NGINX Open Source porque o controller comunitário `ingress-nginx` encerrou sua manutenção em março de 2026. São projetos distintos. A implementação escolhida mantém NGINX no roteamento, sem licença NGINX Plus. A infraestrutura AWS continua sujeita a cobrança.

- https://kubernetes.io/blog/2025/11/11/ingress-nginx-retirement/
- https://docs.nginx.com/nginx-ingress-controller/install/helm/open-source/

## Custos e encerramento

EKS, EC2/EBS, RDS, ElastiCache, NLB/IPv4 e o consumo de outros serviços podem gerar cobranças. Desligar o computador não encerra esses recursos. Após a gravação e entrega, remova o balanceador pelo Kubernetes enquanto o controller estiver ativo, confira sua exclusão e então remova aplicações, bancos/cache, fila/tabela, imagens/repositórios e cluster/rede. Confira snapshots, backups retidos, discos e demais recursos residuais do projeto. Não exclua recursos alheios ao ToggleMaster.

## Origem do código e entrega

Código-base acadêmico: https://github.com/FIAP-TCs — repositórios auth-service, flag-service, targeting-service, evaluation-service e analytics-service. Os históricos originais foram preservados separadamente no computador e não integram o primeiro commit deste repositório. Nenhuma licença adicional é atribuída ao código de terceiros.

O relatório final deve incluir participantes, RM, usernames Discord, link deste repositório e vídeo de até 20 minutos. O relatório e roteiro final de gravação serão concluídos separadamente.
