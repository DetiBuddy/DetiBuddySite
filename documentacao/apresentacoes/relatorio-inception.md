# DetiBuddy — Relatório de Inception

**Projeto em Engenharia Informática (43675)** · Licenciatura em Engenharia Informática · DETI, Universidade de Aveiro · 2026/2027
**Metodologia:** OpenUP (fase de Inception) com práticas ágeis

---

## 1. Introdução e âmbito da fase

Este relatório documenta a fase de **Inception** do DetiBuddy, cujo objetivo é perceber *o que construir*: definir a visão e as fronteiras do sistema, identificar os stakeholders, escolher os requisitos mais críticos, propor uma solução candidata, avaliar a viabilidade técnica e fazer uma estimativa de alto nível de custo, calendário e riscos.

Complementa a apresentação de 8 minutos, o calendário do projeto e o plano de comunicação exigidos à equipa.

## 2. Contexto

Os estudantes do DETI usam no dia a dia vários serviços dispersos da Universidade de Aveiro: PACO (horários, propinas), e-learning (disciplinas e avaliações), MySAS/SAS (refeições, ementas), sistema de parques, senhas de atendimento, email institucional e documentos dispersos (regulamentos, calendário, contactos de docentes). Cada um tem a sua interface e o seu login (a conta @ua.pt dá acesso à maioria).

Em paralelo, os assistentes de IA generalistas são cómodos, mas enviam as conversas para infraestrutura externa, o que é inadequado para dados pessoais e académicos.

## 3. Problema

> Aceder a informação simples do quotidiano académico exige navegar por vários sistemas, e as alternativas com IA não garantem privacidade nem fiabilidade (fontes oficiais).

Consequências: tempo perdido, avisos e prazos falhados, dúvidas sobre regras que ficam sem resposta clara.

## 4. Visão

O DetiBuddy é um assistente conversacional (chatbot) que funciona como um "amigo digital" do estudante: responde em linguagem natural (PT, idealmente EN) sobre horários, datas de avaliação, ementas, parques, senhas, regulamento e contactos do DETI, **mantendo conversas e dados pessoais dentro da infraestrutura da UA**.

### 4.1 Fronteiras do sistema

| Dentro do âmbito | Fora do âmbito (agora) |
|---|---|
| Autenticação com conta UA | Alterar dados nos serviços UA (só leitura) |
| Conectores para serviços UA (PACO, e-learning, SAS, parques, senhas) | Substituir ou modificar os sistemas da UA |
| Base de conhecimento do DETI com citação de fontes (RAG) | Aconselhamento fora de fontes oficiais |
| Pelo menos um canal (web + Discord/Slack/Signal) | Suporte a todos os canais em simultâneo |
| Notificações proativas (senhas, parques) | Emails/propinas (prioridade *Could*) |
| Administração da base de conhecimento e dos conectores | App Android (prioridade *Could*) |

## 5. Objetivos

1. Garantir que dados e conversas não saem da infraestrutura da universidade.
2. Ligar o assistente a serviços reais da UA (PACO, SAS, parques, reitoria, STIC) de forma modular e extensível.
3. Dar ao assistente conhecimento fiável sobre o DETI (regulamentos, calendário, contactos), com respostas fundamentadas em fontes oficiais.
4. Disponibilizar o assistente em pelo menos um canal já usado pelos estudantes, com possível complemento por app Android.
5. Testar com estudantes reais: utilidade, usabilidade e confiança.

## 6. Stakeholders e critérios de sucesso

| Stakeholder | Interesse | Critério de sucesso |
|---|---|---|
| Estudantes do DETI | Informação rápida e fiável | Respostas corretas, uso simples, confiança na privacidade |
| Administradores / DETI | Manter conhecimento e integrações | Atualizar documentos e adicionar conectores sem tocar no núcleo |
| STIC | Segurança e integridade dos sistemas UA | Integração autorizada, sem sobrecarga nem exposição de credenciais |
| Docente orientador / curso | Qualidade técnica e processo | Entregas e marcos cumpridos, avaliação com utilizadores |
| Equipa | Aprendizagem e resultado | MVP funcional e demonstrável |

## 7. Requisitos críticos (resumo)

Detalhados em `docs/requisitos.md` (26 user stories, 11 use cases, 6 NFR). Prioridades MoSCoW; o **MVP** corresponde às histórias *Must*.

| Épico | Must (MVP) | Should / Could |
|---|---|---|
| E1 Privacidade e autenticação | US-01 login UA, US-02 dados na infraestrutura UA | US-03 apagar histórico |
| E2 Serviços UA | US-04 horário, US-05 datas de testes, US-06 ementa | Saldo, parques, senhas (S); propinas, email (C) |
| E3 Conhecimento DETI | US-12 regulamento, US-13 contacto docente, US-15 fonte oficial | US-14 calendário escolar |
| E4 Canais | US-16 interface web | US-17 Discord/Slack/Signal (S), US-18 Android (C) |
| E5 Notificações | — | US-19, US-21 (S), US-20 (C) |
| E6 Administração | US-22 base de conhecimento, US-23 novo conector | US-24 métricas |
| E7 Avaliação | US-26 validação com estudantes reais | US-25 feedback por resposta |

**Requisitos não funcionais decisivos:** privacidade (NFR-01), nunca inventar dados de serviços (NFR-02), resposta típica < 10 s (NFR-03), extensibilidade dos conectores (NFR-04), segurança de credenciais e logs sem dados pessoais (NFR-05).

## 8. Solução candidata e arquitetura de alto nível

### 8.1 Visão geral

```
 Estudante ──► Canal (Web / Discord·Slack·Signal)
                    │
                    ▼
            ┌───────────────┐        ┌──────────────────┐
            │  Backend /    │◄──────►│ Skynet (LLM)     │
            │  Orquestrador │        └──────────────────┘
            │  (servidor    │        ┌──────────────────┐
            │   DETI)       │◄──────►│ Base conhecimento│
            └──────┬────────┘        │ (RAG + índice)   │
                   │                 └──────────────────┘
        Interface comum de conectores
   ┌────────┬───────┼────────┬─────────┬──────────┐
  PACO  e-learning  SAS   Parques   Senhas    (novos…)
```

*O diagrama formal será desenhado em draw.io e guardado em `docs/`.*

### 8.2 Componentes

- **Canais:** interface web (MVP) + um canal de mensagens (a decidir).
- **Orquestrador:** recebe a mensagem, o LLM identifica a intenção e chama o conector ou o RAG apropriado (*tool calling*).
- **Conectores:** módulos com uma interface comum (nome, descrição das capacidades, permissões); registar um novo não altera o núcleo (NFR-04).
- **RAG:** extração de texto, *chunking*, indexação e resposta apenas com base nos excertos, com citação (documento + secção).
- **Autenticação:** login UA delegado, sem guardar a password do estudante (a validar com o STIC).
- **Notificações:** *scheduler* que monitoriza condições (senha, parque) e envia para o canal do estudante.
- **Alojamento:** agente no servidor do DETI; modelo servido pelo skynet.

### 8.3 Decisões a fechar na Elaboration

Tecnologia de backend e de vetores, canal de mensagens do MVP, mecanismo de autenticação delegada, estratégia por serviço (API vs. automação web), formato de cache.

> As escolhas tecnológicas concretas são propostas da equipa e ficam validadas pelo protótipo técnico.

## 9. Viabilidade técnica

| Área | Avaliação | Ação |
|---|---|---|
| LLM no skynet | Provável (infraestrutura da UA) | Confirmar modelo, quotas, latência e suporte a *tool calling* |
| APIs dos serviços UA | **Incerta** — pode não haver API | Perguntar ao STIC; plano B: automação web (Selenium/Playwright) |
| Autenticação sem guardar password | **Incerta** | Investigar SSO/OAuth/CAS da UA com o STIC |
| RAG sobre documentos do DETI | Viável, técnica madura | Protótipo com 1–2 regulamentos |
| Resposta < 10 s | Plausível para consultas simples | Medir no protótipo (automação web é o ponto crítico) |
| Canais (Discord/Slack/Signal) | Viável (APIs de bots) | Escolher um e prototipar |

**Protótipos técnicos propostos para esta fase:** (1) pergunta → skynet → resposta; (2) leitura de um serviço UA (ex.: ementa SAS, pública); (3) RAG mínimo com citação.

## 10. Apresentação do projeto (8 min)

| # | Slide | Conteúdo |
|---|---|---|
| 1 | Título e equipa | DetiBuddy; membros |
| 2 | Contexto | Serviços UA dispersos; privacidade |
| 3 | Problema | Secção 3 |
| 4 | Objetivos | Os 5 objetivos |
| 5 | Tarefas | Módulos e responsáveis |
| 6 | Resultados esperados | MVP, avaliação com estudantes |
| 7 | Trabalho relacionado (rascunho) | Secção 13 |
| 8 | Calendário e comunicação | Marcos, entregas, ferramentas |

## 11. Equipa e responsabilidades

| Membro | Nº | GitHub | Papel de gestão | Área técnica |
|---|---|---|---|---|
| Arthur Haerdy | 122886 | @Haerdy | Team Manager | Backend, integração de IA |
| Francisco Amaro | 119869 | @ZbSmasher | Product Owner | Backend, implementação geral e integração |
| Geovana Mayer | 128238 | @mayer678 | Comunicação | Backend, integração de APIs |
| Lisle Faria | 126035 | @LisleAFaria | DevOps master | Frontend, UI/UX |
| Samuel Ramos | 120088 | @16sgfr | Arquiteto | Frontend, UI/UX |

## 12. Calendário do projeto

### 12.1 Lista de tarefas por módulo

Cada tarefa tem relação com uma entrega (secção 12.4).

**Módulo A — Núcleo e orquestração** (Arthur, Francisco) → *Software*
- A1. Serviço do orquestrador e ligação ao skynet (Arthur)
- A2. Gestão de sessão e contexto de conversa (Francisco)
- A3. *Tool calling* e política de "não inventar dados" (Arthur, Francisco)

**Módulo B — Conectores** (Geovana, Francisco) → *Software*
- B1. Interface comum de conectores e registo (Francisco)
- B2. Conector SAS (ementa) e PACO (horário) (Geovana)
- B3. Conector e-learning (datas de testes) (Geovana, Francisco)
- B4. Conectores de parques e senhas (Geovana)

**Módulo C — Conhecimento (RAG)** (Arthur, Samuel) → *Software, base de conhecimento*
- C1. Ingestão, *chunking* e indexação de documentos (Arthur)
- C2. Respostas com citação e fallback "não sei" (Arthur, Samuel)
- C3. Ferramenta de administração para atualizar documentos (Samuel)

**Módulo D — Autenticação e segurança** (Samuel, Geovana) → *Software, relatório de arquitetura*
- D1. Análise do login UA com o STIC (Samuel, Geovana)
- D2. Sessão segura, gestão de credenciais, logs sem dados pessoais (Samuel)

**Módulo E — Frontend e canais** (Lisle, Samuel) → *Software*
- E1. Design UI/UX e protótipo (Lisle, Samuel)
- E2. Interface web de chat, com feedback 👍/👎 (Lisle)
- E3. Integração com canal de mensagens; notificações (Samuel, Lisle)

**Módulo F — DevOps e infraestrutura** (Lisle) → *Software, documentação*
- F1. Repositório, CI/CD e ambientes no servidor DETI (Lisle)
- F2. Monitorização e métricas (Lisle)

**Módulo G — Gestão, comunicação e avaliação** (Arthur, Francisco, Geovana) → *Relatórios, apresentações*
- G1. Plano de projeto, calendário e reuniões (Arthur)
- G2. Backlog e priorização (Francisco)
- G3. Relatórios, apresentações e questionário de avaliação (Geovana)

### 12.2 Cronograma (indicativo)

As datas exatas dependem do calendário da unidade curricular e do evento **Students@DETI**, por confirmar. O plano está organizado em semanas relativas.

| Fase (OpenUP) | Semanas | Foco |
|---|---|---|
| Inception | 1–4 | Visão, requisitos, protótipos técnicos, plano |
| Elaboration | 5–8 | Arquitetura estável, esqueleto ponta a ponta, riscos principais resolvidos |
| Construction | 9–16 | MVP (Must), depois Should; iterações de 2 semanas |
| Transition | 17–20 | Avaliação com estudantes, correções, relatório final, demo (Students@DETI) |

### 12.3 Marcos (4)

| # | Marco | Descrição (< 120 caracteres) |
|---|---|---|
| M1 | Fim da Inception | Visão, requisitos e plano aprovados; protótipos técnicos executados |
| M2 | Fim da Elaboration | Arquitetura validada com fluxo ponta a ponta (login, 1 conector, RAG) |
| M3 | MVP | Todas as histórias Must operacionais no servidor do DETI |
| M4 | Entrega final | Avaliação com estudantes concluída, relatório e demo entregues |

### 12.4 Entregas (*deliverables*)

- Repositório Git com o código e a documentação (`docs/`)
- Pacote de software: backend, conectores, frontend/canal, base de conhecimento
- Documento de requisitos e diagrama de use cases
- Relatório de arquitetura (incluindo segurança e privacidade)
- Relatório de avaliação com estudantes
- Apresentações intermédias e final; relatório final

## 13. Trabalho relacionado (rascunho)

A completar com referências verificadas até à versão final.

- **Assistentes generalistas na nuvem:** boa qualidade de linguagem, mas sem integração com serviços UA e com dados fora da instituição.
- **Frameworks de chatbot e agentes (ex.: Rasa, LangChain/LlamaIndex):** blocos de construção para diálogo, *tool calling* e RAG.
- **Assistentes universitários:** existem casos noutras instituições (ex.: assistentes de apoio a disciplinas e a informação institucional); a analisar quanto a arquitetura e resultados.
- **Aplicações e portais da UA:** cobrem cada serviço isoladamente; o DetiBuddy propõe uma camada conversacional unificada.
- **LLM locais/auto-alojados:** alternativa para garantir privacidade, relevante para o skynet.

## 14. Riscos

| # | Risco | Prob. | Impacto | Mitigação |
|---|---|---|---|---|
| R1 | Serviços UA sem API | Alta | Alto | Automação web como plano B; contactar STIC cedo |
| R2 | Autenticação delegada inviável sem guardar password | Média | Alto | Reunião STIC na Inception; alternativa de escopo (só dados públicos) |
| R3 | Automação web frágil (mudanças de HTML) | Alta | Médio | Isolar em conectores; testes; cache curta |
| R4 | Alucinações do LLM | Média | Alto | RAG com citação; respostas só com excertos; "não sei" |
| R5 | Latência do skynet (> 10 s) | Média | Médio | Medir cedo; *streaming*; modelo mais leve |
| R6 | Disponibilidade do skynet/servidor | Baixa | Alto | Contacto com responsáveis; ambiente de desenvolvimento local |
| R7 | Escopo excessivo | Média | Médio | MoSCoW; MVP primeiro; iterações curtas |
| R8 | Conformidade de privacidade (RGPD) | Média | Alto | Minimizar dados, logs sem dados pessoais, apagar histórico |
| R9 | Baixa adesão na avaliação | Média | Médio | Recrutar cedo; questionário curto |

## 15. Estimativa de alto nível

- **Custo:** sem custos de licenciamento previstos (infraestrutura UA e ferramentas abertas); o principal custo é o esforço da equipa (5 pessoas ao longo do semestre).
- **Calendário:** ~20 semanas em 4 fases, com iterações de 2 semanas na Construction.
- **Maior incerteza:** acesso e autenticação nos serviços UA (R1, R2), a esclarecer nas primeiras semanas.

## 16. Plano de comunicação (foco ágil)

| Tema | Definição |
|---|---|
| Repositório e site | Repositório Git privado (codeUA/GitHub); docentes e orientadores com acesso |
| Documentação | Requisitos em `docs/requisitos.md`, alterações por *pull request* |
| Plano e backlog | Quadro de tarefas (issues/projects) ligado ao repositório |
| Reuniões | *Stand-up* curto 2×/semana; planeamento e retrospetiva a cada iteração de 2 semanas; reunião com orientador |
| Canal da equipa | Chat privado (Discord/Slack) para comunicação diária |
| Relatórios e apresentações | Preparados no repositório; revistos por PR antes de cada entrega |
| Informação privada | Credenciais e dados só em gestores de segredos, nunca no Git |

**Papéis e atividades:** Team Manager (planeamento e acompanhamento), Product Owner (backlog e prioridades), Comunicação (relatórios, apresentações, contacto externo), DevOps (CI/CD e ambientes), Arquiteto (decisões técnicas e coerência do sistema).

**Princípios ágeis:** pessoas e interações acima de processos; software funcional acima de documentação extensa; colaboração com utilizadores; resposta à mudança.

## 17. Questões em aberto

1. Os serviços UA (PACO, e-learning, MySAS) têm API? Caso contrário, é aceitável automação (Selenium) — validar com o STIC.
2. Como fazer autenticação delegada sem guardar a password?
3. Quais os canais do MVP (web + Discord, Slack ou Signal)?
4. Data do Students@DETI e prazos oficiais da unidade curricular.
5. Especificações do skynet: modelo, quotas, acessos.

## 18. Conclusão

O DetiBuddy é um projeto com visão clara, âmbito controlado por MoSCoW e uma arquitetura modular que atende aos objetivos de privacidade e extensibilidade. A viabilidade técnica depende sobretudo do acesso aos serviços UA e da autenticação delegada, riscos a tratar logo no início da Elaboration. Proposta: avançar para a Elaboration com os protótipos técnicos da secção 9.

## Referências

- Eclipse. *Introduction to OpenUP*. https://eclipse.org/epf/general/OpenUP.pdf
- OpenUP — Inception phase e milestones. http://epf.eclipse.org/wikis/openup/
- Scott Ambler (2013). *Going beyond Scrum*. Disciplined Agile Consortium.
- Duncan Haughey. *Project planning: a step by step guide*. http://www.projectsmart.co.uk/project-planning-step-by-step.php
