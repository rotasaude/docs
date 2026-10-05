# Módulo 16 — Modo de prontuário e exportação

- **Estado:** Stub
- **Tipo:** Ciclo 2

## Escopo (intenção)

Configuração por cidade do modo de prontuário — **registro** (o Rota Saúde é o prontuário e substitui o e-SUS PEC) ou **integrado** (convive com o PEC) — e o exportador das fichas LEDI para o PEC da cidade, que repassa ao SIAPS. Inclui terminologias (CID-10, CIAP-2, SIGTAP), identificadores do profissional e da equipe (CNS, CPF, INE, tipo de equipe 70/76, CNES) e o CADSUS alimentando o perfil do cidadão (hoje declarado, módulo 15).

## Integrações externas

e-SUS APS PEC + LEDI → SIAPS (obrigatório no modo registro); CADSUS; CNES; RNDS (credenciamento por estabelecimento). Detalhe e fontes em
[`pesquisa/2026-10-05-mapa-de-integracoes.md`](../pesquisa/2026-10-05-mapa-de-integracoes.md).

## Depende de

Módulos 09, 10, 15.

## Em aberto

- Credencial por cidade ou da plataforma (LEDI, RNDS, CADSUS).
- Como cada cidade-piloto usa o PEC hoje (instalação própria ou centralizadora).
- Manutenção contínua do LEDI (versão nova a cada 2–6 semanas; dado com versão > 12 meses é invalidado).

## Histórico

- 2026-10-05 — Registrado como `Stub` no portfólio do Ciclo 2 (brainstorm de
  novos módulos; decisão de suíte completa).
