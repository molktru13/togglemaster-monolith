# Análise dos 12 Fatores aplicada ao ToggleMaster

Referência oficial: https://12factor.net/pt_br/

Para cada fator, avaliei se a aplicação atual já **atende**, atende **parcialmente** ou **não atende**, e o que precisaria ser ajustado para um ambiente de produção mais robusto.

| # | Fator | Status | Observações |
|---|---|---|---|
| 1 | **Base de Código** (Codebase) | Atende | O projeto está em um único repositório Git (`toggle-master-monolith`). Cada deploy é versionado. |
| 2 | **Dependências** | Atende | Todas as bibliotecas Python estão isoladas em `requirements.txt` com versões pinadas (Flask 2.2.2, psycopg2-binary 2.9.5, gunicorn 20.1.0). O `Dockerfile` instala via `pip install --no-cache-dir -r requirements.txt`, garantindo ambiente reproduzível. |
| 3 | **Configurações** (Config) | Atende | As credenciais do banco (`DB_HOST`, `DB_NAME`, `DB_USER`, `DB_PASSWORD`) vêm de variáveis de ambiente (`os.getenv`). Nenhuma senha hard-coded no `app.py`. **Ajuste para produção:** usar AWS Secrets Manager ou Parameter Store em vez de exportar variáveis manualmente via `export`. |
| 4 | **Serviços de Apoio** (Backing Services) | Atende | O PostgreSQL é tratado como recurso externo, conectável pelo `DB_HOST`. Localmente é o container `db`; em produção é o endpoint do RDS. Trocar o backing service é uma mudança de variável de ambiente, não de código. |
| 5 | **Build, Release, Run** | Parcial | O `Dockerfile` separa bem o **build** (instala dependências, copia código). O **run** é o `entrypoint.sh` + Gunicorn. **Falta** o estágio de **release** explícito: hoje não há pipeline CI/CD que combine o build com a config e gere artefatos imutáveis numerados. **Ajuste:** introduzir GitHub Actions / GitLab CI / Jenkins que gere uma imagem versionada (`togglemaster:1.2.3`) e a publique num registro (ECR). |
| 6 | **Processos** | Atende | A aplicação roda como processos sem estado dentro do Gunicorn. Toda persistência fica no PostgreSQL. Nenhum dado sensível é guardado em memória entre requisições. |
| 7 | **Vínculo de Portas** (Port Binding) | Atende | A aplicação se publica diretamente na porta 5000 via Gunicorn (`--bind 0.0.0.0:5000`). Não depende de servidor de aplicação externo (Apache/Nginx) injetando o runtime. |
| 8 | **Concorrência** | Parcial | O Gunicorn permite escalar via workers, mas o `entrypoint.sh` atual não define `--workers`. Em produção precisaria definir `--workers $(2 * num_cores + 1)` e, idealmente, escalar horizontalmente (mais EC2 atrás de um ALB) em vez de só verticalmente. |
| 9 | **Descartabilidade** (Disposability) | Parcial | O Gunicorn responde corretamente a SIGTERM e finaliza requisições em curso. **Ponto fraco:** o `entrypoint.sh` faz `flask init-db` no startup; se duas instâncias subirem ao mesmo tempo, ambas tentam criar a tabela. Mitigado pelo `CREATE TABLE IF NOT EXISTS`, mas o ideal seria mover migrations para um job separado (Alembic, Flyway). O startup também é lento porque depende do `pg_isready` em loop. |
| 10 | **Paridade entre Desenvolvimento e Produção** | Parcial | Localmente: container Postgres 13 + Gunicorn. Em produção AWS: RDS PostgreSQL 15/16 + Gunicorn. As versões do banco divergem; isso pode causar bugs sutis (extensões, tipos novos). **Ajuste:** alinhar a versão do Postgres do `docker-compose.yaml` à versão escolhida no RDS. |
| 11 | **Logs** | Parcial | A aplicação imprime via `print()` para stdout, o que é correto. O Gunicorn também loga em stdout/stderr. **Ponto fraco:** quando subi com `nohup gunicorn ... > app.log &`, os logs ficam num arquivo dentro da EC2, contrariando o fator (logs deveriam ser tratados como streams). **Ajuste:** enviar os logs para CloudWatch Logs (driver `awslogs` do Docker ou agent do CloudWatch na EC2). |
| 12 | **Processos de Administração** | Parcial | O comando `flask init-db` é registrado como um CLI command (`@app.cli.command("init-db")`), o que está correto — é um processo administrativo separado. **Ponto fraco:** ele é chamado automaticamente no startup, misturando responsabilidades. Em produção rodaria manualmente (ou em job dedicado de migração) antes de subir a aplicação. |

## Resumo executivo

A aplicação atende **plenamente 6 fatores** (1, 2, 3, 4, 6, 7) e **parcialmente 6** (5, 8, 9, 10, 11, 12). Nenhum fator está totalmente desrespeitado, o que é um bom sinal para um MVP — mostra que o time já pensou em fundamentos de cloud-native desde o início.

### Top 3 ajustes prioritários para produção robusta

1. **Logs em CloudWatch** (Fator 11): hoje a observabilidade depende de SSH no servidor, o que não escala.
2. **Pipeline de Build/Release versionado** (Fator 5): garantir imagens imutáveis e rollback rápido.
3. **Secrets em AWS Secrets Manager** (Fator 3): trocar `export DB_PASSWORD=...` por leitura via IAM Role da EC2.

Com esses três ajustes, o ToggleMaster já estaria pronto para receber tráfego real em produção, mantendo a simplicidade do monolito enquanto adota práticas de operação maduras.
