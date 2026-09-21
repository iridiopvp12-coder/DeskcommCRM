# DeskcommCRM – Análise de Arquitetura e Especificações Técnicas

Este documento consolida a análise técnica e a topologia de arquitetura do **DeskcommCRM**, um sistema focado em CRM multicanal e integração nativa com modelos de inteligência artificial (AI Sales OS).

A solução foi projetada como um SaaS multi-tenant, consolidando a interface do usuário, o roteamento assíncrono de eventos e o banco de dados relacional sob uma mesma infraestrutura.

---

## 1. Topologia da Arquitetura e Stack Tecnológica

O ecossistema foi construído para evitar a dependência de orquestradores de automação intermediários (como n8n ou orquestradores em Python).

### 1.1. Core Application (Frontend e Backend)
*   **Framework:** Construído sobre **Next.js 16**, adotando a abordagem full-stack (Server Actions e API Routes).
*   **Linguagem:** TypeScript, assegurando tipagem rigorosa na comunicação de dados entre a camada visual e o banco.
*   **Processamento Assíncrono:** Implementação de arquitetura baseada em filas e *workers* para o processamento de eventos, assegurando que gargalos na entrada/saída de requisições externas não congelem a aplicação principal.

### 1.2. Camada de Dados e Estado em Tempo Real
*   **Banco de Dados:** **PostgreSQL** orquestrado via **Supabase**.
*   **Real-time / Pub-Sub:** Utiliza os canais de WebSockets do **Supabase Realtime** para persistência e propagação imediata de estado. Atualizações em instâncias de atendimento (como movimento de leads no Kanban) são refletidas nos clientes via *broadcast* sem a necessidade de polling.

### 1.3. Integração com Redes de Mensageria
*   **Motor Principal:** **WAHA (WhatsApp HTTP API)**. O sistema utiliza a API do WAHA como ponte para injetar os dados de conectividade, abstraindo as lógicas intrínsecas de protocolos WebSocket da rede da Meta (Baileys/Puppeteer).
*   **Voice Integration:** O repositório engloba uma arquitetura experimental (`WaCalls`) desenhada para tratar eventos de chamadas de voz oriundas da rede.

### 1.4. Arquitetura de Inteligência Artificial e LLMs
A orquestração de IA está profundamente acoplada ao banco de dados, atuando antes da liberação (dispatch) de mensagens de saída (outbound):
*   **RAG e Guardrails:** As requisições direcionadas para APIs de LLM (como OpenAI ou Anthropic) passam por checagens de *guardrails* internos e cruzamento de contexto (RAG) direto das regras estruturadas no PostgreSQL.
*   **Preparação de Protocolos:** A estrutura está sendo moldada para ser compatível com o padrão de integração **MCP (Model Context Protocol)**.

---

## 2. Infraestrutura e Implantação (Deployment)

O modelo de *deployment* é desenhado para conteinerização total, visando escalabilidade horizontal ou implantação limpa em servidores de borda.

*   **Conteinerização:** A topologia sobe e orquestra **7 containers simultâneos** através do **Docker Compose**. A diferenciação de ambientes é feita através de arquivos específicos (`docker-compose.prod.yml` e `docker-compose.build.yml`).
*   **Gateway e Reverse Proxy:** O **Traefik** atua como camada de proxy reverso (`docker-compose.traefik.yml`), lidando com o roteamento dinâmico de portas, balanço de carga e provisionamento de certificados SSL/TLS (Let's Encrypt).
*   **Observabilidade:** Instrumentação nativa do **Sentry**, com configurações separadas para o *backend* (`sentry.server.config.ts`) e para o *Edge* (`sentry.edge.config.ts`), garantindo *tracing* e telemetria de exceções não tratadas.
*   **Automação de Provisionamento:** O script raiz `comecar.sh` automatiza o provisionamento (verificando dependências de SO, criando as variáveis de ambiente e acionando o build) para instâncias baseadas em Debian/Ubuntu.

---

## 3. Estudo de Viabilidade de Refatoração: Substituição do Motor WAHA

O DeskcommCRM foi acoplado fortemente ao padrão de dados do WAHA. Uma eventual substituição desse motor por soluções alternativas (como a **uazapi**) demandaria refatoração em pontos críticos da engenharia de software:

1.  **Lifecycle de Conexão e Autenticação (QR Code):** As rotas e *controllers* no Next.js responsáveis pela requisição e renderização do payload do QR Code precisariam ser reescritos, visto que os *endpoints* e os tipos de dados devolvidos por bibliotecas alternativas divergem do WAHA.
2.  **Middlewares de Parseamento de Webhooks:** O sistema espera um contrato JSON específico nas requisições entrantes. Seria compulsório implementar uma camada de *Adapter* (middleware) para traduzir o payload da nova API para o formato de persistência já consolidado no PostgreSQL.
3.  **Mapeamento de Request/Response de Mídias:** A construção dos *headers* e o encapsulamento de envios `POST` contendo objetos não-textuais (arquivos binários, áudios, contatos) teriam que ser mapeados e reescritos para alinhar às exigências de assinatura da nova API.

**Conclusão Técnica:** O nível de complexidade para a substituição do motor de mensageria é considerado **Alto**. A arquitetura atual favorece a manutenção do motor WAHA, permitindo ao time focar na evolução das camadas de inteligência artificial (LLMs) e na estabilidade do processamento assíncrono.
