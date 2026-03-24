# QA Performance — BlazeDemo

Testes de performance para o fluxo de **compra de passagem aérea** do [BlazeDemo](https://www.blazedemo.com), utilizando **Apache JMeter 5.6.3**.

---

## Cenário Testado

**Fluxo completo de compra:**

```
GET  /              → Página inicial (formulário de origem/destino)
POST /reserve.php   → Selecionar voo (San Diego → Dublin)
POST /purchase.php  → Confirmar compra (dados do passageiro + cartão)
```

**Assertion crítica:** resposta de `/purchase.php` deve conter `"Thank you for your purchase"`.

---

## Critério de Aceitação

|       Métrica         |           Meta            |
|-----------------------|---------------------------|
| Throughput            | ≥ 250 requisições/segundo |
| Tempo de resposta p90 | < 2.000 ms                |
| Taxa de erro          | 0%                        |

---

## Tipos de Teste

### Load Test (`load-test.jmx`)
Simula carga **gradual e sustentada** para verificar o comportamento estável do sistema.

|          Parâmetro       |     Valor    |
|--------------------------|--------------|
| Usuários simultâneos     | 250          |
| Ramp-up                  | 60 segundos  |
| Duração em carga estável | 300 segundos |
| Duração total            | ~6 minutos   |

### Spike Test (`spike-test.jmx`)
Simula um **pico abrupto** de tráfego para avaliar resiliência e capacidade de recuperação.

|      Parâmetro       |         Valor              |
|----------------------|----------------------------|
| Usuários simultâneos | 500                        |
| Ramp-up              | 10 segundos (pico abrupto) |
| Duração em carga     | 60 segundos                |
| Duração total        | ~80 segundos               |

---

## 🛠 Pré-requisitos

| Ferramenta        | Versão               |                          Download                                  |
|-------------------|----------------------|--------------------------------------------------------------------|
| Java (JDK ou JRE) | 8+ (recomendado 11+) | [adoptium.net](https://adoptium.net)                               |
| Apache JMeter     | 5.6.3                | [jmeter.apache.org](https://jmeter.apache.org/download_jmeter.cgi) |

---

---

## Abrindo os scripts no JMeter (modo GUI)

> ⚠️ A GUI é usada apenas para **visualizar e editar** os scripts.  
> Para executar os testes, use sempre o **modo CLI (headless)** — é mais preciso.

1. Abra o JMeter:
   - Windows: execute `C:\apache-jmeter\bin\jmeter.bat`
   - Linux/Mac: execute `jmeter` no terminal

2. No menu: **File → Open**

3. Navegue até a pasta `scripts/` do projeto e abra:
   - `load-test.jmx` para o Load Test
   - `spike-test.jmx` para o Spike Test

4. Explore a estrutura na árvore à esquerda:
   ```
   Test Plan
   ├── HTTP Request Defaults   ← URL base (www.blazedemo.com)
   ├── HTTP Header Manager     ← Headers comuns
   ├── HTTP Cookie Manager     ← Sessão por usuário
   └── Thread Group            ← Configuração de carga
       ├── 01 - GET / Home
       ├── 02 - POST /reserve.php
       ├── 03 - POST /purchase.php
       ├── Summary Report
       └── Aggregate Report
   ```

---

## Executando os Testes

### Opção 1 — Script automático (recomendado)

**Windows:**
```cmd
cd qa-performance-blazedemo
run-tests.bat
```

**Linux/Mac:**
```bash
cd qa-performance-blazedemo
chmod +x run-tests.sh
./run-tests.sh
```

### Opção 2 — Linha de comando manual

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

**Spike Test:**
```bash
rm -rf results/spike-test-results.jtl reports/spike-test

jmeter -n \
  -t scripts/spike-test.jmx \
  -l results/spike-test-results.jtl \
  -e -o reports/spike-test
```

> **Importante:** O JMeter falha se o arquivo `.jtl` ou a pasta de report já existirem.  
> Sempre delete antes de rodar novamente.

### Opção 3 — Gerar dashboard a partir de JTL existente

```bash
jmeter -g results/load-test-results.jtl -o reports/load-test
```

---

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
│   └── spike-test.jmx      ← Spike Test (500 usuários, pico 10s)
├── results/                ← Arquivos .jtl gerados (gitignored)
├── reports/                ← Dashboards HTML gerados (gitignored)
├── run-tests.bat           ← Execução Windows
├── run-tests.sh            ← Execução Linux/Mac
└── .github/workflows/
    └── jmeter.yml          ← Pipeline CI/CD
```

---

## Pipeline CI/CD

O workflow `.github/workflows/jmeter.yml` permite executar os testes via **GitHub Actions**:

- A cada push para `main` executa o Load Test automaticamente
- Via `workflow_dispatch` é possível escolher: `load`, `spike` ou `both`
- Os dashboards HTML são publicados como **artifacts** por 30 dias

---

## Decisões Técnicas

**Think Time:** foram configurados timers aleatórios entre as requisições (1-3s no Load Test, 500ms no Spike Test) para simular comportamento humano real e não sobrecarregar artificialmente.

**Cookie Manager com `clearEachIteration=true`:** garante que cada iteração simula um usuário novo, sem reaproveitar sessão anterior.

**Assertions duplas:** cada passo tem assertion de status HTTP (200) e assertion de corpo (conteúdo esperado), garantindo que a resposta não é apenas um 200 vazio.

**Ramp-up de 60s no Load Test:** sobe 4 usuários por segundo, evitando spike involuntário no início e permitindo que o servidor se aqueça gradualmente.
