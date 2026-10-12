# Ciclo de interface — subprojeto 2: shell dos painéis — design

**Data:** 2026-10-12
**Status:** em revisão
**Afeta:**
- `apps/dashboard` e `apps/admin`:
  - shell novo em `src/shell/` (barra eMAG, barra de topo, faixa de contexto, trilho, gaveta, Início, tela do grupo, trilha de navegação, busca de tela, menu de tema);
  - roteador por hash;
  - módulo de tema e página de acessibilidade;
  - `App.tsx` passa a usar o roteador;
  - saem o `AppHeader.tsx` e o `NavDropdown.tsx`.
- `contracts`: `design-tokens` ganha a fonte Figtree do tema `rota` (`tokens-v1.1.0`).
- `apps/wpda`, `apps/maintenance` e `apps/api`: nada.

**Spec do ciclo:** `docs/superpowers/specs/2026-10-12-interface-cycle-foundation-design.md`. Ela traz as decisões 10 a 15 do ciclo: AA + eMAG, os dois temas, o seletor por pessoa, as três larguras, e os cards com trilho.

**Base:** o subprojeto 1 já está publicado (tokens `tokens-v1.0.0`, tema `govbr`, `Tag`/`StatusDot` com forma, e as guardas de foco e de `--ink4`).

**Regra do ciclo:** só interface. As telas, os dados, as permissões e as chamadas ao api não mudam.

Três mudanças de uso foram decididas pelo usuário nesta conversa:
1. A entrada passa a ser o Início.
2. O botão Voltar do navegador sobe um nível.
3. O endereço passa a refletir a tela.

## 1. Ponto de partida

**Dashboard:**
- `App.tsx` guarda `active` num `useState` e escolhe a tela com um `switch` de 34 casos.
- O período (`period`) vive no `ScopeContext`.
- O cabeçalho (`shell/AppHeader.tsx`) carrega tudo numa linha só e se sobrepõe a partir de cerca de 1280px:
  - marca;
  - 12 grupos em `NavDropdown`;
  - `SignatureSessionBadge`;
  - `TenantChip`;
  - `SegmentedControl` do período;
  - `NotificationCenter`;
  - relógio;
  - `UserMenu`, que só tem "Sair".
- **Não há roteamento.** O Voltar do navegador sai do painel, e recarregar a página volta para a Visão geral.
- A navegação entre telas por atalho é a seguinte:

  | Origem | Destinos |
  |---|---|
  | Overview | queues, health, events |
  | Protocols | security |
  | Team | professionals, security |
  | Attendance | security |
  | Campaigns | security |
  | SignaturePending | signature |
  | 9 telas via `onGoToSecurity` | security |
  | `SignatureSessionBadge` | signature |
  | Alertas do `NotificationCenter` | queues, health |

- Os retornos do prestador de assinatura e do SNCR passam pelo `main.tsx`, que entrega o `landing` ao `App` como `initialModule`. O `return_to` enviado ao api é `"/"+id`.
- O filtro de bairro mora em `?bairro=` e é escrito com `replaceState` a partir de `location.href`, o que preserva o hash.

**Admin:**
- Mesma estrutura do dashboard, com 17 casos no `switch`.
- 3 grupos: Setup (cities, provision_city, register_channel, unknown_channels), Analytics (city_analytics) e e-SUS (city_production).
- Entra em `cities`.
- Não tem `TenantChip` nem sessão de assinatura, e o relógio mostra "BRT" fixo.

**Testes:** nenhum teste renderiza o `App`, o `AppHeader`, o `NavDropdown` ou o `NotificationCenter`. Os testes que restringem o shell são:
- `shell/modules.test.ts` nos dois apps: nomes, ids e filtros de `NAV_GROUPS`/`navGroupsFor`;
- os testes do `SignatureSessionBadge` (props `{active, onSelect, redirect}` e o DOM dele);
- os testes do `SignatureCallback` e do `SncrCallback` (`onDone(landing)`);
- `theme.test.ts` (`<html lang="pt-BR" data-theme="govbr">`);
- `a11y-guards.test.ts`.

## 2. Decisões desta conversa

| # | Decisão |
|---|---|
| 1 | **Barra de topo + faixa de contexto.** A barra de topo é só de navegação: marca, busca, sino e conta. Abaixo dela, uma faixa sempre visível mostra cidade, período e sessão de assinatura (o aviso "simulada" junto). O relógio só aparece a partir de 1280px. |
| 2 | **O Início (cards dos grupos) é a entrada nos dois painéis.** A Visão geral vira um card como os outros. Os retornos de assinatura e de SNCR continuam indo direto à tela de origem. |
| 3 | **Endereço e Voltar.** O Voltar do navegador sobe um nível, e o endereço reflete a tela com `#/…`. Recarregar a página mantém a tela, e o link pode ser compartilhado. Uma tela que o papel não vê, ou um hash inválido, levam ao Início. |
| 4 | **Ícones Tabler** (`@tabler/icons-react`, MIT) nos dois apps, no lugar dos símbolos unicode. |
| 5 | **Abordagem:** shell próprio por app e roteador por hash escrito por nós (`useHashRoute`). O `switch` de telas e os atalhos entre telas são mantidos. O dashboard é feito primeiro, e o admin recebe a mesma estrutura, copiada e adaptada (ADR 0002). `react-router` foi recusado. |

## 3. Navegação e roteamento

### 3.1 Rotas

| Hash | Tela |
|---|---|
| vazio ou `#/` | Início: um card por grupo visível para o papel |
| `#/grupo/<slug>` | Tela do grupo: um sub-card por tela visível do grupo |
| `#/<id>` | A tela de id `<id>` (ids de `ModuleId`, os mesmos de hoje) |
| `#/acessibilidade` | Página de acessibilidade (nova, estática, fora dos grupos) |

O slug do grupo é o rótulo sem acento, em minúsculas, com espaços trocados por hífen:

| Rótulo | Slug |
|---|---|
| Visão geral | `visao-geral` |
| e-SUS | `e-sus` |
| Comunicação | `comunicacao` |

Os rótulos de `NAV_GROUPS` não mudam, porque os filtros por papel comparam esses rótulos.

### 3.2 `useHashRoute`

- **Arquivo:** `src/shell/router.ts`, um em cada app.
- **Tipo:** `Route = { kind: "home" } | { kind: "group"; slug: string } | { kind: "screen"; id: ModuleId } | { kind: "a11y" }`.
- **Funções:**
  - `parseHash(hash): Route | null` é puro e devolve `null` para um hash que não reconhece;
  - `toHash(route): string` também é puro.
- **Hook:** `useHashRoute({ initial, isAllowed })` devolve `{ route, go(route) }`.
  - **Leitura:** na montagem, lê `location.hash`. Se o hash estiver vazio e houver `initial`, que vem do `initialModule` dos retornos, usa `initial` e grava com `replaceState`.
  - **Navegação:** `go` grava com `pushState` (`history.pushState(history.state, "", url-com-hash)`) e atualiza o estado.
  - **Voltar:** o `popstate` relê o hash.
  - **Rota não permitida:** quando `parseHash` dá `null`, ou quando `isAllowed(route)` é falso, a rota vira `home` com `replaceState`.
- **O que `isAllowed` libera:**
  - `home` e `a11y`;
  - um grupo presente em `navGroupsFor(user)`;
  - uma tela presente nos itens visíveis.
  - Também libera as telas fora do menu que hoje só se abrem por atalho. A lista é `OFF_MENU`:
    - no dashboard: vazia;
    - no admin: `setup_mfa`.
  - Um atalho interno (`go` chamado pelo próprio código) não é filtrado. Ele continua abrindo o que abre hoje.
  - O filtro de `isAllowed` vale só para o endereço digitado ou colado e para o Voltar.

### 3.3 O que não muda

- `renderModule` e todas as props de navegação entre telas. Muda só a implementação: `setActive(id)` vira `go({ kind: "screen", id })`.
- O `NotificationCenter` (`onNavigate`) e o `SignatureSessionBadge` (`onSelect`, `active`).
- `initialModule`, `return_to` (`"/"+id`), `moduleFromPath`, `ScopeContext` e `?bairro=`.

### 3.4 Grupo de uma tela só

O card leva direto à tela, e a trilha mostra "Início › Tela":
- dashboard: Visão geral, Análise e Comunicação, além de qualquer grupo que o filtro de papel reduza a uma tela;
- admin: Analytics e e-SUS.

### 3.5 Contagem de pendentes

O rótulo "Pendentes de assinatura (N)" continua vindo do `withPendingCount`. No sub-card, o número aparece como um selo. No trilho, o grupo Atendimento ganha um ponto quando N for maior que 0.

## 4. Layout e componentes

### 4.1 Estrutura

```
EmagBar
TopBar         [Menu (<1024)] Marca  ScreenSearch  ……  NotificationCenter  ContaMenu
ContextStrip   TenantChip  SegmentedControl(período)  ……  SignatureSessionBadge  [Relógio ≥1280]
┌Rail (≥1024)┐ <main id="conteudo">  Breadcrumb (telas internas) · tela
```

No admin, a `ContextStrip` mostra só o período e o relógio.

### 4.2 Componentes novos

Todos ficam em `src/shell/`, um conjunto em cada app.

| Componente | Comportamento |
|---|---|
| `EmagBar` | Três links com `accesskey`: "Ir para o conteúdo" (1, `#conteudo`), "Ir para o menu" (2, o trilho ou o botão Menu) e "Ir para a busca" (3). À direita, "Acessibilidade" (`#/acessibilidade`) e o botão "Alto contraste". Fonte de 12px. É o primeiro elemento focável da página. |
| `TopBar` | A marca é um link para o Início. Também tem o `ScreenSearch`, o `NotificationCenter` atual sem mudança e o `ContaMenu`. Abaixo de 1024px aparece o botão "Menu", que abre a `Drawer`. |
| `ContaMenu` | Substitui o `UserMenu`. Mantém o botão `aria-label="Conta"` e o item "Sair", que chama `auth.logout()` como hoje. Ganha o grupo "Tema" com três itens `menuitemradio` e `aria-checked`: gov.br, Rota Saúde e Alto contraste. |
| `ContextStrip` | `TenantChip`, `SegmentedControl` e `SignatureSessionBadge`, os três como estão hoje, só mudando de lugar. O relógio (`LiveClock`) aparece só a partir de 1280px. A faixa quebra linha abaixo de 1024px. |
| `Rail` | Largura de 60px, `<nav aria-label="Grupos">`. O primeiro ícone é o Início e os outros são os grupos visíveis. Cada ícone é um `<a href>` com `aria-label` igual ao rótulo do grupo e uma dica no hover e no foco. Leva à tela do grupo, ou direto à tela no grupo de uma tela só. O grupo atual fica com `aria-current="page"`. |
| `Drawer` | Abaixo de 1024px, abre pelo botão Menu. É um `<nav aria-label="Menu">` num diálogo (`role="dialog"`, `aria-modal`) com a lista completa de grupos e telas. Fecha com Esc, com clique fora e ao escolher uma tela. Prende o foco enquanto está aberta e devolve o foco ao botão Menu quando fecha. |
| `HomeCards` | Mostra "Bom dia/Boa tarde/Boa noite" e um card por grupo visível. O card tem ícone em quadro de 44px, título e descrição. A grade tem 2, 3 ou 4 colunas a partir de 768, 1024 e 1280px. Cada card é um `<a href="#/…">`. |
| `GroupCards` | O cabeçalho repete o ícone e a descrição do grupo. Mostra um sub-card por tela visível, no mesmo visual dos cards do Início, com selo de contagem quando houver. |
| `Breadcrumb` | Aparece nas telas internas: "Início › Grupo › Tela". O último item é um botão que abre um menu com as outras telas do grupo. Em grupo de uma tela só, mostra "Início › Tela". |
| `ScreenSearch` | Segue o padrão combobox do ARIA 1.2 (`role="combobox"`, `aria-expanded`, `aria-activedescendant`, lista `role="listbox"`). Busca no rótulo, no grupo e na descrição das telas visíveis, sem diferenciar acentos nem maiúsculas. Mostra até 8 resultados. Enter ou clique abre a tela, e Esc limpa o campo. |
| `A11yPage` | A página de acessibilidade descrita na seção 5.3. |

### 4.3 Conteúdo dos cards

O conteúdo fica em `src/shell/catalog.ts`: ícone Tabler e descrição por grupo e por tela. Os rótulos continuam vindo de `NAV_GROUPS`.

**Dashboard: grupos**

| Grupo | Ícone | Descrição |
|---|---|---|
| Visão geral | `IconLayoutDashboard` | Indicadores do dia na cidade |
| Aquisição | `IconInbox` | Conversas e consentimentos |
| Triagem | `IconListCheck` | Triagens, classificação e relatórios |
| Análise | `IconChartLine` | Analytics da cidade |
| Governança | `IconFileCertificate` | Protocolos e auditoria |
| Operação | `IconServerCog` | Filas e saúde do sistema |
| Atendimento | `IconStethoscope` | Agenda, prontuário e consultas |
| Comunicação | `IconSpeakerphone` | Campanhas por SMS |
| Equipe | `IconUsers` | Pessoas, papéis e assinaturas |
| Cidade | `IconBuildingCommunity` | Território e documentos |
| e-SUS | `IconArrowsExchange` | Integrações, CNES e produção |
| Conta | `IconUserCircle` | Segurança, perfil e assinatura |

**Dashboard: telas**

| id | Ícone | Descrição |
|---|---|---|
| overview | `IconLayoutDashboard` | Triagens, filas e alertas de hoje |
| ingestion | `IconDownload` | Volume de mensagens recebidas |
| conversations | `IconMessages` | Conversas do canal do cidadão |
| consent | `IconShieldCheck` | Aceites e revogações do termo |
| triages | `IconListCheck` | Triagens concluídas e em andamento |
| classification | `IconChartPie` | Distribuição por risco e prioridade |
| reports | `IconFileText` | Relatórios entregues ao cidadão |
| analytics | `IconChartLine` | Tendências por semana e bairro |
| protocols | `IconFileCertificate` | Versões, assinaturas e ativação |
| protocol-editor | `IconPencil` | Escrever e revisar rascunhos |
| events | `IconHistory` | Quem fez o quê e quando |
| queues | `IconStack2` | Filas de processamento e falhas |
| health | `IconHeartbeat` | Frescor dos dados e alertas do sistema |
| attendance | `IconStethoscope` | Fila de chamada e desfecho |
| my-agenda | `IconCalendar` | Seus horários de hoje e da semana |
| signature-pending | `IconSignature` | Documentos esperando sua assinatura |
| clinical-record | `IconNotes` | Histórico clínico do cidadão |
| my-consultations | `IconClipboardList` | Consultas que você registrou |
| professional-consultations | `IconUsersGroup` | Consultas da equipe por pessoa |
| open-drafts | `IconFilePencil` | Consultas iniciadas e não concluídas |
| campaigns | `IconSpeakerphone` | Criar e enviar campanhas por SMS |
| team | `IconUsers` | Usuários, convites e papéis |
| professionals | `IconIdBadge2` | Vínculos, CBO e turnos |
| signature-overview | `IconCertificate` | Certificados digitais da equipe |
| sncr-overview | `IconPill` | Numeração de receita controlada da equipe |
| territory | `IconMapPin` | Bairros e unidades de referência |
| clinical-documents-admin | `IconFileDescription` | Modelos de atestado e receita |
| integrations | `IconPlugConnected` | Conexão com o PEC e-SUS |
| cnes | `IconBuildingHospital` | Estabelecimentos e profissionais do CNES |
| production | `IconUpload` | Envio da produção ao e-SUS |
| security | `IconLock` | Senha e verificação em duas etapas |
| my-profile | `IconUser` | Seus dados profissionais |
| signature | `IconCertificate` | Seu certificado para assinar |
| sncr | `IconPill` | Sua numeração de receita controlada |

**Admin**

| Grupo ou tela | Ícone | Descrição |
|---|---|---|
| Setup | `IconSettings` | Cidades, canais e provisionamento |
| cities | `IconBuildingCommunity` | Catálogo das cidades da plataforma |
| provision_city | `IconPlus` | Criar uma cidade nova |
| register_channel | `IconPhonePlus` | Ligar um número de canal a uma cidade |
| unknown_channels | `IconPhoneOff` | Mensagens de números sem cidade |
| Analytics | `IconChartDots` | Indicadores de todas as cidades |
| city_analytics | `IconChartDots` | Indicadores de todas as cidades |
| e-SUS | `IconArrowsExchange` | Produção enviada pelas cidades |
| city_production | `IconUpload` | Produção enviada pelas cidades |

### 4.4 Larguras

| Largura | Trilho | Gaveta | Cards por linha | Relógio | Faixa de contexto |
|---|---|---|---|---|---|
| 768–1023 | não | sim | 2 | não | quebra em duas linhas |
| 1024–1279 | sim | não | 3 | não | uma linha |
| 1280 ou mais | sim | não | 4 | sim | uma linha |

As regras de largura ficam em CSS (`@media`) num arquivo `src/shell/shell.css`, com classes. Não usam `matchMedia` em JavaScript. O conteúdo das telas não muda: em 768px, as tabelas largas rolam para o lado até o subprojeto 3. Os alvos de toque do shell têm 44px ou mais.

### 4.5 O que sai

`shell/AppHeader.tsx`, `shell/NavDropdown.tsx` e o `UserMenu`. O `TenantChip`, o `Brand` e o `LiveClock` saem do `AppHeader` e vão para arquivos próprios, sem mudar o comportamento.

## 5. Temas, eMAG e acessibilidade

### 5.1 Tema

`src/theme/theme.ts`, em cada app, exporta `THEME_KEY = "rotasaude:theme"`, `getTheme()`, `setTheme(name)` e `useTheme()`.

- **Temas disponíveis:** `govbr` (padrão), `rota` e `alto-contraste`. O `govbr-escuro` existe nos tokens, mas não é oferecido nos painéis.
- **Persistência:** `localStorage`, sempre dentro de `try/catch`. Se a leitura falhar ou trouxer um valor que não é dessa lista, vale `govbr`.
- **`setTheme`:** aplica `document.documentElement.dataset.theme`, salva e avisa os ouvintes.
- **Botão "Alto contraste" da `EmagBar`:** se o tema atual não é `alto-contraste`, guarda o tema atual e liga o alto contraste. Se já é, volta ao tema guardado, ou a `govbr`.
- **Antes do React montar:** um script inline no `index.html` aplica o tema salvo, sem flash. O atributo `data-theme="govbr"` continua no `<html>` como padrão.

### 5.2 Figtree no contracts (`tokens-v1.1.0`)

- **Arquivos:** Figtree 400, 600 e 700 em woff2, tirados do pacote npm `@fontsource/figtree` (subconjunto latin), com o `LICENSE` (OFL) ao lado em `dist/fonts/`.
- **Download:** precisa da permissão do usuário na execução.
- **Declaração:** `roles.json` ganha as faces da Figtree, e o teste de faces existente passa a cobrir as duas famílias.
- **Distribuição:** os dois apps sincronizam com `tokens-v1.1.0`.

### 5.3 eMAG

- **Atalhos:** `accesskey` 1, 2 e 3 na `EmagBar`, conforme o padrão eMAG. O modificador depende do navegador: Alt no Chrome e no Edge, Alt+Shift no Firefox.
- **Página `#/acessibilidade`:**
  - nível declarado: WCAG 2.2 AA e eMAG 3.1;
  - tabela dos atalhos;
  - alto contraste e temas;
  - foco visível;
  - nível de risco também indicado por forma;
  - compatibilidade com leitor de tela;
  - para relatar problema: "fale com o administrador da sua prefeitura" no dashboard, e "fale com a equipe da plataforma" no admin.

### 5.4 Acessibilidade do shell

- **Landmarks:** `header` (barra eMAG, barra de topo e faixa), `nav` com rótulos distintos ("Grupos" no trilho, "Menu" na gaveta) e `main#conteudo`.
- **Troca de rota:**
  - `document.title` passa a ser "<Tela> · Rota Saúde" ("Início · Rota Saúde" no Início);
  - o foco vai para o `h1` da página com `tabIndex=-1`. O `h1` é o do Início, o da tela do grupo, ou o primeiro `h1` dentro de `main`. Se a tela não tiver `h1`, o foco vai para `main`.
- **Contador do `NotificationCenter`:** o texto `"white"` fixo passa para `var(--on-accent)` quando o selo é `--accent`. Para as cores de tom, usa uma cor de texto que atinja 4,5:1 em todos os temas: `--panel` sobre `--down`/`--warn`/`--info` no govbr e no rota, e `--bg` (preto) no alto contraste. O valor fica num papel novo, `--on-tone`, em `tokens-v1.1.0`, coberto pelo teste de contraste. Esse foi o adiado "badge branco fixo" do subprojeto 1.
- **Sem `outline: "none"`.** A guarda do subprojeto 1 já cobre os arquivos novos.

## 6. Testes e critério de pronto

**Garantia de funcionalidade:** as suítes atuais dos dois apps passam **sem nenhuma asserção editada**. Os `modules.test.ts`, os testes do `SignatureSessionBadge` e os testes dos retornos de assinatura e SNCR continuam como estão.

**Testes novos** (Vitest e Testing Library, nos dois apps quando o componente existe nos dois):

- **Roteador:**
  - `parseHash` e `toHash` vão e voltam para cada tipo de rota;
  - um slug ou id desconhecido dá `null`;
  - `go` faz `pushState` e o `popstate` atualiza a rota;
  - uma rota não permitida vira `home` com `replaceState`, sem entrar no histórico;
  - `initial` com hash vazio usa `replaceState`;
  - um atalho interno não é filtrado.
- **Navegação entre telas:** renderizando o `App` com os módulos de origem simulados, cada atalho muda o hash para a tela de destino. Os atalhos cobertos são:
  - Overview → queues, health, events;
  - Team → professionals, security;
  - `onGoToSecurity` → security;
  - SignaturePending → signature;
  - os alertas do `NotificationCenter`.
- **Início e tela do grupo:** os grupos visíveis batem com `navGroupsFor` para os papéis municipal_admin, health_professional, analyst, campaign_manager e operador. Um grupo de uma tela só leva direto à tela. O selo de pendentes aparece.
- **Trilho e gaveta:**
  - `aria-current` no trilho;
  - a gaveta fecha com Esc e com clique fora, prende o foco e devolve o foco ao botão Menu.
- **Trilha de navegação:** "Início › Grupo › Tela", e o menu do último item troca de tela.
- **Busca:** "agenda" acha "Minha agenda"; "producao" acha "Produção e-SUS"; uma tela invisível para o papel não aparece; as setas e o Enter funcionam.
- **Tema:**
  - `getTheme` com armazenamento vazio, com valor inválido e quando o armazenamento lança erro;
  - `setTheme` aplica o tema e salva;
  - o botão de alto contraste liga e depois volta ao tema anterior;
  - o script inline do `index.html` aplica o tema salvo.
- **Acessibilidade do shell:** `document.title` e o foco no `h1` ao trocar de rota.
- **contracts:** o teste de faces cobre a Figtree. O teste de contraste cobre `--on-tone` sobre `--down`, `--warn`, `--info` e `--accent` em todos os temas.

**Ajuste em teste novo do subprojeto 1:** a guarda de `--ink4` em `a11y-guards.test.ts` passa a mirar `shell/TopBar.tsx` e `shell/ContextStrip.tsx`, porque o `shell/AppHeader.tsx` deixa de existir. Esse teste foi criado neste ciclo; a regra de não editar asserções vale para os testes anteriores ao ciclo.

**Conferência visual:** capturas de antes e depois:
- em 768, 1024 e 1280px, no Início, numa tela de grupo (Atendimento) e numa tela interna (Minha agenda);
- no 1280px, também nos três temas;
- no admin, no Início e em Cidades.

O script de captura do subprojeto 1 ganha as larguras e os temas.

**Pronto quando:**
1. `tokens-v1.1.0` está publicado e as suítes dos dois apps estão verdes, sem asserção antiga editada;
2. o usuário aprovou as capturas;
3. o merge e o push foram feitos em contracts, dashboard, admin e docs;
4. a sessão do módulo 19 foi avisada antes de começar, porque a rodada especial (dashboard#12–14) toca telas que passam pelo `App.tsx`.

## 7. Riscos

- **Navegação entre telas.** São cerca de 20 chamadas, e todas passam a usar `go`. Os testes de navegação entre telas cobrem cada origem.
- **Script inline no `index.html` e CSP.** Se o nginx de produção mandar uma CSP sem `'unsafe-inline'`, o script é bloqueado. Nesse caso, o tema é aplicado no `main.tsx`, com um flash breve. O plano confere o `nginx.conf` dos dois apps antes de escolher.
- **`accesskey` conflitando com atalhos do navegador ou do leitor de tela.** É o padrão do eMAG e fica documentado na página de acessibilidade.
- **Hábito da Visão geral como entrada.** É uma mudança de uso decidida pelo usuário.
- **Conflito de merge com a rodada especial do módulo 19,** no `App.tsx` e nas telas do Atendimento. Coordenar a ordem com a sessão do módulo 19.
