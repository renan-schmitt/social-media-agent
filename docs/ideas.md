# Social Media Agent — Ideias

> Documento vivo com as ideias discutidas para automatizar a divulgação dos apps (Android/iOS) nas redes sociais.
> Última atualização: 2026-11-03

## Objetivo

Ter um programa/script/agente que mantenha a divulgação dos apps **sempre ativa**: o conteúdo é criado no Canva e publicado automaticamente, numa frequência definida, nas redes sociais de cada app.

## Fluxo inicial (visão do Renan)

1. **Criação**: conteúdo gerado no Canva
2. **Organização**: saber o que está pronto para publicar (e o que já foi publicado)
3. **Publicação**: postar nas redes sociais de cada app
4. **Agendamento**: publicar com uma frequência definida

## Requisitos (respostas de 2026-11-03)
- **Apps:** entre 2 e 10.
- **Conteúdo:** só imagens.
- **Custo:** zero, nenhuma ferramenta paga.
- **Papel do agente:** só publica o que está pronto, sem criar nem sugerir conteúdo.
- **Aprovação:** nenhuma, publica direto.
- **Redes (v1):** Instagram e Facebook.
- **Frequência:** cada app tem uma frequência de posts. Quando o conteúdo novo acaba, **repete** o que já foi publicado, para a conta nunca ficar parada.
- **Puxar do Canva:** **manual**. O Renan roda um comando no Claude Code quando quiser. Não vai usar o Claude Desktop.
- **Publicação:** o GitHub Actions pode rodar com pouca frequência (algumas vezes por dia, ou uma).
- **Armazenamento:** Firebase (já tem conta).

---

## Arquitetura proposta (v2)

```
                    (manual, sob demanda)                         (nuvem, sempre)
┌──────────────┐  Claude Code:         ┌───────────────────────────┐  GitHub Actions  ┌─────────────────┐
│ Canva        │  /enfileirar          │ Firebase                  │  cron 1–3x/dia   │ Meta Graph API  │
│ <App>/Pronto │ ────────────────────► │  Storage: imagens (URL    │ ───────────────► │  ├─ FB Page     │
│      │       │  exporta JPEG, sobe,  │           pública p/ IG)  │  publisher       │  └─ IG Business │
│      ▼       │  cria doc no          │  Firestore: biblioteca,   │  escolhe o post  └─────────────────┘
│ <App>/       │  Firestore, move o    │             histórico     │  e registra a
│  Biblioteca  │  design de pasta      └───────────────────────────┘  publicação
└──────────────┘
```

Separação de responsabilidades:
- **Ingestão** (manual): Canva → Firebase. Pode ficar dias ou semanas sem rodar sem problema, porque a biblioteca tem conteúdo para repetir.
- **Publicação** (automática): GitHub Actions → Meta. Roda sempre, mesmo com o computador desligado.

### 1. Ingestão: comando manual no Claude Code
- Um comando do projeto (skill/slash command `/enfileirar`) que você roda no Claude Code quando tiver designs novos prontos. Ele usa o conector Canva que já está ligado.
- O que o comando faz:
  1. Para cada app em `apps.yaml`, lista os designs da pasta `<App>/Pronto` no Canva.
  2. Exporta cada um em JPEG.
  3. Sobe a imagem no Firebase Storage em `posts/<app>/<designId>.jpg`.
  4. Cria um documento em `posts` no Firestore.
  5. Move o design para `<App>/Biblioteca` no Canva. É isso que evita importar o mesmo design duas vezes.
  6. Mostra um resumo: quantos posts novos por app e o tamanho da biblioteca.
- Se um dia quiser automatizar: `claude -p "/enfileirar"` via cron/launchd, ou uma rotina do Claude na nuvem. Não precisa mudar mais nada.

### 2. Biblioteca + frequência (conteúdo "evergreen")
O modelo deixa de ser uma "fila com data marcada" e passa a ser uma **biblioteca por app + uma grade de horários**.

**Config por app** (`apps.yaml` no repo, versionado):
```yaml
apps:
  app-x:
    nome: App X
    facebook_page_id: "123..."
    instagram_user_id: "178..."
    timezone: America/Sao_Paulo
    dias: [seg, qua, sex]      # dias em que publica (1 post por dia)
    repetir_apos_dias: 30      # não repete o mesmo post antes disso
```

**Post na biblioteca** (Firestore `posts/{id}`):
```yaml
app: app-x
canva_design_id: DAxxxx
image_path: posts/app-x/DAxxxx.jpg
image_url: https://firebasestorage.googleapis.com/...
legendas: ["...", "..."]       # 1 ou mais; alterna na repetição
ativo: true                    # desligar sem apagar
criado_em: ...
vezes_publicado: 0
ultima_publicacao: null
```

**Slot** = (app, dia). O horário é definido pelo cron do workflow, que é global. Se precisar de horários diferentes por app, dá para evoluir depois.

**Algoritmo de escolha** (a cada slot vencido de um app):
1. Posts **nunca publicados**, do mais antigo para o mais novo. Conteúdo novo tem prioridade.
2. Se não houver nenhum, posts publicados há mais de `repetir_apos_dias` dias, começando pelo que está **há mais tempo sem sair** (rodízio).
3. Se nada for elegível (biblioteca pequena demais), relaxa a regra e pega o mais antigo mesmo. Nunca pula o slot.
4. Evita repetir o mesmo post duas vezes seguidas.

**Histórico** (Firestore `publicacoes/{app}_{slot}`):
- O ID do documento é o próprio slot (ex.: `app-x_2026-11-10`). Isso garante **idempotência**: se o cron rodar duas vezes, ou atrasar, o slot é publicado uma única vez (create-if-not-exists).
- Status por rede (`facebook: publicado | erro`, `instagram: ...`), com os IDs dos posts e os erros. O retry refaz só a rede que falhou.
- Se o publisher ficar fora do ar por muito tempo, publica só o **último** slot perdido e descarta os anteriores. Assim não sai uma rajada de posts.

**Sobre repetir conteúdo:** a Meta não proíbe repostar conteúdo próprio, mas posts idênticos tendem a ter menos alcance. As mitigações são o intervalo mínimo (`repetir_apos_dias`) e as legendas alternativas, que você escreve. O agente não gera texto.

### 3. Publicação: GitHub Actions + Meta Graph API
- Workflow com `cron` **1x por dia** no horário de postagem (ex.: `0 21 * * *` UTC = 18h em Brasília). Opcionalmente, uma 2ª execução horas depois como **retry**: ela só refaz slots do dia que falharam ou não rodaram, e a idempotência garante que nada duplica. Também dá para disparar à mão (`workflow_dispatch`).
  - Lê `apps.yaml`, verifica se hoje é dia de publicar para cada app, escolhe o post, publica e grava em `publicacoes`.
  - **Facebook:** `POST /{page-id}/photos` com `url` (a URL do Firebase) e `caption`.
  - **Instagram:** `POST /{ig-user-id}/media` com `image_url` e `caption`, depois `POST /{ig-user-id}/media_publish`. Exige JPEG em URL pública, e a URL do Firebase Storage resolve isso.
- O cron do GitHub pode atrasar de minutos até cerca de 1h. Para este caso, é aceitável.
- Como o estado fica no Firestore, o workflow **não precisa fazer commit** no repo. Isso também evita que o GitHub desative o workflow agendado: em repos públicos, ele faz isso depois de 60 dias sem atividade, e vale monitorar.
- Secrets no GitHub: o token da Meta (System User) e a service account do Firebase.

### 4. Firebase
- **Firestore:** biblioteca e histórico. O volume é minúsculo, bem dentro da cota gratuita.
- **Cloud Storage:** as imagens. ⚠️ **Desde 03/02/2026, o Storage exige o plano Blaze** (pay-as-you-go, com cartão cadastrado). O Blaze mantém uma cota gratuita, e com algumas centenas de imagens o custo esperado é **US$ 0**. Recomendações:
  - Criar o bucket numa região com cota gratuita (ex.: `us-central1`).
  - Configurar um **alerta de orçamento** (ex.: US$ 1). O alerta avisa, mas não bloqueia o gasto.
  - Se não quiser cadastrar cartão: guardar as imagens num **repo público no GitHub** (URLs raw servem para o IG) e manter só o Firestore (plano Spark).
- A URL de download do Firebase (com token) é pública e funciona como `image_url` no Instagram.

### 5. Meta: setup único
- As contas do Instagram precisam ser **Business/Creator** e estar ligadas à Página do Facebook de cada app.
- Criar **um app Meta** e **um Business Manager** com todas as Páginas.
- Usar um **System User** do Business Manager. O token dele não expira e cobre todas as páginas.
- Como só vamos publicar nas próprias contas, o app deve funcionar em **modo de desenvolvimento**, sem App Review. *(confirmar na prática)*
- Permissões: `pages_manage_posts`, `pages_read_engagement`, `instagram_basic` e `instagram_content_publish`.
- O Instagram aceita proporções de 4:5 até 1.91:1. A ingestão deve validar isso antes de enviar.

---

## Alternativas avaliadas

### Canva Content Planner: descartado
- Exige plano pago (Canva Pro/Teams).
- Limite de cerca de **8 contas sociais conectadas** no Pro.
- Não tem API e não suporta repetição automática.

### Outras ferramentas
- Buffer, Later, Metricool e Ayrshare: pagas ou limitadas no plano gratuito.
- Postiz self-hosted: gratuito, mas exige um servidor sempre ligado.

### Formas de puxar do Canva
| Opção | Automático? | Observações |
|---|---|---|
| **Comando manual no Claude Code** | Não | **Escolhida.** Zero setup de autenticação no Canva. |
| Tarefa agendada do Claude Desktop | Sim, com o Mac ligado | Descartada: o Renan não usa o Desktop. |
| `claude -p` via launchd | Sim, com o Mac ligado | Evolução possível. |
| Rotina do Claude na nuvem | Sim, sempre | Confirmar o acesso ao conector Canva. |
| Script + Canva Connect API | Sim, sempre | Sem LLM, mas com OAuth e refresh token rotativo, chato de manter. |
| Manual (exportar e subir) | Não | Fallback e testes. |

---

## Perguntas em aberto
- **Legenda:** onde você escreve? Opções: (a) um campo de texto dentro do design no Canva, que a ingestão lê; (b) o título do design; (c) editar o documento no Firestore depois. Como suportar várias legendas por post?
- **Firebase Blaze:** ok cadastrar o cartão (custo esperado US$ 0), ou prefere imagens no GitHub público?
- **Repo:** público ou privado?
- **Linguagem:** Python (proposta) ou TypeScript?
- **Frequência inicial:** quais dias por app, e qual horário global do cron?

## Decisões
- 2026-11-03: Canva Content Planner descartado (pago e limitado a cerca de 8 contas).
- 2026-11-03: o agente só publica, sem gerar conteúdo e sem aprovação manual.
- 2026-11-03: v1 cobre só Instagram e Facebook, via Meta Graph API.
- 2026-11-03: modelo de **biblioteca + frequência** com repetição, em vez de fila com data marcada.
- 2026-11-03: ingestão do Canva **manual**, via comando `/enfileirar` no Claude Code (sem Claude Desktop).
- 2026-11-03: publicação via **GitHub Actions**, com cron **1x por dia** (mais um retry opcional) e no máximo 1 post por app por dia.
- 2026-11-03: estado (biblioteca e histórico) no **Firestore**. Imagens no **Firebase Storage** (pendente: ok com o Blaze).
