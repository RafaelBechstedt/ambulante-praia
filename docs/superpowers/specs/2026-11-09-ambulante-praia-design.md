# Ambulante Praia: design

**Data:** 2026-11-09
**Status:** aprovado pelo Rafael em 2026-11-09

Neste documento, **(decisão do Claude)** marca uma escolha feita durante a conversa de design que o Rafael não pediu explicitamente. Ele pode trocá-la. Todo o resto foi decidido por ele.

---

## 1. Intenção e motivo

**O que é.** Um app Android para quem está na praia ver **onde estão os ambulantes agora**. Cada ambulante aparece no mapa com um título curto que ele mesmo escreve ("VENDO MILHO"). Ao tocar nele, abre um cartão com um texto livre: cardápio, preços, recado. Com isso a pessoa decide se espera, se vai atrás ou se compra em outro lugar.

O recurso "tenho interesse" é um **extra**: o cliente avisa um ambulante que há gente interessada numa área. O ambulante muitas vezes nem olha o celular, só caminha pela praia como sempre. O valor central é o mapa.

**Por que existe.** É um projeto de **estudo**: o Rafael quer praticar desenvolvimento (é o primeiro app mobile dele, vindo de React, TypeScript e NestJS) e testar o fluxo de trabalho do superpowers. O código fica num **repositório público**. Não há plano de vender o app, mas ele deve ser **construído como se pudesse virar produto**: arquitetura limpa e pontos de troca já previstos, sem infraestrutura antes da hora.

**Como é o sucesso:**
- quem clonar o repositório sobe tudo com `docker compose` e um build do app, **sem criar conta em nenhum serviço**;
- um ambulante no emulador anda por uma rota simulada e aparece se movendo, ao vivo, no mapa de um cliente;
- o código mostra decisões de produto real: peças trocáveis, limites, privacidade desde o desenho.

## 2. Perfis

- **Ambulante:** tem conta (celular + código). Compartilha a localização em segundo plano, com o celular no bolso, enquanto está vendendo.
- **Cliente:** não tem conta. Vê o mapa. Na etapa 2, é identificado só por um ID anônimo do aparelho, usado para o anti-spam.

O app é **um só**, com os dois perfis.

## 3. Metas, não-metas e anti-metas

**Metas**
- Etapa 1: cadastro e login do ambulante; "Estou vendendo" com hora de término; localização em segundo plano; mapa do cliente ao vivo com o cartão de cada ambulante; simulador de ambulantes.
- Etapa 2: "tenho interesse", com localização aproximada do cliente, expiração, agrupamento por área, anti-spam e notificação com vibração para o ambulante.
- Etapa 3, de estudo: duas instâncias da API + Redis, para viver o problema da escala horizontal e resolvê-lo.

**Não-metas (agora)**
- iOS e Expo Go.
- Login do cliente, favoritos, aviso de "ambulante por perto". Ficam para o futuro.
- SMS ou WhatsApp de verdade: o envio do código é uma peça trocável, mas só a versão de desenvolvimento é implementada.
- Fluxo de aprovação manual de ambulantes: o campo existe, mas não é usado.
- Pagamento, chat, avaliações, lista de ambulantes por distância.
- Publicar em produção ou na Play Store. Tudo roda localmente.

**Anti-metas** (o projeto falhou se acontecer qualquer uma destas, mesmo que o resto funcione)
- O servidor receber a **posição exata** do cliente, ou o ambulante conseguir identificar quem mandou um "tenho interesse".
- O ambulante continuar visível **fora** de uma sessão de venda, por exemplo mostrando o caminho até a casa dele.
- Guardar **histórico de trajeto** de alguém.
- Exigir chave de API, conta em nuvem ou cartão de crédito para rodar o projeto.
- O ambulante precisar olhar o celular durante o dia para continuar aparecendo no mapa.

## 4. Restrições

- **Stack:** React Native + Expo (um app); NestJS; PostgreSQL + PostGIS; Socket.IO, que roda sobre WebSocket.
- **Só Android.** Os testes acontecem no emulador Android no Ubuntu do Rafael. Tudo roda num **development build** do Expo (um APK do próprio app, com recarga ao vivo), nunca no Expo Go.
- **Sem chaves nem contas:** nada de Google Maps nem Firebase. O mapa usa MapLibre com os mapas gratuitos do OpenFreeMap. O backend sobe com `docker compose`.
- **Repositório público:** nenhum segredo no repositório. Configuração por variáveis de ambiente, com um `.env.example` versionado.
- O Rafael pediu que as decisões de mobile sejam **explicadas de forma didática**. O README e os comentários dos pontos não óbvios, como permissões, serviço em primeiro plano e tarefas em segundo plano, devem explicar o porquê.

## 5. Abordagens consideradas

A pergunta era: onde fica a posição ao vivo dos ambulantes, e como ela chega aos clientes.

| | Abordagem | Trade-off |
|---|---|---|
| **Escolhida** | **Só Postgres + uma instância da API, com pontos de troca.** A posição atual fica numa tabela do PostGIS com índice espacial. As atualizações ao vivo vão por Socket.IO, em salas por célula do mapa, distribuídas em memória. | Poucas peças e fácil de entender. Aguenta com folga a escala de um protótipo (mil ambulantes a cada 25 s dão cerca de 40 gravações por segundo). Ao subir a segunda instância, basta ligar um adaptador do Socket.IO (Redis ou Postgres), sem reescrever nada. |
| Descartada | **Postgres + Redis desde o início.** Posições no Redis (GEO, com expiração automática) e adaptador Redis no Socket.IO. | Arquitetura com cara de produção, mas cria duas fontes de verdade e quase não usa o PostGIS. Os ganhos reais (repassar mensagens entre instâncias, contadores compartilhados, volume muito alto de gravações) só aparecem com várias instâncias. Virou a **etapa 3**, de propósito: viver o problema antes de aplicar a solução. |
| Descartada | **Serviço pronto** (Supabase Realtime + PostGIS). | Tiraria o NestJS do meio, e ele é parte do que o Rafael quer praticar. |

## 6. Arquitetura

```
┌──────────────── apps/mobile (Expo, Android) ────────────────┐
│ Cliente: mapa (MapLibre) ◀── Socket.IO (salas por célula)   │
│ Ambulante: tarefa em 2º plano ── HTTP POST /me/position ──┐ │
└───────────────────────────────────────────────────────────┼─┘
                                                            ▼
┌──────────────── apps/api (NestJS, 1 instância) ─────────────┐
│ auth · vendors · presence · interests (etapa 2) · realtime  │
│        │                     │                              │
│   PostgreSQL + PostGIS     MinIO (fotos, API do S3)         │
└─────────────────────────────────────────────────────────────┘
packages/shared: tipos e schemas de validação usados pelos dois lados
```

### 6.1 Repositório
- **pnpm workspaces**: `apps/mobile`, `apps/api` e `packages/shared`. **(decisão do Claude)**
- `packages/shared` guarda os schemas (zod) e os tipos dos contratos HTTP e dos eventos do Socket.IO. A API valida com os mesmos schemas que o app usa. **(decisão do Claude)**
- `docker-compose.yml` sobe o Postgres com PostGIS, o MinIO e a API. A API também roda fora do Docker em modo de desenvolvimento. **(decisão do Claude)**

### 6.2 Fluxo de dados
- **O ambulante envia a posição por HTTP**, e não por WebSocket. Com o celular no bolso, o Android "acorda" a tarefa em segundo plano, ela envia e volta a dormir. Uma conexão aberta não aguenta bem esse ciclo. **(decisão do Claude)**
- **Os clientes recebem por Socket.IO.** O Socket.IO já traz reconexão automática e salas, e isso importa numa praia com internet fraca. **(decisão do Claude)**
- **Salas por célula do mapa:** células **geohash de precisão 6** (cerca de 1,2 km × 0,6 km). O cliente entra nas salas das células que cobrem a área visível do mapa, com uma pequena margem. Quando um ambulante se move, o servidor emite só para a sala da célula dele. Se ele mudar de célula, a célula antiga recebe uma remoção e a nova recebe a atualização. **(decisão do Claude)**
- **Zoom afastado demais:** se a área visível exigir mais de cerca de 12 células, o app sai das salas e mostra "Aproxime o mapa para ver os ambulantes". **(decisão do Claude)**
- **Carga inicial:** ao entrar numa sala, o cliente recebe a lista atual dos ambulantes daquela célula. Primeiro ele assina a sala, depois pede a lista, para não perder nenhuma atualização no meio do caminho. O formato exato (resposta do próprio Socket.IO ou uma rota REST) fica com quem construir.
- **Eventos:** um evento de atualização com dados públicos do ambulante e posição (`vendorId`, nome, título, URL da foto, lat, lng, `updatedAt`) e um evento de remoção (`vendorId`). Os nomes exatos ficam com quem construir.
- **Ponto de troca para escalar:** a distribuição dos eventos usa o adaptador padrão do Socket.IO, que trabalha em memória. Na etapa 3, entra um adaptador Redis sem mudar o código que emite os eventos.

### 6.3 Dados (PostgreSQL + PostGIS)
Tabelas e campos principais. Os nomes finais ficam com quem construir:
- **vendors:** id, telefone no formato E.164 (único), nome, URL ou chave da foto, título (até 30 caracteres), descrição (até 500), **status** (`pendente` | `aprovado` | `bloqueado`), datas de criação e atualização.
  - O padrão de status hoje é `aprovado`. Nas etapas 1 e 2, o status **não tem efeito** nenhum. Ele existe para uma verificação manual no futuro, e `bloqueado` permite banir alguém. **(decisão do Claude: status com três valores, em vez de um campo sim/não)**
- **selling_sessions** (ou a presença do ambulante): id do ambulante, início, **vende_até** (hora de término), posição atual (`geography(Point, 4326)` com índice GiST), `position_updated_at`, fim (quando ele para).
  - Só a posição **atual** é guardada. Nenhum histórico de trajeto (anti-meta).
  - Quando a sessão termina, a posição é **apagada** (fica nula). Uma sessão encerrada não guarda onde o ambulante estava por último. **(decisão do Claude)**
- **Contadores de limite** (pedidos de código e, na etapa 2, anti-spam) ficam no Postgres. Na etapa 3, eles passam para o Redis. **(decisão do Claude)**
- **otp_codes:** telefone, **hash** do código, data de expiração, tentativas, data de uso.
- **refresh tokens** ou equivalente, conforme o desenho de autenticação.
- Etapa 2, **interests:** id do ambulante, **hash do ID do aparelho** (nunca o ID puro, ver seção 8), centro da célula da grade (`geography(Point)`), data de criação, expira em.
- Etapa 2: os pedidos expirados são apagados depois de 24 h. Esse tempo cobre a janela de anti-spam de 1 hora, com folga. **(decisão do Claude)**

### 6.4 Ferramentas e bibliotecas (decisão do Claude, salvo onde indicado)
- Mapa: `@maplibre/maplibre-react-native` com o estilo do OpenFreeMap (decidido pelo Rafael). A URL do estilo é configurável, para trocar de fornecedor de mapa no futuro.
- Banco: **Drizzle ORM**, com migrações pelo drizzle-kit. Ele dá acesso direto ao SQL espacial, enquanto o Prisma lida mal com tipos do PostGIS.
- Navegação: Expo Router.
- Localização: `expo-location` + `expo-task-manager`.
- Notificações locais: `expo-notifications`. Só notificações **locais**, sem Firebase nem FCM.
- Tokens no celular: `expo-secure-store`.
- ID do aparelho (etapa 2): `expo-application`, `getAndroidId()`. Esse ID continua o mesmo mesmo se o app for reinstalado.
- Fotos: o celular reduz a imagem antes de enviar (`expo-image-manipulator`). O envio vai direto para o MinIO por uma **URL pré-assinada** que a API gera. Tudo atrás de uma interface de armazenamento compatível com S3, para usar AWS S3 ou R2 no futuro.
  - O bucket de fotos tem **leitura pública**, porque as fotos aparecem para qualquer cliente.
  - O endereço público do MinIO é configurável. No emulador, ele é `10.0.2.2`, o endereço que o emulador usa para chegar ao computador. Isso vale tanto para as URLs pré-assinadas quanto para as URLs das fotos.

## 7. Etapa 1: comportamento

### 7.1 Entrada no app (decisão do Claude)
- O app abre **no mapa do cliente**, sem pedir login.
- Uma entrada discreta "Sou ambulante" leva ao login.
- Depois de logado, o app abre na tela do ambulante, que também consegue ver o mapa.

### 7.2 Login do ambulante (decidido pelo Rafael, números sugeridos pelo Claude)
- Celular + código de 6 dígitos.
- O envio do código é uma **peça trocável**: uma interface `CodeSender` injetada por token no NestJS. A implementação de desenvolvimento, `ConsoleCodeSender`, escreve o código no log da API. As futuras são SMS e WhatsApp. A escolha é feita por variável de ambiente.
- O código **vale 5 minutos**. São **no máximo 3 pedidos de código por número por hora**. Depois de **5 tentativas erradas**, o código é cancelado.
- **Números de teste** (decisão do Claude): uma faixa reservada de números aceita um código fixo, para o simulador e testes manuais.
  - Uma variável de ambiente liga o recurso. No código ela vem **desligada** por padrão.
  - O `.env.example` e o `docker-compose.yml` de desenvolvimento a ligam, para que quem clonar o repositório tenha o simulador funcionando sem nenhum passo extra.
  - Uma configuração de produção nunca a liga.
- Sessão: JWT de acesso de curta duração + refresh token, guardados no `expo-secure-store`. **(decisão do Claude)**
- **Renovação em segundo plano** (pedido do Rafael): a tarefa em segundo plano renova o access token sozinha quando ele expira, sem o ambulante abrir o app.
  - O refresh token dura **mais que o tempo máximo de venda**: 30 dias, contra 14 h de sessão. **(decisão do Claude)**
  - **Uma renovação por vez:** o app aberto e a tarefa em segundo plano usam a mesma rotina, que não deixa duas renovações correrem juntas. O servidor troca o refresh token a cada uso, mas aceita o anterior por alguns segundos, para que uma corrida entre as duas não derrube a sessão. **(decisão do Claude)**
  - **Se a renovação falhar de vez**, por exemplo com o token revogado: a tarefa para de enviar posições, encerra a venda no app e mostra uma notificação local "Entre de novo para continuar visível".
- Números brasileiros por padrão (+55), normalizados para o formato E.164.

### 7.3 Perfil do ambulante
- Nome, foto, **título** de até 30 caracteres (aparece no mapa) e **descrição livre** de até 500 caracteres (aparece no cartão). Os limites são do Claude.
- O ambulante pode editar o perfil enquanto vende. A mudança chega aos clientes na hora, pelo evento de atualização.

### 7.4 "Estou vendendo"
- A tela principal tem a foto, o nome e o título, um botão grande "ESTOU VENDENDO" / "PARAR" e um status ("Visível no mapa · posição enviada há 12 s").
- **Ao ligar, ele escolhe a hora de término** ("até 17h"). O padrão é o último horário usado, então costuma ser um toque. O máximo é 14 horas a partir de agora **(decisão do Claude)**.
- **Lembrete** 15 minutos antes do fim: uma notificação local "Você sai do mapa às 17h", com a ação "+1 hora". Se ele ignorar, o app desliga na hora marcada.
- **O servidor também aplica a hora de término:** depois de `vende_até`, ele recusa posições daquela sessão e remove o ambulante do mapa. Isso cobre o caso de o Android matar o app. **(decisão do Claude)**
- **Permissões**, cada uma com uma tela de explicação antes:
  1. localização "durante o uso";
  2. localização "o tempo todo". No Android 11 em diante, essa opção não aparece na janelinha: o app leva o ambulante às configurações e explica o que marcar;
  3. notificações (Android 13 em diante).
- **Serviço em primeiro plano** com a notificação fixa "Você está visível no mapa". O Android exige isso para usar a localização em segundo plano.
- **Frequência de envio:** a cada cerca de 20 a 30 s **ou** quando ele anda uns 15 m, o que acontecer antes. O ajuste fino de precisão contra bateria fica com quem construir, com estes valores como referência.
- **Ambulante parado continua enviando.** Um ambulante parado na barraca precisa continuar aparecendo como "agora". O envio não pode depender só da distância percorrida: se o filtro de distância do Android segurar as atualizações, é preciso um envio de "ainda estou aqui" a cada cerca de 25 s. **(decisão do Claude)**
- **Parar:** pelo botão, ou automaticamente na hora de término. Os dois encerram a sessão no servidor, que remove o ambulante do mapa.

### 7.5 Mapa do cliente (opção A, escolhida pelo Rafael no mockup)
- O mapa ocupa a tela inteira. Cada ambulante aparece com uma **etiqueta com o título** e um pequeno avatar.
- Ambulantes próximos demais se **agrupam** num círculo com o número, que se separa com o zoom (agrupamento nativo do MapLibre).
- Uma faixa no topo mostra "N vendendo agora", contando os ambulantes que estão **na área visível** do mapa. Há também um botão para centralizar no cliente.
- **Tocar num ambulante abre o cartão** (uma folha que sobe de baixo): foto, nome, título, distância (se o cliente deu a localização), há quanto tempo a posição foi enviada e o texto livre.
- **Localização do cliente é opcional.** Sem ela, o mapa abre na última área vista, ou na praia padrão do simulador, e não mostra distâncias.
- **Frescor** (os números são do Claude):

  | Tempo desde a última posição | Como aparece |
  |---|---|
  | até 90 s | normal, "agora" |
  | de 90 s a 10 min | apagado, "há X min" |
  | mais de 10 min | some do mapa |

  - Um job do servidor, rodando a cada minuto, emite a remoção dos ambulantes que passaram de 10 min sem sinal ou da hora de término.
  - Quem construir decide como lidar com a diferença de relógio entre o celular e o servidor.
- Tudo se atualiza ao vivo, sem precisar puxar a tela.

### 7.6 Simulador de ambulantes (decisão do Claude)
- Um script, por exemplo `pnpm simulate`, cria alguns ambulantes falsos (usando os números de teste) e os faz andar por uma rota numa praia real (Copacabana), enviando a posição pelo **mesmo endpoint HTTP** do app.
- Ele serve para testar o lado do cliente sem um segundo emulador, e para quem clonar ver o mapa vivo na hora.
- O repositório também inclui um **arquivo GPX** com uma caminhada pela praia, para o emulador reproduzir no teste do ambulante de verdade (Extended Controls → Location → Routes).

## 8. Etapa 2: "tenho interesse"

- **No cartão do ambulante aparece o botão "Tenho interesse".** Ele só funciona se o cliente der a localização e se o ambulante estiver vendendo.
- **A posição é arredondada no celular:** antes de enviar, o app encaixa a localização do cliente numa **grade fixa de cerca de 40 m** e manda só o centro da célula. O servidor nunca recebe a posição exata.
  - Grade fixa, e não desvio aleatório: com um desvio aleatório, vários pedidos permitiriam tirar a média e descobrir o ponto exato.
  - O servidor encaixa de novo na mesma grade. Não muda nada no resultado e protege contra um cliente alterado.
- **O pedido não leva identidade.** O ambulante nunca recebe o ID do aparelho, só a célula e a hora de expiração.
- **Expira em 15 minutos.** O cliente vê "Zé foi avisado · vale até 14:32".
- **Anti-spam**, por ID do aparelho (os números são do Claude):
  - cada aparelho pode ter **um pedido ativo por ambulante**;
  - no máximo **5 pedidos por hora**.
  - O ID vem do `getAndroidId()` e é enviado num cabeçalho.
  - **O servidor guarda só um hash do ID** (pedido do Rafael). É um **HMAC-SHA256 com uma chave secreta do servidor**, configurada por variável de ambiente, e não um hash simples. Com o hash simples, quem tivesse o banco e o Android ID de alguém poderia calcular o hash e achar os pedidos dessa pessoa. O ID puro não é gravado em lugar nenhum, nem em log. **(decisão do Claude: HMAC)**
- **Distância máxima** de 3 km entre o cliente e o ambulante. Pedidos de mais longe são recusados. **(decisão do Claude)**
- **Mapa do ambulante:** os pedidos ativos aparecem como áreas agrupadas ("5 interessados aqui"), usando o mesmo agrupamento nativo do MapLibre. Elas somem à medida que os pedidos expiram. Com o app aberto, as atualizações chegam por Socket.IO.
- **Notificação com vibração, agrupada** (decidido pelo Rafael):
  - A **resposta** do `POST` de posição traz um resumo dos pedidos ativos, por exemplo a quantidade e a distância do grupo mais próximo.
  - Quando há pedidos novos desde o último aviso, a tarefa em segundo plano mostra uma notificação **local** com vibração ("3 interessados a 200 m").
  - É **no máximo um aviso a cada 3 minutos** **(decisão do Claude)**.
  - Não precisa de Firebase: o ambulante só recebe pedidos enquanto está vendendo, e nesse período o app já está rodando.

## 9. Etapa 3: escalar (estudo, futuro)

1. Subir duas instâncias da API (`docker compose up --scale api=2`) com um nginx na frente como balanceador.
2. Observar o problema: um ambulante cujo `POST` cai na instância 1 não aparece se movendo para o cliente conectado na instância 2.
3. Resolver com o **adaptador Redis do Socket.IO**, que repassa os eventos entre instâncias, e levar os **contadores de limite** (código, anti-spam) para o Redis.

As posições continuam no PostGIS. Esta etapa não é desenhada em detalhe agora. A etapa 1 só precisa não fechar essa porta: nada de estado importante guardado só na memória da API, além das salas do Socket.IO.

## 10. Testes e documentação (decisão do Claude)

- **API:** testes de integração contra um Postgres + PostGIS de verdade (Testcontainers ou um serviço de teste no compose), cobrindo:
  - autenticação e limites;
  - regras da sessão de venda (hora de término e frescor);
  - consultas espaciais;
  - eventos do Socket.IO por sala.
- **Lógica pura no app e no `packages/shared`:** testes unitários para o encaixe na grade, os cálculos de frescor, a hora de término padrão e as células geohash visíveis.
- **Fluxos nativos** (permissões, segundo plano, notificações): roteiro de teste manual no emulador, descrito no README. Ele inclui:
  - **deixar o ambulante vendendo, com o app fechado, por mais tempo que a validade do access token**, e conferir que ele continua aparecendo no mapa. Para o teste não demorar, a validade pode ser reduzida por uma variável de ambiente;
  - o lembrete 15 minutos antes da hora de término, e o desligamento automático nessa hora.
- **README** com:
  - como subir tudo do zero, sem contas;
  - como gerar e instalar o development build;
  - como usar o emulador com rota GPX;
  - como rodar o simulador;
  - explicações didáticas dos conceitos de mobile.
- Atualizar o **CLAUDE.md** com os comandos de build, lint e teste (inclusive como rodar um teste só) e com a arquitetura.

## 11. Modo de trabalho (decidido pelo Rafael)

- **Os agentes escrevem todo o código. O Rafael revisa.**
- **Tarefas pequenas, sempre com commit.** Cada tarefa vira um commit pequeno e fácil de revisar, que o Rafael lê antes de seguir para a próxima.
- **Nada de mudanças gigantes** de milhares de linhas de uma vez. A exceção é configuração ou código gerado que seja grande por natureza, como lockfile, scaffold do Expo ou migração gerada. Nesse caso, o commit traz só isso e avisa na mensagem.
- Como o Rafael aprende revisando, os commits e os comentários de código explicam o **porquê** dos pontos de mobile que não são óbvios (ver seção 4).

## 12. O que fica com quem construir

- **Um plano por etapa.** A etapa 1 é planejada e construída primeiro. A etapa 2 ganha um plano próprio depois.

- Versões das bibliotecas, de preferência a versão estável mais recente do Expo SDK.
- Estrutura interna das pastas em cada app e os módulos do NestJS.
- Nomes exatos de tabelas, rotas e eventos.
- Formato da carga inicial ao entrar numa sala, seguindo a ordem "primeiro assina, depois pede a lista".
- Ajuste fino da precisão do GPS contra bateria, partindo dos valores da seção 7.4.
- Como lidar com a diferença de relógio no cálculo de "há X min".
- Visual detalhado das telas, seguindo a estrutura do mockup da opção A e a tela do ambulante descrita na seção 7.4.
