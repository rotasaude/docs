# Rollout — acolhimento (escuta inicial) (módulo 18)

Runbook do operador para publicar o módulo 18 (ADR 0030), ligar o acolhimento
numa cidade e fazer as fichas da escuta saírem para o e-SUS PEC. Base da
exportação: [`rollout-modo-de-prontuario.md`](rollout-modo-de-prontuario.md).

## 1. Publicação

Ordem obrigatória:

1. `contracts` com a tag `protocols-v1.6.0` (variante `kind: "screening"`; a
   raiz escolhe a forma por `if`/`then`/`else` sobre `kind`).
2. `api` (`eb732e8` ou posterior). Por cidade, antes de cortar o tráfego:
   1. publique a imagem nova;
   2. pare o worker (`Ledi::DeliverJob` não pode gravar na fila no meio da
      conversão);
   3. **faça backup do banco da cidade** — a `20261007300002` é irreversível;
   4. rode `bin/rails city:migrate:all` **da imagem nova**:
      - `20261007300001` cria `screenings`, `screening_revisions`,
        `ledi_generation_failures`, `health_units.screening_scope` (padrão
        `walk_in`), os desfechos `scheduled_from_screening` e `oriented`, o
        pedido `kind = screening` e a exceção na trava do módulo 13;
      - `20261007300002` converte `ledi_outbox.last_error` em
        `last_error_codes` e **apaga o texto**. Cria também
        `last_attempted_at` e `replaces_outbox_id`;
   5. religue o worker.

   Nunca migre uma cidade fora do rake: ela fica em 503. A guarda da fila
   tolera a janela entre a 300001 e a 300002.
3. `dashboard` (`4bf895b` ou posterior), **logo depois** do `api`. O formato de
   `GET /production` mudou (`last_error_codes`, `rejections` por campo e
   código, `replaces_outbox_id`). O dashboard antigo não quebra, mas mostra "—"
   no lugar dos erros.

Jobs novos no `recurring.yml`:

- `Ledi::PurgeStalePayloadsJob`, diário às 5h15, apaga o `payload` de fichas
  `rejected`/`failed` com a última tentativa há mais de 90 dias;
- `Ledi::ScreeningFichaSweepJob`, a cada hora, só age às 23h no fuso da cidade.

Publicar não muda o atendimento de nenhuma cidade. Toda unidade fica em
`walk_in`. Sem protocolo `acolhimento` ativo, não há cor sugerida e a fila do
profissional segue a ordem do módulo 13. Fichas só saem onde a exportação já
está ligada (§3).

## 2. Ligar o acolhimento numa cidade

1. **Terminologia CIAP-2 (plataforma, uma vez por versão).** A queixa exige
   CIAP-2. Sem release ativa, a busca responde `503 terminology_unavailable`:
   ```bash
   bin/rails 'terminology:import[ciap2,<versão>,<zip ou pasta>]'
   ```
   A lista de 32 códigos da semente (`db/seeds/terminology/ciap2`) é só de dev.
2. **Protocolo de acolhimento.** No Editor de protocolo, o autor escolhe "Novo
   acolhimento (modelo)". O nome é o reservado `acolhimento`, com
   `kind: "screening"`. O autor ajusta as regras de cor e confere no simulador.
   Depois vêm as duas assinaturas e a ativação (ADR 0016). Só uma versão fica
   ativa.
   **As regras do modelo são ponto de partida e não têm revisão clínica.** A
   cidade revisa e assina as dela antes de usar.
3. **Escopo da unidade.** Em Unidades, o `municipal_admin` escolhe `walk_in`
   (só demanda espontânea, o padrão) ou `all`.
4. **Quem faz a escuta.** Precisa de vínculo ativo na unidade (módulo 10) com
   CBO permitido (`config/scheduling/screening_cbos.yml`):
   - nível superior da equipe (médico, enfermeiro e os demais grupos da lista);
   - técnico ou auxiliar de enfermagem, por exemplo o técnico CBO `322205`.

   CBO fora da lista recebe `403 cbo_not_allowed`.

A recepção vê só a cor, o destino, a espera e o marcador "aguardando
acolhimento". Ela nunca vê queixa nem sinais vitais.

## 3. Fichas da escuta (e-SUS PEC)

A ficha nasce no fechamento do atendimento, com a última revisão da escuta:

- nível superior gera o Atendimento Individual de escuta inicial
  (`tipoAtendimento 4`);
- técnico (`3222xx`) gera a Ficha de Procedimentos com as aferições, ou só com
  `statusEscutaInicialOrientacao = true` quando não houve aferição.

Para a ficha sair, a cidade precisa de:

- modo de prontuário `integrated` ou `record`, credencial do PEC e `ledi_export`
  ligado ([`rollout-modo-de-prontuario.md`](rollout-modo-de-prontuario.md) §4);
- CNES na unidade;
- o profissional numa equipe com INE (CNES confirmado) e com CNS;
- o cidadão com nascimento e sexo no perfil.

Com a exportação inutilizável, nada nasce e o varredor tenta depois (abaixo).
Sem CNES, equipe, CNS ou perfil, a ficha não nasce e entra em e-SUS → Produção e-SUS →
"Fichas que não puderam ser geradas", com o motivo:
`unit_without_cnes`, `professional_without_team`, `professional_without_cns`,
`citizen_without_birth_date`, `citizen_without_sex` ou `unknown_ciap2`.

Depois de corrigir o cadastro, use "Gerar de novo" (step-up). Ficha recusada
pelo PEC e corrigida na origem é regerada por "Reenviar". A ficha nova recebe
outro `uuid` e aponta para a recusada (`replaces_outbox_id`).

**Varredor das 23h.** Ele gera a ficha de duas situações:

- escutas concluídas de atendimentos ainda abertos;
- atendimentos já fechados que ficaram sem ficha e sem "não gerada", porque a
  exportação estava inutilizável no fechamento.

O segundo caso só alcança escutas da competência atual e da anterior. Se a
exportação continua inutilizável, ele tenta de novo na noite seguinte.

**Modo `integrated`:** a escuta registrada no Rota Saúde já vai ao PEC. Oriente
a unidade a **não** registrar a mesma escuta também no PEC, para não contar
duas vezes.

## 4. Gates e pendências antes de uma cidade real

| Item | Bloqueia? |
|---|---|
| rotasaude/api#48 — confirmar os SIGTAP `0214010015` (glicemia capilar) e `0301100039` (pressão arterial) na competência vigente, e o aceite da Ficha de Procedimentos só com a marca num PEC real | **Sim** (Alta, antes do go-live da exportação) |
| rotasaude/api#41 — aceite (2xx) num PEC com CNES real (módulo 16) | **Sim** |
| Regras de cor revisadas e assinadas pela cidade (o modelo não tem revisão clínica) | **Sim** |
| CIAP-2 importado do arquivo oficial (os títulos da semente vieram de memória; só valem em dev) | **Sim** |
| rotasaude/api#49 — o Analytics não conta `scheduled_from_screening` nem `oriented` | Não |
| rotasaude/api#50 — a escuta continua apontando para o pedido `moved` depois de esvaziar a unidade | Não |

## 5. Acompanhar

- **Fila do acolhimento** (Atendimento → Acolhimento): quem precisa de escuta
  pelo escopo, por chegada.
- **Fila do profissional:** primeiro quem tem escuta "no dia", por cor
  (vermelho no topo) e chegada; depois quem não precisa de escuta. Por último,
  marcado, quem ainda aguarda acolhimento. Essa última faixa só existe com
  protocolo ativo.
- **Produção e-SUS:** fichas com códigos de erro (`campo · código`), "não
  geradas" e reenvio.
- **Trilha:** abrir a escuta de alguém (detalhe ou chamada) publica
  `screening.viewed`.
