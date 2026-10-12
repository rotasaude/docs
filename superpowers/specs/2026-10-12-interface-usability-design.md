# Ciclo de interface — subprojeto 6: usabilidade — design

**Data:** 2026-10-12
**Status:** em revisão
**Afeta:** `apps/dashboard`, `apps/admin`, `apps/maintenance` e `apps/wpda`.
**Spec do ciclo:** `docs/superpowers/specs/2026-10-12-interface-cycle-foundation-design.md`. Lá o subprojeto 6 é só "usabilidade dos fluxos: levantamento por tela".

## 1. Escopo decidido

| # | Decisão |
|---|---|
| 1 | **Escopo:** fechar as pendências adiadas dos subprojetos 1–5 e do módulo 15 no wpda (**parte A**, 28 itens), mais os atritos dos 4 fluxos críticos (**parte C**, 74 itens). Entram todos os itens, inclusive os de severidade baixa. |
| 2 | **Fluxos críticos:** fila e chamada, atendimento, triagem do cidadão no wpda e protocolos. |
| 3 | **Limite:** só front-end, sem mudar regra. Pode mudar comportamento de interface: foco, ordem, textos, confirmações, atalhos e estados de carregamento, erro e vazio. Não pode mudar API, dados, permissões nem regra de negócio. O que exigir API vira card no board #2. |
| 4 | **Organização:** 5 fatias. Cada uma tem plano, execução, capturas, aprovação, merge e push. |
| 5 | **Dado clínico:** nunca fica guardado no navegador. Rascunho que se perde vira card de API. No front, só se faz salvar mais cedo e avisar antes de sair. |

### 1.1 Origem dos itens

- **Percursos:** cada fluxo foi percorrido no dev em 2026-10-12, só lendo a tela e capturando. Os achados ficam em `rota-saude/evidencias/2026-10-12-usabilidade/<fluxo>/achados.md` (fora do git), com capturas, efeito para quem usa e correção proposta. As pastas são `fila-chamada/`, `atendimento/`, `wpda/` e `protocolos/`.
- **Parte A:** está em `evidencias/2026-10-12-usabilidade/pendencias-A.md`.
- **Correção:** por padrão vale a proposta de cada achado, com os ajustes das seções 3 e 4.
- **Contagem:**

| Fluxo | Alta | Média | Baixa |
|---|---|---|---|
| Fila e chamada (FC) | 5 | 10 | 4 |
| Atendimento (AT) | 5 | 11 | 4 |
| wpda (WP) | 4 | 11 | 3 |
| Protocolos (PR) | 5 | 8 | 4 |

## 2. Fatias e ordem

| Fatia | App | Itens |
|---|---|---|
| 6a — padrões comuns | dashboard + admin | Seção 3, mais A-01 a A-12 |
| 6b — fila e chamada | dashboard | FC-01 a FC-19 |
| 6c — atendimento | dashboard | AT-01 a AT-20 |
| 6d — protocolos | dashboard + maintenance | PR-01 a PR-17, mais A-21 a A-28 |
| 6e — cidadão | wpda | WP-01 a WP-18, mais A-13 a A-20 |

A ordem é 6a → 6b → 6c → 6d, porque as três últimas usam os padrões da 6a. A 6e é independente e pode correr a qualquer momento. O módulo de protocolos do admin, que está fora da navegação, fica como está.

## 3. Fatia 6a — padrões comuns (dashboard e admin)

1. **`useReveal(ref)`**
   - Quando um painel, confirmação ou formulário aparece por causa de um clique, rola até ele com `scrollIntoView({ block: "nearest" })`.
   - Põe o foco no título do painel, que recebe `tabIndex=-1`.
   - Resolve FC-03, FC-04, FC-11, AT-08 e PR-04. No maintenance, o PR-04 usa a mesma ideia, implementada lá.
2. **`useDialog`** para painéis que cobrem a tela (o painel lateral de Protocolos e a gaveta):
   - `role="dialog"` com `aria-modal`;
   - foco preso dentro do painel;
   - Escape fecha e o foco volta para quem abriu.
   - Painéis em linha também passam a fechar com Escape (FC-13). Resolve PR-02/06 e A-10.
3. **Foco que não se perde**
   - O botão em execução troca `disabled` por `aria-disabled="true"` e mostra "…", ignorando cliques repetidos. Assim mantém o foco.
   - Ao terminar, o foco vai para a mensagem de resultado ou para o próximo passo.
   - Resolve FC-05, AT-07 e PR-06.
4. **`ActionMessage`**
   - A mensagem de sucesso ou erro aparece logo abaixo do grupo de ações que a gerou e some quando outra ação começa.
   - Usa o mesmo `role="status"`/`alert` de hoje, sem criar papel novo.
   - Resolve FC-02, AT-08 e PR-04.
5. **Confirmação em dois passos** para ação irreversível sem código
   - O primeiro clique troca o botão por "Confirmar <ação>?" com Confirmar e Cancelar; só o segundo clique executa.
   - Ação sensível continua no `SensitiveAction`, que ganha um texto obrigatório dizendo o que acontece (PR-03/05).
6. **Motivo do bloqueio**
   - Botão desabilitado por regra mostra o porquê logo abaixo, ligado por `aria-describedby`.
   - Segue o modelo que já existe em Protocolos. Resolve FC-12 e AT-12.
7. **Fim das reticências**
   - `.dt-cell` (em `components/datatable.css`) passa a quebrar linha, com `white-space: normal` e `overflow-wrap: anywhere`.
   - `KeyValue` e `Overview` também passam a quebrar linha.
   - Corte com reticências só onde a coluna optar explicitamente, e então com o texto inteiro em `title`.
   - Uma guarda de fonte proíbe `textOverflow: "ellipsis"` sem `title`.
   - Resolve A-12, FC-07, AT-01, AT-02, AT-15 e a tabela de eventos dos protocolos.
8. **Alvos de toque:** abaixo de 1024px, botões, rádios, caixas de seleção e abas ficam com 44px ou mais, por CSS em `responsive.css`. Resolve FC-14 e AT-17.

Os padrões 1–6 e 8 também valem para o admin, onde houver o mesmo caso.

## 4. Decisões por fatia (ajustes às correções propostas)

### 4.1 Fatia 6b — fila e chamada

- **FC-01:** "Saiu sem atendimento" passa a usar a confirmação em dois passos e mostra o resultado na linha. Não há "desfazer", porque isso seria regra nova.
- **FC-03, FC-04 e FC-11:** os painéis abrem com `useReveal` e dizem de quem são, pelo nome ou pelo CPF mascarado.
- **FC-08:** a espera longa ganha destaque só visual quando a chegada é de outro dia ("chegou ontem", "há 2 dias"). Não há limite de horas, porque isso seria regra. A espera passa a aparecer como "3 d 8 h".
- **FC-10:** a página ganha no topo um índice com âncoras para Pedidos, Agenda e Balcão. A lista de pedidos vem recolhida, com os 5 primeiros e "ver todos (N)".
- **FC-19:** a lista passa a usar chave estável.
- **Vira card:** FC-09, se a API não trouxer o nome legível do protocolo, e FC-18 (estado da escuta na fila).

### 4.2 Fatia 6c — atendimento

- **AT-04:** o autosave salva na hora quando o campo perde o foco, quando a aba fica oculta (`visibilitychange`) e quando a tela fecha. Com algo pendente, aparece o aviso `beforeunload`. Nada é guardado no navegador.
- **AT-03:** vira card de API (rascunho da escuta no servidor). No front, só o aviso `beforeunload` com a escuta preenchida.
- **AT-06:** trocar de documento com o formulário preenchido usa a confirmação em dois passos.
- **AT-16:** o seletor mostra primeiro as unidades com vínculo, se a API já informar o vínculo. Senão, vira card.
- **AT-19:** "Chamar próximo" fica desabilitado, com o motivo, quando a fila está vazia. A parte "em escuta e chamável" é FC-18 (card).
- **Vira card:** AT-05 (tabela CIAP-2 de dev), AT-13 ("nunca abriu sessão" contra "sessão venceu") e os acentos do AT-18.

### 4.3 Fatia 6d — protocolos

- **PR-01:** não é defeito. As assinaturas de ativação contam só depois da última ativação (`Protocols::Signatures`), e o "0/2" de uma versão em uso significa "para a próxima ativação". Nas versões em uso, a linha passa a dizer "em uso desde dd/mm". O contador "0/2" só aparece em versões publicadas que aguardam ativação.
- **PR-08:** os erros do editor são traduzidos para frases em português, com caminho legível ("Pergunta 1 › tipo de resposta"). Salvar fica bloqueado com o motivo enquanto houver erro de validação local.
- **PR-09:** uma versão publicada abre só para leitura, com a ação "Criar nova versão a partir desta".
- **PR-10:** depois de salvar, aparece o link "Ir para o ciclo de vida", que abre a versão salva.
- **PR-14:** um vocabulário só nos dois apps: "em uso". No maintenance, só o rótulo de `active` muda.
- **PR-12:** Aposentar e Reverter ganham estilo secundário ou de perigo.
- **PR-15:** a ajuda do código inválido do maintenance é copiada para o dashboard.
- **Vira card:** PR-11 (quem agiu e o motivo nos eventos).
- **A-21 a A-28:** pendências do maintenance, entram nesta fatia.

### 4.4 Fatia 6e — cidadão (wpda)

- **WP-01 e A-17:** o 192 vira uma faixa fixa no rodapé, "Emergência? Ligar 192", sempre visível. No resultado continua em destaque.
- **WP-02:** a cada troca de tela, o foco vai para o `h1`.
- **WP-03:** durante a espera aparece "Preparando seu resultado…" com indicador. As ações só aparecem com o resultado pronto. Se a espera desistir, aparece "Tentar de novo".
- **WP-11:** o CPF aparece mascarado no perfil.
- **WP-14:** a lista de bairros ganha campo de busca no topo, com "Prefiro não informar" em primeiro.
- **WP-15:** a ação principal passa a ser mostrar o código. "Gerar outro código" vira secundária, com confirmação em dois passos.
- **Vira card:** WP-06 (pular o termo com consentimento vigente) e WP-09 (apelido por pessoa).

## 5. Cards de API (board #2, "Pendências de Ciclo")

Os cards são criados com permissão do usuário, na fatia que os gera:

- **AT-03 e AT-04:** rascunho da escuta e do SOAP no servidor.
- **AT-05 e AT-18:** tabela CIAP-2 de dev e acentos.
- **AT-13:** distinguir "nunca abriu sessão" de "sessão venceu".
- **FC-09:** nome legível do protocolo na fila, se a API não tiver.
- **FC-18:** estado da escuta na fila.
- **PR-11:** quem agiu, o motivo e a versão nos eventos de protocolo.
- **WP-06:** pular o termo quando há consentimento vigente.
- **WP-09:** apelido por pessoa no mesmo celular.
- **AT-16:** vínculo do profissional com a unidade, se a API não informar.

## 6. Testes e critério de pronto

**Testes por fatia:**
- **Vitest:**
  - A suíte existente passa.
  - Uma asserção só muda quando o comportamento muda de propósito (por exemplo, a confirmação nova do FC-01). Cada mudança dessas fica registrada no plano, com o achado que a motiva.
  - Cada achado corrigido ganha um teste que falha antes da correção, sempre que der para testar no jsdom: foco, Escape, confirmação, mensagem limpa, rótulo, salvamento imediato, chave estável. Quando não der, a prova é a captura.
- **Ponta a ponta:**
  - No maintenance (6d) e no wpda (6e), os Playwright existentes passam.
  - O wpda ganha um caso: o 192 visível sem rolar em 360×640.
- **Varredura de corte (6a–6d):**
  - Um script Playwright conta, nas telas dos 3 fluxos de painel, em 1024 e 1280px, os elementos com `scrollWidth > clientWidth` e corte por reticências sem `title`.
  - Esperado: zero.
- **Capturas antes e depois** dos mesmos passos dos percursos, comparadas lado a lado para aprovação.
- **Revisão final** do branch pelo revisor mais capaz, com os achados da fatia como lista de checagem.

**Pronto, por fatia:**
1. Todos os itens estão corrigidos ou viraram card.
2. Vitest, e2e e varredura estão verdes.
3. O usuário aprovou as capturas.
4. Merge e push estão feitos.

O subprojeto 6, e com ele o ciclo de interface, termina quando as 5 fatias estiverem prontas e os cards criados.

## 7. Riscos

- **Confirmações novas mudam o ritmo de quem já usa** ("Saiu sem atendimento" em dois cliques). Isso é intencional e merece aviso a quem opera o posto.
- **Quebra de linha no lugar de reticências** deixa as linhas mais altas em 1024px. Isso é aceito e conferido nas capturas.
- **Sessões paralelas no dashboard** (módulos 19/20):
  - o trabalho usa worktree em `rota-saude/.worktrees/`;
  - antes de cada push, conferir `origin/main..main`;
  - antes do merge, avisar a sessão dona.
- **Dados que os percursos deixaram no dev:** triagens, chamadas, documentos de teste e o protocolo `u6-teste-193133` em uso em Curitiba. Isso é aceito em dev e nada é apagado.
