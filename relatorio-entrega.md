# Relatório de Entrega — Tech Challenge Fase 1

**Projeto:** ToggleMaster — Plataforma de Feature Flags
**Curso:** Pós-graduação em DevOps e Arquitetura Cloud
**Turma:** 4DCLT
**Fase:** 01

---

## 1. Participantes do grupo

| Nome | RM | E-mail |
|---|---|---|
| [Marcílio Alves Galindo] | [374871] | [marcilio@workmail.com] |

---

## 2. Links de entrega

- **Repositório do código:** https://github.com/molktru13/togglemaster-monolith
- **Vídeo de demonstração (até 15 min):** https://drive.google.com/file/d/1Km1gQSTCsHw09qhzy_haFgRlLV4Ukr6w/view
- **Diagrama de arquitetura (diagrams.net):** https://drive.google.com/file/d/1nzhV61Hm2wRZitOxoyUFHU_Rpak-Svve/view?usp=sharing
- **Estimativa de custos AWS (link da calculadora):** https://calculator.aws/#/estimate?id=30dbd95bbc09537580ff126a65d028d04cd32121

---

## 3. Resumo da solução

A primeira fase do projeto ToggleMaster consistiu em estudar a aplicação monolítica fornecida (Python + Flask + PostgreSQL), executá-la localmente via Docker Compose e, em seguida, provisionar manualmente a infraestrutura necessária na AWS para colocá-la em produção.

A arquitetura escolhida segue o padrão mínimo recomendado pelo desafio:

- **VPC** dedicada (`10.0.0.0/16`) com sub-rede pública (EC2) e privadas (RDS), em duas AZs.
- **EC2 t3.micro** (Amazon Linux 2023) rodando o Gunicorn na porta 5000.
- **RDS PostgreSQL db.t3.micro** em sub-rede privada, sem acesso público.
- **Security Groups** segmentados:
  - `sg-togglemaster-ec2`: aceita SSH (22) apenas do IP do desenvolvedor e HTTP (5000) do mundo.
  - `sg-togglemaster-rds`: aceita PostgreSQL (5432) **exclusivamente** do `sg-togglemaster-ec2`.
- **Credenciais** do banco gerenciadas via variáveis de ambiente (`DB_HOST`, `DB_NAME`, `DB_USER`, `DB_PASSWORD`), exportadas na sessão SSH — atendendo ao Fator 3 (Config) do 12-Factor App.

---

## 4. Discussão sobre os 12 Fatores

O documento completo está em `12-factor-app.md` no repositório.

**Resumo:** a aplicação atende plenamente 6 fatores (Codebase, Dependências, Config, Backing Services, Processos, Port Binding) e parcialmente 6 (Build/Release, Concorrência, Descartabilidade, Dev/Prod parity, Logs, Admin processes).

Os três ajustes prioritários identificados para uma produção robusta são:
1. Centralizar logs em **CloudWatch Logs** (hoje ficam em arquivo na EC2).
2. Criar um **pipeline CI/CD com imagens versionadas** (hoje o build/release é manual).
3. Substituir `export DB_PASSWORD=...` por **AWS Secrets Manager** + IAM Role.

---

## 5. Análise do monolito

O documento completo está em `analise-monolito.md`. Em resumo, o ToggleMaster é um monolito clássico (código único, processo único, deploy único). Para um MVP, essa escolha é **acertada**, pois prioriza velocidade de validação e simplicidade de operação. Os pontos de dor (escalabilidade granular, isolamento de falhas, ciclo de deploy) tornam-se relevantes conforme o produto cresce em tráfego e em complexidade — exatamente o que as próximas fases do curso vão endereçar.

---

## 6. Estimativa de custos (AWS Pricing Calculator)

**Link da estimativa:** https://calculator.aws/#/estimate?id=30dbd95bbc09537580ff126a65d028d04cd32121

![Texto descritivo](togglemaster-monolith/calc amz.jpeg)

**Custo mensal estimado:** 23,67 USD /mês

**Composição:**
- Amazon EC2: US$ 8,23
- RDS db.t3.micro PostgreSQL Single-AZ + 20 GB gp3: US$ 15,44

> Estimativa feita para a região `us-east-1`. Em `sa-east-1` os valores são ~ 30% maiores.

---

## 7. Desafios encontrados e decisões tomadas

### Desafio 1 — `curl.exe` no Windows quebrando o JSON e gerando HTTP 400
Na fase de validação dos endpoints tentei usar `curl.exe` (que já vem instalado no Windows 10/11) para disparar as requisições POST e PUT com body JSON. O resultado foi que o Flask retornava **HTTP 400 Bad Request** em todas as chamadas, mesmo com o JSON aparentemente correto.

Investigando, descobri a causa: o `curl.exe` do Windows exige que as aspas internas do JSON sejam escapadas com `\"`. Porém, quando colei o comando no PowerShell, o próprio PowerShell **tenta interpretar esses escapes uma segunda vez** e entrega para o `curl.exe` uma string com `\"` literal em vez de `"`. Resultado: o servidor recebia um JSON inválido.

**Antes (quebrado):**
```bash
curl.exe -X POST -H "Content-Type: application/json" -d "{\"name\":\"new-feature\",\"is_enabled\":true}" http://localhost:5000/flags
```

**Decisão:** abandonar o `curl.exe` e usar o cmdlet nativo do PowerShell **`Invoke-RestMethod`**, que:
- Aceita JSON com **aspas simples por fora** (`'{"name":"x","is_enabled":true}'`).
- **Não exige nenhum escape** — o PowerShell não reinterpreta as aspas porque o conteúdo está em uma string de aspas simples.
- Lê automaticamente o `Content-Type: application/json` quando recebe um objeto via `-Body`.
- Retorna objetos **deserializados** (não texto cru), o que facilita a leitura nos prints.

**Depois (funcionando):**
```powershell
Invoke-RestMethod -Uri http://localhost:5000/flags -Method Post -ContentType 'application/json' -Body '{"name":"new-feature","is_enabled":true}'
```


### Desafio 2 — IP público da EC2 mudava a cada stop/start
Durante a fase de deploy e validação, percebi que o **IP público da EC2 mudava sempre que a instância era parada e iniciada** (seja por manutenção, ajuste de Security Group, ou reinicialização acidental). Isso quebrava:
- Os testes automatizados no PowerShell local (variável `$EC2` desatualizada)
- O link do navegador para demonstração
- Qualquer documentação ou script que tivesse o IP *hardcoded*

Na primeira vez, o IP era `54.210.20.2`. Após um stop/start, virou `3.91.32.197`. Cada mudança exigia atualizar comandos, refazer prints e reconectar SSH.

**Decisão:** criar um **Elastic IP** no console AWS (EC2 → Elastic IPs → Allocate → Associate com a instância `togglemaster-ec2`) e associá-lo à instância. O Elastic IP **não muda** enquanto estiver associado a uma instância rodando — mesmo após stop/start/reboot.

**Resultado:** o IP fixo passou a ser **`100.51.72.110`**. Todos os comandos, scripts, SSH e navegador passaram a usar essa constante. O custo é **$0,00/hora** enquanto associado a uma instância running (cobrança só se o EIP ficar *desassociado*).

**Lição:** para qualquer workload que precise de endpoint estável (API pública, webhook, DNS), **sempre provisione Elastic IP** no início. Evita retrabalho e garante reprodutibilidade dos testes.
