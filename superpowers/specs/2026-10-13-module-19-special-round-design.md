# Rodada especial do módulo 19 — design

**Data:** 2026-10-13
**Origem:** decisão do usuário de 2026-10-11 ("rodada especial ao final do módulo 20") para as pendências do módulo 19 no board #2 que não dependem de serviço externo.
**Caminho:** uma spec, quatro fatias em sequência (R1 → R4 → R2 → R3). Cada fatia segue o ciclo plano → execução nesta sessão → revisão final → aprovação do usuário → merge e push.

## 1. Objetivo

Fechar as pendências do módulo 19 que podem ser resolvidas só com código, configuração e documentação, antes do go-live da assinatura digital e dos documentos clínicos.

**Critério de sucesso:**
- os 9 cards em escopo ficam em "Resolvido" no board #2, cada um com a evidência do seu critério de encerramento;
- os gates externos continuam abertos e intocados.

## 2. Escopo e decisões

| Card | Fatia | Decisão do usuário (2026-10-13) |
|---|---|---|
| signer#1 | R1 | Os 9 endurecimentos agora |
| api#74 | R4 | Conta + configuração + runbook. A prova com carga em staging vira gate de go-live. A conta fecha para **5 cidades** |
| api#52 | R2 | Registrar falha de geração nos dois ramos mudos (opção 1 do card) |
| dashboard#12 | R2 | A autora vê a lista completa de problemas do paciente no adendo da própria consulta, com acesso registrado como `author` |
| dashboard#13 | R2 | Invalidar a leitura do prontuário depois de finalizar e de acrescentar adendo |
| dashboard#14 | R2 | Só front: depois das releituras, "aguardando assinatura — veja Pendentes" |
| api#59 | R3 | Lista, reimpressão e cancelamento pela recepção. **Restrito à unidade**, e a unidade vem do seletor da tela (não há vínculo de recepção com unidade no api) |
| api#60 | R3 | A busca no catálogo para o `municipal_admin` exige só `clinical_record` |
| api#61 | R3 | Histórico de versões no api e no painel do dashboard |

### Fora do escopo

- **api#75** (dentista emite atestado e receita): fica para o módulo 28, por decisão do usuário de 2026-10-12. O card continua no board #2.
- **Gates externos de go-live:** api#53, api#63, api#56, api#57, api#41, api#58, `production.yml.enc` e a evidência de staging do api#62.
- **Vínculo de recepção com unidade.** A restrição do api#59 usa a unidade escolhida na tela. Isso evita erro no dia a dia, mas não é trava de segurança; a trava exigiria mexer no cadastro de equipe e no admin.
- **PgBouncer.** Fica registrado no runbook como próximo passo, não implantado.

## 3. Restrições globais

- **Mudanças de regra:** só as decididas acima.
  - O dashboard#12 amplia o acesso da autora a dados do prontuário, e só à lista de problemas.
  - O api#59 restringe a emissão à unidade do seletor.
  - O api#60 relaxa o interruptor só para a busca do admin.
- **Testes:**
  - todo item corrigido ganha um teste que falha antes da correção;
  - as suítes existentes passam;
  - uma asserção só muda quando o comportamento muda de propósito, com uma linha `Ruling:` no ledger.
- **Suíte do api:**
  - `docker compose stop worker` antes da suíte completa e `start` depois;
  - bancos de teste com sufixo (`ROTA_TEST_DB_SUFFIX=_r19`), apagados no fechamento.
- **Git:**
  - commits em Conventional Commits, em inglês, com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`;
  - `/opt/homebrew/bin/git`;
  - worktrees de frontend em `rota-saude/.worktrees/`; o do api em `apps/api/.claude/<wt>`, com `master.key` copiada e `exec -w` no container.
- **Coordenação:** avisar as outras sessões (módulo 20, curadora) antes de cada merge no api e no dashboard, e conferir `origin/main..main`.
- **Contrato com o api:** a R1 não muda o contrato do signer com o api, nem os campos que o api lê do `/verify`.

## 4. Fatia R1 — signer#1 (repo `signer`)

1. **Hora no futuro.** `signedAt` maior que agora mais 5 minutos dá `invalid` com `signed_in_future`. Vale para o CAdES (`signingTime`) e para o PAdES (`/M`).
2. **CPF e nome só com cadeia confiável.** `/verify` e `/certificates/check` devolvem `signer_cpf` e `signer_name` só quando o certificado encadeia numa âncora confiável. Sem cadeia, malformado ou forjado, os dois vão nulos.
   - Com isso, o `indeterminate` (cadeia boa, LCR fora do ar) continua trazendo o CPF.
   - O api continua forçando `invalid` quando o CPF não bate.
3. **SSRF da LCR.**
   - A checagem de confiança (`Chain.of`) roda antes do `CAdESChecker`/`PAdESChecker` do Demoiselle. Sem cadeia confiável, o resultado é `invalid` sem nenhum download de LCR nem acesso a destino tirado do certificado. (A atualização das âncoras da ICP-Brasil pelo próprio Demoiselle vai a destinos fixos do ITI e do Serpro e fica fora desta regra; ver o README do signer.)
   - O download de LCR aceita só `http`/`https` e recusa destino que resolva para endereço privado, loopback ou link-local.
4. **Auto-verificação.**
   - Na partida, o serviço confere que as âncoras de confiança carregaram (em produção, ICP-Brasil); se não, não sobe.
   - O `/assemble` verifica a assinatura que produziu antes de devolvê-la. Se ela não der `valid`, responde 422 `assembled_signature_invalid`.
5. **Concorrência e tamanho.**
   - Semáforo de requisições simultâneas (`SIGNER_MAX_CONCURRENCY`, padrão 8). O excesso recebe 503 com `Retry-After`.
   - `Content-Length` acima de 48 MiB é recusado antes da leitura, com o mesmo 400 de hoje.
6. **Cache de LCR.** Cache em memória por URL, válido até o `nextUpdate`, com limite de entradas. É compartilhado pela nossa checagem e pela do Demoiselle.
7. **ByteRange estrito.** Exige:
   - 4 valores, todos não negativos, com `range[0] = 0`;
   - `range[0] + range[1] < range[2]`;
   - o vão ser exatamente a string hexadecimal do `/Contents`, entre `<` e `>`;
   - `range[2] + range[3]` igual ao tamanho do arquivo.

   Qualquer desvio dá `invalid`.
8. **Imagem.**
   - Os `FROM` passam a usar `@sha256`.
   - Entra um `GET /livez` sem token, que só responde `ok`.
   - O `HEALTHCHECK` chama o `/livez` por uma classe Java mínima, sem instalar curl.
9. **LCR indisponível persistente.**
   - Log `WARN` só com o host da LCR, no máximo um por host por minuto.
   - O `/health` ganha `revocation_unavailable_count` e `revocation_unavailable_since`.

**Testes (JUnit, `mvn -B verify`):**
- um teste por item, com os fixtures do `TestPki`;
- no item 3, um servidor HTTP falso conta os acessos e prova que zero requisições saem quando a cadeia não é confiável;
- o build da imagem segue na CI.

## 5. Fatia R4 — api#74 (api + docs)

1. **Calculadora** `Ops::ConnectionBudget` e rake `ops:connection_budget[cidades]`.
   - Leem `config/queue.yml`, `config/puma.rb` e `deploy/production/deploy.yml` (hosts, `WEB_CONCURRENCY`, `RAILS_MAX_THREADS`).
   - Devolvem o teto por parte (web, workers por cidade, plataforma) e o total.
   - O teto de cada pool é o menor entre o tamanho do pool e as threads que o usam.
   - Cada processo de worker conta os pools Solid Queue padrão, Solid Queue por cidade, `CityRecord` e plataforma.
   - Cada processo web conta plataforma, cache e 2 pools por cidade.
2. **Configuração.**
   - Pools das cidades com `idle_timeout` de 60 s (`CityConnection.database_config`).
   - Pool do worker no mínimo exigido pelo Solid Queue: threads do maior worker mais 2.
   - `max_connections` do `deploy.yml` e `CITY_ROLE_CONNECTION_LIMIT` derivados da calculadora para **5 cidades**, com 20% de folga.
   - O comentário velho do `deploy.yml` é substituído pela referência ao runbook.
3. **Spec de orçamento** em `spec/config/`. Recalcula com os arquivos reais e falha se o total para 5 cidades passar de 80% do `max_connections` configurado.
4. **Runbook** `docs/operacao/conexoes-postgres.md`:
   - fórmula e tabela de 1 a 20 cidades;
   - como recalcular;
   - quando subir o limite ou adotar PgBouncer (modo transação);
   - prova com carga em staging registrada como gate de go-live.

   A seção "Orçamento de conexões" do README do api passa a apontar para o runbook. Os itens bloqueantes de `max_connections` nos runbooks de rollout (assinatura e documentos clínicos) passam a citar o novo runbook, mantendo a prova em staging como pendente.

**Testes:**
- calculadora com cenários fixos (1, 4 e 10 cidades);
- spec de orçamento sobre os arquivos reais;
- `solid_queue_configuration_spec` continua passando.

## 6. Fatia R2 — consulta e assinatura (api + dashboard)

**api#52**
- **Ficha recusada.** Em `Ledi::ConsultationFicha#refresh!`, quando o `Ledi::Enqueue` da ficha recusada devolve nil, registra `LediGenerationFailure` com o motivo `regeneration_refused`. O `case` ganha `else`, que registra o mesmo motivo.
- **Job que desiste.** `Ledi::ConsultationFichaJob` ganha bloco no `retry_on InFlight`: depois da 10ª tentativa, registra `still_sending`.
- **Motivos e rótulos.** Os dois motivos entram em `LediGenerationFailure::REASONS`. O dashboard (`GenerationFailures.tsx`) ganha os rótulos "o reenvio da ficha recusada não foi aceito" e "a ficha seguia em envio; gere de novo".
- **Gerar de novo.** O "gerar de novo" existente resolve as duas falhas.

**dashboard#12**
- **api.**
  - `GET /attendance/consultations/:id/problems` devolve só a lista de problemas do paciente (formato de `ClinicalRecord::Json`).
  - Exige `ClinicalRecord::Access.for_consultation` com grant `author`: consulta finalizada e autora logada. Os outros casos recebem 403.
  - Registra `clinical_record.viewed` com `access: author` e o `consultation_id`.
- **dashboard.**
  - Em "Minhas consultas", a lista é buscada ao abrir o formulário de adendo e passada como `patientProblems`, no lugar do `[]` de hoje.
  - Erro ao buscar: o formulário segue com os problemas da consulta e mostra o aviso "não foi possível carregar a lista completa".

**dashboard#13.** `ConsultationWorkspace` invalida `[RECORD_KEY, attendanceId]` no `onFinalized` e no `onAddendumAdded`, como a abertura justificada já faz com o `JUSTIFIED_KEY`.

**dashboard#14.** Esgotadas as 3 releituras com bloco ainda "pendente sem motivo", o marcador passa de "assinando…" para "aguardando assinatura — veja Pendentes", com link para a tela de Pendentes.

**Testes:**
- **api:** os dois ramos do api#52 com falha registrada e listada; a rota de problemas (autora, outra pessoa, consulta aberta, registro de acesso).
- **dashboard:** problema antigo no adendo de "Minhas consultas"; painel atualizado sem recarregar depois de finalizar e de acrescentar adendo; marcador depois da 3ª releitura.

## 7. Fatia R3 — documentos clínicos (api + dashboard + docs)

**api#59**
- **Emissão.** `POST /attendance/attendances/:id/declarations` exige `health_unit_id`. Quando o atendimento é de outra unidade, responde 422 `attendance_outside_unit`. Vale para a recepção e para o profissional.
- **Listas.**
  - `GET /attendance/attendances/:id/declarations`.
  - `GET /attendance/declarations?health_unit_id=&date=` (do dia, pela unidade do atendimento).
  - Cada item traz status, quem emitiu e `issued_at`, sem conteúdo cifrado. A listagem registra leitura.
- **Reimpressão.** Pelo `print` existente; declaração cancelada continua dando 409.
- **Cancelamento.** Pelo `POST /attendance/documents/:id/cancel`, com motivo de 10 a 500 caracteres e step-up, como hoje. Para declaração, pode cancelar:
  - quem emitiu;
  - a recepção (`citizen_verifier`) que informa o mesmo `health_unit_id` do atendimento.

  A conferência pública mostra "cancelada".
- **dashboard.**
  - `DeclarationPanel` lista as declarações do atendimento, com "Reimprimir" e "Cancelar" (motivo e código).
  - A fila (`UnitQueue`) ganha "Declarações de hoje" da unidade.
  - Toda chamada envia o `health_unit_id` do seletor.

**api#60.** `MedicationSearchesController` exige `clinical_record` para todos e `clinical_documents` só para quem não é `municipal_admin`. Quem tem os dois papéis segue a regra do admin.

**api#61**
- **api.**
  - `GET /clinical_documents/nursing_protocols/:id/versions` lista as versões em ordem decrescente, com `version`, `valid_from`, `valid_until`, itens com dose máxima e `situation` (`agendada`, `vigente` ou `encerrada`, calculada para hoje no fuso da cidade).
  - Mesmos papéis de leitura do índice; sem migração.
  - `ClinicalDocuments::Json.protocol` passa a usar um helper por versão.
- **dashboard.**
  - `NursingProtocolsPanel` ganha "Ver versões" com a lista e os itens.
  - Versão futura aparece como "agendada a partir de dd/mm/aaaa".
  - Sem versão vigente hoje, mostra qual é a próxima, quando houver.
- **Contrato.** `docs/superpowers/plans/2026-10-09-module-19c-documents-contracts.md` ganha as rotas novas do api#59 e do api#61.

**Testes:**
- **api:** emissão fora da unidade; listas; cancelamento por quem emitiu, pela recepção da unidade e recusado para outra unidade; busca do admin com o interruptor desligado e recusada para profissional; versões nas três situações.
- **dashboard:** reimprimir e cancelar pela lista; "Declarações de hoje"; histórico com versão agendada.

## 8. Fechamento

Por fatia:
- revisão final por revisor independente (o mais capaz), com Critical e Important corrigidos com TDD;
- aprovação do usuário;
- merge e push;
- card movido para "Resolvido" no board #2, com comentário de evidência.

O dashboard#12 e o api#61 registram a decisão no comentário. O api#75 recebe comentário com a decisão do módulo 28 e continua aberto.

No fim da rodada:
- `DROP` dos bancos com sufixo `_r19`;
- memória `rodada-especial-pos-modulo-20` atualizada;
- página de status da rodada no padrão das páginas de módulo.
