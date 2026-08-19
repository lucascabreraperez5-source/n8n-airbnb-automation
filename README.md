# Vacation Rental Automation (Airbnb / Booking)
### n8n · Airtable · Evolution API (WhatsApp) · JavaScript

I built this to stop running the same rental admin by hand every day. With it
running, the host only has to start a booking. Everything after that (guest
messages, cleaning alerts, error monitoring) runs on its own. In my own use it
took the daily admin from around **4 hours down to about 1**.

It's a low-cost setup that replaces paid channel managers and pricing tools for
short-term rental hosts.

<img width="739" height="1600" alt="WhatsApp Image 2026-08-19 at 19 10 49" src="https://github.com/user-attachments/assets/40f3d2f9-7c40-4f0d-9e25-05c5b61e72b9" />

*A check-in message the system sends on its own, 5 days before arrival.*

🇧🇷 **Versão em português mais abaixo** ⬇️ · [Ir para o português](#-automação-de-aluguéis-de-temporada-airbnb--booking)

---

## The problem

Airbnb / Booking hosts lose hours every day answering guests, coordinating
cleaning and tracking bookings in a spreadsheet. This system does the repetitive
part for them and keeps every booking in one place (Airtable), so there's a
single source of truth instead of scattered notes.

## Architecture

I split the system into separate workflows, one per trigger type. That wasn't
accidental: a daily schedule, an incoming webhook, an error handler and an email
reader are four different jobs, and keeping them apart makes each one easy to
test and fix without touching the others.

<img width="954" height="415" alt="architecture" src="https://github.com/user-attachments/assets/286d58cb-10be-4c05-97e3-fc35188c5a40" />


| Workflow file | Module | Trigger | Status |
|---|---|---|---|
| [`workflows/01-regua-comunicacao.json`](workflows/01-regua-comunicacao.json) | Guest messaging cadence | Schedule | ✅ Done & tested |
| [`workflows/02-recebe-whatsapp.json`](workflows/02-recebe-whatsapp.json) | Inbound WhatsApp handling | Webhook | ✅ Done & tested |
| [`workflows/03-alerta-erro.json`](workflows/03-alerta-erro.json) | Global error alerting | Error | ✅ Done & tested |
| [`workflows/04-parser-email-WIP.json`](workflows/04-parser-email-WIP.json) | Email → booking parser | IMAP | 🚧 In development |

## Tech stack

- **n8n** for orchestration (schedule, webhook, error and email triggers)
- **Airtable** as the database, with linked tables, lookups and formulas
- **Evolution API** to send WhatsApp over REST, called from HTTP Request nodes
- **JavaScript + Regex** in Code nodes to parse data and build the messages

## What each module does

### Module 1: Guest messaging cadence ✅
Runs once a day, checks every booking against today's date and sends the right
message at the right time: 5 days before check-in (location), 1 day before
(access code, Wi-Fi), on the day (welcome), 1 day before check-out (heads-up to
the cleaning crew) and on check-out day (thanks + review request).

A few things I had to get right:
- Date math is done in UTC, otherwise a timezone offset sends the wrong message on the wrong day
- Each message ticks a checkbox in Airtable once sent, and the search skips anything already ticked, so nobody gets the same message twice
- Cancelled bookings are skipped automatically
- The message text is built from two linked tables (booking + property) via lookup

### Module 2: Inbound WhatsApp handling ✅
A webhook fires whenever a guest sends a message. It pulls the booking code and
phone number out of the text with Regex, finds the booking in Airtable and marks
the contact as confirmed.

- Getting the webhook to reach n8n from Docker needed `host.docker.internal` (more on that below)
- Handles both code formats: Airbnb's `HM########` and Booking's numeric IDs
- If a message has no valid code, it gets an automatic "please resend" reply instead of crashing the flow
- WhatsApp now masks the real number of unknown contacts, so I read the number from the message body itself (opt-in)

<img width="1158" height="380" alt="module-2-whatsapp" src="https://github.com/user-attachments/assets/ea656e5f-41a9-4f62-8b08-8575414ef65f" />

### Module 3: Global error alerting ✅
If any workflow breaks, I get a WhatsApp within seconds with the workflow name,
the node that failed, the error message and a link to the exact run. No more
finding out three days later that messages quietly stopped going out.

<img width="666" height="284" alt="module-3-error" src="https://github.com/user-attachments/assets/7d6f1763-6330-43b9-868c-5fa5ae94b32f" />

### Module 4: Email → booking parser 🚧
Reads the inbox over IMAP and creates bookings in Airtable straight from the
platform confirmation emails, telling Airbnb and Booking formats apart. Still in
progress: I need more real email samples to make the Regex reliable.

## Real problems I ran into (and fixed)

This is the part I'm most proud of. None of it is in a tutorial.

1. **Evolution API 1.8.2 kept rejecting my sends.** I only figured it out by
   reading the API's own error response: this version wants a `textMessage`
   field, not `text`. One word, hours of my life.
2. **The webhook never arrived.** Evolution's `localhost` inside its container
   isn't n8n's `localhost`. Swapping it for `host.docker.internal` fixed it.
3. **The guest's real number was masked** (WhatsApp's `@lid`). I worked around
   it by capturing the number from the message text the guest sends, not from
   the sender field.
4. **Long messages broke the JSON body** because real line breaks aren't valid
   inside JSON. I normalize them to `\n` and format the dates in a Code node
   first.
5. **Airtable wasn't showing new fields** because the node had cached the old
   column list. Forcing a table refresh brought them back.

## Data model (Airtable)

- **`Properties`** holds the fixed stuff per apartment: name, iCal links,
  address, map link, access instructions, check-in/out times, weekday and
  weekend rates, cleaning fee, platform commissions.
- **`Bookings`** holds the moving parts: booking ID, guest, phone, dates,
  status, the message-control checkboxes and formula fields for the money
  (platform fee, net profit). It pulls property data in via lookup.

Keeping the fixed property data separate from the per-booking data means I never
repeat myself and each apartment has one source of truth.

## Running it yourself

> ⚠️ Every credential, phone number and Airtable ID in these files was replaced
> with a placeholder (`SUA_EVOLUTION_API_KEY`, `tblRESERVAS_AQUI`, etc). Nothing
> here is a live secret.

1. In n8n, use **Import from File** and pick a workflow from `workflows/`.
2. Reconnect the credentials (Evolution key, Airtable token, IMAP) in each node.
   The placeholders replaced the original references.
3. Drop in your own Airtable base/table IDs and your WhatsApp number.

## Status & roadmap

Three modules are done and tested. Two are on the way: the email parser above,
and a rule-based pricing engine (Module 5) that reads the real iCal calendar and
suggests weekend / holiday / last-minute prices over WhatsApp. Once the whole
pipeline is connected I'll add a short end-to-end demo video here.

The three finished modules already cover what most automation work needs:
orchestration, REST APIs, webhooks, a relational database, Regex parsing, error
handling and idempotency.

## Hire me

Freelance n8n automation, focused on WhatsApp integrations, REST APIs and
Airtable.
Lucas Cabrera Perez
lucas.cabreraperez5@gmail.com
tel: 11 97294-4496

---
---

# 🇧🇷 Automação de Aluguéis de Temporada (Airbnb / Booking)
### n8n · Airtable · Evolution API (WhatsApp) · JavaScript

Fiz esse sistema pra parar de fazer a mesma rotina de aluguel na mão todo dia.
Com ele rodando, o anfitrião só precisa iniciar a reserva. Todo o resto
(mensagens pro hóspede, aviso de faxina, monitoramento de erro) acontece
sozinho. No meu uso, isso levou o trabalho diário de cerca de **4 horas pra mais
ou menos 1**.

É uma solução de baixo custo que substitui channel managers e ferramentas de
precificação pagas.

<img width="739" height="1600" alt="WhatsApp Image 2026-08-19 at 19 10 49" src="https://github.com/user-attachments/assets/b34d0dbf-2aaa-428e-8104-cf894216992d" />

*Uma mensagem de check-in que o sistema envia sozinho, 5 dias antes da chegada.*

## O problema

Quem administra Airbnb / Booking perde horas todo dia respondendo hóspede,
combinando faxina e controlando reserva na planilha. Esse sistema faz a parte
repetitiva sozinho e mantém todas as reservas num lugar só (Airtable), em vez de
anotações espalhadas.

## Arquitetura

Separei o sistema em workflows independentes, um por tipo de gatilho. Não foi à
toa: um agendamento diário, um webhook que chega, um tratador de erro e um
leitor de e-mail são quatro trabalhos diferentes, e mantê-los separados deixa
cada um fácil de testar e consertar sem mexer nos outros.

<img width="954" height="415" alt="architecture" src="https://github.com/user-attachments/assets/37a8fe26-41b1-43d2-bf7d-6d18feb20947" />

| Arquivo do workflow | Módulo | Gatilho | Status |
|---|---|---|---|
| [`workflows/01-regua-comunicacao.json`](workflows/01-regua-comunicacao.json) | Régua de comunicação | Schedule | ✅ Pronto e testado |
| [`workflows/02-recebe-whatsapp.json`](workflows/02-recebe-whatsapp.json) | Recebe WhatsApp | Webhook | ✅ Pronto e testado |
| [`workflows/03-alerta-erro.json`](workflows/03-alerta-erro.json) | Alerta de erro global | Error | ✅ Pronto e testado |
| [`workflows/04-parser-email-WIP.json`](workflows/04-parser-email-WIP.json) | Parser de e-mail | IMAP | 🚧 Em desenvolvimento |

## Stack

- **n8n** pra orquestração (gatilhos de tempo, webhook, erro e e-mail)
- **Airtable** como banco, com tabelas ligadas, lookups e fórmulas
- **Evolution API** pra enviar WhatsApp via REST, chamada dos nós HTTP Request
- **JavaScript + Regex** em nós Code pra fazer o parsing e montar as mensagens

## O que cada módulo faz

### Módulo 1: Régua de comunicação ✅
Roda uma vez por dia, compara cada reserva com a data de hoje e manda a mensagem
certa na hora certa: 5 dias antes do check-in (localização), 1 dia antes (código
de acesso, Wi-Fi), no dia (boas-vindas), 1 dia antes do check-out (aviso pra
faxina) e no dia do check-out (agradecimento + pedido de avaliação).

Umas coisas que precisei acertar:
- A conta de datas é feita em UTC, senão o fuso manda a mensagem errada no dia errado
- Cada mensagem marca um checkbox no Airtable ao ser enviada, e a busca ignora o que já está marcado, então ninguém recebe a mesma mensagem duas vezes
- Reservas canceladas são puladas automaticamente
- O texto da mensagem é montado de duas tabelas ligadas (reserva + imóvel) via lookup

### Módulo 2: Recebe WhatsApp ✅
Um webhook dispara toda vez que o hóspede manda mensagem. Ele tira o código da
reserva e o telefone do texto com Regex, acha a reserva no Airtable e marca o
contato como confirmado.

- Fazer o webhook chegar no n8n a partir do Docker exigiu o `host.docker.internal`
- Trata os dois formatos de código: o `HM########` do Airbnb e o numérico do Booking
- Se a mensagem não tem código válido, recebe uma resposta automática pedindo reenvio, sem quebrar o fluxo
- O WhatsApp passou a esconder o número real de contatos desconhecidos, então leio o número do próprio texto que o hóspede manda (opt-in)

<img width="1158" height="380" alt="module-2-whatsapp" src="https://github.com/user-attachments/assets/1d66fcc6-a476-4337-abbb-377180f9b9c1" />

### Módulo 3: Alerta de erro global ✅
Se qualquer workflow quebra, recebo um WhatsApp em segundos com o nome do
workflow, o nó que falhou, a mensagem de erro e o link pra execução exata. Chega
de descobrir três dias depois que as mensagens pararam de sair caladas.

<img width="666" height="284" alt="module-3-error" src="https://github.com/user-attachments/assets/b8585cbc-c35d-4bba-a32a-30b0ed425949" />

### Módulo 4: Parser de e-mail 🚧
Lê a caixa de entrada por IMAP e cria as reservas no Airtable direto dos e-mails
de confirmação das plataformas, distinguindo o formato do Airbnb e do Booking.
Ainda em andamento: preciso de mais e-mails reais pra deixar os Regex confiáveis.

## Problemas reais que apareceram (e resolvi)

Essa é a parte que mais me orgulho. Nada disso está em tutorial.

1. **A Evolution API 1.8.2 recusava meus envios.** Só descobri lendo a resposta
   de erro da própria API: essa versão quer o campo `textMessage`, não `text`.
   Uma palavra, várias horas da minha vida.
2. **O webhook nunca chegava.** O `localhost` da Evolution dentro do container
   não é o `localhost` do n8n. Trocar por `host.docker.internal` resolveu.
3. **O número real do hóspede vinha mascarado** (o `@lid` do WhatsApp).
   Contornei capturando o número do texto que o próprio hóspede manda, em vez do
   campo de remetente.
4. **Mensagens longas quebravam o JSON**, porque quebra de linha real não é
   válida dentro de JSON. Normalizo pra `\n` e formato as datas num nó Code antes.
5. **O Airtable não mostrava campos novos** porque o nó tinha guardado a lista
   antiga de colunas. Forçar o refresh da tabela trouxe eles de volta.

## Modelagem de dados (Airtable)

- **`Imóveis`** guarda o que é fixo de cada apartamento: nome, links de iCal,
  endereço, link de mapa, instruções de acesso, horários de check-in/out, diária
  de semana, diária de fim de semana, taxa de limpeza e comissões.
- **`Reservas`** guarda o que muda: ID da reserva, hóspede, telefone, datas,
  status, os checkboxes de controle de mensagem e os campos de fórmula do
  dinheiro (taxa da plataforma, lucro líquido). Puxa os dados do imóvel via lookup.

Separar o dado fixo do imóvel do dado de cada reserva evita repetição e dá uma
fonte de verdade única pra cada apartamento.

## Como rodar

> ⚠️ Toda credencial, número e ID do Airtable nesses arquivos foi trocado por um
> placeholder (`SUA_EVOLUTION_API_KEY`, `tblRESERVAS_AQUI`, etc). Nada aqui é
> segredo real.

1. No n8n, use **Import from File** e escolha um workflow em `workflows/`.
2. Reconecte as credenciais (chave da Evolution, token do Airtable, IMAP) em
   cada nó. Os placeholders substituíram as referências originais.
3. Coloque seus próprios IDs de base/tabela do Airtable e seu número.

## Status e roadmap

Três módulos estão prontos e testados. Dois estão a caminho: o parser de e-mail
acima e um motor de precificação por regras (Módulo 5) que lê o iCal real e
sugere preço de fim de semana / feriado / última hora pelo WhatsApp. Quando o
pipeline inteiro estiver ligado, coloco aqui um vídeo curto de ponta a ponta.

Os três módulos prontos já cobrem o que a maioria dos trabalhos de automação
pede: orquestração, APIs REST, webhooks, banco relacional, parsing com Regex,
tratamento de erro e idempotência.

## Contato

Automação freelance com n8n, focado em integrações WhatsApp, APIs REST e Airtable.
Lucas Cabrera Perez
lucas.cabreraperez5@gmail.com
tel: 11 97294-4496

