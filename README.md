# QA Performance — BlazeDemo

Testes de performance para o fluxo de **compra de passagem aérea** do [BlazeDemo](https://www.blazedemo.com), utilizando **Apache JMeter 5.6.3**.

**Relatório publicado:** [franciscogeorger.github.io/qa-performance-blazedemo](https://franciscogeorger.github.io/qa-performance-blazedemo/)

---

## Pré-requisitos

| Ferramenta        | Versão               |
|-------------------|----------------------|
| Java (JDK ou JRE) | 8+ (recomendado 11+) |
| Apache JMeter     | 5.6.3                |

---

## Cenário Testado
 
**Fluxo completo de compra (4 passos):**
 
| # | Método | Endpoint | Descrição |
|---|--------|----------|-----------|
| 01 | GET | `/` | Página inicial |
| 02 | POST | `/reserve.php` | Selecionar voo (San Diego → Dublin) |
| 03 | POST | `/purchase.php` | Escolher voo (flight 234, United Airlines, $432.98) |
| 04 | POST | `/confirmation.php` | Confirmar compra com dados do passageiro e cartão |
 
**Assertion crítica:** resposta de `/confirmation.php` deve conter `"Thank you for your purchase today!"`.
 
**Assertions de status:** todos os 4 passos validam HTTP 200.

---

## Critério de Aceitação

|       Métrica         |           Meta            |
|-----------------------|---------------------------|
| Throughput            | ≥ 250 requisições/segundo |
| Tempo de resposta p90 | < 2.000 ms                |
| Taxa de erro          | 0%                        |

---

## Configuração do Load Test (`load-test.jmx`)
 
| Parâmetro | Valor |
|-----------|-------|
| Usuários simultâneos | 250 |
| Ramp-up | 60 segundos |
| Duração total | 360 segundos (~6 minutos) |
| Erro | Continua execução (`continue`) |
| Cookie | Limpo a cada iteração |
| Timeout de conexão | 10.000 ms |
| Timeout de resposta | 20.000 ms |
| Protocolo | HTTPS |
| Host | www.blazedemo.com |

---

## Resultado da Execução
 
Resultados extraídos do **Aggregate Report** gerado pelo JMeter:
 
| Endpoint | Amostras | Média (ms) | Mediana (ms) | p90 (ms) | p95 (ms) | p99 (ms) | Mín (ms) | Máx (ms) | Erro % | Throughput |
|----------|----------|------------|--------------|----------|----------|----------|----------|----------|--------|------------|
| 01 - GET / Home | 19.833 | 1.055 | 413 | 2.942 | 5.048 | 8.722 | 159 | 10.552 | 0,17% | 54,1/s |
| 02 - POST /reserve.php | 19.770 | 1.052 | 418 | 2.923 | 5.069 | 8.620 | 160 | 10.482 | 0,19% | 54,2/s |
| 03 - POST /purchase.php | 19.723 | 1.024 | 411 | 2.606 | 4.805 | 8.638 | 138 | 10.553 | 0,20% | 54,1/s |
| 04 - POST /confirmation.php | 19.654 | 1.072 | 418 | 3.013 | 5.195 | 8.882 | 159 | 10.515 | 0,13% | 53,9/s |
| **TOTAL** | **78.980** | **1.051** | **415** | **2.882** | **5.081** | **8.721** | **158** | **10.553** | **0,17%** | **215,0/s** |
 
---
 
## Análise do Critério de Aceitação
 
| Critério | Meta | Resultado | Status |
|----------|------|-----------|--------|
| Throughput | ≥ 250 req/s | 215,0 req/s | ❌ Não atingido |
| Tempo de resposta p90 | < 2.000 ms | 2.882 ms | ❌ Não atingido |
| Taxa de erro | 0% | 0,17% | ❌ Não atingido |
 
### Conclusão
 
O critério de aceitação **não foi satisfeito** em nenhuma das três métricas avaliadas. A seguir, a análise de cada ponto:
 
**Throughput — 215,0 req/s (meta: ≥ 250 req/s)**
O throughput total ficou 14% abaixo da meta. Cada um dos 4 endpoints entregou entre 53,9 e 54,2 req/s individualmente, somando 215 req/s. Isso indica que o servidor não conseguiu absorver a carga de 250 usuários simultâneos dentro do tempo de resposta esperado — parte das requisições ficou enfileirada ou com latência alta demais para compor o throughput exigido.
 
**p90 — 2.882 ms (meta: < 2.000 ms)**
O percentil 90 ficou 44% acima do limite. Isso significa que 10% das requisições demoraram mais de 2,8 segundos — quase 1 segundo acima do tolerado. O endpoint mais crítico foi o `/confirmation.php` com p90 de 3.013 ms, seguido do `/` com 2.942 ms. A mediana de 415 ms mostra que a maior parte das requisições foi rápida, mas picos expressivos de latência (p99 acima de 8,7 segundos, máximo de 10,5 segundos) elevaram o p90 para além do aceitável.
 
**Taxa de erro — 0,17% (meta: 0%)**
Embora baixa, a taxa de erro de 0,17% sobre 78.980 amostras representa aproximadamente 134 requisições com falha. O endpoint com maior índice de erros foi o `/purchase.php` com 0,20%. Esses erros provavelmente são timeouts ou respostas HTTP 5xx causadas pela sobrecarga do servidor de demonstração, que é um ambiente público e compartilhado não dimensionado para carga real de 250 usuários.
 
### Causas Prováveis
 
O ambiente alvo (`www.blazedemo.com`) é um **site de demonstração público**, sem SLA de disponibilidade ou infraestrutura dedicada. Os resultados refletem as limitações do próprio servidor-alvo, e não necessariamente uma falha na configuração do teste. Em um ambiente de produção devidamente dimensionado, os mesmos scripts JMeter poderiam produzir resultados dentro do critério de aceitação.

---

## Como Executar o Script
 
1. Clone este repositório:
 
```bash
git clone https://github.com/franciscogeorger/qa-performance-blazedemo.git
```
 
2. Execute o teste com o seguinte comando:
 
```bash
jmeter -n -t ./scripts/load-test.jmx -l ./results/load-test-results.jtl -e -o ./results/dashboard -f
```

## Visualizando o Relatório
 
Após a execução, abra no navegador:
```
reports\load\index.html
```
 
| Seção no dashboard | O que verificar |
|--------------------|-----------------|
| Statistics | p90, Throughput e Error % — métricas do critério de aceitação |
| Response Times Over Time | Estabilidade da latência ao longo do teste |
| Active Threads Over Time | Curva de ramp-up (0 → 250 em 60s) |
| Transactions per Second | Se o throughput se manteve estável |
| Errors | Tipo e frequência das falhas |
 
---

## Estrutura do Projeto

```
qa-performance-blazedemo/
├── scripts/
│   ├── load-test.jmx       ← Load Test (250 usuários, 6min)
├── results/                ← Arquivos .jtl gerados (gitignored)
├── reports/                ← Dashboards HTML gerados (gitignored)
└── .github/workflows/
    └── jmeter.yml         ← Pipeline CI/CD → publica no GitHub Pages
```

---

## Decisões Técnicas

**Cookie Manager com `clearEachIteration=true`:** cada iteração simula um usuário novo, garantindo independência entre requisições.
 
**Ramp-up de 60s:** sobe ~4 usuários por segundo, evitando spike involuntário no início.
 
**`on_sample_error=continue`:** em caso de falha em um passo o teste continua, garantindo que os erros sejam registrados sem interromper a medição.

