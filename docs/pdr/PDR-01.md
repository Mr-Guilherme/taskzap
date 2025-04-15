# Product Design Review (PDR)  
**Projeto:** WhatsApp Task Bot com OpenAI e Google Calendar  
**Data:** 12/04/2025  
**Autor:** Guilherme Rodrigues

---

## 1. Visão Geral do Produto

O WhatsApp Task Bot é uma aplicação que permite a um usuário enviar mensagens de áudio para um número de WhatsApp. A aplicação escuta o áudio, transcreve utilizando o modelo Whisper da OpenAI, extrai tarefas com um modelo de linguagem (GPT), envia as tarefas de volta para o usuário para confirmação via WhatsApp e, após confirmação, adiciona os eventos no Google Calendar.

Este sistema é ideal para usuários que preferem se organizar por voz, com foco em produtividade rápida via mobile e integração fluida com calendário.

---

## 2. Público-alvo

- Profissionais que usam o WhatsApp como canal principal de comunicação  
- Usuários que preferem entrada de dados por voz  
- Pessoas que precisam organizar rotinas e compromissos com agilidade  

---

## 3. Objetivos do Projeto

- Reduzir o atrito na organização de tarefas a partir de comandos de voz  
- Automatizar o fluxo: voz → texto → tarefas → calendário  
- Usar ferramentas acessíveis e populares (WhatsApp, Google Calendar)  
- Minimizar custos e complexidade de infraestrutura  

---

## 4. Arquitetura e Componentes

### 4.1 Stack Tecnológica

| Componente           | Ferramenta / Libs                                    |
|----------------------|------------------------------------------------------|
| Backend              | Node.js + NestJS                                     |
| WhatsApp Bot         | [Baileys](https://github.com/WhiskeySockets/Baileys) |
| Transcrição de áudio | OpenAI Whisper (REST API via `fetch`)                |
| Extração de tarefas  | GPT-3.5 (REST API via `fetch`)                       |
| Integração de agenda | Google Calendar API                                  |
| Infraestrutura       | Docker, VPS, GitHub Actions CI/CD                    |
| Armazenamento        | Supabase (self-hosted)                               |
| Monitoramento        | Uptime Kuma (self-hosted)                            |

---

### 4.2 Fluxo de Dados

1. Usuário envia um áudio para o número do bot no WhatsApp  
2. Bot escuta com Baileys e baixa o áudio localmente  
3. Áudio é convertido via FFmpeg e enviado à OpenAI Whisper  
4. Texto transcrito é enviado à GPT-3.5 para extração de tarefas  
5. Bot responde com uma lista de tarefas formatadas no WhatsApp  
6. Usuário confirma ou edita tarefas via mensagem  
7. Eventos confirmados são inseridos no Google Calendar  
8. Áudio original é deletado do sistema após o processamento  

---

### 4.3 Diagrama de Componentes

```text
WhatsApp User
     |
     v
  [Baileys Bot]
     |
     v
[Transcrição (Whisper API)]
     |
     v
[Extração de Tarefas (GPT)]
     |
     v
[Confirmação via WhatsApp]
     |
     v
[Google Calendar API]
```

---

### 4.4 Fluxo de Autenticação com Google Calendar

O bot utiliza o fluxo OAuth2 do Google para obter permissão de acesso ao calendário de cada usuário individualmente. Após a autorização, os tokens são salvos no Supabase (self-hosted), associados ao número de telefone do usuário do WhatsApp.

#### Passos do fluxo:

1. O bot recebe uma mensagem de áudio
2. Verifica se o número do WhatsApp já possui token salvo no Supabase
3. Se não houver token, envia um link de autorização do Google Calendar
4. O usuário clica, autoriza e o Google redireciona para o servidor
5. O servidor troca o `code` por `access_token` e `refresh_token`
6. Os tokens são salvos no Supabase, vinculados ao número de WhatsApp
7. Com a permissão válida, o bot pode criar eventos no Google Calendar do usuário

---

### 4.5 Supabase (self-hosted)

#### Uso previsto:

- Armazenamento de tokens de autenticação por usuário
- Relacionamento com o número de WhatsApp
- Permite reuso dos tokens para chamadas futuras

#### Tabela `google_tokens`

| Campo            | Tipo       | Observações                           |
|------------------|------------|---------------------------------------|
| `whatsapp_number`| TEXT       | Chave do usuário (número WhatsApp)    |
| `access_token`   | TEXT       | Token temporário de acesso            |
| `refresh_token`  | TEXT       | Usado para renovar o access_token     |
| `expires_at`     | TIMESTAMP  | Quando o access_token expira          |
| `google_email`   | TEXT       | (Opcional) Email da conta Google      |
| `created_at`     | TIMESTAMP  | Data de criação do registro           |

> **Segurança**: tokens devem ser criptografados no backend antes de salvar.

---

### 4.6 Diagrama Atualizado

```text
WhatsApp User
     |
     v
  [Baileys Bot]
     |
     v
  [NestJS Server]
     |      \
     |       -----> [Verifica tokens no Supabase]
     |                    |
     |                [Tem token?]
     |                 /      \
     |          [Sim]         [Não]
     |             |              \
     |       Cria evento        Envia link OAuth
     |                            |
     |                    Usuário autoriza
     |                            |
     |                 /auth/callback recebe código
     |                            |
     |                Troca por token, salva no Supabase
     |                            |
     |                    Fluxo normal continua
     |
     v
[OpenAI APIs] → Transcrição + GPT
     |
     v
[Google Calendar API]
```

---

## 5. Infraestrutura

- **Hospedagem**: VPS (1 vCPU, 2GB RAM)
- **Deploy**: Docker + GitHub Actions
- **Supabase**: instância self-hosted, com banco Postgres local
- **Monitoramento**: Uptime Kuma (para health, HTTP, WebSocket)

---

## 6. Segurança

- Os áudios são mantidos apenas localmente e deletados após o processamento
- Tokens sensíveis são criptografados antes de serem armazenados
- Comunicação com APIs via HTTPS
- Sessões do WhatsApp armazenadas de forma segura

---

## 7. Custos Estimados

| Item                  | Custo por Usuário | Custo Total (1.000 usuários) |
|-----------------------|-------------------|------------------------------|
| OpenAI Whisper        | $0.54             | $540                         |
| OpenAI GPT-3.5        | $0.06             | $60.75                       |
| Google Calendar API   | $0.00             | $0                           |
| VPS + Infra           | —                 | ~$10                         |
| Supabase (self-hosted)| —                 | ~$0 (já incluso na VPS)      |
| Monitoramento         | —                 | ~$5                          |
| **Total**             | **~$0.60**        | **~$615.75/mês**             |

---

## 8. Roadmap Inicial

| Fase                                 | Data Estimada     |
|--------------------------------------|-------------------|
| MVP (transcrição + resposta simples) | Abril 2025        |
| Autenticação Google por usuário      | Abril/Maio 2025   |
| Confirmação de tarefas via WhatsApp  | Maio 2025         |
| Integração Supabase                  | Maio 2025         |
| Integração com Google Calendar       | Maio/Junho 2025   |
| Painel de tokens/logins (opcional)   | Junho 2025        |
| Lançamento público (beta)            | Julho 2025        |

---

## 9. Riscos e Mitigações

| Risco                                  | Mitigação                                                   |
|----------------------------------------|-------------------------------------------------------------|
| Baileys é uma lib não-oficial          | Monitoramento ativo e fallback para Twilio se necessário    |
| Custo variável com OpenAI              | Rate limiting, caching parcial                              |
| Interpretação incorreta de tarefas     | Confirmação manual no WhatsApp antes do envio ao calendário |
| Queda da sessão WhatsApp               | Reautenticação automática e backup da sessão                |
| Token OAuth expirado                   | Renovação automática com `refresh_token`                    |

---

## 10. Conclusão

Este projeto combina tecnologias acessíveis e de baixo custo para criar um assistente virtual de tarefas via WhatsApp. Com uma abordagem pragmática (monolito, REST, Docker), ele foca em entregar valor rapidamente com uma boa experiência de usuário.

---
