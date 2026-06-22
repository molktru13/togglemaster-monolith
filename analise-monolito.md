# Análise da Arquitetura Monolítica — ToggleMaster

## Por que o ToggleMaster é um monolito?

Analisando os arquivos `app.py`, `Dockerfile`, `requirements.txt` e `docker-compose.yaml`, observei as características clássicas de uma aplicação monolítica:

1. **Código-fonte único e indivisível.** Todo o comportamento da API (rotas HTTP, regras de negócio, acesso ao banco de dados e inicialização do schema) está concentrado em um único arquivo `app.py` com 143 linhas. Não há separação por camadas (controller / service / repository), nem por contextos de negócio.

2. **Processo único de execução.** O `Dockerfile` define um único `ENTRYPOINT` (`entrypoint.sh`) que dispara um único processo Gunicorn (`gunicorn --bind 0.0.0.0:5000 app:app`). Toda funcionalidade roda dentro do mesmo processo, na mesma porta.

3. **Banco compartilhado e acoplado ao processo.** A função `get_db_connection()` é chamada diretamente em cada handler HTTP. Se eu mudasse o esquema de persistência ou trocasse o PostgreSQL por outra tecnologia, teria que alterar todo o código de uma vez.

4. **Deploy unitário.** Subir uma correção em qualquer endpoint exige reconstruir e republicar a imagem inteira. Não há como atualizar “somente o módulo de criação de flag”.

5. **Stack tecnológica única.** Toda a aplicação está em Python/Flask. Não há liberdade para escolher uma tecnologia diferente para um subdomínio específico.

## Vantagens dessa abordagem para um MVP

- **Velocidade de desenvolvimento.** O time consegue colocar a primeira versão no ar em horas, validando a hipótese de negócio antes de investir em uma arquitetura mais complexa.
- **Simplicidade de operação.** Um único processo, um único deploy, um único log. Reduz a carga cognitiva sobre o time de DevOps em fase inicial.
- **Baixo custo de infraestrutura.** Uma EC2 t3.micro + um RDS db.t3.micro são suficientes para validar o produto.
- **Debug e testes locais triviais.** `docker compose up` sobe o ambiente completo. Um desenvolvedor júnior consegue contribuir desde o primeiro dia.
- **Transações ACID nativas.** Como tudo acessa um único banco, transações são simples e não exigem padrões distribuídos como Saga.
- **Menor sobrecarga de comunicação.** Não há latência de rede entre componentes — tudo acontece em memória, no mesmo processo.

## Desvantagens conhecidas (e quando elas começam a doer)

- **Acoplamento alto.** Mudar um campo da tabela `flags` exige rebuild do binário inteiro.
- **Escalabilidade grosseira.** Para suportar mais tráfego, escalar a aplicação inteira, mesmo que apenas um endpoint (por exemplo, `GET /flags/<name>`) seja o gargalo.
- **Risco de “big bang failure”.** Um bug numa rota pode derrubar o processo e tirar a API inteira do ar.
- **Tecnologia engessada.** Adotar Go ou Node.js para um pedaço específico exige reescrita.
- **Ciclo de deploy mais lento conforme cresce.** Em um time de 30 pessoas tocando o mesmo código, cada release vira um evento.
- **Difícil isolar performance.** CPU/memória são compartilhadas — uma rota lenta afeta todas as outras.

## Conclusão

Para o estágio atual (MVP de validação), o monolito é a **decisão correta**: maximiza velocidade de entrega e minimiza custo. Conforme o ToggleMaster crescer em usuários, número de times consumindo e complexidade de regras (segmentação por usuário, percentual de rollout, A/B testing), os pontos de dor citados acima vão pressionar a arquitetura.
