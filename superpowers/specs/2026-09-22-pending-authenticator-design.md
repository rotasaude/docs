# Troca de autenticador com segredo pendente — design

**Data:** 2026-09-22
**Status:** aprovado em conversa (2026-09-22), aguardando revisão do texto
**Afeta:** `apps/api` (banco de cada cidade, `MfaController`, `User`) e `apps/dashboard` (textos e mensagens da página Segurança).

**Origem:** a revisão final da fatia 1 das telas de assinatura do dashboard (`2026-09-22-dashboard-protocol-signatures-design.md`, §4.2) achou que trocar o autenticador desliga o antigo antes de o novo ser confirmado. Isso foi registrado como bloqueio antes da fatia 2 e do go-live.

## 1. O problema

`Mfa::Enroll` grava `otp_secret`, substitui `otp_recovery_codes` e põe `otp_enabled = false` na mesma chamada. Como `User#mfa_enrolled?` é `otp_enabled? && otp_secret.present?`, a conta passa a contar como **não cadastrada** no instante em que a troca começa. Daí em diante:

- o autenticador antigo e os códigos de recuperação antigos deixam de valer imediatamente;
- o estado não tem prazo: quem fecha a aba no meio da troca fica sem segundo fator;
- e, nesse estado, `POST /mfa/enroll` não exige step-up (a guarda só vale para conta cadastrada), então **quem tiver apenas a senha** cadastra o próprio autenticador e passa a aprovar ações sensíveis.

O login da cidade é só senha (`SessionsController#create`), então "ter a senha" é o mesmo que "ter uma sessão". A fatia 1 tornou essa troca um caminho normal da interface, o que transforma um canto teórico em rotina.

Sobrou também da fatia 1 o **replay do TOTP** do usuário da cidade: `Mfa::Verify::DRIFT` mantém três passos válidos ao mesmo tempo, e o mesmo código serve mais de uma vez. O mantenedor já se protege disso (`Maintainer#consume_totp!`); o usuário da cidade, não.

## 2. Decisões

1. **O spec cobre os dois:** segredo pendente e replay. É a mesma tabela, a mesma migração e o mesmo trecho de controller.
2. **O pendente vale 15 minutos.** Depois disso, a confirmação responde "cadastro expirado" e a pessoa começa de novo.
3. **Um caminho só:** `enroll` propõe, `confirm` efetiva — inclusive no primeiro cadastro. `enroll` nunca toca no segredo ativo.
4. **Serviço novo, só para o usuário da cidade:** `Mfa::PendingEnrollment.start` / `.confirm`. `Mfa::Enroll` fica como está, atendendo o convite do mantenedor e o operador, que não têm este problema (lá o cadastro nasce de um convite, e o reenvio zerar tudo é o caminho de recuperação).

   Descartadas: uma bandeira `pending:` no `Mfa::Enroll` (quem chamasse sem ela continuaria sobrescrevendo o segredo ativo — o erro que estamos consertando, deixado à mão de quem chama); e mover mantenedor e operador para o pendente (mexe em convite e recuperação que funcionam, sem ganho).
5. **Um pendente por conta.** Um cadastro novo substitui o anterior.
6. **Os códigos de recuperação pendentes entram com hash de bcrypt**, como os ativos, e não são cifrados por cima.

## 3. Dados

Uma migração em `db/city_migrate` acrescenta quatro colunas a `users`, no banco de cada cidade:

| coluna | tipo | para quê |
|---|---|---|
| `otp_pending_secret` | string, com `encrypts` como `otp_secret` | o segredo proposto, ainda não confirmado |
| `otp_pending_recovery_codes` | jsonb, default `[]`, não nulo | os códigos do cadastro proposto, com hash |
| `otp_pending_at` | datetime | quando o cadastro começou, para o prazo de 15 min |
| `last_otp_step` | integer | o passo de 30 s do último código aceito |

- `otp_secret`, `otp_recovery_codes` e `otp_enabled` **não mudam**. Enquanto houver pendente, os dois convivem: o ativo continua valendo, o proposto espera.
- Nenhuma linha existente é tocada: as colunas nascem nulas ou vazias, e conta já cadastrada segue funcionando.
- `db/city_schema.rb` é regerado. A spec `spec/services/city_schema_spec.rb` ("produces from the migrations exactly the schema of db/city_schema.rb") é quem cobre essa paridade.
- **`User#consume_totp!`** copia o desenho de `Maintainer#consume_totp!`: pega o passo com `Mfa::Verify.totp_step_for`, e grava com um `UPDATE` condicional (`last_otp_step IS NULL OR last_otp_step < ?`) cujo retorno 1 **é** o teste. Passo repetido ou mais velho é recusado, e duas requisições simultâneas com o mesmo código não passam as duas.

## 4. Endpoints

### `POST /mfa/enroll`

Continua exigindo step-up quando a conta tem autenticador ativo, e continua limitado a 10 tentativas por 3 minutos. Passa a gravar **só** o pendente (segredo, códigos com hash, `otp_pending_at = Time.current`) e devolve `otpauth_uri` e `recovery_codes` em texto, uma vez. Chamar de novo substitui o pendente.

### `POST /mfa/confirm`

Verifica o código contra o **segredo pendente**, nunca contra o ativo, e só por TOTP (`Mfa::Verify.totp_valid?`), como já decidido na fatia 1: confirmar tem de provar que o autenticador novo foi lido.

| situação | resposta |
|---|---|
| TOTP do pendente confere | `200 { ok: true }`; promove: `otp_secret` e `otp_recovery_codes` viram os pendentes, `otp_enabled = true`, pendente apagado, passo registrado |
| não há pendente | `422 { error: "no_pending_enrollment" }` |
| pendente com mais de 15 min | `422 { error: "enrollment_expired" }`, e o pendente é apagado |
| código errado | `422 { error: "invalid_code" }` |
| código já usado (passo repetido ou anterior) | `422 { error: "code_reused" }` |

A promoção é uma transação: ou os cinco campos mudam juntos, ou nenhum muda.

### `POST /mfa/step_up`

Verifica contra o segredo **ativo**, como hoje, e passa a registrar o passo: código repetido responde `422 { error: "code_reused" }`. O código de recuperação continua valendo como alternativa, e continua sendo consumido.

### O que isso garante

- Abandonar a troca não custa nada: o autenticador antigo segue valendo até a confirmação do novo.
- Quem tem só a senha não planta mais um segredo confirmável: numa conta ativa, `enroll` exige step-up, que exige o autenticador atual.
- Um mesmo código não serve duas vezes, nem entre endpoints diferentes.

### O limite que continua

Numa conta que **nunca** cadastrou TOTP, quem tiver a senha cadastra o próprio autenticador: não há fator anterior a pedir. Fechar isso exige cadastro obrigatório no convite, ou aviso por e-mail ao cadastrar. Fica fora deste spec, registrado como pendência de go-live.

## 5. Dashboard

A fatia 1 escreveu textos que este spec torna falsos. Junto com a API mudam:

- **Sai** o aviso "Seu autenticador anterior já não vale — confirme o novo para voltar a aprovar ações." (e o estado `viaReplace` que só existia para ele).
- **Volta** a descrição correta da troca: "O autenticador atual continua valendo até você confirmar o novo."
- `void auth.reload()` depois do enroll da troca deixa de ser necessário (a sessão não muda mais no enroll) e sai.
- **Duas mensagens novas**, mapeadas onde hoje ficam `invalid_code` e afins:
  - `enrollment_expired` → "cadastro expirado — comece de novo";
  - `code_reused` → "código já usado — espere o próximo".
  - `no_pending_enrollment` → a frase genérica que já existe; é um estado que a tela não provoca (ela só confirma o que acabou de propor).

## 6. Estratégia de teste

- **Modelo (`User`):** `consume_totp!` aceita código novo, recusa o repetido e o de passo anterior; duas chamadas simultâneas com o mesmo código, só uma vence.
- **Serviço (`Mfa::PendingEnrollment`):**
  - `start` grava o pendente e não toca em `otp_secret`, `otp_recovery_codes` nem `otp_enabled`;
  - `start` de novo substitui o pendente;
  - `confirm` promove tudo e limpa o pendente, e o código de recuperação pendente só passa a valer depois disso;
  - `confirm` recusa sem pendente, com pendente vencido, com o código do segredo **ativo** e com código repetido.
- **Requisição (`/mfa/*`):** cada linha das tabelas do §4, mais `enroll` numa conta ativa sem step-up (segue `401 mfa_required`) e o rate limit intacto.
- **Esquema:** regerar `db/city_schema.rb`; a spec de paridade que já existe cobre.
- **Dashboard (Vitest):** as duas mensagens novas, o texto corrigido da troca e a ausência do aviso removido.
- **Verificação no navegador**, uma vez, com o stack de dev: cadastrar, trocar sem confirmar (o antigo continua valendo, provado por um step-up), depois confirmar a troca e usar o novo.

## 7. Aplicação nas cidades

- A migração roda com `rails "city:migrate[slug]"`, ou `rails city:migrate:all` para todas. Enquanto uma cidade não migra, o host dela responde `city_schema_behind`, como em toda migração de cidade.
- **Ordem:** migrar antes de publicar o código novo. A migração só acrescenta colunas nulas, e o código atual convive com elas.
- Nada a migrar em dado existente.

## 8. Entrega

Um plano, três tarefas:

1. migração de cidade, `db/city_schema.rb` regerado e `User#consume_totp!`;
2. `Mfa::PendingEnrollment` e o `MfaController` (`enroll`, `confirm`, `step_up`);
3. os textos e as mensagens do dashboard.

## 9. Fora de escopo

- Cadastro obrigatório de TOTP ao aceitar convite, e aviso por e-mail ao cadastrar ou trocar.
- Mantenedor e operador, que seguem com `Mfa::Enroll`.
- Login da cidade com TOTP.
- Vários cadastros pendentes ao mesmo tempo.
