# Módulo 24a — Cadastro individual e domiciliar — design

**Data:** 2026-10-23
**Status:** aprovado (2026-10-23)
**Afeta:**
- `apps/api`: cidade — `micro_areas`, `households`, `families`, `individual_registrations` e as versões imutáveis de cadastro; paciente sem CPF (justificativa, CNS); fichas LEDI de cadastro individual e domiciliar; equipe da pessoa pelo cadastro; óbito pela saída do cadastro; interruptor `territory_registry`; ACS e agente de endemias sem conselho (módulo 10).
- `apps/dashboard`: microáreas da equipe; domicílios, famílias e moradores; ficha de cadastro individual; possíveis duplicados; validade de 24 meses e quem falta cadastrar.
- `apps/maintenance`: rótulo do interruptor (mecanismo genérico).

**ADR:** `docs/adr/0042.md` · **Módulo:** `docs/modulos/24--domiciliar-e-coletivo.md` · **Pesquisa:** `docs/pesquisa/2026-10-05-integracoes/frente-1-aps-sisab.md` (§2 fichas, §5 componente de Vínculo, §7 identificação) · **Divisão:** ADR 0041

**Fora desta entrega:** visita domiciliar e atividade coletiva (24b); app de campo offline (24c); cadastro pelo cidadão no wpda; junção de dois pacientes da mesma pessoa; georreferenciamento do imóvel (vem com o 24c); trabalho do agente de endemias (35).

## 1. Intenção

Toda pessoa do território da equipe tem cadastro individual e domiciliar válido no SIAPS, porque o componente de Vínculo do repasse federal depende disso: só conta o cadastro individual atualizado nos últimos 24 meses, e o fator sobe de 0,75 para 1,5 com o cadastro domiciliar também atualizado (NT 30/2025). O cadastro rápido do módulo Cidadão não conta.

**Sucesso:** a equipe sabe, por microárea, quem tem cadastro válido, quem vence logo e quem falta cadastrar; o cadastro feito na web vira ficha LEDI aceita pelo SIAPS; a pessoa passa a pertencer à equipe da microárea onde mora.

**Antiobjetivos:** duas fichas da mesma pessoa (duplicado); cadastro de alguém que recusou sem o termo de recusa; condição de saúde ou situação de rua legível no banco, no log ou na URL; exigir CPF de quem não tem.

## 2. Decisões

| # | Decisão |
|---|---|
| 1 | **Paciente sem CPF** (escolha do usuário, opção A): o paciente — a pessoa da cidade — pode existir sem CPF, com a justificativa da ficha (recém-nascido, situação de rua, outra) e CNS quando houver. O cidadão do wpda (par CPF + telefone) não muda: o CPF continua obrigatório lá. |
| 2 | Ao informar o CPF de um paciente sem CPF, o sistema recusa se já existir paciente com esse CPF (`cpf_taken`). Antes de criar paciente sem CPF, mostra possíveis duplicados por nome, nascimento e nome da mãe. A junção de pacientes é pendência à parte. |
| 3 | **Microárea:** cada equipe (tipos 70/76) tem microáreas com código de 2 dígitos e o ACS responsável. O administrador municipal cria e altera, com segundo fator. |
| 4 | **Domicílio e família:** o domicílio é o imóvel (endereço, tipo, microárea, condições de moradia); dentro dele, uma ou mais famílias, cada uma com responsável, renda e número de membros. Gera a ficha de cadastro domiciliar e territorial. |
| 5 | **Cadastro individual** no paciente: dados sociodemográficos, condições de saúde, situação de rua, família e responsável. Gera a ficha de cadastro individual. |
| 6 | **Recusa** do cidadão registrada com o termo (campo da própria ficha). **Saída** do cadastro com motivo: mudança de território ou óbito. |
| 7 | **Validade de 24 meses:** lista por equipe e microárea com cadastros vencidos, a vencer em 60 dias e pessoas da equipe sem cadastro. |
| 8 | **ACS (CBO 515105) e agente de endemias (515140)** entram no cadastro de profissionais sem registro em conselho (exceção no módulo 10). |
| 9 | **Quem cadastra:** o ACS da microárea e o enfermeiro ou médico da equipe. Lista e leitura nominal gravam trilha. |
| 10 | Interruptor `territory_registry`, que requer `clinical_record` (só modo registro, onde as fichas vão para o SIAPS). |
| 11 | O cadastro é retido na exclusão LGPD, como os demais registros enviados ao SIAPS; recusa, condições de saúde, situação de rua e condições de moradia ficam cifradas. |
| 12 | wpda fora: o cidadão não vê nem edita o cadastro pelo celular. |
| 13 | **O cadastro cria o paciente direto** (escolha do usuário, opção A): o ACS ou o enfermeiro digitam nome, nascimento, sexo e nome da mãe, sem validação no posto. Quando a mesma pessoa valida o CPF no posto depois, os dados validados com documento substituem os digitados (o `refresh!` do `Patients::Resolve` continua valendo) e o cadastro mostra o que mudou, gerando nova versão e nova ficha. |

Decisões 1, 2 e 13 (paciente sem CPF; cadastro cria o paciente) são do usuário; as demais são do coordenador, apresentadas e aprovadas.

**Decisões do coordenador (desenho):**
- **Versões imutáveis.** Cada salvamento do cadastro individual ou domiciliar grava uma versão imutável (trigger em `db/city_triggers.sql`, com a exceção `rota.reencrypting`) e enfileira uma ficha no `ledi_outbox`; a ficha de atualização aponta a originadora como o LEDI pede. O registro "atual" é a última versão.
- **Uma fonte por dado.** Raça/cor, etnia e endereço continuam no paciente (módulo 22); o cadastro individual lê e grava esses campos no paciente, não cópia.
- **Equipe da pessoa** (muda a regra do 23, P18): cadastro individual ativo → equipe da microárea do domicílio; sem cadastro, as regras de hoje (inscrição, consulta recente com equipe, bairro). Saída por mudança tira a pessoa da equipe; saída por óbito vira a mesma marca de óbito do 23 (`Facts.deceased`), somada ao encerramento de inscrição por `death`.
- **Busca de duplicados** por nome normalizado, nascimento e nome da mãe, com dados cifrados: blind index (HMAC) de nome normalizado + nascimento, nunca texto claro.
- **CPF e condições só no corpo** das requisições (padrão P20 do 23).
- **Primeira task do api é prova técnica:** ler as regras oficiais das fichas de cadastro individual e domiciliar (documentação LEDI 8.7, `/ledi/documentacao/regras/`), conferir os campos obrigatórios e os domínios contra o IDL já vendorizado (`vendor/ledi/8.7.0/idl/ras/cadastro_*.thrift`) e fixar os códigos de tipo de ficha.

**Decisões do coordenador (revisão da spec):**
- **Endereço:** o domicílio é a fonte do endereço dos moradores com cadastro ativo. Salvar o domicílio atualiza o endereço desses pacientes; no módulo 22, o endereço de paciente com cadastro ativo aparece só para leitura ("do cadastro domiciliar"). Não gera fichas individuais em cascata (a ficha individual não leva endereço).
- **Saída por mudança:** tira o cadastro como primeira fonte da equipe; as regras antigas (consulta recente com equipe, bairro) continuam valendo. Cadastro gravado define a equipe mesmo antes da resposta do SIAPS.
- **Sem rascunho:** nem no servidor nem no navegador (dado de saúde não vai para o `localStorage`); cada salvamento é versão e ficha. O formulário só guarda o que está na tela enquanto aberto.
- **Família e versões:** troca de responsável, renda, número de membros, entrada ou saída de família geram nova versão do domicílio e nova ficha domiciliar. Morador que muda só o próprio cadastro individual gera só ficha individual.
- **"Enviado" para a exclusão LGPD:** ficha enfileirada no `ledi_outbox` (uma vez na fila, sai). Paciente sem CPF não tem par no wpda; o pedido de exclusão dele é pelo fluxo no posto (ADR 0026), adaptado para paciente — e, com cadastro enviado, ele é retido de qualquer forma.
- **Quem falta cadastrar:** a população da equipe do 23 (`TeamPopulation`), pacientes e pares validados; o cadastro de um par validado sem paciente cria o paciente por `Patients::Resolve`.
- **Cabeçalho da ficha:** CNS e CBO de quem salvou, INE da equipe e microárea do domicílio. CNS continua obrigatório para todo profissional que cadastra (a exceção do módulo 10 é só do conselho); ACS sem CNS não pode ser responsável por microárea.
- **Paciente sem CPF nos outros envios:** ficha de atendimento e BPA-I usam o CNS quando não há CPF; sem CPF e sem CNS, o envio cai no caminho de falha que já existe (`ledi_generation_failures` e pendência do faturamento), nunca com CPF inventado.

## 3. Interruptor

`territory_registry` (requer `clinical_record`). Desligado: nenhuma tela, rota de cadastro ou ficha nova; o que já foi gravado fica. A regra da equipe pelo cadastro só vale com o interruptor ligado.

## 4. Dados

Cidade:
- `micro_areas`: equipe, código (01–99, único por equipe), nome, ACS responsável (membro da equipe com CBO 515105), ativa.
- `households` + versões: microárea, endereço (CEP, logradouro, número, complemento, bairro do 11, ponto de referência), tipo de imóvel, condições de moradia (cifradas), telefone de contato (cifrado), recusa, situação (ativo, inativo).
- `families`: domicílio, responsável (paciente), renda familiar (faixa da ficha), número de membros, reside desde, mudou-se.
- `individual_registrations` + versões: paciente (um cadastro ativo por paciente), família, é responsável, dados sociodemográficos e condições de saúde (cifrados), situação de rua (cifrada), recusa, saída (motivo, data).
- `patients`: CPF passa a ser opcional, com `no_cpf_reason` e `cns` (cifrado); índice cego para duplicados.

Ficha LEDI: tipos novos no `FichaTypes` (cadastro individual e domiciliar), carregando os tipos Thrift já gerados.

## 5. Telas (dashboard)

- **Microáreas** (administrador): lista por equipe, criar, renomear, trocar ACS, desativar.
- **Território da equipe:** microáreas → domicílios → famílias → moradores, com situação do cadastro de cada um.
- **Ficha de cadastro individual** e **ficha de domicílio**, em seções, com validação igual à do api; aviso de possíveis duplicados antes de criar paciente sem CPF.
- **Validade:** vencidos, a vencer em 60 dias e sem cadastro, por microárea, com atalho para atualizar.

## 6. Segurança e LGPD

Trilha `clinical_record.viewed` em leitura nominal (sem polling); contagens sem trilha. Nada pessoal em `inspect`, `to_s`, `pretty_print`, `as_json` ou log. Ordem de travas estável (paciente → domicílio → família → cadastro). Exclusão LGPD: retém quem tem cadastro enfileirado ou enviado. Eventos novos de `Platform.audit` declarados em R18.

## 7. Testes

- api: modelo e trigger de imutabilidade; regras da ficha (obrigatórios, domínios) contra a prova técnica; ficha Thrift gerada conferida campo a campo com exemplo de referência; duplicados e `cpf_taken`; equipe pelo cadastro (com a regra do 23); óbito; exclusão LGPD; ordem de travas; escopo por equipe; nada na URL.
- dashboard: telas com testes de componente; validação espelhada; nenhum código cru.
- Suíte completa do api no fim (migração de cidade e mudança em `patients`).

## 8. Entrega

api → dashboard → maintenance; migração de cidade irreversível (CPF opcional no paciente); runbook `operacao/rollout-cadastro-territorial.md`.

## 9. F-IDs (propostos)

| ID | Funcionalidade | Superfície |
|---|---|---|
| F-24.1 | Interruptor e microáreas | api, dashboard, maintenance |
| F-24.2 | ACS e agente de endemias sem conselho | api, dashboard |
| F-24.3 | Paciente sem CPF e possíveis duplicados | api, dashboard |
| F-24.4 | Cadastro domiciliar e família | api, dashboard |
| F-24.5 | Cadastro individual, recusa e saída | api, dashboard |
| F-24.6 | Validade de 24 meses e quem falta cadastrar | api, dashboard |
| F-24.7 | Fichas LEDI de cadastro | api |
| F-24.8 | Equipe e óbito pelo cadastro | api |

## 10. Riscos

- Regras de validação das fichas ainda não lidas (mitigação: prova técnica na Task 0).
- Paciente sem CPF toca buscas por CPF dos módulos 17, 19, 22 e 23 (mitigação: inventário dos leitores de `patients.cpf` no plano do api).
- Duplicados entre paciente sem CPF e paciente com CPF (mitigação: aviso na criação, `cpf_taken`, pendência de junção).
- Ficha extensa: cansaço no preenchimento web (mitigação: seções; o app de campo vem no 24c).
