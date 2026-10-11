# e-SUS PEC — "Sistemas externos": parâmetros e assinatura (lido em 2026-10-11)

Fonte: https://integracao.esusaps.bridge.ufsc.tech/sistemas_externos/ (páginas index, iframe,
chave_assinatura e parametros_dinamicos; "Alterado em" 11/09/2026, 16/09/2025, 09/02/2026 e 19/06/2026).

## Cadastro
- Feito pelo Administrador da Instalação em Integração > Sistemas externos, só em instalação com HTTPS.
- Dados: URL de acesso, nome, CPF ou CNPJ, e-mail e nome do sistema; "Local de acesso": Atendimento
  (aba no atendimento individual e odontológico) ou menu principal.
- Ao cadastrar, o PEC gera a chave de assinatura; o administrador a repassa ao integrador. Chave perdida
  não se recupera: novo cadastro.

## Parâmetros na URL
- URL entregue ao administrador: `https://<host>/pec/timeline?documento-profissional=[documentoProfissional]&documento-cidadao=[documentoCidadao]&cnes=[CNES]`.
- `documento-profissional` e `documento-cidadao`: CPF, ou CNS quando não há CPF. `documento-cidadao` só
  com "Local de acesso" = Atendimento.
- `cnes`: CNES do estabelecimento da lotação do profissional.
- O PEC acrescenta `&pec_timestamp=<segundos Unix>&pec_sign=<assinatura>`.

## Assinatura
- HMAC-SHA256 com a chave do cadastro, saída hexadecimal minúscula.
- Mensagem assinada: a URL completa (protocolo, host, caminho e query) até `pec_timestamp` inclusive —
  tudo antes de `&pec_sign=`. A URL assinada é canônica.
- Janela do exemplo oficial: 30 min em valor absoluto. O Rota Saúde usa 5 min (ADR 0035), uso único.

## iframe
- Não carrega com `X-Frame-Options` DENY/SAMEORIGIN nem com `frame-ancestors 'none'`.

## Consequências no Rota Saúde
- Chaves da query com hífen (contrato do módulo 20, Divergência D1).
- Documento do cidadão por CNS não identifica ninguém (CNS do par é cifrado não determinístico).
- URL conferida = `request.base_url + request.original_fullpath`, cortada em `&pec_sign=`: o proxy
  de produção precisa repassar Host e X-Forwarded-Proto (gate de go-live).
