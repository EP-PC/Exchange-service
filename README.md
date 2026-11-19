# Exchange-service

Microsserviço de **câmbio** do [store-pma](https://github.com/EP-PC/store-pma) (Insper, Platforms,
Microservices, DevOps and APIs, 2026.2). Em **Python** (FastAPI), usando como API externa o
**PTAX do Banco Central do Brasil** (pública, sem chave).

O serviço foi escrito pelo grupo seguindo o contrato combinado com o Order
([`EXCHANGE.md`](https://github.com/EP-PC/store-pma/blob/main/EXCHANGE.md)). Ele **não** é o
"BACEN Service" pronto do site da disciplina.

## Endpoints

| Método e path | Login | Resposta |
|---|---|---|
| `GET /exchange/{from}/{to}` | sim (`id-account`) | `{"sell", "buy", "date", "id-account"}` |
| `GET /exchange/currencies` | sim | moedas suportadas (`["AUD", "BRL", ...]`) |
| `GET /exchange/health-check` | não | `200 {}` (probes do Kubernetes) |

O path é **singular** (`/exchange/...`) porque é o que o gateway roteia (`Path=/exchange/**`). O
enunciado usa as duas grafias, e o próprio exemplo dele é o singular.

```json
GET /exchange/USD/BRL   (header id-account: 0195ae95-...)
{"sell": 5.8102, "buy": 5.8096, "date": "2026-11-18 13:08:44", "id-account": "0195ae95-..."}
```

- `sell` e `buy` são a cotação de **1 `from` em `to`**: `sell` é a taxa de venda (o que a casa de
  câmbio cobra) e `buy` a de compra. O Order usa `sell` para converter os pedidos.
- `date` é a data e hora do boletim PTAX usado. Fim de semana e feriado não têm boletim: vale o
  último dia útil (o serviço busca os últimos 7 dias).
- O header `X-Cache` diz se a cotação veio do PTAX (`miss`), da memória (`hit`) ou é uma cotação
  antiga servida porque o PTAX falhou (`stale`).

| Situação | Status |
|---|---|
| sem `id-account` (ou vazio) | 401 |
| moeda que o PTAX não cota (ex.: `XYZ`, `REAL`) | 404 |
| PTAX fora do ar, lento ou com resposta inválida, sem cotação em memória | 503 |

## Como funciona a cotação

O PTAX só cota moedas **em reais**: compra `c` e venda `v` de 1 unidade em BRL. Para `USD → EUR`
o serviço busca as duas e faz a **taxa cruzada**:

```
buy  = c(USD) / v(EUR)     o mercado compra seus dólares e te vende euros
sell = v(USD) / c(EUR)     o mercado te vende dólares e compra seus euros
```

O BRL vale 1, então `USD → BRL` é a própria cotação do dólar. As moedas suportadas vêm de
`/Moedas` do PTAX (AUD, CAD, CHF, DKK, EUR, GBP, JPY, NOK, SEK, USD) mais o BRL.

**Por que PTAX:** o site avisa que algumas APIs gratuitas bloqueiam requisições que vêm da AWS (o
serviço funciona no notebook e falha no EKS). O PTAX é público e responde de qualquer lugar.

**Timeout:** no pior caso uma requisição faz 3 chamadas ao PTAX em sequência (lista de moedas,
`from` e `to`). Com `PTAX_TIMEOUT=1.5`, isso dá no máximo 4,5 s, abaixo dos 5 s que o Order espera.

## Configuração (variáveis de ambiente)

| Variável | Padrão | O que faz |
|---|---|---|
| `PTAX_URL` | URL do BCB | base da API do PTAX |
| `PTAX_TIMEOUT` | `1.5` | segundos de espera por chamada ao PTAX |
| `CACHE_SECONDS` | `300` | por quanto tempo uma cotação é reaproveitada (`0` desliga o cache) |
| `RETRY_SECONDS` | `30` | com o PTAX fora, quanto tempo serve a cotação antiga antes de tentar de novo |
| `MAX_STALE_SECONDS` | `86400` | idade máxima de uma cotação antiga; mais velha que isso, responde 503 |

## Rodar e testar

```bash
python -m venv .venv && .venv/Scripts/pip install -r requirements-dev.txt   # Linux/macOS: .venv/bin/pip
.venv/Scripts/python -m pytest -q                                           # 14 testes, PTAX simulado
.venv/Scripts/python -m uvicorn main:app --port 8080

curl -i -H 'id-account: 0195ae95-5be7-7dd3-b35d-7a7d87c404fb' http://127.0.0.1:8080/exchange/USD/BRL  # 200
curl -i http://127.0.0.1:8080/exchange/USD/BRL                                                        # 401
curl -i -H 'id-account: 0195ae95-5be7-7dd3-b35d-7a7d87c404fb' http://127.0.0.1:8080/exchange/USD/XYZ  # 404
curl -i http://127.0.0.1:8080/exchange/health-check                                                   # 200
```

Com Docker: `docker build -t exchange . && docker run --rm -p 8080:8080 exchange`. Os testes rodam
dentro do build com `docker build --target test .`.

Os testes (`tests/test_exchange.py`) trocam o PTAX por um `httpx.MockTransport`, sem rede, e usam
um relógio falso para testar a expiração do cache sem esperar.

## Bottlenecks

### 1. Cache das cotações (com cotação antiga quando o PTAX cai)

**Problema:** cada conversão chamava o PTAX, ~230 ms por requisição. O serviço também ficava
preso ao limite e à disponibilidade de um serviço público: se o BCB caísse, toda conversão de
pedido do Order virava 503. O site pede para não fazer teste de carga contra o PTAX.

**Solução** (`ptax.py`):

- cache em memória por moeda, com TTL de 5 min (`CACHE_SECONDS`);
- **um lock por moeda**: quando várias requisições chegam juntas com o cache vazio, só uma vai ao
  PTAX e as outras esperam e usam o valor que ela trouxe;
- se o PTAX falhar e houver uma cotação de até 24 h, o serviço responde com ela (`X-Cache: stale`)
  em vez de 503, e só tenta o PTAX de novo a cada 30 s;
- a lista de moedas fica em cache por 24 h.

**Medição** (`bench/bench.py`): 100 × `GET /exchange/USD/BRL`, 5 em paralelo, serviço recém-iniciado,
em 18/11/2026, Windows 11, contra o PTAX real:

| | Sem cache (`CACHE_SECONDS=0`) | Com cache | Diferença |
|---|---|---|---|
| Latência p50 | 233,9 ms | 3,7 ms | −98% |
| Latência p95 | 279,7 ms | 5,5 ms | −98% |
| Vazão | 20 req/s | 347 req/s | 17× |
| Chamadas ao PTAX | 101 | 2 | −98% |

Com cache, as 5 primeiras requisições chegaram juntas e só **uma** foi ao PTAX (a outra chamada
é a lista de moedas). O fallback para cotação antiga está coberto por testes
(`test_ptax_fora_serve_cotacao_antiga`, `test_cotacao_antiga_demais_vira_503`).

**Limite:** o cache vive na memória do processo. Com mais de uma réplica, cada pod tem o seu, o
que dá no máximo uma chamada ao PTAX por moeda a cada 5 min por pod.

### 2. Varredura de vulnerabilidades no pipeline

**Problema:** uma dependência com vulnerabilidade conhecida (ou uma imagem base desatualizada)
chegava ao Docker Hub e ao EKS sem ninguém perceber.

**Solução** (`Jenkinsfile`, `Dockerfile`):

- **`Test & Audit`**: o estágio `test` do Dockerfile roda `pytest` e
  [`pip-audit`](https://github.com/pypa/pip-audit) nas dependências fixadas em `requirements.txt`.
  Teste falhando ou vulnerabilidade conhecida param o build;
- **`Image Scan`**: o [Trivy](https://trivy.dev) varre a imagem final (pacotes do Debian da
  imagem base + pacotes Python) e **barra a publicação** se houver vulnerabilidade HIGH ou
  CRITICAL que já tenha correção;
- versões fixas em `requirements.txt`, `docker build --pull` (imagem base sempre atualizada) e
  imagem final sem root (`USER 10001` e `runAsNonRoot` no Kubernetes).

**Medição (`pip-audit`, 18/11/2026):**

| Dependências | Resultado |
|---|---|
| antigas: `fastapi==0.109.0`, `uvicorn==0.27.0`, `httpx==0.26.0` | **15 vulnerabilidades em 2 pacotes** (fastapi e starlette): o pipeline para |
| atuais: `fastapi==0.165.0`, `uvicorn==0.54.0`, `httpx==0.28.1` | **nenhuma**: o pipeline segue |

**Achado real do Trivy (18/11/2026).** A primeira versão da imagem foi **barrada** (exit code 1)
com 2 vulnerabilidades HIGH que já tinham correção:

| Pacote | CVE | De onde vinha | Correção no `Dockerfile` |
|---|---|---|---|
| `zlib1g` 1.3.1-1+b1 | CVE-2026-85091 | Debian da imagem base, ainda sem o patch `deb13u1` | `apt-get upgrade` no estágio base |
| `setuptools` 70.3.0 | CVE-2025-47273 | embutido no próprio `pip` (`pip/_vendor`) | imagem final sem `pip` (não é usado em produção) |

Depois da correção, o mesmo comando do Trivy passou com **exit code 0**.

## Docker, Kubernetes e CI/CD

- `Dockerfile`: `python:3.13-slim`, 3 estágios (base, test, final).
- `k8s/k8s.yaml`: Deployment + Service `ClusterIP` `exchange:8080`, readiness e liveness no
  health-check, requests 64Mi/50m e limits 128Mi/200m. Sem Secret: o PTAX não usa chave.
- `Jenkinsfile`: `Test & Audit` → `Image Scan` → `Build & Push Image` (amd64 + arm64) →
  `Deploy to EKS` (pula se o cluster estiver desligado).
