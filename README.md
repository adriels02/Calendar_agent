# Calendario WhatsApp

Assistente de agenda no WhatsApp. A pessoa conversa com o número conectado na Evolution API, o n8n entende o pedido e usa o Google Calendar. A cada 5 minutos, o mesmo fluxo avisa os clientes ativos sobre compromissos que começam em um intervalo de 30 minutos.

O workflow importável está em `workflows/Calendario WhatsApp.json`.

## Como sobe

`docker compose up -d` sobe quatro serviços:

| Serviço | Para que serve | Acesso |
|---|---|---|
| n8n | Workflow | `http://localhost:5678` |
| Evolution API | WhatsApp | `http://localhost:8080` |
| Postgres | Banco da Evolution | requisito para Evolution API |
| Redis | Cache da Evolution | requisito para Evolution API |

O fuso dos containers é `America/Sao_Paulo`. O n8n fica no volume `n8n_data`.

Copie `.env.example` para `.env` e preencha `EVOLUTION_API_KEY` e `POSTGRES_PASSWORD`. A primeira é a chave global da Evolution. A segunda é a senha do Postgres dela.

Na Evolution, o evento `MESSAGES_UPSERT` precisa chegar ao webhook do n8n. Com o workflow ativo, o caminho é `http://n8n:5678/webhook/whatsapp`.

## Conversa

A mensagem entra pelo webhook. O fluxo guarda o número, o nome e o texto e só chama o agente se aquele número for de um cliente ativo.

O agente lista, cria, altera e apaga eventos na agenda cujo id é o `email` desse usuário. As datas seguem o horário de Brasília. Antes de marcar, ele olha se o horário está livre. Se a pessoa não disser o fim, o compromisso dura uma hora. A resposta volta no mesmo chat.

Quem não é cliente ativo recebe um convite para cadastro. Falha do agente e o histórico da conversa estão em [Logs e erros](#logs-e-erros). A regra de quem pode usar o sistema está em [Autenticação e autorização](#autenticação-e-autorização).

## Lembretes

A cada 5 minutos o fluxo lê os usuários ativos. Para cada um, busca na agenda dele os eventos dos próximos 30 minutos, em `America/Recife`, incluindo ocorrências de eventos recorrentes.

Ignora dia inteiro e evento cancelado. Envia o lembrete quando faltam 30 minutos para o início e o compromisso ainda não começou. A instância da Evolution nesse envio é `Agent_Calendar`.

Antes de enviar, consulta `reminder_sent` pelo id do evento e pelo horário daquela ocorrência. Se o aviso já saiu, passa para o próximo. Se não, um segundo agente monta categoria, briefing e texto. O WhatsApp recebe essa mensagem e o envio é registrado, com o id da mensagem quando a Evolution devolve.

Se a geração do briefing falhar, o compromisso ainda é avisado por uma mensagem padrão. O detalhe está em [Logs e erros](#logs-e-erros).

## Logs e erros

Quando um agente é acionado, o n8n grava o histórico na tabela padrão `n8n_chat_histories`, no Supabase. Cada conversa fica separada pelo número de quem falou.

Erros do sistema vão para `agent_errors` e o fluxo segue a partir desse registro.

No agente de calendário, a falha é gravada com a instância, o contato, o nome e a mensagem que chegou. O WhatsApp responde que a equipe vai analisar e pede para a pessoa tentar de novo, ou fazer outra operação.

Na geração do briefing do lembrete, a falha também é gravada. O texto registrado dessa mensagem é o aviso automático. Em seguida o usuário ainda recebe o lembrete padrão do compromisso, com título, descrição, data e hora. Esse envio entra em `reminder_sent`, para não repetir no próximo ciclo.

## Autenticação e autorização

Para usar o sistema, a pessoa precisa ser cliente. No início da conversa o fluxo procura o número em `users`. Só segue para o agente quando `is_active` é verdadeiro. Esse é o controle de quem está autorizado.

Quem não está cadastrado, ou não está ativo, não chega na agenda. Recebe a mensagem de que ainda não é cliente e o caminho para se cadastrar.

Os lembretes usam a mesma regra: a cada 5 minutos só entram usuários com `is_active` verdadeiro. A agenda consultada é o `email` dessa linha, e o WhatsApp do aviso é o `number`.

## Credenciais

O arquivo `.env.example` lista o que precisa ser preenchido. `EVOLUTION_API_KEY` e `POSTGRES_PASSWORD` são lidas pelo Docker Compose. As demais são coladas nas credenciais do n8n.

| Credencial | Uso | Onde gerar | Variáveis |
|---|---|---|---|
| Google Calendar | Acessar a API do Google Calendar nos quatro nós da agenda e na busca dos lembretes | Google Cloud | `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET` |
| Evolution | Enviar as respostas e os lembretes no WhatsApp | Chave global da Evolution | `EVOLUTION_API_KEY` |
| Postgres da Evolution | Banco e cache de sessão da própria Evolution | Definida no projeto | `POSTGRES_PASSWORD` |
| Supabase | Clientes, memória da conversa, erros do agente e controle de lembretes já enviados | Supabase, na conexão pooling | `SUPABASE_HOST`, `SUPABASE_PORT`, `SUPABASE_DATABASE`, `SUPABASE_USER`, `SUPABASE_PASSWORD` |
| xAI | Modelo do agente que conversa sobre a agenda (`grok-4.3`) | Console da xAI | `XAI_API_KEY` |

A credencial chamada `False` no agente do lembrete é um teste. Ela faz esse agente falhar de propósito, para o fluxo de erro e o lembrete padrão serem usados. Não entra no `.env`.

## Tabelas

No Supabase, schema `public`:

```sql
CREATE TABLE public.users (
    id BIGINT GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
    email TEXT,
    is_active BOOLEAN,
    number TEXT,
    name TEXT
);

CREATE TABLE public.reminder_sent (
    id BIGINT GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
    event_id TEXT,
    occurrence_start TIMESTAMPTZ,
    message_id TEXT,
    summary TEXT,
    number TEXT,
    sent_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE public.agent_errors (
    id BIGINT GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    instance TEXT,
    user_contact TEXT,
    user_name TEXT,
    user_message TEXT,
    error_message TEXT,
    error_description TEXT
);
```

`users.email` é a agenda do Google. `users.number` é o WhatsApp. `users.is_active` define quem conversa com o agente e quem recebe lembrete.

## Melhorias futuras

Integrar uma base real de clientes, ou acrescentar um fluxo de cadastro no primeiro contato. Hoje quem não está em `users` só recebe o convite para se cadastrar.

Gerar a imagem do briefing. O agente do lembrete já devolve o prompt em `visual_prompt`. O passo que falta é usar esse texto em outro nó, gerar a imagem e enviá-la no WhatsApp junto com o aviso.

Avaliar o quanto o modelo acerta. Os testes iniciais foram positivos, mas sem métrica não dá para saber a taxa de acerto. Um dos focos é produzir essas medidas:

- Classificação manual, com acurácia e F1-score.
- Avaliação por LLM: um modelo generativo atua como juiz e classifica o resultado como 0 ou 1.
- Nota do usuário, de 0 a 5, sobre o quanto está satisfeito com o sistema.
