# QA Performance — BlazeDemo

Testes de performance para o fluxo de **compra de passagem aérea** do [BlazeDemo](https://www.blazedemo.com), utilizando **Apache JMeter 5.6.3**.

---

## Pré-requisitos

| Ferramenta        | Versão               |
|-------------------|----------------------|
| Java (JDK ou JRE) | 8+ (recomendado 11+) |
| Apache JMeter     | 5.6.3                |

---

## Cenário Testado

**Fluxo completo de compra (4 passos):**

```
| # | Método | Endpoint | Descrição |
|---|--------|----------|-----------|
| 01 | GET | `/` | Página inicial |
| 02 | POST | `/reserve.php` | Selecionar voo (San Diego → Dublin) |
| 03 | POST | `/purchase.php` | Escolher voo (flight 234, United Airlines, $432.98) |
| 04 | POST | `/confirmation.php` | Confirmar compra com dados do passageiro e cartão |
```

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
 
**Listeners configurados no script:**
- `Summary Report` → salvo em `results/load-test-results.jtl`
- `Aggregate Report` → salvo em `results/load-test-aggregate.jtl`
- `View Results Tree` → apenas para uso na GUI (sem arquivo de saída)
 
---

## Executando os Testes


**Load Test:**
```bash
# Limpa resultados anteriores (obrigatório — JMeter não sobrescreve)
rm -rf results/load-test-results.jtl reports/load-test

# Executa e gera dashboard HTML
jmeter -n \
  -t scripts/load-test.jmx \
  -l results/load-test-results.jtl \
  -e -o reports/load-test
```

## Visualizando o Relatório

Após a execução, abra no navegador:

```
reports/load-test/index.html
reports/spike-test/index.html
```

**Métricas principais no dashboard:**

| Seção                        | O que observar                        |
|-------                       |----------------                       |
| **Statistics**               | Throughput (req/s), p90, p95, Error % |
| **Response Times Over Time** | Estabilidade ao longo do teste        |
| **Active Threads Over Time** | Curva de ramp-up                      |
| **Transactions per Second**  | Se atingiu 250 req/s                  |
| **Errors**                   | Tipo e frequência dos erros           |

---

## Estrutura do Projeto

```
qa-performance-blazedemo/
├── scripts/
│   ├── load-test.jmx       ← Load Test (250 usuários, 6min)
├── results/                ← Arquivos .jtl gerados (gitignored)
├── reports/                ← Dashboards HTML gerados (gitignored)
└── .github/workflows/
    └── jmeter.yml          ← Pipeline CI/CD
```

---

## Relatório de Execução (CI/CD)

O workflow `.github/workflows/jmeter.yml` permite executar os testes via **GitHub Actions**:

---

