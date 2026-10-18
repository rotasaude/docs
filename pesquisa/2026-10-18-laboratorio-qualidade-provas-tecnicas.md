# Módulo 21c — prova técnica da HL7 do controle (2026-10-18)

## Convenção HL7 v2.5.1 para resultado de controle de qualidade
- Fontes (URL e data de consulta 2026-10-18):
  - IHE PaLM TF Vol. 2b, rev. 11.0 (2024-04-08): https://ihe.net/uploadedFiles/Documents/PaLM/IHE_PaLM_TF_Vol2b.pdf (PDF baixado e lido com pdftotext; fonte principal).
  - HL7 tabela 0369: https://terminology.hl7.org/CodeSystem-v2-0369.html — página sem a lista de códigos no acesso feito (inconclusivo; o IHE traz a tabela do papel do espécime, com Q como controle).
  - HL7 tabela 0384: https://terminology.hl7.org/CodeSystem-v2-0384.html — idem, sem lista no acesso feito; o IHE manda usar CO para material de controle.
  - Norma cap. 7 e 13 / Caristix / middleware público: não consultados (sem acesso necessário; o IHE cobre os cinco pontos).
- Marca do controle: SPM-11 (tabela 0369) = Q — confere. O IHE exige Q quando a ordem de trabalho é de controle, P para paciente, L para pool; U só em resposta negativa de consulta.
- Material de controle: INV no grupo SAC, INV-3 = CO, INV-12 validade, INV-16 lote — confere. INV é condicional: só pode vir quando SPM-11 = Q; INV-3 vira CO para controle; validade e lote são "se disponíveis no analisador". (Diferença menor: o IHE também prevê INV-4, série do frasco; não usamos.)
- SPM-4 vazio para Q — confere (vazio também para U; preenchido só para P e L).
- Aparelho: OBX-18 (EI, repetível; modelo^fabricante e número de série na LAW) — confere. 1ª repetição: modelo (id da entidade) e fabricante (namespace); 2ª: número de série e fabricante; 3ª em diante: dados opcionais de fornecedor.
- EQU fora do OUL^R22 na 2.5.1 — confere: o texto do IHE para a transação de resultados não inclui nenhum segmento EQU; o aparelho é carregado só pelo OBX-18. Divergência Q1 mantida (contrato §6 cita EQU-1).
- SPM-2 (Specimen ID, EIP): identificador do espécime, "obrigatório se disponível"; para controle serve de etiqueta/nível — confere como suporte à decisão abaixo.
- Decisão do Rota Saúde: controle pelo SPM-2 = código de barras do nível do lote cadastrado; SPM-11 = Q recomendado
  (Q com código que não é de controle é recusado); INV aceito e ignorado; aparelho pela repetição de OBX-18 cujo
  componente 1 casa com hl7_equipment_id.
- Resultado: CONFIRMADO (IHE PaLM TF-2b rev. 11.0; as páginas das tabelas 0369/0384 não deram a lista no acesso feito, mas não contradizem).
