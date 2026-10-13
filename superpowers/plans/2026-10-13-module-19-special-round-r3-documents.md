# Rodada especial do módulo 19 — fatia R3: documentos clínicos — plano de implementação

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Fechar três cards do módulo 19c:
- **api#59:** a recepção lista, reimprime e cancela declarações de comparecimento, restritas à unidade do seletor;
- **api#60:** o admin prepara a REMUME e os protocolos com os documentos clínicos ainda desligados;
- **api#61:** histórico de versões do protocolo de enfermagem, no api e no painel.

**Architecture:**
- **api, sem migração:**
  - um parâmetro `health_unit_id` (a unidade escolhida no seletor) nas rotas da declaração;
  - um grant novo de leitura, `:unit_reception`, só para declaração;
  - o cancelamento aberto à recepção da mesma unidade;
  - a busca de medicamentos com o interruptor por papel;
  - uma rota de versões.
- **dashboard:** o painel de declaração ganha a lista do atendimento, e a fila ganha "Declarações de hoje", as duas com reimprimir e cancelar (reaproveitando `CancelDocument`). O painel de protocolos ganha "Ver versões".

**Tech Stack:** Rails 8 e RSpec (api); React 18, TanStack Query 5, Vitest e Testing Library (dashboard).

**Spec:** `docs/superpowers/specs/2026-10-13-module-19-special-round-design.md` (§7). Cards: rotasaude/api#59, #60 e #61.

## Global Constraints

- **Unidade da declaração (api#59):**
  - vem do seletor da tela, no parâmetro `health_unit_id`;
  - o api só confere que o atendimento é daquela unidade. Não há vínculo da recepção com a unidade: isso não é trava de segurança (decisão do usuário, 2026-10-13);
  - falta de `health_unit_id`, ou unidade diferente da do atendimento, dá 422 `attendance_outside_unit` na emissão, nas listas, na leitura e no cancelamento feitos pela recepção que não é a autora.
- **Grant `:unit_reception`:**
  - só para `attendance_declaration`, para quem tem `citizen_verifier`, com o `health_unit_id` igual ao do atendimento;
  - registro: `clinical_record.viewed` com `access: "unit_reception"`;
  - a autora segue com o grant `:author`.
- **Listas:**
  - `GET /attendance/attendances/:id/declarations?health_unit_id=`;
  - `GET /attendance/declarations?health_unit_id=&date=AAAA-MM-DD`, com o dia no fuso da cidade e a data padrão hoje;
  - itens: `{ id, status, issued_at, short_code, cancelled_at, author: { id, name }, attendance_id, patient: { display_name } }`, sem conteúdo nem motivo de cancelamento;
  - **Ruling:** a listagem não grava `clinical_record.viewed`, porque só mostra o que a fila já mostra (nome e horário). Ler e imprimir o documento continuam gravando. O spec pedia o registro também na listagem; isso custaria um evento por linha.
- **Cancelamento:** `POST /attendance/documents/:id/cancel` com `{ reason, health_unit_id? }`, step-up e motivo de 10 a 500 caracteres, como hoje. Quem pode cancelar:
  - a autora, como hoje;
  - a recepção (`citizen_verifier`) com o `health_unit_id` do atendimento, **só** em `attendance_declaration`.
- **Busca de medicamentos (api#60):**
  - `clinical_record` vale para todos;
  - `clinical_documents` só para quem **não** é `municipal_admin`;
  - quem tem os dois papéis segue a regra do admin, porque a busca não emite nada.
- **Versões (api#61):**
  - `GET /clinical_documents/nursing_protocols/:id/versions` → `{ items: [{ id, version, valid_from, valid_until, situation, items: [{ catalog_item, max_dose }] }] }`;
  - ordem decrescente de `version`;
  - `situation` é `"agendada"` (início futuro), `"vigente"` (a versão de `current_version` hoje) ou `"encerrada"` (o resto), com hoje no fuso da cidade;
  - mesmos papéis de leitura do índice.
- **Testes:**
  - todo item ganha um teste que falha antes;
  - as suítes do api e do dashboard passam;
  - api com `ROTA_TEST_DB_SUFFIX=_r19` e `docker compose stop worker` antes da suíte completa;
  - rota GET nova em `/attendance` entra na lista de permissões de `spec/invariants/attendance_invariants_spec.rb`. Nenhuma leva CPF.
- **Git:**
  - `/opt/homebrew/bin/git`;
  - Conventional Commits em inglês, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`.
- **Worktrees:**
  - api em `apps/api/.claude/r3`, com `master.key` e `exec -w /rails/.claude/r3`;
  - dashboard em `rota-saude/.worktrees/dashboard-r3`;
  - branch `fix/module-19-round-r3` a partir de `origin/main`.

## Review Focus

1. **Recepção da UBS A emite e, no dia seguinte, a da UBS B tenta cancelar.**
   - Esperado: 422 `attendance_outside_unit` quando o seletor está na UBS B; com o seletor na UBS A, cancela.
   - Prova: Task 3.
2. **Declaração cancelada na lista.**
   - Esperado: aparece como cancelada, sem "Reimprimir" nem "Cancelar"; a reimpressão dá 409.
   - Prova: Tasks 3 e 4.
3. **Profissional emite declaração de atendimento de outra unidade** (vínculo em outra UBS).
   - Esperado: 422 `attendance_outside_unit`, não `missing_role`.
   - Prova: Task 3.
4. **Admin com o interruptor `clinical_documents` desligado busca na REMUME.**
   - Esperado: 200. O profissional, com o mesmo interruptor desligado, recebe 403 `feature_disabled`.
   - Prova: Task 1.
5. **Protocolo só com versão futura.**
   - Esperado: o painel diz "sem versão vigente hoje — próxima a partir de dd/mm/aaaa", e o histórico mostra "agendada".
   - Prova: Tasks 2 e 5.

---

### Task 0: Preparação

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git
$G -C $R/apps/api fetch -q && $G -C $R/apps/api worktree add -q $R/apps/api/.claude/r3 -b fix/module-19-round-r3 origin/main
cp $R/apps/api/config/master.key $R/apps/api/.claude/r3/config/master.key
$G -C $R/apps/dashboard fetch -q && $G -C $R/apps/dashboard worktree add -q $R/.worktrees/dashboard-r3 -b fix/module-19-round-r3 origin/main
(cd $R/.worktrees/dashboard-r3 && npm ci --silent && npx vitest run 2>&1 | tail -2)
docker exec -w /rails/.claude/r3 -e ROTA_TEST_DB_SUFFIX=_r19 api-dev bundle exec rspec spec/requests/clinical_documents_spec.rb spec/requests/medication_search_spec.rb spec/requests/clinical_documents 2>&1 | tail -2
```

Expected: verde nos dois.

---

### Task 1: api#60 — busca do admin com o interruptor desligado

**Files:**
- Modify: `app/controllers/medication_searches_controller.rb`
- Test: `spec/requests/medication_search_spec.rb`

- [ ] **Step 1: testes que falham.** No padrão do exemplo `"switch off → 403; reception → 403"` (linha ~43):
  - **Admin, `clinical_documents` desligado e `clinical_record` ligado:** 200 com itens.
  - **Profissional com `clinical_documents` desligado:** 403 `feature_disabled`, como hoje.
  - **Admin com `clinical_record` desligado:** 403 `feature_disabled` com `feature: "clinical_record"`.
  - **Usuário com os dois papéis e `clinical_documents` desligado:** 200.

  Run: `docker exec -w /rails/.claude/r3 -e ROTA_TEST_DB_SUFFIX=_r19 api-dev bundle exec rspec spec/requests/medication_search_spec.rb`
  Expected: FAIL nos casos do admin.

- [ ] **Step 2: implementar.**

```ruby
include ClinicalRecordGate
before_action :require_clinical_record!
before_action :require_searcher
before_action :require_documents_for_professionals

# api#60: o municipal_admin monta a REMUME e os protocolos antes de ligar os
# documentos clínicos (só clinical_record); o profissional, que emite, segue
# exigindo clinical_documents. Com os dois papéis vale a regra do admin.
def require_documents_for_professionals
  return if CitizenVerificationPolicy.new(Current.user, nil).manage?

  require_clinical_documents!
end
```

  Confira os nomes do concern (`ClinicalRecordGate#require_clinical_record!`) no `ClinicalDocumentsController`, que já usa os dois. Atualize o comentário do topo do arquivo.

  Run: o mesmo.
  Expected: PASS.

- [ ] **Step 3: commit** (`fix: let the municipal admin search medications before enabling clinical documents`).

---

### Task 2: api#61 — rota de versões

**Files:**
- Modify: `config/routes.rb` (junto das rotas de `nursing_protocols`: `get "nursing_protocols/:id/versions", to: "nursing_protocols#versions"`), `app/controllers/clinical_documents/nursing_protocols_controller.rb`, `app/services/clinical_documents/json.rb`
- Test: `spec/requests/clinical_documents/nursing_protocols_spec.rb`

**Interfaces:**
- Produces:
  - `ClinicalDocuments::Json.protocol_version(version, catalog:, situation:)`, usado também por `Json.protocol` para a vigente;
  - a rota dos Global Constraints.

- [ ] **Step 1: testes que falham.** Um protocolo com 3 versões (encerrada, vigente e agendada, com início daqui a 10 dias) no `travel_to` de uma data fixa:
  - **Ordem e situação:** a resposta traz as 3 em ordem decrescente de `version`, com `situation` `agendada`, `vigente` e `encerrada`, e os itens com `max_dose`.
  - **Leitura:** profissional e admin leem; recepção recebe 403. Id inválido dá 404.
  - **Índice:** o `index` continua devolvendo só `current_version`, a mesma forma de hoje.

  Run: `docker exec -w /rails/.claude/r3 -e ROTA_TEST_DB_SUFFIX=_r19 api-dev bundle exec rspec spec/requests/clinical_documents/nursing_protocols_spec.rb`
  Expected: FAIL (rota inexistente).

- [ ] **Step 2: implementar.**

```ruby
# controller
before_action :require_reader_role, only: %i[index versions]
before_action :require_admin_role, except: %i[index versions]
before_action :require_step_up!, except: %i[index versions]

# api#61: todas as versões, para auditoria e para ver a agendada.
def versions
  protocol = uuid?(params[:id]) && NursingProtocol.find_by(id: params[:id])
  return not_found unless protocol

  versions = protocol.versions.includes(:items).order(version: :desc).to_a
  catalog = MedicationCatalogItem.where(id: versions.flat_map { |v| v.items.map(&:catalog_item_id) }.uniq).index_by(&:id)
  today = Time.zone.today
  current = protocol.current_version(on: today)
  render json: { items: versions.map { |v| Json.protocol_version(v, catalog: catalog, situation: Json.version_situation(v, current, today)) } }
end
```

```ruby
# json.rb
def version_situation(version, current, today)
  return "vigente" if current && version.id == current.id
  version.valid_from > today ? "agendada" : "encerrada"
end

def protocol_version(version, catalog:, situation: nil)
  { id: version.id, version: version.version, valid_from: version.valid_from.iso8601, valid_until: version.valid_until&.iso8601,
    items: version.items.sort_by { |i| [ i.created_at, i.id ] }.filter_map do |item|
      entry = catalog[item.catalog_item_id]
      entry && { catalog_item: catalog_item(entry),
                 max_dose: item.max_quantity && { quantity: number(item.max_quantity), unit: item.max_quantity_unit } }
    end }.tap { |h| h[:situation] = situation if situation }
end
```

  `Json.protocol` passa a usar `protocol_version(version, catalog:)` para a vigente, sem `situation`, para manter a forma de hoje.

  - O `Time.zone` do api é o da cidade? Confira `config/application.rb` (`America/Sao_Paulo`). Se a cidade tiver fuso próprio, use o mesmo `today` do `Json.protocol`.
  - **Rota em `/attendance`?** Não: `nursing_protocols` mora em `/clinical_documents`. Confira se há allowlist equivalente com `grep -rn "nursing_protocols" spec/invariants spec/architecture`.

  Run: o mesmo, mais `spec/requests/clinical_documents`.
  Expected: PASS.

- [ ] **Step 3: commit** (`feat: list every nursing protocol version with its situation`).

---

### Task 3: api#59 — declaração restrita à unidade, listas e cancelamento pela recepção

**Files:**
- Modify:
  - `config/routes.rb`: no escopo `/attendance`, `get "attendances/:id/declarations", to: "clinical_documents#attendance_declarations"` e `get "declarations", to: "clinical_documents#declarations"`;
  - `app/controllers/clinical_documents_controller.rb`;
  - `app/commands/clinical_documents/issue.rb` (recebe `health_unit_id:`);
  - `app/commands/clinical_documents/cancel.rb` (recebe `health_unit_id:`);
  - `app/services/clinical_documents/read.rb` (grant `:unit_reception`);
  - `app/services/clinical_documents/json.rb` (`Json.declaration_row`);
  - `spec/invariants/attendance_invariants_spec.rb` (lista de permissões: as duas rotas GET).
- Test: `spec/requests/clinical_documents_spec.rb` (bloco das declarações, linhas ~69–138), `spec/requests/clinical_document_cancel_spec.rb`, `spec/requests/clinical_document_print_spec.rb`

**Interfaces:**
- Produces:
  - `ClinicalDocuments::Issue.call(..., health_unit_id: nil)`. Na declaração, falta ou diferença dá `Result.fail(:attendance_outside_unit)`;
  - `ClinicalDocuments::Cancel.call(document_id:, by:, reason:, health_unit_id: nil)`;
  - `ClinicalDocuments::Read.document_grant(user:, document:, health_unit_id: nil)`;
  - erro `attendance_outside_unit` → 422 no `ERROR_STATUS`.

- [ ] **Step 1: testes que falham:**
  - **Emissão:**
    - a recepção com `health_unit_id` da unidade do atendimento emite (201);
    - sem o parâmetro ou com outra unidade: 422 `attendance_outside_unit`;
    - o profissional vinculado a outra UBS: 422 `attendance_outside_unit` (Review Focus 3).
  - **Lista do atendimento:**
    - a recepção vê as declarações daquele atendimento (emitidas por ela e por colegas) com os campos dos Global Constraints, sem `content` nem `cancel_reason`;
    - outra unidade: 422;
    - nenhuma `clinical_record.viewed` nova.
  - **Lista do dia:** duas declarações hoje na UBS A e uma na UBS B. `?health_unit_id=A` devolve as duas; `&date=` de ontem, nenhuma.
  - **Leitura e impressão:**
    - uma colega da recepção (não autora), com o `health_unit_id` certo, imprime (200 PDF) e grava `clinical_record.viewed` com `access: "unit_reception"`;
    - com outra unidade: 403;
    - cancelada: 409, como hoje.
  - **Cancelamento:**
    - a colega da recepção, com step-up e a unidade certa, cancela;
    - com outra unidade: 422 (Review Focus 1);
    - a recepção não cancela atestado nem receita, nem com a unidade certa: 403 `not_author`;
    - a autora segue cancelando, como hoje.
  - **Invariantes:** as duas rotas GET novas passam na lista de permissões.

  Run: `docker exec -w /rails/.claude/r3 -e ROTA_TEST_DB_SUFFIX=_r19 api-dev bundle exec rspec spec/requests/clinical_documents_spec.rb spec/requests/clinical_document_cancel_spec.rb spec/requests/clinical_document_print_spec.rb spec/invariants/attendance_invariants_spec.rb`
  Expected: FAIL.

- [ ] **Step 2: implementar.**
  - **`Issue`:** no ramo da declaração, antes do `Issuers.declaration_staff`:

    ```ruby
    return Result.fail(:attendance_outside_unit) unless health_unit_id.present? && attendance.health_unit_id == health_unit_id
    ```

  - **`Read.document_grant`:** depois do ramo da autora:

    ```ruby
    # api#59: a recepção da unidade do atendimento (a escolhida no seletor) lê a declaração.
    if document.kind == "attendance_declaration" && user.has_role?("citizen_verifier") &&
       health_unit_id.present? && document.attendance.health_unit_id == health_unit_id
      return Grant.new(kind: :unit_reception, opening: nil, reason: nil)
    end
    ```

  - **`Cancel`:** troque `next Result.fail(:not_author) unless document.author_user_id == by.id` por:

    ```ruby
    unless document.author_user_id == by.id
      next Result.fail(:not_author) unless document.kind == "attendance_declaration" && by.has_role?("citizen_verifier")
      next Result.fail(:attendance_outside_unit) unless health_unit_id.present? && document.attendance.health_unit_id == health_unit_id
    end
    ```

  - **Controller:**
    - `require_staff` passa a liberar para `verify?` também `cancel`, `attendance_declarations` e `declarations`;
    - `declaration`, `cancel`, `show` e `print` repassam `params[:health_unit_id]` (string; ignore se não for UUID);
    - `ERROR_STATUS` ganha `attendance_outside_unit: :unprocessable_entity`.
  - **Listas:**

    ```ruby
    # api#59: declarações do atendimento, para a recepção reimprimir e cancelar. Só metadados.
    def attendance_declarations
      attendance = find(Attendance, params[:id])
      return not_found unless attendance
      return render_failure(Result.fail(:attendance_outside_unit), ERROR_STATUS) unless unit_matches?(attendance.health_unit_id)

      rows = ClinicalDocument.where(kind: "attendance_declaration", attendance_id: attendance.id)
                             .includes(:patient, :author_user).order(issued_at: :desc, id: :desc)
      render json: { items: rows.map { |d| ClinicalDocuments::Json.declaration_row(d) } }
    end

    def declarations
      unit_id = uuid_param(:health_unit_id)
      return render_failure(Result.fail(:attendance_outside_unit), ERROR_STATUS) unless unit_id && HealthUnit.exists?(id: unit_id)

      day = (Date.iso8601(params[:date].to_s) rescue Time.zone.today)
      rows = ClinicalDocument.joins(:attendance).where(kind: "attendance_declaration", attendances: { health_unit_id: unit_id })
                             .where(issued_at: day.in_time_zone.all_day).includes(:patient, :author_user).order(issued_at: :desc, id: :desc)
      render json: { items: rows.map { |d| ClinicalDocuments::Json.declaration_row(d) } }
    end
    ```

  - **`Json.declaration_row(d)`:**

    ```ruby
    { id:, status:, issued_at:, cancelled_at:, short_code: Codes.display(d.short_code), attendance_id:,
      author: { id: d.author_user.id, name: Screenings::Json.staff_name(d.author_user) },
      patient: d.patient && { display_name: d.patient.display_name } }
    ```

    Se o paciente for nulo (par não validado), use o `display_name` do cidadão do atendimento, como faz o restante do módulo.
  - **Nomes reais:** verifique os nomes de associação (`author_user`, `attendance`) e o helper de UUID do controller (`find`, `uuid?`) antes de usar.

  Run: o mesmo do Step 1.
  Expected: PASS.

- [ ] **Step 3: commit** (`feat: let reception list, reprint and cancel declarations of its unit`).

---

### Task 4: api#59 (dashboard) — lista no painel e "Declarações de hoje"

**Files:**
- Modify:
  - `src/lib/api.ts`:
    - `issueAttendanceDeclaration(attendanceId, content, healthUnitId)`;
    - `listAttendanceDeclarations(attendanceId, healthUnitId)`;
    - `listUnitDeclarations(healthUnitId, date)`;
    - `cancelDocument(id, reason, healthUnitId?)`;
    - `fetchDocumentPdf` e `printDocument` aceitando `healthUnitId?` (query `?health_unit_id=`);
    - o tipo `DeclarationRow`.
  - `src/modules/documents/CancelDocument.tsx` (prop opcional `healthUnitId`);
  - `src/modules/documents/printDocument.ts`;
  - `src/modules/attendance/DeclarationPanel.tsx` (prop `unitId`);
  - `src/modules/attendance/UnitQueue.tsx` (passa `unit.id`; seção nova);
  - Create: `src/modules/attendance/UnitDeclarations.tsx`.
- Test:
  - `src/lib/api.clinicalDocuments.test.ts`;
  - `src/modules/attendance/UnitQueue.declaration.test.tsx`;
  - Create: `src/modules/attendance/UnitDeclarations.test.tsx`.

**Interfaces:**
- Produces:
  - `interface DeclarationRow { id: string; status: DocumentStatus; issued_at: string; cancelled_at: string | null; short_code: string | null; attendance_id: string; author: { id: string; name: string }; patient: { display_name: string } | null }`;
  - `UnitDeclarations({ unitId: string; onGoToSecurity?(): void })`.

- [ ] **Step 1: testes que falham.**
  - **`api.clinicalDocuments.test.ts`:** cada cliente novo chama a rota certa, com `health_unit_id` na query (GET) ou no corpo (`POST` da emissão e do cancelamento).
  - **`UnitQueue.declaration.test.tsx`:**
    - abrir a declaração de uma linha lista as declarações daquele atendimento;
    - uma emitida tem "Reimprimir" e "Cancelar";
    - uma cancelada aparece como "cancelada", sem botões (Review Focus 2);
    - "Reimprimir" chama a impressão com o id e a unidade;
    - "Cancelar" abre o `CancelDocument` (motivo e código) e, ao concluir, relê a lista;
    - emitir uma nova também relê a lista.
  - **`UnitDeclarations.test.tsx`:**
    - "Declarações de hoje" começa recolhida;
    - ao abrir, busca `listUnitDeclarations(unitId, hoje)` e mostra paciente, hora, autor e status, com os mesmos botões;
    - lista vazia diz "nenhuma declaração emitida hoje nesta unidade".

  Run: `npx vitest run src/lib/api.clinicalDocuments.test.ts src/modules/attendance`
  Expected: FAIL.

- [ ] **Step 2: implementar.**
  - **`DeclarationPanel`:**
    - recebe `unitId`;
    - `useQuery(["attendanceDeclarations", row.id, unitId], …)`;
    - abaixo do formulário (ou do aviso de emitida), uma lista compacta;
    - "Reimprimir" usa `printDocument(doc.id, documentError, unitId)`;
    - "Cancelar" renderiza `CancelDocument` com `doc` (adaptando a linha para o tipo que ele espera; se precisar de `ClinicalDocument` completo, busque com `getClinicalDocument(id, unitId)` ao clicar) e `healthUnitId`;
    - ao concluir, `invalidateQueries(["attendanceDeclarations", …])`.
  - **`UnitDeclarations`:** a mesma lista para o dia, num `<details>` com `summary` "Declarações de hoje", que monta a query só ao abrir.
  - **`UnitQueue`:** passa `unit.id` ao painel e renderiza `<UnitDeclarations unitId={unit.id} />` quando `clinicalDocuments` está ligado.
  - **Rótulos:** "emitida", "cancelada" (reaproveite o que existir em `lib/clinicalDocuments.ts`), e a hora com `fmtHourMinute`.
  - **Erro `attendance_outside_unit`** em `documentError`: "esta declaração é de um atendimento de outra unidade — troque a unidade no seletor".

  Run: `npx vitest run 2>&1 | tail -3 && npx tsc --noEmit`
  Expected: verde.

- [ ] **Step 3: commit** (`feat: list, reprint and cancel the unit's attendance declarations`).

---

### Task 5: api#61 (dashboard) — histórico de versões no painel

**Files:**
- Modify: `src/lib/api.ts` (`listNursingProtocolVersions(id)` e o tipo `NursingProtocolVersionRow`, que é a versão com `situation`), `src/modules/clinicalDocumentsAdmin/NursingProtocolsPanel.tsx`
- Test: `src/modules/clinicalDocumentsAdmin/NursingProtocolsPanel.test.tsx`

- [ ] **Step 1: testes que falham:**
  - **"Ver versões"** de um protocolo lista as versões com número, vigência ("de dd/mm/aaaa a dd/mm/aaaa" ou "a partir de dd/mm/aaaa"), situação ("vigente", "agendada a partir de dd/mm/aaaa" ou "encerrada") e os itens com dose máxima.
  - **Protocolo sem vigente e com uma agendada** (Review Focus 5): a coluna de vigência diz "sem versão vigente hoje — próxima a partir de dd/mm/aaaa". A lista só traz `current_version`, então a próxima vem do histórico, buscado sob demanda. Ruling: o texto da próxima aparece depois de "Ver versões". Sem essa abertura, segue "sem versão vigente hoje".
  - **Erro ao buscar** mostra a frase de erro do painel, sem quebrar a tabela.

  Run: `npx vitest run src/modules/clinicalDocumentsAdmin/NursingProtocolsPanel.test.tsx`
  Expected: FAIL.

- [ ] **Step 2: implementar.** Botão "Ver versões" na linha (ou no cartão). Abre um painel com `useReveal`, com a lista (`useQuery(["nursingProtocolVersions", id])`) e "Fechar". Siga o estilo do painel existente (`DataTable` ou lista). Use as datas com `fmtDay`, como o painel já faz.

  Run: `npx vitest run 2>&1 | tail -3 && npx tsc --noEmit && npm run build`
  Expected: verde.

- [ ] **Step 3: commit** (`feat: show the nursing protocol version history`).

---

### Task 6: Contrato, suítes, prova, revisão e publicação

- [ ] **Contrato.** `docs/superpowers/plans/2026-10-09-module-19c-documents-contracts.md` ganha:
  - as duas rotas de lista;
  - o `health_unit_id` na emissão, na leitura, na impressão e no cancelamento da declaração;
  - o grant `unit_reception`;
  - a rota de versões;
  - a regra nova da busca.

  Commit no docs: `docs: update the module 19c contract for the special round`.
- [ ] **Suítes completas:**
  - api: `docker compose stop worker`; `docker exec -w /rails/.claude/r3 -e ROTA_TEST_DB_SUFFIX=_r19 api-dev bundle exec rspec`; `docker compose start worker`;
  - dashboard: `npx vitest run`, `npx tsc --noEmit` e `npm run build`.
- [ ] **Revisão final** por revisor independente, no modelo mais capaz, com o Review Focus. Critical e Important são corrigidos com TDD.
- [ ] **Aprovação do usuário.**
- [ ] **Publicação.**
  - avise as sessões;
  - `merge --ff-only` e `push` do api, depois do dashboard e do docs;
  - confira as CIs;
  - prova no navegador em dev, num navegador headless com `recepcao@curitiba.demo` e `admin@curitiba.demo`: emitir, listar, reimprimir e cancelar uma declaração; histórico de um protocolo. Capturas em `evidencias/2026-10-13-rodada-19/r3/`;
  - mova api#59, #60 e #61 para "Resolvido", com comentário, e feche.
- [ ] **Fechamento da rodada:**
  - `DROP` dos bancos de teste com sufixo `_r19` (conferindo conexões);
  - memória atualizada;
  - página de status da rodada (Artifact) no padrão das páginas de módulo;
  - comentário no api#75 com a decisão do módulo 28.
