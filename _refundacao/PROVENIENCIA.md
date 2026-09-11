# Proveniência da refundação

Esta pasta documenta **como o corpus v2 foi construído a partir do v1**. Não é
parte do corpus de governança em si — é o registro de construção. **Pode ser
arquivada** (ou removida) depois que a refundação concluir e o de-para (Etapa 4)
estiver embutido no corpus.

## Conteúdo

| Arquivo | Etapa | O que é |
|---|---|---|
| `00-destilado.md` | 0 | Destilado dos 24 ADRs v1 + 0025 em **estado final** (emendas dobradas), 15 invariantes extraídos, 10 contradições sinalizadas. |
| `01-mapa-consolidacao.md` | 1 | Mapa de consolidação **aprovado** pelo autor: 26 origens → 15 ADRs v2, com splits, flags resolvidos e ordem justificada. |
| `PROVENIENCIA.md` | 2 | Este arquivo. |

## Cadeia de construção

```
v1 (docs/adr/, 26 docs, cadeia de emendas)  ── read-only, intocado ──┐
                                                                     │ Etapa 0 (lê)
                                                                     ▼
                                              00-destilado.md (estado final)
                                                                     │ Etapa 1 (consolida)
                                                                     ▼
                                              01-mapa-consolidacao.md (APROVADO, N=15)
                                                                     │ Etapa 2 (scaffold)
                                                                     ▼
                                              docs/ (esqueleto + placeholders)  ◄── você está aqui
                                                                     │ Etapa 3 (escreve ADRs)
                                                                     │ Etapa 4 (de-para v1→v2)
                                                                     │ Etapa 5 (módulos/ciclo/operação)
                                                                     │ Etapa 6 (auditoria de invariantes)
                                                                     ▼
                                              corpus v2 completo e auditado
```

## Decisões fixas do v2 (não reabrir)

1. Corte limpo — história de emendas descartada no corpus novo.
2. Re-numeração do zero (`0001`–`0015`).
3. Idioma **tudo em inglês** para identificadores.
4. Chave de AR Encryption ancorada exclusivamente no env do `api`.

## Nota sobre nomenclatura

Os artefatos foram renomeados de `etapa-N-*.md` para `0N-*.md` na Etapa 2, para
alinhar com a convenção de proveniência usada pelos prompts das etapas seguintes.
O conteúdo é o mesmo; apenas o nome do arquivo mudou.
