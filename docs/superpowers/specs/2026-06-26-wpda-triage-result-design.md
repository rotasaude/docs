# F-03.17 — Resultado da triagem renderizado no wpda — Design

**Data:** 2026-06-26
**Feature:** F-03.17 (mod-03, board #1). Só `wpda` (frontend).
**Repo:** `rotasaude/wpda` (local `apps/wpda`).

## Contexto

F-04.5 entregou a página pública do relatório no `wpda` (lê `?token`, busca `GET /r/:token`,
renderiza). F-03.17 (superfície `wpda` do mod-03) é o **mesmo destino**: o paciente vê o resultado
da triagem (doc mod-03: "tier, recomendação, sem dado clínico de terceiros"). Não há rota separada —
o único caminho do cidadão é o `/r/:token`.

Ao explorar F-03.17 descobrimos **um bug no F-04.5**: o `summary` do payload **não é uma string** —
o `GenerateReportJob#build_payload` define `summary` como o **trail** (`triage.outcome["trail"].map
{ step:, answer: }`, um array). O `Report.tsx` do F-04.5 renderiza `<p>{r.summary}</p>`, o que com
dado real renderiza `[object Object]` / quebra. O teste do F-04.5 passou por mockar `summary: "ok"`.

Além disso, **`tier` é definido pelo protocolo** (thresholds per-protocolo, ex.: `{baixa:0, media:4,
alta:8}` em `weighted.rb`) — não um enum fixo. Logo um mapa `tier → recomendação` no frontend é
**não-confiável e arriscado** (conselho de saúde errado pra tier desconhecido). Decisão: **não**
inventar recomendação por tier no cliente; isso é follow-up de backend.

## Decisões fechadas

1. **Só frontend (`wpda`)**, sem tocar o motor de protocolo.
2. **Corrigir o tipo/render do `summary`:** é o trail (`Array<{step, answer}>`), não string. **Não
   renderizar** (step-ids crípticos; trail amigável é do dashboard, F-03.16).
3. **`tier` renderizado como vem** (rótulo do protocolo). **Sem** mapa de recomendação por tier no
   frontend.
4. **Nota genérica não-prescritiva** no lugar do summary-string (texto aprovado pelo usuário).
5. **Trail omitido** da página do paciente.

## Mudanças (`apps/wpda/src`)

**`lib/report.ts`** — corrigir o tipo (o `/r/:token` JSON é o mesmo; só o tipo estava errado):
```ts
export interface TrailEntry { step: string; answer: string }
export interface Report {
  tier: string | null;
  priority: string | null;
  summary: TrailEntry[] | null;   // era string — é o trail
  completed_at: string | null;
  expires_at: string | null;
}
```
`tokenFromUrl` e `fetchReport` ficam iguais.

**`modules/Report.tsx`** — no estado `ok`, trocar a renderização do corpo:
- **Remover** `{r.summary && <p>{r.summary}</p>}`.
- **Manter/ajustar:** badge de `tier` ("Classificação: \<tier\>", valor como vem), `priority`
  secundário, `completed_at` ("Realizado em ..."), `expires_at` ("Válido até ...").
- **Adicionar** a nota genérica (corpo principal):
  > Este é o resultado da sua triagem. Siga as orientações da sua unidade de saúde. Se os sintomas
  > piorarem, procure atendimento.
- Manter a nota de privacidade ("não compartilhe este link").
- Estados loading/invalid/error inalterados.

**`lib/report.test.ts`** — atualizar o mock do happy-path: `summary` passa a ser um array
(`[{ step: "s1", answer: "sim" }]`) e o assert verifica `r?.tier`/`r?.summary?.length` em vez de
`summary === "ok"`.

## Data flow / erros

Sem mudança vs F-04.5: `?token` → `fetchReport` → 200 (render) / 404 (inválido/expirado) / throw
(erro). A correção é só **o que** se renderiza no estado `ok`.

## Testes

- `lib/report.test.ts` (atualizado): `fetchReport` em 200 devolve `Report` com `summary` array;
  404 → null; 500 → throw. `tokenFromUrl` inalterado.
- Verificação: `npm run typecheck` + `npm run build` + smoke do 404 (já coberto). Happy-path com
  token real **se** houver `Triage` em dev (mintar snapshot com `summary` = trail real e abrir a
  página → não deve aparecer `[object Object]`).

## Critério de aceite

1. Com payload real (`summary` = array de trail), a página renderiza **sem** `[object Object]` e sem
   quebrar — mostra tier, nota genérica, data.
2. Nenhum conselho de saúde específico por tier é inventado no cliente.
3. Trail/step-ids não aparecem na página do paciente.
4. `npm run test` (report atualizado), typecheck e build verdes.
5. F-04.5 fica consistente (o bug do summary corrigido sob esta fatia).

## Out-of-scope

- Recomendação específica por tier (backend / o protocolo define o texto) — follow-up.
- Render amigável do trail (resolver prompts via definição) — é F-03.16 (dashboard).
- Qualquer outra tela do wpda; mudar o endpoint `/r/:token`.
