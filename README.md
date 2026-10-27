# Observability Stack

Infraestrutura centralizada de observabilidade para aplicações Java/Spring Boot, servidores Linux e bancos PostgreSQL, utilizando o ecossistema Grafana.

## Objetivo

Monitorar múltiplas aplicações a partir de um servidor dedicado, reunindo métricas, logs, traces e alertas. O ambiente será validado localmente com Docker Compose antes da implantação em uma VM Linux.

> **Escopo inicial:** Grafana + Prometheus + métricas do Spring Boot Actuator. Loki, Alloy e Tempo serão incorporados gradualmente.

## Arquitetura

```text
Servidores de aplicação (Apache HTTP Server/Tomcat + Java/Spring Boot)
  ├── Spring Boot Actuator + Micrometer -> /actuator/prometheus
  ├── node_exporter (futuro)           -> métricas do Linux
  ├── postgres_exporter (futuro)       -> métricas do PostgreSQL
  └── Grafana Alloy (futuro)           -> logs e telemetria
                       |
                       | Rede privada
                       v
Servidor central de observabilidade (Docker Compose)
  ├── Prometheus -> coleta e armazena métricas
  ├── Loki       -> logs (futuro)
  ├── Tempo      -> traces (futuro)
  └── Grafana    -> dashboards, exploração e alertas
```

As aplicações Java permanecem nos seus servidores atuais, sem necessidade de executá-las em contêineres. O Apache pode atuar como proxy reverso ou o ambiente pode utilizar Tomcat; os endpoints de coleta devem ser ajustados à implantação real.

## Pré-requisitos

- Docker Engine e Docker Compose Plugin (`docker compose`).
- Aplicação Spring Boot acessível pela máquina que executa o Prometheus.
- Endpoint `/actuator/prometheus` habilitado e protegido.
- Portas locais `3000` (Grafana) e `9090` (Prometheus) disponíveis.

### Dependências Spring Boot (Maven)

Caso ainda não estejam presentes:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
```

Exemplo de `application.yml` (ajustar conforme a versão do Spring Boot e a política de segurança):

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,prometheus
  metrics:
    tags:
      application: conecta
```

> Em algumas versões do Spring Boot, propriedades de métricas podem mudar. Confirme a compatibilidade com a versão utilizada. Não exponha endpoints administrativos à internet.

## Estrutura do repositório

```text
observability/
├── README.md
├── compose.yaml
├── .env.example
├── .gitignore
├── prometheus/
│   └── prometheus.yml
├── grafana/
│   └── provisioning/
│       ├── datasources/
│       │   └── datasources.yml
│       └── dashboards/
├── loki/                    # fase 2
│   └── loki.yaml
├── tempo/                   # fase 3
│   └── tempo.yaml
└── alloy/                   # fase 2
    └── config.alloy
```

## Configuração inicial

### 1. `compose.yaml`

```yaml
services:
  prometheus:
    image: prom/prometheus:v3.5.0
    restart: unless-stopped
    command:
      - --config.file=/etc/prometheus/prometheus.yml
      - --storage.tsdb.retention.time=15d
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - prometheus_data:/prometheus
    ports:
      - "127.0.0.1:9090:9090"
    extra_hosts:
      - "host.docker.internal:host-gateway"
    networks:
      - observability

  grafana:
    image: grafana/grafana:12.1.1
    restart: unless-stopped
    environment:
      GF_SECURITY_ADMIN_USER: ${GRAFANA_ADMIN_USER}
      GF_SECURITY_ADMIN_PASSWORD: ${GRAFANA_ADMIN_PASSWORD}
    volumes:
      - grafana_data:/var/lib/grafana
      - ./grafana/provisioning:/etc/grafana/provisioning:ro
    ports:
      - "127.0.0.1:3000:3000"
    networks:
      - observability

volumes:
  prometheus_data:
  grafana_data:

networks:
  observability:
```

As imagens acima são referências fixadas para o exemplo. Antes de publicar em produção, valide versões suportadas e atualizações de segurança.

### 2. `prometheus/prometheus.yml`

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: spring-boot-local
    metrics_path: /actuator/prometheus
    static_configs:
      - targets:
          - host.docker.internal:8080
        labels:
          application: conecta
          environment: development
```

O exemplo pressupõe que o Spring Boot esteja acessível na porta `8080` do host. Em Linux, o mapeamento `host.docker.internal:host-gateway` permite resolver o endereço do host a partir do contêiner; confirme também que a aplicação aceita conexões nessa interface. Se estiver acessível apenas em `127.0.0.1`, adapte a rede ou o bind da aplicação **sem expor o Actuator publicamente**.

Para monitorar outras aplicações na VM, substitua os targets por endereços DNS/IP privados e utilize rótulos consistentes (`application`, `environment` e `instance`).

### 3. `grafana/provisioning/datasources/datasources.yml`

```yaml
apiVersion: 1

datasources:
  - name: Prometheus
    uid: prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
    editable: false
```

### 4. `.env.example`

```dotenv
GRAFANA_ADMIN_USER=admin
GRAFANA_ADMIN_PASSWORD=substitua-por-uma-senha-forte
```

Copie o arquivo para `.env` e defina uma senha segura:

```bash
cp .env.example .env
```

### 5. `.gitignore`

```gitignore
.env
*.log
```

Nunca versione credenciais reais ou dados sensíveis.

## Executando localmente

Na raiz do repositório:

```bash
docker compose config
docker compose up -d
docker compose ps
```

Acessos locais:

| Serviço | Endereço |
| --- | --- |
| Grafana | http://localhost:3000 |
| Prometheus | http://localhost:9090 |

Entre no Grafana com as credenciais definidas no `.env`. A fonte Prometheus é provisionada automaticamente.

### Validação da coleta

1. Confirme que `http://localhost:8080/actuator/prometheus` responde a partir da máquina da aplicação.
2. No Prometheus, acesse **Status → Targets** e verifique o target `spring-boot-local`.
3. Execute a consulta PromQL:

```promql
up{job="spring-boot-local"}
```

O resultado `1` indica coleta bem-sucedida; `0` indica falha na coleta. Ausência de série indica que o target ainda não foi carregado ou que há outro problema de configuração.

Consultas úteis:

```promql
jvm_memory_used_bytes
```

```promql
hikaricp_connections_active
```

A disponibilidade e os nomes exatos das métricas dependem da instrumentação e da versão do Spring Boot/Micrometer.

Para consultar logs dos contêineres:

```bash
docker compose logs -f prometheus grafana
```

Para parar os serviços sem apagar os dados:

```bash
docker compose down
```

> Evite `docker compose down -v`, pois remove os volumes persistentes.

## Dashboards iniciais

- Visão geral de aplicações: disponibilidade, taxa de requisições, erros HTTP e latência P95/P99.
- JVM: heap, garbage collection, threads e CPU.
- HikariCP: conexões ativas, ociosas e pendentes.
- Linux (após instalar node_exporter): CPU, memória, disco e rede.
- PostgreSQL (após instalar postgres_exporter): conexões, locks e indicadores do banco.

Utilize variáveis e rótulos para filtrar dashboards por `application`, `environment` e `instance`.

## Migração para VM centralizada

1. Provisionar VM Linux dedicada e instalar Docker Engine + Compose Plugin.
2. Clonar este repositório e criar o `.env` com credenciais seguras.
3. Ajustar os targets do Prometheus para IPs/DNS privados das aplicações.
4. Garantir conectividade de rede da VM de observabilidade até os endpoints de métricas.
5. Configurar DNS, HTTPS, autenticação e regras de firewall para acesso ao Grafana.
6. Manter Prometheus, Loki e Tempo restritos à rede interna; não publicar seus endpoints indiscriminadamente.
7. Definir retenção, dimensionamento de armazenamento, backup e monitoramento da própria stack.
8. Testar dashboards, alertas e recuperação após reinicialização.

Os bindings de portas `127.0.0.1` deste exemplo são adequados ao uso local. Na VM, publique o Grafana preferencialmente por um proxy reverso HTTPS com acesso restrito, em vez de abrir diretamente a porta `3000` para a internet.

## Próximas fases

### Fase 2 — Logs

- Adicionar Loki ao Compose central.
- Instalar Grafana Alloy nos servidores de aplicação.
- Coletar logs do Java e do Apache/Tomcat, preferencialmente estruturados em JSON.
- Adicionar rótulos de serviço, ambiente e instância, evitando rótulos de alta cardinalidade (como IDs de usuário ou requisição).
- Definir retenção e mascaramento de dados sensíveis.

### Fase 3 — Tracing

- Adicionar Grafana Tempo ao Compose central.
- Instrumentar Java com OpenTelemetry, avaliando a compatibilidade com a versão do Java/Spring Boot.
- Encaminhar traces via OTLP para o coletor e, depois, para o Tempo.
- Correlacionar logs e traces por `traceId` e `spanId`.
- Ajustar amostragem para controlar volume e custo.

### Fase 4 — Operação e alertas

- Criar alertas para indisponibilidade, aumento de HTTP 5xx, latência e saturação de recursos.
- Adicionar postgres_exporter, node_exporter e métricas do Apache/Tomcat conforme o ambiente.
- Estabelecer rotinas de atualização, backup, retenção e revisão de permissões.

## Segurança

- Nunca exponha `/actuator/prometheus` ou endpoints administrativos publicamente.
- Restrinja a coleta à rede privada, com firewall e autenticação quando necessário.
- Não registre senhas, tokens ou dados pessoais em logs e traces.
- Não armazene segredos em Git; utilize variáveis de ambiente ou mecanismo de segredos apropriado.
- Fixe e atualize versões de imagens após testes.
- Controle o acesso ao Grafana por perfis e, em produção, HTTPS.

## Referências oficiais

- [Grafana](https://grafana.com/docs/grafana/latest/)
- [Prometheus](https://prometheus.io/docs/)
- [Spring Boot Actuator](https://docs.spring.io/spring-boot/reference/actuator/)
- [Grafana Loki](https://grafana.com/docs/loki/latest/)
- [Grafana Tempo](https://grafana.com/docs/tempo/latest/)
- [Grafana Alloy](https://grafana.com/docs/alloy/latest/)
- [OpenTelemetry Java](https://opentelemetry.io/docs/languages/java/)
