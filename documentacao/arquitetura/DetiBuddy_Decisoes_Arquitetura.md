# DetiBuddy — Documento de Decisões de Arquitetura (Architecture Decision Records - ADRs)
## Engenharia de Software — Fase de Elaboração (Milestone 2)

---

### Resumo Executivo da Arquitetura v2

A arquitetura do **DetiBuddy** foi consolidada e validada durante a Fase de Elaboração (Milestone 2), evoluindo da visão preliminar da *Inception* para a **Arquitetura v2**, uma topologia modular, assíncrona e desacoplada baseada em microsserviços e no protocolo **MCP (*Model Context Protocol*)**.

O sistema está confinado à infraestrutura soberana da Universidade de Aveiro (servidores DETI e cluster Skynet), garantindo conformidade rigorosa com o RGPD e total soberania dos dados dos estudantes e da comunidade académica.

---

### ADR-01: Topologia de Borda e Cliente Web (Traefik + React PWA)
* **Estado:** Aprovado
* **Contexto:** Os estudantes acedem ao assistente através de smartphones, computadores portáteis e tablets tanto na rede do campus como no exterior.
* **Decisão:** Adoção do proxy reverso de borda **Traefik** e de uma aplicação web progressiva **React PWA** (*Progressive Web App*).
* **Justificação:**
  * Permite instalação direta no ecrã inicial do smartphone sem fricção de lojas de aplicações.
  * Suporte nativo a modos claro/escuro (*Dark/Light Mode*) e design responsivo fluido.
  * O Traefik gere terminação TLS/SSL, encaminhamento de rotas por subdomínio e *rate limiting* preventivo.
* **Consequências:** A aplicação nativa Android passa a requisito de expansão (*Could Have*), garantindo cobertura universal no MVP através da web móvel.

---

### ADR-02: Backend Assíncrono com Server-Sent Events (FastAPI + SSE)
* **Estado:** Aprovado
* **Contexto:** Assistentes baseados em LLM geram texto token a token. O tempo até à primeira resposta (*Time To First Token - TTFT*) é crítico para a experiência conversacional.
* **Decisão:** Desenvolvimento do backend em Python utilizando a framework assíncrona **FastAPI** com streaming de respostas via **SSE (*Server-Sent Events*)**.
* **Justificação:**
  * O SSE opera sobre HTTP standard, contorna firewalls institucionais sem bloqueios de WebSockets e suporta reconexão automática nativa no browser.
  * O FastAPI assegura tipagem estrita com Pydantic v2 e alto desempenho em I/O assíncrono.
* **Consequências:** Menor overhead operacional do que sockets bidirecionais permanentes, com streaming fluido das respostas do LLM.

---

### ADR-03: Desacoplamento de Serviços Externos via Model Context Protocol (MCP)
* **Estado:** Aprovado
* **Contexto:** O DetiBuddy integra múltiplos sistemas universitários (PACO, Moodle, SAS, M365, IPMA, Parques, Bibliotecas). Alguns dependem de autorização dos STIC e outros disponibilizam APIs abertas.
* **Decisão:** Cada domínio funcional é isolado no seu próprio **Servidor MCP dedicado**, comutável via variável de ambiente entre `mock`, `real` e `scraper`.
* **Justificação:**
  * O MCP padroniza as *tool calls* do agente de IA através de esquemas JSON-RPC estruturados.
  * Permite testar o sistema de ponta a ponta com mocks realistas enquanto decorrem as aprovações institucionais de segurança (Planos de Contingência A, B e C).
* **Consequências:** Qualquer alteração na API de um serviço externo afeta apenas o respetivo servidor MCP, sem impacto no núcleo conversacional do agente.

---

### ADR-04: Autenticação Institucional e IdP Simulado (Keycloak OIDC)
* **Estado:** Aprovado
* **Contexto:** A autenticação final requer integração com o IdP dos STIC (OpenID Connect / SAML). Durante o desenvolvimento, a equipa necessita de testar fluxos com múltiplos perfis de estudante e docente.
* **Decisão:** Incorporação de uma instância Docker do **Keycloak** a replicar o IdP institucional da UA em modo OIDC.
* **Justificação:**
  * Permite emitir tokens JWT com os mesmos *claims* (NMEC, perfil, curso) e testar a política de acessos RBAC sem depender de credenciais reais dos STIC.
* **Consequências:** A transição para o IdP de produção exige apenas a alteração dos endpoints de autorização e chaves públicas no `.env`.

---

### ADR-05: Persistência Híbrida e Base de Dados Vetorial (PostgreSQL + pgvector)
* **Estado:** Aprovado
* **Contexto:** O sistema necessita de armazenar dados relacionais (sessões, utilizadores, histórico efêmero de conversas) e realizar pesquisa semântica em regulamentos pedagógicos e bibliografia.
* **Decisão:** Unificação da persistência em **PostgreSQL** com a extensão **pgvector**.
* **Justificação:**
  * Evita a complexidade operacional e risco de desfasamento de manter uma base relacional e uma base vetorial dedicada (como Qdrant ou Pinecone) em contentores separados.
  * Suporta **Busca Híbrida** combinando similaridade densa de embeddings vetoriais com pesquisa léxica BM25 / trigramas para códigos de UC e nomes de docentes.
* **Consequências:** Simplificação arquitetural, integridade transacional ACID estrita e backups unificados num único volume persistente.

---

### ADR-06: Cache, Filas de Mensagens e Workers Assíncronos (Redis + ARQ)
* **Estado:** Aprovado
* **Contexto:** Consulta frequente de ementas, estado de ocupação de parques e processamento pesado de indexação de PDFs regulamentares e envio de notificações push.
* **Decisão:** Adoção de **Redis** em memória associado a um worker assíncrono **ARQ**.
* **Justificação:**
  * Implementação de estratégia *quickload* com TTL (cache de curta duração para parques e ementas).
  * O worker ARQ descarrega o processo principal da API durante o chunking e embedding de regulamentos e no despacho de notificações.
* **Consequências:** Redução drástica da latência para chamadas repetidas e resposta instantânea para utilizadores concorrentes.

---

### ADR-07: Inferência Soberana de LLM (Cluster Skynet do DETI)
* **Estado:** Aprovado
* **Contexto:** Proteção de dados pessoais e regulamentação europeia (RGPD) no tratamento de conversas e dados académicos de estudantes universitários.
* **Decisão:** O modelo de linguagem (LLM) corre localmente no cluster de servidores **Skynet** do Departamento de Eletrónica, Telecomunicações e Informática (DETI).
* **Justificação:**
  * Garantia absoluta de que nenhuma mensagem, dúvida académica ou dado de contexto transita para servidores de clouds comerciais (OpenAI, Anthropic ou Google).
  * API de inferência de alta velocidade compatível com o padrão OpenAI via rede interna segura.
* **Consequências:** Soberania técnica completa e custos de operação nulos em subscrições externas de APIs proprietárias.

---

### ADR-08: Alinhamento de Âmbito e Gestão de Riscos (Elaboração vs. Inception)
* **Estado:** Aprovado
* **Contexto:** Na fase de Inception foram identificadas ideias alargadas, incluindo consulta de saldo de cartões bancários SAS e pagamento de propinas.
* **Decisão:** Exclusão fundamentada de transações bancárias e saldos monetários do âmbito do projeto (*Won't Have*).
* **Justificação:**
  * Operações financeiras acarretam requisitos de certificação bancária (PCI-DSS) e autorizações institucionais complexas que inviabilizariam o cronograma semestral de PEI.
  * Foco prioritário na agregação pedagógica, RAG de regulamentos, horários, avaliações e serviços de campus.
* **Consequências:** Redução de riscos críticos e garantia de entrega de um MVP robusto, funcional e perfeitamente testável.
