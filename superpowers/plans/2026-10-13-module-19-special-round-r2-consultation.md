# Rodada especial do módulo 19 — fatia R2: consulta e assinatura — plano de implementação

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Fechar o api#52 (adendo que some da ficha LEDI sem aviso), o dashboard#12 (lista completa de problemas no adendo de "Minhas consultas"), o dashboard#13 (painel de problemas desatualizado) e o dashboard#14 ("assinando…" que não sai).

**Architecture:** Duas mudanças no api e quatro no dashboard.
- **api:**
  - os dois ramos mudos da regeneração da ficha passam a registrar uma falha de geração (`LediGenerationFailure`), que já aparece em "Produção › não geradas";
  - uma rota nova devolve à autora só a lista de problemas do paciente, com registro de acesso `author`.
- **dashboard:** invalida a leitura do prontuário depois de finalizar ou de acrescentar adendo; busca a lista de problemas ao abrir o adendo de "Minhas consultas"; troca o "assinando…" quando as releituras acabam.

**Tech Stack:** Rails 8 e RSpec (api); React 18, TanStack Query 5, Vitest e Testing Library (dashboard).

**Spec:** `docs/superpowers/specs/2026-10-13-module-19-special-round-design.md` (§6). Cards: rotasaude/api#52, rotasaude/dashboard#12, #13 e #14.

## Global Constraints

- **Motivos novos de falha de geração:**
  - `regeneration_refused`, com o rótulo "o reenvio da ficha recusada não foi aceito";
  - `still_sending`, com o rótulo "a ficha seguia em envio; gere de novo".
- **Rota nova:** `GET /attendance/consultations/:id/problems`.
  - Resposta: `{ "problems": [<problem>] }`, no formato de `ClinicalRecord::Json.problem`, na ordem de `ClinicalRecord::Json.record` (ativos primeiro, depois código).
  - Só com `ClinicalRecord::Access.for_consultation` de grant `:author` (consulta finalizada e autora logada). Fora disso, 403 com o `forbid` do controller: o motivo do grant, ou `not_author` quando o grant for de contexto ou abertura (a rota é da autora; quem está em contexto já lê o prontuário pela rota do atendimento).
  - Registro: `clinical_record.viewed` com `access: "author"` e `consultation_id`.
- **Marcador:** pendente sem motivo, com as releituras esgotadas, mostra o detalhe "aguardando assinatura" e o link "veja Pendentes" para `#/signature-pending`.
- **Testes:**
  - todo item ganha um teste que falha antes;
  - as suítes do api e do dashboard passam;
  - api com `ROTA_TEST_DB_SUFFIX=_r19` e `docker compose stop worker` antes da suíte completa.
- **Git:**
  - `/opt/homebrew/bin/git`;
  - Conventional Commits em inglês, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`.
- **Worktrees:**
  - api em `apps/api/.claude/r2`, com `master.key` copiada e `exec -w /rails/.claude/r2`;
  - dashboard em `rota-saude/.worktrees/dashboard-r2`, com `npm ci`.

  Os dois no branch `fix/module-19-round-r2`, a partir de `origin/main`.

## Review Focus

1. **Adendo feito em "Minhas consultas" de um paciente sem problemas.**
   - Esperado: o formulário abre normal, com a lista vazia, sem erro.
   - Prova: Task 4.
2. **Rota de problemas chamada por um profissional em contexto que não é a autora.**
   - Esperado: 403 `not_author` e nenhum registro de leitura gravado.
   - Prova: Task 2.
3. **"Gerar de novo" numa falha `still_sending`, depois que a ficha saiu do envio.**
   - Esperado: a ficha é regravada (ou regerada) e a falha fica resolvida.
   - Prova: Task 1.
4. **Duas finalizações seguidas no mesmo atendimento (escuta e consulta) com o painel aberto.**
   - Esperado: o painel relê depois de cada uma, sem perder a consulta aberta.
   - Prova: Task 3.
5. **Consulta antiga, pendente sem motivo, aberta sem finalizar agora** (as releituras não foram armadas).
   - Esperado: mostra "aguardando assinatura" de cara, não "assinando…".
   - Prova: Task 5.

---

### Task 0: Preparação

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git
$G -C $R/apps/api fetch -q && $G -C $R/apps/api worktree add -q $R/apps/api/.claude/r2 -b fix/module-19-round-r2 origin/main
cp $R/apps/api/config/master.key $R/apps/api/.claude/r2/config/master.key
$G -C $R/apps/dashboard fetch -q && $G -C $R/apps/dashboard worktree add -q $R/.worktrees/dashboard-r2 -b fix/module-19-round-r2 origin/main
(cd $R/.worktrees/dashboard-r2 && npm ci --silent && npx vitest run 2>&1 | tail -3)
docker exec -w /rails/.claude/r2 -e ROTA_TEST_DB_SUFFIX=_r19 api-dev bundle exec rspec spec/services/ledi spec/jobs/ledi spec/requests/consultation_reading_spec.rb 2>&1 | tail -2
```

Expected: verde nos dois. Anote no ledger.

---

### Task 1: api#52 — falha registrada nos dois ramos

**Files:**
- Modify: `app/services/ledi/consultation_ficha.rb` (`refresh!` e um método público `still_sending!(consultation)`), `app/jobs/ledi/consultation_ficha_job.rb` (`retry_on` com bloco), `app/models/ledi_generation_failure.rb` (`REASONS`)
- Test: `spec/services/ledi/consultation_ficha_spec.rb`, `spec/jobs/ledi/consultation_ficha_job_spec.rb`

**Interfaces:**
- Produces:
  - `Ledi::ConsultationFicha.still_sending!(consultation)`, que devolve `:failed`;
  - `LediGenerationFailure::REASONS` passa a incluir `regeneration_refused` e `still_sending`.

- [ ] **Step 1: testes que falham.**

```ruby
# spec/services/ledi/consultation_ficha_spec.rb
it "ficha recusada cujo reenvio não é aceito vira 'não gerada', não some" do
  consultation = finalized!
  described_class.generate(consultation)
  entry = LediOutboxEntry.sole
  entry.reject!([ { "field" => "condutas", "code" => "invalid" } ])
  addendum!(consultation, "conducts" => [ 9 ])
  allow(Ledi::Enqueue).to receive(:call).and_return(nil)

  expect(described_class.refresh!(consultation.reload)).to eq(:failed)
  failure = LediGenerationFailure.unresolved.sole
  expect(failure).to have_attributes(source_type: "Consultation", source_id: consultation.id, reason_codes: [ "regeneration_refused" ])
end

it "falha 'seguia em envio': gerar de novo leva o adendo à ficha e resolve" do
  consultation = finalized!
  described_class.generate(consultation)
  addendum!(consultation, "conducts" => [ 1, 9 ])
  described_class.still_sending!(consultation)
  failure = LediGenerationFailure.unresolved.sole
  expect(failure.reason_codes).to eq([ "still_sending" ])

  described_class.retry!(failure, by: doctor)

  expect(failure.reload).to be_resolved
  expect(ficha_of(LediOutboxEntry.sole).atendimentosIndividuais.sole.condutas).to eq([ 1, 9 ])
end
```

```ruby
# spec/jobs/ledi/consultation_ficha_job_spec.rb
it "depois da 10ª tentativa com a ficha em envio, registra a falha 'seguia em envio'" do
  consultation = finalized_consultation!(unit: unit, doctor: doctor, citizen: verified_citizen!(1))
  allow(Ledi::ConsultationFicha).to receive(:refresh!).and_raise(Ledi::ConsultationFicha::InFlight)
  job = described_class.new(city_slug: city.slug, consultation_id: consultation.id, reason: "addendum")
  job.executions = 9 # a próxima é a 10ª

  job.perform_now

  expect(LediGenerationFailure.unresolved.find_by(source_id: consultation.id).reason_codes).to eq([ "still_sending" ])
end
```

  - Se `retry!` exigir `by:` com outro tipo, use o que a spec existente de `retry!` usa (`grep -n "retry!" spec/services/ledi/consultation_ficha_spec.rb`).
  - O `perform_now` de um job com `executions = 9` passa pelo handler do `retry_on`: confira na suíte que ele não reenfileira. Se o handler precisar de `ActiveJob::Base.queue_adapter` de teste, ele já é o padrão.

  Run: `docker exec -w /rails/.claude/r2 -e ROTA_TEST_DB_SUFFIX=_r19 api-dev bundle exec rspec spec/services/ledi/consultation_ficha_spec.rb spec/jobs/ledi/consultation_ficha_job_spec.rb`
  Expected: FAIL, porque hoje dá `:unusable` sem falha, `still_sending!` não existe e o job não registra nada.

- [ ] **Step 2: implementar.**

```ruby
# refresh!, dentro do case:
when "rejected"
  Ledi::Enqueue.call(ficha, city: city, replaces: latest) ? :regenerated : record_failure!(consultation, [ "regeneration_refused" ])
when "accepted" then correction!(latest, ficha, city)
else record_failure!(consultation, [ "regeneration_refused" ])
end
# e: resolve_failure!(consultation) unless %i[unusable failed].include?(outcome)

# api#52: o job desistiu com a ficha em envio; o adendo ainda não está nela.
def still_sending!(consultation) = record_failure!(consultation, [ "still_sending" ])
```

```ruby
# consultation_ficha_job.rb
retry_on Ledi::ConsultationFicha::InFlight, wait: 1.minute, attempts: 10 do |job, _error|
  job.still_sending!
end

def still_sending!
  city_slug, consultation_id = arguments.first.values_at(:city_slug, :consultation_id)
  with_city(city_slug) do
    consultation = Consultation.find_by(id: consultation_id)
    Ledi::ConsultationFicha.still_sending!(consultation) if consultation
  end
end
```

  - `REASONS` ganha os dois códigos.
  - Confira se `still_sending!` precisa entrar no `private_class_method` ou não: é público.

  Run: o mesmo do Step 1, mais `spec/requests/production_generation_failures_spec.rb`.
  Expected: PASS.

- [ ] **Step 3: commit** (`fix: record a generation failure when an addendum cannot reach the consultation ficha`).

---

### Task 2: dashboard#12 (api) — problemas para a autora

**Files:**
- Modify: `config/routes.rb` (depois de `consultations/:id/addenda`: `get "consultations/:id/problems", to: "consultations#problems"`), `app/controllers/consultations_controller.rb`
- Test: `spec/requests/consultation_reading_spec.rb`

**Interfaces:**
- Produces: `GET /attendance/consultations/:id/problems` → `{ problems: [...] }` (Global Constraints).

- [ ] **Step 1: testes que falham.** No padrão do exemplo `"a autora finaliza … com trilha author"` (linha 26):
  - **Autora:** com o paciente tendo um problema antigo (de outra consulta) e o da consulta, recebe 200 com os dois, ativos primeiro. O evento `clinical_record.viewed` tem `access: "author"` e `consultation_id`.
  - **Outra profissional em contexto** no atendimento do paciente: 403 `not_author`, e nenhum `clinical_record.viewed` novo.
  - **Consulta em rascunho**, mesmo sendo da autora: 403.
  - **Sem `health_professional`:** 403.

  Run: `docker exec -w /rails/.claude/r2 -e ROTA_TEST_DB_SUFFIX=_r19 api-dev bundle exec rspec spec/requests/consultation_reading_spec.rb`
  Expected: FAIL (404, rota inexistente).

- [ ] **Step 2: implementar.**

```ruby
# Adendo da própria consulta (dashboard#12; decisão do usuário 2026-10-13): só a
# lista de problemas do paciente, só para a autora, com trilha author.
def problems
  return forbid("not_author") if @consultation.draft?

  grant = read_grant
  return forbid(grant.reason.to_s) unless grant.allowed?
  return forbid("not_author") unless grant.kind == :author

  patient = @consultation.patient
  ClinicalRecord::Trail.viewed!(patient: patient, user: Current.user, grant: grant, consultation_id: @consultation.id)
  problems = patient.problems.order(Arel.sql("status = 'active' DESC"), :code).map { |p| ClinicalRecord::Json.problem(p) }
  render json: { problems: problems }
end
```

  `problems` não pode entrar no `set_consultation` excluído: o `before_action :set_consultation` já cobre, porque `except` lista só `options create mine`.

  Run: o mesmo.
  Expected: PASS.

- [ ] **Step 3: commit** (`feat: let the author read the patient's problem list for an addendum`).

---

### Task 3: dashboard#13 — painel relê depois de finalizar ou de adendo

**Files:**
- Modify: `src/modules/consultation/ConsultationWorkspace.tsx`
- Test: `src/modules/consultation/ConsultationWorkspace.test.tsx`

- [ ] **Step 1: teste que falha.** Use os stubs de API do arquivo (`vi.mock("../../lib/api")` ou o padrão local):
  1. `getAttendanceRecord` devolve primeiro um prontuário sem problemas e, na segunda chamada, um com `K86` ativo;
  2. finalize a consulta pelo editor (como os testes existentes de finalizar);
  3. confira que `getAttendanceRecord` foi chamado 2 vezes e que o painel mostra `K86` sem recarregar.

  Faça o mesmo para o adendo: `onAddendumAdded` dispara a segunda leitura.

  Run: `npx vitest run src/modules/consultation/ConsultationWorkspace.test.tsx`
  Expected: FAIL (1 chamada só).

- [ ] **Step 2: implementar.**

```tsx
const queryClient = useQueryClient();
const rereadRecord = () => void queryClient.invalidateQueries({ queryKey: [ RECORD_KEY, props.attendanceId ] });
// onFinalized: (c) => { setConsultation(c); restartRereads(); rereadRecord(); props.onFinalized(); }
// onAddendumAdded: (addendum) => { setConsultation(...); restartRereads(); rereadRecord(); }
```

  Run: `npx vitest run src/modules/consultation`
  Expected: PASS.

- [ ] **Step 3: commit** (`fix: refresh the patient panel after finalizing a consultation or adding an addendum`).

---

### Task 4: dashboard#12 (front) — lista completa no adendo de "Minhas consultas"

**Files:**
- Modify: `src/lib/api.ts` (`getConsultationProblems(id): Promise<PatientProblem[]>`, que lê `GET /attendance/consultations/:id/problems` e devolve `.problems`), `src/modules/consultation/ConsultationView.tsx` (prop opcional `loadPatientProblems?: () => Promise<PatientProblem[]>`, repassada ao `AddendumForm`), `src/modules/consultation/AddendumForm.tsx`, `src/modules/MyConsultations.tsx`
- Test: `src/modules/consultation/AddendumForm.test.tsx`, `src/modules/MyConsultations.test.tsx`, `src/lib/api.consultation.test.ts`

**Interfaces:**
- Produces:
  - `getConsultationProblems(id: string): Promise<PatientProblem[]>`;
  - `AddendumForm` e `ConsultationView` com a prop `loadPatientProblems?: () => Promise<PatientProblem[]>`.

- [ ] **Step 1: testes que falham.**
  - **`api.consultation.test.ts`:** `getConsultationProblems("c1")` chama `GET /attendance/consultations/c1/problems` e devolve o array.
  - **`AddendumForm.test.tsx`, com `loadPatientProblems`:**
    - o formulário mostra um problema antigo (`L03`, de outra consulta) que só veio da função;
    - sem a função, só os da consulta, como hoje;
    - se a função rejeitar, o formulário continua com os da consulta e mostra o aviso "não foi possível carregar a lista completa de problemas".
  - **`MyConsultations.test.tsx`:** abrir uma consulta e clicar em "Acrescentar adendo" chama `getConsultationProblems` com o id. Só abrir a consulta não chama.
  - **Paciente sem problemas:** a função devolve `[]`; o formulário abre sem erro, com a lista da consulta (Review Focus 1).

  Run: `npx vitest run src/modules/consultation/AddendumForm.test.tsx src/modules/MyConsultations.test.tsx src/lib/api.consultation.test.ts`
  Expected: FAIL.

- [ ] **Step 2: implementar.**
  - **`AddendumForm`:**

    ```tsx
    useQuery({ queryKey: [ "consultationProblems", consultation.id ], queryFn: loadPatientProblems!, enabled: !!loadPatientProblems, gcTime: 0, staleTime: 0 })
    ```

    A base do `ProblemsEditor` vira `full.data ?? patientProblems`. Com `full.isError`, mostra `<p role="status">não foi possível carregar a lista completa de problemas; seguem os desta consulta</p>`.
  - **`ConsultationView`:** repassa a prop.
  - **`MyConsultations`:**

    ```tsx
    <ConsultationLoader … patientProblems={[]} loadPatientProblems={() => getConsultationProblems(viewing)} />
    ```

    A função só é chamada quando o `AddendumForm` monta, isto é, ao clicar em "Acrescentar adendo".

  Run: `npx vitest run src/modules src/lib`
  Expected: PASS.

- [ ] **Step 3: commit** (`feat: show the patient's full problem list in an addendum from My consultations`).

---

### Task 5: dashboard#14 — "aguardando assinatura" depois das releituras

**Files:**
- Modify: `src/lib/signature.ts` (`signatureMarker(block, opts?: { settled?: boolean })` e o tipo `Marker`, que ganha `link?: { href: string; label: string }`), `src/modules/signature/SignatureMarker.tsx` (prop `settled?: boolean`; renderiza `marker.link` como `<a href>`), `src/modules/consultation/ConsultationView.tsx` (prop `signatureSettled?: boolean`, repassada aos dois `SignatureMarker`), `src/modules/consultation/ConsultationWorkspace.tsx`
- Test: `src/lib/signature.test.ts`, `src/modules/consultation/ConsultationWorkspace.signature.test.tsx`

**Interfaces:**
- Produces:
  - `signatureMarker(block, { settled: true })` para `pending` sem motivo devolve `{ label: "assinatura pendente", tone: "warn", detail: "aguardando assinatura", link: { href: "#/signature-pending", label: "veja Pendentes" } }`;
  - sem `settled`, segue "assinando…".

- [ ] **Step 1: testes que falham.**
  - **`signature.test.ts`:** o caso acima, e o caso com motivo, que não muda.
  - **`ConsultationWorkspace.signature.test.tsx`**, no padrão do `"para depois de 3 tentativas, mesmo sem assentar"`: depois da 3ª releitura sem motivo, o texto "aguardando assinatura" e o link "veja Pendentes" (`href="#/signature-pending"`) aparecem, e "assinando…" some.
  - **Consulta antiga pendente sem motivo**, aberta sem finalizar (releituras não armadas): mostra "aguardando assinatura" de cara (Review Focus 5).

  Run: `npx vitest run src/lib/signature.test.ts src/modules/consultation/ConsultationWorkspace.signature.test.tsx`
  Expected: FAIL.

- [ ] **Step 2: implementar.**
  - **Workspace:** `const signatureSettled = !settling || rereads.current >= SIGNATURE_REREAD_LIMIT;`. O `rereadTick` já faz re-render a cada releitura. Passe `signatureSettled` ao `ConsultationView`.
  - **`ConsultationView` e `ConsultationLoader`:** o padrão é `true` quando a prop não vem, porque fora do atendimento ninguém relê. Assim "Minhas consultas" e a abertura justificada já mostram "aguardando assinatura".
  - **`signatureMarker`:** o ramo `pending` sem `reason_code` usa `opts?.settled ? { detail: "aguardando assinatura", link } : { detail: "assinando…" }`.
  - **`SignatureMarker`:** `{marker.link && <a href={marker.link.href} style={linkButton}>{marker.link.label}</a>}`.

  Run: `npx vitest run 2>&1 | tail -3 && npx tsc --noEmit`
  Expected: verde.

- [ ] **Step 3: commit** (`fix: tell the professional a signature is waiting once the rereads give up`).

---

### Task 6: Rótulos, prova, revisão e publicação

- [ ] **Rótulos.** `src/lib/production.ts` → `GENERATION_REASON` ganha `regeneration_refused` e `still_sending`, com os rótulos dos Global Constraints, e um teste em `src/lib/production.test.ts`. Commit no dashboard: `feat: label the new LEDI generation failure reasons`.
- [ ] **Suítes completas:**
  - api: `docker compose stop worker`; `docker exec -w /rails/.claude/r2 -e ROTA_TEST_DB_SUFFIX=_r19 api-dev bundle exec rspec`; `docker compose start worker`;
  - dashboard: `npx vitest run`, `npx tsc --noEmit` e `npm run build`.
- [ ] **Prova no navegador** (dev, Curitiba): finalizar uma consulta com problema novo e ver o painel atualizar. Em "Minhas consultas", abrir o adendo e ver um problema antigo. Sem sessão de assinatura, acrescentar um adendo e ver "aguardando assinatura — veja Pendentes" depois de cerca de 9 s. Capturas em `evidencias/2026-10-13-rodada-19/r2/`.
- [ ] **Revisão final** por revisor independente, no modelo mais capaz, com o Review Focus. Critical e Important são corrigidos com TDD.
- [ ] **Aprovação do usuário.**
- [ ] **Publicação.**
  - avise as sessões do api e do dashboard;
  - `merge --ff-only` e `push` do api, depois do dashboard (o dashboard chama a rota nova), e do docs;
  - confira as CIs;
  - remova os worktrees;
  - mova api#52, dashboard#12, #13 e #14 para "Resolvido", com comentário de evidência, e feche as issues.
