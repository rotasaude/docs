# Corpus v1 — arquivo read-only

Os 25 ADRs originais (com cadeia de emendas) que a refundação consolidou nos 15
ADRs lineares de [`docs/adr/`](../docs/adr/README.md). Preservados como
**arquivo histórico**: não são decisões vigentes e não devem ser citados por
código novo.

Para saber onde cada ADR v1 foi parar, veja
[`docs/adr/DE-PARA.md`](../docs/adr/DE-PARA.md). Para ver como a consolidação
foi feita, veja [`_refundacao/`](../_refundacao/PROVENIENCIA.md).

O ADR v1 0026 nunca existiu como arquivo — era uma decisão não-arquivada,
absorvida direto pelo v2 0015 (contratos e versionamento).

## O que NÃO veio junto, e por quê

- **`prompts/`** (prompts de relatório semanal) ficou de fora: é material de
  processo, não decisão de arquitetura. Por isso o link `prompts/` no
  [`adr/README.md`](adr/README.md) daqui não resolve. O original segue no
  arquivo do projeto, fora de repositório.

## Sobre links quebrados aqui dentro

Estes documentos foram escritos quando o projeto era um monorepo, então alguns
links apontam para caminhos que não existem neste repositório — por exemplo
`../../apps/api/Dockerfile`, no ADR 0002. **Isso é de propósito.** Arquivo é
arquivo: reescrever os links falsificaria o registro histórico. Quando precisar
do alvo, ele está no repositório da aplicação correspondente.
