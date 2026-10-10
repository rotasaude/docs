# Pesquisa 19d — receita de controlados e antimicrobianos integrada ao SNCR

Data: 2026-10-10. Pesquisa só de leitura. Complementa `2026-10-09-documentos-clinicos-19c.md`, sem repetir o que já está lá: Lei 14.063, CATMAT e a receita comum.

Legenda: **[CONFIRMADO]** = fonte oficial lida (Anvisa, RDC, manuais). **[SECUNDÁRIO]** = conselho, imprensa ou fornecedor. **[INFERÊNCIA]** = conclusão nossa.

Fontes primárias lidas por inteiro:
- P&R RDC 1.000, 2ª edição (out/2026): https://www.gov.br/anvisa/pt-br/assuntos/medicamentos/controlados/sncr/farmacias/perguntas-e-respostas-rdc-1000-2025-2a-edicao.pdf
- Manual da API SNCR, 3ª edição (set/2026): https://www.gov.br/anvisa/pt-br/assuntos/medicamentos/controlados/sncr/documentos-do-sncr/sncr_manual_api_v5.pdf
- Instruções de Integração API SNCR v1.0 (jul/2026): https://www.gov.br/anvisa/pt-br/assuntos/medicamentos/controlados/sncr/documentos-do-sncr/instrucoes-de-integracao-api-sncr-v1-0.pdf
- Manual do SNCR para VISAs, 7ª edição: https://www.gov.br/anvisa/pt-br/assuntos/medicamentos/controlados/sncr/vigilancias-sanitarias/manual-sncr-7ed.pdf
- Manual do SNCR para Farmácias, 1ª edição: https://www.gov.br/anvisa/pt-br/assuntos/medicamentos/controlados/sncr/farmacias/manual_sncr_farmacias_1ed_2026_formatado_vf_publicacao.pdf
- Modelos eletrônicos: https://www.gov.br/anvisa/pt-br/assuntos/medicamentos/controlados/sncr/receituario-eletronico/modelos-de-receituarios-eletronicos
- RDC 1.000/2025 consolidada (Datalegis): https://anvisalegis.datalegis.net/action/ActionDatalegis.php?acao=abrirTextoAto&tipo=RDC&numeroAto=00001000&seqAto=000&valorAno=2025&orgao=RDC%2FDC%2FANVISA%2FMS
- Notícia "nova etapa do SNCR": https://www.gov.br/anvisa/pt-br/assuntos/noticias-anvisa/2026/inicio-de-nova-etapa-do-sncr-o-que-muda-a-partir-de-30-de-setembro
- Informes da API, de 05/10 e 09/10: https://www.gov.br/anvisa/pt-br/assuntos/medicamentos/controlados/sncr/documentos-do-sncr

> Dica: a página "documentos-do-sncr" carrega a lista de arquivos por JavaScript. A lista aparece em `https://www.gov.br/anvisa/++api++/pt-br/assuntos/medicamentos/controlados/sncr/documentos-do-sncr`.

---

## 1. SNCR: o que é, situação e API

### O que é e em que situação está

- O SNCR foi criado pela RDC 873/2024 e está no ar desde 18/07/2024. Até agora servia só para numerar as notificações em papel. [CONFIRMADO]
- Na versão 2.0, desde **30/09/2026**, ele faz três coisas: entrega numeração por API a plataformas de prescrição, cuida das faixas eletrônicas e registra o uso da receita na farmácia. O prazo vem do art. 16 da RDC 1.000, com a redação da RDC 1.028/2026, que adiou a data de 01/06 para 30/09. [CONFIRMADO]
- Está em produção, mas ainda instável. Os informes oficiais registram três problemas:
  - 05/10: API fora do ar.
  - 09/10: a faixa de numeração de RCE e RET **se esgotou em algumas UFs**.
  - 09/10: a API **entregou números vinculados a outro prescritor**, inclusive de outra UF. Esses números serão cancelados e as receitas emitidas com eles precisam ser reemitidas.
  [CONFIRMADO]

### Datas por tipo (resolve o conflito entre 30/09 e 30/10)

| Tipo | Papel | Eletrônico |
|---|---|---|
| Notificação A (amarela), B e B2 (azuis), retinoides, talidomida | Continua válido. Numeração dada pela VISA. Não há registro no SNCR. | **Só a partir de 30/09/2026 e só por plataforma integrada.** Usa a faixa eletrônica atribuída pela VISA. Nunca houve emissão eletrônica sem SNCR. |
| RCE branca (C1, C5, adendos A1/A2/B1) | Continua válida. Não leva numeração SNCR. | Transição: pode sair sem número SNCR até o fim de out/2026. Depois disso, o número SNCR é obrigatório. |
| RET (antimicrobiano, GLP-1) | Continua válida. Não leva numeração SNCR (P&R 29). | Mesma transição da RCE. |

[CONFIRMADO]

- O **30/09** é a data em que as funções ficaram disponíveis. O **30/10** é o fim da tolerância para RCE e RET eletrônicas sem número, prevista no art. 18 da RDC: "até 30 dias após o início da disponibilização do registro de utilização".
- As fontes discordam em um dia:
  - A notícia da Anvisa e o informe de 09/10 dizem que a transição vai "até 29/10" e que a exigência começa em 30/10.
  - A P&R 2ª ed. (itens 30 e 50) diz que a transição vai de 30/09 a 30/10 e que o número é obrigatório "a partir de 31/10/2026".
  - **Para o desenho, a data dura é 30/10/2026.** [CONFIRMADO, com conflito de um dia entre fontes oficiais]
- A emissão eletrônica nunca é obrigatória. O papel continua válido para todos os tipos. [CONFIRMADO]
- Títulos que falam em adiamento "para 2027" não têm respaldo oficial. [SECUNDÁRIO, descartado]

### Quem se credencia

- **A plataforma não se credencia.**
  - Instruções v1.0, §3: "O credenciamento de plataformas não será necessário".
  - Manual da API, §2.2.2: não há chave nem credencial de parceiro.
  - Há um aviso: "em breve" passará a haver validação da "organização mantenedora" da plataforma. [CONFIRMADO]
- **O prescritor** se cadastra uma vez. O cadastro é automático no primeiro login gov.br no SNCR e exige inscrição ativa no **CFM, CFMV ou CFO**. [CONFIRMADO]
- Para Notificações, o prescritor também precisa de **saldo de numeração eletrônica atribuída pela VISA** ao seu CPF.
  - Ele faz o pedido à VISA local pelo canal que ela já usa (presencial, formulário etc.).
  - A página de pedido dentro do SNCR ainda não existe.
  - Os números eletrônicos são pessoais: não existe perfil "administrador de clínica" (P&R 55). [CONFIRMADO]
- **A farmácia** (inclusive a pública) entra pelo Cadastro Anvisa. A pública é identificada pelo CNES, a privada pela AFE. O gestor atribui o perfil "SNCR-Farmácia" e os colaboradores entram com gov.br. [CONFIRMADO]

### A API

- É REST/JSON e há Swagger só no ambiente de homologação. [CONFIRMADO]
  - Homologação: `https://api-gateway.hmg.apps.anvisa.gov.br/api-sncr/api/v1/`
  - Swagger: `https://api-gateway.hmg.apps.anvisa.gov.br/api-sncr/swagger-ui/index.html`
  - Produção: `https://api-gateway.prd.apps.anvisa.gov.br/api-sncr/api/v1/`
  - Portal de treinamento: https://sncr-treinamento.anvisa.gov.br/ (com contas gov.br de teste)
  - Suporte: sncr.integracao@anvisa.gov.br
- Em homologação, o prescritor consegue números de Notificação **sem aprovação da VISA**. Em produção, não. [CONFIRMADO]
- **Autenticação** é OAuth2/OIDC com **o gov.br do próprio prescritor**, via Keycloak da Anvisa. [CONFIRMADO]
  - Fluxo: o navegador vai para `GET /auth/login?client_url=…`, passa pelo gov.br e volta com `?session_id` (vale 30 s e só uma vez). Em seguida, `GET /auth/token?session_id` devolve o JWT Bearer.
  - O **access token expira em 30 s e não há refresh token**: cada lote de pedidos exige novo login gov.br.
  - O `client_url` precisa estar em domínio **`.br`**. Em desenvolvimento, `localhost` e `127.0.0.1` são aceitos.
  - O session_id fica preso ao Origin que iniciou o fluxo.
  - O nível da conta gov.br (bronze, prata, ouro) não é verificado "na etapa inicial".
  - **Não há certificado ICP do sistema nem do estabelecimento na autenticação.**
- **Endpoints.** Só existem dois, e os dois **apenas fornecem números**. [CONFIRMADO]
  - `POST numeracoes/notificacao-receita`
    - Entrada: `{receita: NRA|NRB|NRB2|NRR|NRT, conselho: CRM|CRMV|CRO, uf, documento, quantidade 10–50}`.
    - Saída: lista de números (ex.: `2411.1-00.0000001`), `saldoReceitas` e aviso quando o saldo fica abaixo de 50.
    - Limite: 50 números por tipo, por prescritor, por dia. O consumo é limitado pelo saldo que a VISA atribuiu.
  - `POST numeracoes/receita-especial-retencao`
    - Entrada: `{conselho, tipo: RCE|RET, documento, uf, cnpj}`. O `cnpj` é o da **empresa responsável pela plataforma**.
    - Saída: um **bloco de 1.000 números** (`inicio`, `fim`).
    - Limite: 3 pedidos por mês por inscrição, ou seja, 3.000 números por tipo por mês.
    - Não depende da VISA. A numeração de RCE e RET é "exclusivamente eletrônica".
  - O gov.br autenticado precisa ser **o mesmo CPF** da inscrição informada. Se não for, a API devolve 404 "Inscrição fornecida é diferente da autenticada".
- **O que a API não faz.** Não há endpoint para registrar a emissão, enviar o conteúdo da receita, cancelar um número ou consultar o status de uso. [CONFIRMADO pela ausência no manual 3ª ed.] Consequências:
  - O SNCR não guarda o conteúdo clínico. Na consulta, a farmácia vê só o número, o tipo, o meio (físico ou eletrônico), a situação ("Autorizada", "Utilizada", "Utilizada parcialmente") e o prescritor ao qual o número foi concedido (P&R 79). [CONFIRMADO]
  - **O documento que vale é o PDF da plataforma.** O art. 5º, III, da RDC obriga a plataforma a "garantir a disponibilização do receituário eletrônico ao paciente" e a manter rastreabilidade. [CONFIRMADO]
  - Cancelar uma numeração é ato da VISA no SNCR: cancelamento definitivo ou de atribuição. [CONFIRMADO]
  - Uma receita emitida por engano não tem como ser "descancelada" na API. [INFERÊNCIA] A plataforma precisa marcá-la como inválida na própria página de conferência e reemitir com outro número.
- **Na farmácia** (Manual Farmácias):
  1. Consulta o número em sncr.anvisa.gov.br. Nas Notificações, um QR leva direto à consulta.
  2. Clica em "Utilizar Receita".
  3. Informa o **CPF do comprador**. O sistema valida na Receita Federal; sem CPF, a receita eletrônica não pode ser usada (P&R 73).
  4. Informa o registro do produto (13 dígitos), lote e caixas, ou a fórmula magistral com DCB.
  5. Na RET, informa também a **data de emissão digitada**: o SNCR não conhece a receita e calcula a validade a partir dela.
  - Notificações e RCE: só uso integral. RET: uso parcelado permitido.
  - O cancelamento do uso só pode ser feito pelo próprio estabelecimento que registrou.
  - Se o SNCR estiver fora do ar, **não há dispensação** da receita eletrônica (P&R 75).
  [CONFIRMADO]

---

## 2. Conteúdo, quantidades, validade e prescritores

**Modelo eletrônico.** O mesmo layout vale para NRA, NRB, NRB2 e RCE. [CONFIRMADO]
- Cabeçalho: tipo + "ELETRÔNICA" + N.º.
- **Emitente:** nome, inscrição no conselho/UF, **endereço completo** (art. 35 da Lei 5.991; endereço da instituição quando ela emite) e telefone.
- **Paciente:** nome, **CPF** (ou passaporte; "não possui" é aceito, P&R 24) e **endereço completo**.
- **Prescrição:** medicamento ou substância, concentração, forma farmacêutica, quantidade e posologia.
- Rodapé:
  - **"Identificação da plataforma"**
  - "Registro da assinatura digital e data"
  - "Valide essa assinatura em validar.iti.gov.br"
  - **"Esta prescrição pode ser verificada em: [URL da plataforma]"**
  - "Verifique a numeração e registre a utilização desta prescrição em sncr.anvisa.gov.br"

**Regras de forma**
- Nas Notificações, o leiaute oficial não pode ser alterado. A RCE pode ter leiaute próprio, desde que mantenha o título "Receita de Controle Especial" e os campos obrigatórios (P&R 7 e 67). [CONFIRMADO]
- Os modelos eletrônicos são **diferentes** dos de impressão e só podem ser usados por plataforma integrada. Ninguém pode preencher o número "à mão" (P&R 48 e 49). [CONFIRMADO]
- O campo UF saiu do modelo: o próprio número SNCR traz o código IBGE da UF (P&R 26). [CONFIRMADO]
- Na emissão eletrônica: [CONFIRMADO, RDC 1.000, arts. 8–12]
  - A data da assinatura é a data de emissão.
  - RCE e RET dispensam as 2 vias.
  - A Notificação dispensa a receita que normalmente a acompanha.
- Os **dados do comprador e do fornecedor** saíram do corpo do documento. Passam a ser registrados no ato da dispensação: no SNCR, se a receita é eletrônica; no verso, se é de papel (P&R 6, 10 e 37). [CONFIRMADO]
- A **RET (RDC 471)** não exige CPF do paciente (P&R 23). Não há modelo eletrônico oficial de RET. [CONFIRMADO P&R; "sem modelo de RET" SECUNDÁRIO, página do CFM]
- Prescrição emergencial continua só em papel (P&R 31). [CONFIRMADO]
- Termos de consentimento (talidomida, retinoides, B2/sibutramina) seguem duas regras: [CONFIRMADO]
  - Se forem natos digitais, prescritor e paciente assinam digitalmente, com assinatura avançada ou qualificada.
  - É vedado imprimir o documento assinado digitalmente pelo prescritor para o paciente assinar à mão (P&R 27).

**Limites da Portaria 344/98** [CONFIRMADO no texto da Portaria, versão republicada; os artigos sobre o leiaute foram alterados pela RDC 1.000]

| Tipo | Validade | Onde vale | Quantidade máxima | Itens por receita |
|---|---|---|---|---|
| NRA | 30 dias | Todo o país; fora da UF exige justificativa (art. 41) | 5 ampolas ou 30 dias de tratamento (art. 43) | 1 substância |
| NRB | 30 dias | Só na UF (art. 45) | 5 ampolas ou 60 dias de tratamento (art. 46) | 1 substância |
| NRB2 | 30 dias | — | 30 dias de tratamento [SECUNDÁRIO; RDC 58/2007 teve dispositivos revogados pela RDC 1.000, art. 25 — conferir] | 1 substância |
| RCE (C1 e C5) | 30 dias | Todo o país (art. 52) | 5 ampolas ou 60 dias; antiparkinsonianos e anticonvulsivantes até 6 meses (art. 59) | Até 3 substâncias C1 (art. 57) |
| RET (antimicrobiano) | 10 dias | — | Ver relatório do 19c | — |
| RET (GLP-1) | 90 dias | — | — | — |

- Nas Notificações, quantidade acima do limite exige justificativa com CID ou diagnóstico.
- No SNCR, a validade da RET é calculada pela data de emissão digitada pela farmácia.

**Quem prescreve** [CONFIRMADO, Portaria 344, arts. 1 e 38; Manual da API]
- **Médico:** todas as listas.
- **Cirurgião-dentista e veterinário:** só para uso odontológico ou veterinário.
- **A API aceita só CRM, CRMV e CRO.** **Enfermeiro não consegue numeração SNCR**, nem de RET.
- Daí resulta: [INFERÊNCIA forte]
  - Antimicrobiano prescrito por enfermeiro (Cofen 801/2026, SNGPC aceita Coren) só poderá sair **em papel** depois de 30/10/2026.
  - A RET em papel não leva número SNCR (P&R 29).
  - Médico com RMS (Mais Médicos) usa o RMS no campo do CRM (P&R 36). [CONFIRMADO]

---

## 3. Assinatura e formato

- **Notificações e RCE** exigem assinatura **qualificada ICP-Brasil, sem exceção**. Avançada e gov.br **não valem** (P&R 39–41; Lei 14.063, art. 13; RDC 1.000, art. 8º, I). [CONFIRMADO]
- **RET** (antimicrobiano, GLP-1) aceita assinatura avançada ou qualificada. A avançada não precisa ser do ITI, desde que comprove autoria e integridade (P&R 41, 43 e 51). [CONFIRMADO]
- **Verificação da assinatura** (P&R 42): [CONFIRMADO]
  - ICP-Brasil e gov.br: pelo validar.iti.gov.br.
  - Avançada privada: pelo validador indicado pela plataforma.
  - O art. 13 da RDC manda a farmácia verificar a assinatura "por meio do serviço do ITI".
  - [INFERÊNCIA] Para a farmácia não travar, use ICP mesmo na RET, ou pelo menos gov.br. Uma avançada "própria" cria atrito no balcão.
- **Formato**
  - A Anvisa **não define XML, JSON nem FHIR** para a receita. A RDC não cita formato, e o "modelo" é um leiaute visual em PDF. [CONFIRMADO]
  - [INFERÊNCIA] O documento é um **PDF nato-digital com assinatura PAdES**, que é o que o validar.iti verifica. O PAdES AD-RB do 19b serve.
  - Escanear papel ou assinar depois um PDF que veio de papel **não** é receita eletrônica (RDC 1.000, art. 2º). [CONFIRMADO]
- **QR code**
  - O texto da RDC não fala de QR. [CONFIRMADO]
  - A notícia oficial diz que "nas Notificações eletrônicas, o QR Code facilita a consulta" no SNCR. [CONFIRMADO]
  - O modelo traz uma linha para a URL de verificação da plataforma. [CONFIRMADO]
  - [INFERÊNCIA] Coloque dois códigos: um QR para a página pública de conferência (a do 19c) e um para a consulta do número no SNCR. O formato exato da URL de consulta do SNCR **não foi encontrado**.

---

## 4. Como as plataformas fazem hoje

- **CFM Prescrição Eletrônica** (gratuita para médicos) já está integrada ao SNCR. Tem acordo de cooperação técnica com a Anvisa para validar o CRM e fornece certificado ICP gratuito. https://portal.cfm.org.br/noticias/notificacoes-de-receitas-amarela-azul-e-outras-poderao-ser-emitidas-eletronicamente-a-partir-desta-quarta-feira-30 — [SECUNDÁRIO, fonte institucional do CFM]
- **Memed**: quando falta numeração, o médico se autentica no gov.br dentro do fluxo de emissão e a numeração "é solicitada automaticamente". O PDF identifica a Memed como "gráfica/plataforma" (CNPJ e endereço). Retinoides e talidomida não são 100% digitais, porque os termos precisam ser impressos. https://suporte-medico.memed.com.br/hc/pt-br/articles/56357536042651 — [SECUNDÁRIO] Isso bate com o desenho da API.
- **Whitebook/PEBMED, Afya, ProDoctor, Nexodata** anunciam integração, mas não foi possível ler os detalhes (403 ou não encontrado). [NÃO CONFIRMADO]
- **Não há lista de "sistemas homologados".** Não existe credenciamento de plataforma, e a imprensa diz que a Anvisa não publicou lista nenhuma. O Manual da VISA usa uma vez a expressão "plataformas… homologados", mas sem nenhum processo de homologação descrito. [CONFIRMADO quanto à ausência de credenciamento; SECUNDÁRIO quanto à lista]
- **PEC e-SUS APS**: não foi achado nenhum anúncio de integração com o SNCR nem de prescrição eletrônica de controlado. Prefeituras (SP, Curitiba) orientam a imprimir controlado e antimicrobiano saídos do e-SUS. [NÃO CONFIRMADO / SECUNDÁRIO] Pela P&R 72, receita emitida pelo e-SUS segue as mesmas regras que qualquer outra. [CONFIRMADO]

---

## 5. Transição, papel e talonário da VISA

- **Papel continua válido.** [CONFIRMADO]
  - Os novos modelos impressos (Versão 2) são obrigatórios em impressões feitas desde 18/05/2026. O estoque antigo segue válido por prazo indeterminado.
  - A VISA não precisa mais imprimir a NRA. O prescritor ou a instituição pode pegar só a numeração e imprimir em gráfica (P&R 12–17).
  - **A numeração física de NRA e NRB pode ser atribuída a uma instituição (CNPJ ou CNES)** (Manual da VISA, §7).
  - [INFERÊNCIA] Por isso a SMS pode receber numeração física em nome da UBS.
- **Número físico não pode ser usado em receita eletrônica, e vice-versa.** São séries separadas no SNCR. [CONFIRMADO] Daí resulta:
  - Não é permitido que o sistema imprima número de talonário físico numa receita eletrônica (P&R 49).
  - O que é possível: o sistema **imprime a NR em papel, no modelo físico V2, com número físico atribuído pela VISA, e o prescritor assina à mão.** [INFERÊNCIA a partir dos itens 11–14 e 48–49 da P&R] Isso é receita física e precisa seguir o leiaute oficial à risca.
  - Se a VISA local exigir gráfica cadastrada (P&R 5), imprimir na UBS pode não ser aceito. **Confirmar com a VISA de cada município.**
- **Concessão de NRA:** depende da pactuação estadual. A VISA municipal só concede se houver descentralização (Manual da VISA, 1.3). [CONFIRMADO]
- **RCE e RET eletrônicas sem número**, até 30/10: o estabelecimento guarda o PDF original e uma cópia impressa, onde anota comprador, fornecedor e quantidade (P&R 30). [CONFIRMADO]

---

## 6. Assistência farmacêutica municipal e SNGPC

- **Farmácia de UBS ou da rede municipal**
  - Entra no SNCR pelo Cadastro Anvisa, usando o **CNES**. Um farmacêutico pode estar ligado a vários CNES (P&R 60 e 66). [CONFIRMADO]
  - **Não é obrigatório integrar e-SUS AF, Hórus ou sistema próprio** por API. A baixa pode ser feita no navegador (P&R 76). [CONFIRMADO]
  - **Não existe endpoint público de registro de uso** para sistemas dispensadores. [INFERÊNCIA pela ausência no manual da API]
  - Sem acesso ao SNCR, a farmácia **não consegue dispensar** a receita eletrônica (P&R 46 e 61). [CONFIRMADO]
- **BNAFAR e e-SUS AF** não mudam. O registro no SNCR é uma obrigação paralela (P&R 77). [CONFIRMADO]
- **SNGPC**
  - É complementar ao SNCR e não há cronograma de integração entre os dois (P&R 53). No SNGPC, informa-se o número da receita, seja física ou eletrônica (P&R 35). [CONFIRMADO]
  - O SNGPC é obrigatório só para farmácias e drogarias **privadas**. A pública fica fora enquanto não houver módulo próprio (RDC 22/2014, art. 3º, parágrafo único, conforme ofício da Anvisa ao Cofen). A pública mantém a escrituração da Portaria 344 em livro ou em sistema aprovado pela VISA. [SECUNDÁRIO] https://www.macae.rj.gov.br/midia/uploads/RDC%2022-2014.pdf
- [INFERÊNCIA] Quando a farmácia da própria UBS dispensa uma receita da própria plataforma, a baixa no SNCR é obrigatória do mesmo jeito. Marcar como "dispensada" só no Rota Saúde não substitui a baixa (P&R 30: "registros mantidos exclusivamente em plataformas digitais intermediárias não substituem…").

---

## 7. Implicações para o desenho do 19d [INFERÊNCIA]

1. **O SNCR é um "cofre de números", não um registro de receitas.** A plataforma guarda os números obtidos, consome um por receita assinada, mantém o documento e a página de conferência. A baixa fica com a farmácia. O modelo é este:
   - Lote de numeração: tipo, prescritor, faixa, origem, data.
   - Número consumido: ligado ao documento.
   - Número inutilizado: emissão abortada ou cancelada.
2. **A obtenção de número é um ato interativo do prescritor no navegador**, com gov.br e token de 30 s sem refresh. O job em background não consegue fazer isso. Estratégia:
   - Buscar o lote antes de prescrever, como faz a Memed.
   - RCE/RET: blocos de 1.000, até 3 por mês.
   - NR: 10–50 por dia, dentro do saldo da VISA.
   - Ver depois se o JWT pode ir do browser para o backend e ser usado server-side dentro dos 30 s. **Não foi confirmado**, e o Origin fica preso ao fluxo.
3. **Domínio `.br` é obrigatório** para o `client_url`. É preciso registrar e usar `*.rotasaude.<algo>.br` em produção e staging. Localhost funciona em dev contra o ambiente de homologação.
4. **O `cnpj` do pedido é o da empresa responsável pela plataforma.** Num SaaS municipal pode ser o fornecedor ou o município, e a validação "da organização mantenedora" ainda vai chegar. **Decisão em aberto: perguntar a sncr.integracao@anvisa.gov.br.**
5. **Assinatura:** ICP qualificada para Notificações e RCE, usando o PAdES do 19b. Na RET, recomenda-se ICP também. Enfermeiro fica com RET em papel.
6. **Papel continua como caminho padrão e de contingência** (SNCR fora do ar, faixa esgotada, enfermeiro, emergência, comprador sem CPF). A RCE e a RET em papel do 19c servem. A NR em papel depende de talonário ou numeração física da VISA.
7. **Ordem sugerida:**
   - Primeiro RET e RCE eletrônicas. A numeração é automática, sem VISA.
   - Depois NRB/NRB2 e por último NRA. Dependem de saldo atribuído pela VISA a cada médico e, na NRA, de pactuação estadual.
   - Retinoides e talidomida ficam fora (termos com assinatura do paciente).

## O que NÃO foi possível confirmar

- O formato da URL de consulta do SNCR para o QR, e se o QR é obrigatório.
- Se a API aceita um token obtido no browser e usado pelo backend. A documentação mostra só chamadas feitas pelo frontend.
- Qual CNPJ usar num SaaS municipal e quando chega a "validação da organização mantenedora".
- O conteúdo do Swagger de homologação: não foi acessado (exige o ambiente).
- O plano do PEC e-SUS APS para o SNCR.
- A lista de plataformas integradas.
- Regra atual de duração máxima da B2 depois da RDC 1.000.
- O dia exato do fim da transição: 29/10 ou 30/10.
- Se a VISA de cada município aceita NR física impressa pelo sistema na UBS.
