# Rollout — assinatura digital do prontuário (módulo 19b)

Runbook do operador para publicar o módulo 19b (ADR 0032), ligar a assinatura
digital ICP-Brasil numa cidade, operar o PSC simulado fora de produção e
acompanhar as assinaturas. Base do prontuário:
[`rollout-prontuario.md`](rollout-prontuario.md).

O 19b está **Entregue**, não Fechado. Ele foi verificado contra o **PSC
simulado** e a AC de teste do `signer`. A prova com prestador real é o gate
rotasaude/api#53 (§7). Até esse gate passar, nenhuma cidade real dispensa o
papel.

## 1. Publicação

Ordem obrigatória:

1. `contracts` com a tag `clinical-v1.0.0`: o JSON canônico da consulta e do
   adendo, que é o que se assina.
2. **`signer`** (`rotasaude/signer`, `70475f7` ou posterior), serviço interno
   em Java com Demoiselle, na porta 8090, sem estado. Configuração:
   - `SIGNER_ENV=production`, obrigatório; `prod` é recusado;
   - `SIGNER_TOKEN` com pelo menos 32 caracteres, o mesmo do api.

   Em produção são proibidos `SIGNER_EXTRA_TRUST_ANCHORS`, `SIGNER_DEV_PKI_DIR`
   e `-Dsigner.extraTrustAnchors`: o boot falha. A rede de saída precisa
   alcançar só as LCRs das ACs.
3. `api` (`c9ccd88` ou posterior). **Imagem nova**. Por cidade, antes de cortar o
   tráfego:
   1. publique a imagem;
   2. pare o worker;
   3. faça backup do banco da cidade;
   4. rode a migração de plataforma (`20261008500001`, `signature_provider_checks`);
   5. rode `bin/rails city:migrate:all` **da imagem nova**. A `20261008500001`
      cria certificados, sessões, pedidos e assinaturas, com as guardas de
      imutabilidade, e é **irreversível**;
   6. religue o worker.

   O `config/queue.yml` ganha um worker de cidade dedicado à fila `signatures`.
   Ele é um processo a mais por cidade, com cerca de 5 conexões em produção, e
   entra na conta do `max_connections` (§7).
   Variáveis do api: `SIGNER_URL` e `SIGNER_TOKEN`.
4. `dashboard` (`8a0c0de` ou posterior), depois do api.
5. `maintenance` (`0f0a779` ou posterior), depois do dashboard.

Publicar não muda nenhuma cidade: `digital_signature` nasce desligado e, sem
ele, tudo segue no papel do 19a.

## 2. Prestadores (PSC em nuvem)

As credenciais de cada prestador ficam nas credenciais cifradas do api, por
ambiente, em `signature.providers.<chave>`. Cada uma tem `client_id`,
`client_secret`, `base_url` (com `/v0`) e `authorize_base_url`. As chaves são
`vidaas`, `birdid`, `safeid`, `neoid` e `remoteid`. Prestador sem credencial não
é oferecido aos profissionais.

- Cadastre a aplicação em cada PSC, conforme o DOC-ICP-17.01; em produção, o
  PSC exige certificado SSL ICP-Brasil da plataforma.
- O endereço de retorno é um por cidade:
  `https://<host do dashboard da cidade>/dashboard/signature/callback`.
- O maintenance mostra, só para leitura, quais prestadores têm credencial, a
  última checagem e o estado do `signer`.

## 3. Ligar numa cidade

1. Pré-requisitos do 19a: modo `record` e `clinical_record` ligado.
2. O mantenedor liga `digital_signature` no `maintenance` (aba Funcionalidades).
3. Cada profissional que for assinar:
   - precisa ter CPF no cadastro de profissional (sem ele:
     `professional_cpf_missing`);
   - em Conta → Assinatura digital, procura e vincula o certificado em nuvem,
     com step-up e autorização no app do PSC. O CPF do certificado é conferido
     com o do profissional;
   - a cada turno abre a **sessão de assinatura** (selo no topo), que dura o
     menor entre o tempo concedido pelo PSC e 12 h.

Quem não tem certificado continua no papel. A adoção é por profissional.

## 4. Como a assinatura acontece

- Ao **finalizar** uma consulta, ou ao acrescentar um **adendo**, com a autora
  tendo certificado ativo e `digital_signature` utilizável, o pedido de
  assinatura nasce na mesma transação e já aparece como "pendente". O
  `SignJob` roda na fila `signatures`. Finalizar nunca depende do PSC nem do
  `signer`.
- Com sessão aberta, a assinatura sai sozinha: CAdES do JSON canônico mais
  PAdES do PDF, na política AD-RB. O PSC ou o `signer` fora do ar contam como
  falha transitória, com 3 tentativas (30 s e 2 min).
- Sem sessão, o documento vai para **Pendentes de assinatura**, onde se resolve
  de dois jeitos:
  - **"Assinar todas"**: uma aprovação só no PSC (`multi_signature`);
  - **"Voltar ao papel"**: exige motivo de 10 a 500 caracteres, e o impresso
    volta a ter espaço para a assinatura à mão.
- A **varredura** roda a cada 10 min por cidade e devolve ao papel os pendentes
  da cidade que desligou `digital_signature`, `clinical_record` ou o modo
  `record`.
- **Leitura**: segue as regras do 19a. A autora lê, baixa o PDF e o pacote
  `.p7s` e revalida sempre. Outro profissional faz o mesmo em contexto ou com
  abertura justificada. O `municipal_admin` lê só o conteúdo assinado, com
  step-up, e não baixa nem revalida.
- **Impresso**: sai o PAdES puro quando a consulta é digital, não tem adendo e
  a validação é `valid`. Nos outros casos sai o impresso com o estado de cada
  parte e espaço para assinatura à mão.

## 5. PSC simulado (só fora de produção)

O interruptor de cidade `signature_psc_mock` ("PSC simulado
(desenvolvimento)") requer `digital_signature`. Ele só existe em
development, test e staging: em produção o catálogo não o oferece e o api
recusa ligá-lo.

- **Ligado:** a cidade assina só com o PSC simulado (o `fake-psc` do compose,
  `FAKE_PSC_URL` e `FAKE_PSC_PUBLIC_URL`), com e-CPF da AC de teste do
  `signer`. Toda assinatura sai marcada `simulated: true`, e o dashboard, o
  maintenance, o PDF e o pacote (`documento-assinado-simulado.zip`, com um
  arquivo de aviso) dizem "Assinatura simulada — sem validade jurídica".
- **Desligado:** a cidade usa só os prestadores reais configurados no
  ambiente.

Invariante do ADR 0032: nenhuma assinatura `simulated` em produção.

Em dev, a semente liga `digital_signature` e `signature_psc_mock` em Curitiba.

## 6. Acompanhar

- **Painel de assinatura** (`municipal_admin`, Equipe):
  - profissionais com e sem certificado e certificados vencendo em 30 dias;
  - pendentes há mais de 24 h;
  - documentos por modo (digital, pendente, papel);
  - assinaturas inválidas ou indeterminadas na última validação.
- **Revalidar** confere de novo a cadeia e a revogação. LCR indisponível ou não
  autenticável dá `indeterminate`, nunca `valid`.

## 7. Gates e pendências antes de uma cidade real

| Item | Bloqueia? |
|---|---|
| rotasaude/api#53 — assinatura com PSC real (VIDaaS de homologação ou outro) aprovada no validar.iti.gov.br | **Sim** (Crítica) |
| rotasaude/signer#1 — endurecimento do `signer` (hora futura, SSRF de LCR, concorrência, ByteRange, imagem etc.) | **Sim** (Alta) |
| Cadastro da aplicação em cada PSC, com o SSL ICP-Brasil da plataforma | **Sim** |
| `max_connections` com o worker `signatures` de cada cidade | **Sim** |
| Chamadas ao PSC e ao `signer` dentro da transação com lock: no pior caso um lote prende linhas por minutos | Não; acompanhar |
| Usuário desativado mantém a sessão de assinatura por até 12 h | Não; acompanhar |
| rotasaude/dashboard#14 — marcador do adendo fica "assinando…" até reabrir quando não há sessão | Não (Baixa) |
