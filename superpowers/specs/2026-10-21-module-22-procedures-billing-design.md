# Módulo 22 — Procedimentos e faturamento ambulatorial (BPA) — design

**Data:** 2026-10-21
**Status:** aprovado (2026-10-21)
**Afeta:**
- `apps/api`: cidade — raça/cor, etnia, CEP e endereço em `patients` (cifrados); `performed_procedures`, `billing_periods`, `billing_exports`; papel `billing_clerk`; interruptor `procedure_billing`; críticas SIGTAP; gerador do arquivo de interface do BPA Magnético; fornecedor `Timeline::Sources::Procedures`; origem automática a partir dos laudos liberados do 21b.
- `apps/dashboard`: raça/cor, etnia, CEP e endereço na recepção (check-in e cadastro do paciente) e na pendência do faturista; registro de procedimento (no atendimento, na consulta e avulso), lista de procedimentos com crítica, fechamento da competência (resumo, pendências, geração, reabertura, download).
- `apps/maintenance`: interruptor (mecanismo genérico).
- `contracts`: perfil do arquivo BPA (`billing/bpa-interface.md`) e exemplos — tag `billing-v1.0.0`.

**ADR:** `docs/adr/0039.md` · **Módulo:** `docs/modulos/22--procedimentos.md` · **Pesquisa:** `docs/pesquisa/2026-10-05-integracoes/frente-4-faturamento-regulacao.md` (§2.1, §2.2, §3 camada 1, §4)

**Fora desta entrega:** APAC e OCI (Grupo 09); FPO e teto; conciliação do retorno do SIA (aprovado/glosado); RAAS; RAC/CMD na RNDS; faturamento de prestador credenciado (ele fatura a própria produção); AIH (linha hospitalar).

## 1. Intenção

O faturamento BPA/SIGTAP aparece nos 15 editais analisados e é o que a secretaria precisa para receber pela produção de média complexidade feita nas unidades próprias. Hoje o faturista digita no BPA Magnético o que o sistema já sabe. O módulo registra o procedimento realizado já conferido pela SIGTAP e entrega, no fim do mês, o arquivo de interface que o BPA Magnético importa — como fazem o e-SUS APS, o SISREG e os sistemas privados.

**Sucesso:** o faturista gera o arquivo da competência sem redigitar, vê antes o que não vai passar na consistência do SIA e o arquivo importa no BPA Magnético sem rejeição de layout.

**Antiobjetivos (falha mesmo funcionando):** produção da atenção primária duplicada entre LEDI/SISAB e BPA; atendimento travado por regra de faturamento; arquivo com dado do paciente baixado sem trilha.

## 2. Decisões

| # | Decisão |
|---|---|
| 1 | Escopo da camada 1 da pesquisa: registro com críticas SIGTAP + exportação BPA-I/BPA-C. APAC/OCI, FPO, conciliação e RNDS depois. |
| 2 | O BPA só sai de estabelecimentos que **não são de atenção primária** (`health_units.kind` `upa`, `hospital`, `other`) e do laboratório municipal (21b); unidade `ubs` nunca gera BPA (sua produção vai por LEDI/PEC). Dentro desses estabelecimentos, entra o procedimento cujo instrumento na SIGTAP da competência é BPA-I ou BPA-C. |
| 3 | Origens: procedimento registrado no atendimento/consulta ou avulso pela unidade; exames liberados pelo laboratório municipal (21b) automaticamente; lançamento manual pelo faturista. Prestador credenciado do 21a fica fora. |
| 4 | Crítica SIGTAP marca e não bloqueia: o procedimento fica "não faturável" com motivo. |
| 5 | Papel novo `billing_clerk` (faturista) fecha a competência por estabelecimento (CNES); reabrir exige motivo e gera nova versão do arquivo; download com trilha. |
| 6 | Paciente identificado por CPF no BPA-I; CNS quando não houver CPF (nunca os dois). |
| 7 | Layout lido na Task 0 a partir do PDF vigente do DATASUS; teste byte a byte contra arquivo de referência; importação real no BPA Magnético é gate de go-live. |
| 8 | Dados do paciente que o cadastro não tem: o arquivo usa o que existe, sem captar dados novos — "sem informação" onde o layout aceita (raça/cor, etnia), município de residência = o da cidade, endereço e telefone do cidadão do wpda quando houver; o que o layout não aceita vazio vira pendência no fechamento. |
| 9 | Revisão da decisão 8 após a Task 0 (layout 05.00): raça/cor "sem informação" (99) é recusada desde 2023 e CEP/endereço são criticados desde 2013. O paciente passa a ter **raça/cor, etnia (quando indígena), CEP e endereço**, captados na recepção (check-in e cadastro do paciente, campos opcionais incentivados) e pelo faturista na pendência do fechamento (com trilha); o que faltar no fechamento continua pendência. |

As decisões 2 a 7 foram propostas pelo coordenador e aprovadas pelo usuário; as decisões 2 (revista) e 8 foram escolhidas pelo usuário depois da revisão da spec.

**Decisões do coordenador (revisão da spec):**
- **Crítica recalculada:** `billable`/`blocking_reasons` são calculados na gravação e **recalculados no fechamento** com o cadastro do momento (CNS, CBO, identificação); corrigir o cadastro basta, sem cancelar e relançar. O arquivo gerado congela os dados usados.
- **Estabelecimento:** a competência é por estabelecimento faturante — unidade de saúde não-APS ou laboratório municipal (`exam_providers` com `uses_municipal_lab`), identificado pelo CNES; liberador sem CNS ou sem CBO vira pendência.
- **Cabeçalho:** órgão de destino, indicador de gestão (M/E) e CNPJ de origem são configuração da cidade (admin municipal), sem step-up de maintenance; habilitação/serviço/INE exigidos por alguns procedimentos não são criticados nesta entrega (o SIA continua consistindo).
- **SIGTAP ausente:** sem a release da competência, vale a última release ativa, com aviso; o fechamento recalcula com a release da competência se ela já tiver sido importada.

## 3. Interruptor e papel

- `procedure_billing` (sem pré-requisito de modo; vale nos dois modos de prontuário): liga registro, críticas e fechamento.
- `Membership` `billing_clerk` (privilegiado): lê procedimentos e competências da cidade, lança manualmente, fecha, reabre e baixa arquivos. O `municipal_admin` lê.

## 4. Dados (cidade)

- `performed_procedures`:
  - quem e onde: `patient_id`, estabelecimento (unidade de saúde ou laboratório municipal, com CNES), `professional_user_id`, `cbo_code`, `performed_on`;
  - o quê: `sigtap_code`, `sigtap_competence` (competência da tabela usada na crítica), `quantity` (≥ 1), `cid10` (opcional), `character` (`eletivo` | `urgencia`, padrão eletivo), `authorization_number` (opcional, para o futuro);
  - origem: `source` (`attendance` | `consultation` | `lab_report` | `manual`) e a referência (`attendance_id`, `consultation_id`, `lab_report_item_id`);
  - crítica: `instrument` (`bpa_i` | `bpa_c` | `not_bpa`), `billable` (bool), `blocking_reasons` (lista de códigos: `cbo_not_allowed`, `age_out_of_range`, `sex_not_allowed`, `cid_required`, `cid_not_allowed`, `not_in_sigtap`, `missing_professional_cns`, `missing_professional_cbo`, `missing_patient_id`, `establishment_is_aps`, `missing_required_patient_field`), calculados na gravação e recalculados no fechamento;
  - estado: `status` (`recorded` | `cancelled`), `cancel_reason`, `billing_period_id` (quando incluído num arquivo).
  - só acréscimo para conteúdo clínico; cancelamento por motivo; correção = cancelar e registrar de novo.
- `billing_periods`: `establishment` (unidade de saúde não-APS ou laboratório municipal; `cnes`), `competence` (AAAAMM), `status` (`open` | `closed`), `closed_by`, `closed_at`, `reopened_reason`, `version`.
- `billing_settings` (cidade): `destination_body`, `management` (`M` | `E`), `origin_cnpj`, `origin_name` — editados pelo admin municipal.
- `billing_exports` (só acréscimo): `billing_period_id`, `version`, `file` (cifrado), `sha256`, `lines_bpa_i`, `lines_bpa_c`, `control_field`, `generated_by`, `generated_at`; downloads registrados na trilha.

## 5. Fluxo

1. **Registro.** No atendimento e na consulta, o profissional acrescenta procedimentos realizados (busca na SIGTAP da competência; quantidade; CID quando a tabela exige). A unidade registra procedimento avulso para o paciente. A crítica roda na gravação e a tela mostra "não faturável — motivo" sem impedir.
2. **Laboratório municipal (21b).** Cada item de laudo liberado vira `performed_procedure` com `source: lab_report`, unidade = estabelecimento do laboratório, profissional = liberador, CBO do liberador. Retificação não gera procedimento novo; cancelamento do exame antes da liberação não gera nada.
3. **Lançamento manual.** O faturista lança produção feita fora do sistema (mesma crítica).
4. **Fechamento.** Por estabelecimento e competência, com as críticas recalculadas: resumo (BPA-I, BPA-C, não faturáveis por motivo), pendências corrigíveis (CNS do profissional, CBO, CPF/CNS do paciente), geração do arquivo, competência `closed`. Procedimento registrado depois do fechamento com data daquela competência entra na competência aberta seguinte (regra do SIA de apresentação em atraso) e é sinalizado.
5. **Reabertura.** Com motivo; o próximo arquivo é nova versão; o anterior permanece no histórico.
6. **Download.** Arquivo de texto no layout de interface; cada download gravado na trilha (`billing_export.downloaded`).

## 6. Arquivo

- Layout de interface texto do BPA Magnético vigente (PDF do DATASUS de 09/07/2026, conferido na Task 0): cabeçalho `01` com `#BPA#`, competência, número de linhas e folhas, campo de controle, órgão de origem/destino, indicador M/E e versão; linhas `02` (BPA-C: CNES, competência, CBO, folha/sequência — até 20 por folha —, procedimento, idade, quantidade, origem) e `03` (BPA-I: CNES, competência, CNS do profissional, CBO, data, folha/sequência, procedimento, identificação do paciente, sexo, IBGE de residência, CID, idade, quantidade, caráter, autorização, origem, nome, nascimento, raça/cor, etnia e demais campos do layout).
- BPA-C agrega por CNES × CBO × procedimento × idade; BPA-I sai item a item.
- Campos do paciente sem dado no cadastro: código de "sem informação" quando o layout aceita (raça/cor 99, etnia vazia quando não indígena), IBGE de residência = município da cidade, endereço e telefone do cidadão do wpda quando houver; campo obrigatório sem valor aceitável → pendência no fechamento (o item não entra no arquivo até ser resolvido).
- Largura fixa, CR+LF, codificação do layout; teste "golden file" byte a byte; perfil documentado em `contracts/billing/bpa-interface.md`.

## 7. Linha do tempo

`Timeline::Sources::Procedures` (fecha api#71 para procedimentos): o profissional vê o procedimento realizado; o cidadão vê nome e data, sem crítica nem faturamento.

## 8. Segurança e LGPD

- Arquivo cifrado com a chave da cidade; download só `billing_clerk`, com trilha; nunca em log.
- Eventos só com ids: `performed_procedure.recorded`, `.cancelled`, `billing_period.closed`, `.reopened`, `billing_export.generated`, `.downloaded`.
- Procedimento e arquivo contam como documento clínico na exclusão LGPD (regra da api#84).

## 9. Testes

- **api:** unidade `ubs` nunca gera BPA; cada crítica SIGTAP (CBO, idade, sexo, CID exigido e compatível, instrumento) com casos que passam e que falham; procedimento da APS fora do BPA; origem do 21b sem duplicar e sem retificação; fechamento, reabertura e versões; atraso de competência; arquivo golden byte a byte (BPA-I e BPA-C, folhas de 20, campo de controle); trilha do download; guarda R18.
- **dashboard:** Vitest.
- **Prova no navegador:** procedimento no atendimento com crítica ok e um não faturável; exames do laboratório entrando; lançamento manual; fechamento com pendências; geração e download; reabertura com motivo.

## 10. Entrega

0. Prova técnica: ler o PDF do layout de 09/07/2026 e o `BPA_LEIAME.txt`; registrar o que mudou desde o layout de 2020 (identificação do paciente por CPF, campos novos).
1. `contracts` — perfil do arquivo e exemplos; tag `billing-v1.0.0`.
2. `api`. 3. `dashboard`. 4. `maintenance`. 5. catálogo público.

## 11. F-IDs (propostos)

| F-ID | Funcionalidade | Apps |
|---|---|---|
| F-22.1 | Interruptor e papel de faturista | api, maintenance, dashboard |
| F-22.2 | Registro de procedimento com críticas SIGTAP | api, dashboard |
| F-22.3 | Produção do laboratório municipal no faturamento | api |
| F-22.4 | Lançamento manual pelo faturista | api, dashboard |
| F-22.5 | Fechamento da competência e pendências | api, dashboard |
| F-22.6 | Arquivo BPA-I/BPA-C do BPA Magnético | api, contracts, dashboard |
| F-22.7 | Procedimentos na linha do tempo | api |

## 12. Riscos e gates de go-live

- Importação real no BPA Magnético (Windows) ou pela secretaria piloto — gate.
- Mudança de layout a cada versão do DATASUS: acompanhar o "Leia-me" a cada competência (runbook).
- Transição para RAC/CMD sem data (o BPA é ponte).
- CNS do profissional e CNES da unidade precisam estar cadastrados (módulos 09 e 10).
