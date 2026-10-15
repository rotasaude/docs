# Módulo 21b — provas técnicas (2026-10-15)

Task 0 do plano do api (`superpowers/plans/2026-10-15-module-21b-lab-api.md`). Base do api conferida em `origin/main`
`0e81173`: última migração de cidade `20261013900001_add_exam_tracking.rb`, de plataforma
`20261010700001_add_controlled_prescriptions_platform.rb`, `VALID_RANGE = (1..36)`, sem `barby` no `Gemfile.lock`; os
nomes que o plano usa (`fake_lab` no catálogo de interruptores, `Exams::Transition`, `Exams::RegisterResult`,
`RequestErasure.examined_patients?`, `Signatures::DocumentTypes`/`Signing#store`, `Mfa::Verify.totp_step_for`,
`User#consume_totp_step!`, `Professionals::Create`) existem.

## (a) Perfil HL7 v2.5.1 das mensagens QBP^Q11, RSP^K11, OUL^R22 e ACK

**Resultado: PASSOU.** Nenhuma diferença além das Divergências D1 e D2 do plano.

### Fontes (consultadas em 2026-10-15)

- Norma HL7 v2.5.1 publicada no hl7.eu: capítulo 2 (`https://www.hl7.eu/HL7v2x/v251/std251/ch02.html`) e capítulo 2A,
  tipos de dados (`https://www.hl7.eu/HL7v2x/v251/std251/ch02a.html`).
- Estruturas geradas da especificação v2.5.1 no HAPI HL7v2
  (`https://hapifhir.github.io/hapi-hl7v2/v251/apidocs/ca/uhn/hl7v2/model/v251/`): mensagens `QBP_Q11`, `RSP_K11`,
  `OUL_R22` e `ACK`; segmentos `MSH`, `MSA`, `ERR`, `QPD`, `QAK`, `SPM`, `OBR` e `OBX`; tipo `MSG`.
- Tabelas 0008, 0208 e 0357 nas páginas de vocabulário v2 do HL7 FHIR R4 (`https://hl7.org/fhir/r4/v2/0008/`,
  `https://hl7.org/fhir/R4/v2/0208/`, `https://hl7.org/fhir/r4/v2/0357/`).
- As páginas da Caristix não abriram (aplicação em JavaScript). O product brief da HL7 e o volume 2a do IHE PaLM
  (LAB-27/LAB-29) não foram lidos; as fontes acima cobrem os nove pontos.

### O que foi conferido

| # | Ponto | Resultado |
|---|---|---|
| 1 | `QBP_Q11`: `MSH [SFT] QPD RCP [DSC]`; o código de barras vai num parâmetro do usuário em `QPD-3` | confere |
| 2 | `RSP_K11`: `MSH [SFT] MSA [ERR] QAK QPD` + o grupo de conformidade + `[DSC]`; `QAK-2` da tabela 0208 (`OK`, `NF`, `AE`, `AR`) | confere; o grupo de conformidade genérico é `[RDF {RDT}]`, e o perfil pode trocá-lo pelos segmentos da resposta |
| 3 | `OUL_R22`: `SPM` com `OBR` e `OBX` aninhados; `SPM-2` é EIP; `OBR-4` é o serviço; `OBX-2/3/5/6` tipo, observação, valor e unidade | confere; **`SPM-4` (tipo do espécime) é obrigatório** e o perfil precisa preenchê-lo |
| 4 | `ACK`: `MSH [SFT] MSA [ERR]`; `MSA-1` da tabela 0008 (`AA`, `AE`, `AR`); `MSA-2` é o `MSH-10` respondido | confere; na v2.5.1 o `ERR` é repetível |
| 5 | `ERR` na v2.5.1: `ERR-3` é o código HL7 da tabela 0357, obrigatório; `ERR-4` severidade obrigatória; `ERR-5` código da aplicação | confere; pôr o nosso código em `ERR-3` contrariaria a tabela 0357 → **Divergência D1** |
| 6 | `OBX-3` é CE (identificador, texto, sistema) | confere; a chave do componente vai no identificador e o exame em `OBR-4` → **Divergência D2** |
| 7 | Sistema de código local `99zzz` | confere (tabela 0396, seção 2.17.4) |
| 8 | `MSH-12` = `2.5.1`; terceiro componente de `MSH-9` obrigatório | confere (tipo `MSG`, seção 2.A.1.44: componente 3 obrigatório) |
| 9 | Segmento terminado por `\r`; escapes `\F\ \S\ \T\ \R\ \E\` | confere (seções 2.5.4 e 2.7.1) |
| 10 | Transporte | a norma não define HTTP; o MLLP vem do apêndice de transporte. A entrada por HTTPS com o corpo ER7 puro, sem o envelope MLLP, é escolha nossa |

### Para o perfil (Task 1)

- Preencher `SPM-4` nas mensagens `OUL^R22`, já que é obrigatório na v2.5.1.
- Aceitar `ERR` repetido na leitura e escrever um `ERR` por erro.

## (b) Etiqueta Code 128 em PDF para impressora térmica

**Resultado: PASSOU.**

- barby 0.7.0 (Code 128 automático: 10 dígitos usam o conjunto C, 90 módulos) e Prawn 2.5.0, com as barras desenhadas
  em vetor.
- Página de 50 × 25 mm (141,73 × 70,87 pt, conferido no `pdfinfo`); módulo de 0,375 mm (3 pontos a 203 dpi); barra de
  9 mm; zona de silêncio de 10 módulos de cada lado.
- Leitura com `zbarimg` 0.23.92 (container Debian, sem instalar nada no host), rasterizado pelo `pdftoppm` em preto e
  branco:
  - 203 dpi: página 1 `0000012301`, página 2 `9999999999`;
  - 300 dpi: página 1 `0000012301`, página 2 `9999999999`.
- A olho: o texto não invade as barras e os acentos (É, Ç, Ã) saem certos com a fonte embutida.
- **Para a Task 9:** o nome de 80 caracteres sai cortado ("JOSÉ DE ASSUNÇÃO PEREIRA DOS SANTOS", sem
  "ALBUQUERQUE NETO" e sem reticências), mesmo com `shrink_to_fit`. O código de barras não é afetado. É o caso
  "etiqueta no limite" do Review Focus 5.

### Limite

O leitor físico e a impressora térmica da cidade são gate de go-live.
