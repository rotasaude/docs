# Módulo 35 — Zoonoses e endemias — design

**Data:** 2026-11-14
**Status:** aprovado pelo usuário (2026-11-14)
**Entregas:** **35a** (campo de endemias) primeiro; **35b** (centro de zoonoses) depois. Cada entrega tem contrato entre apps e planos próprios.
**Afeta:**
- `apps/api` (cidade): boletim na visita do agente de endemias, quarteirão e risco do imóvel, ciclos, ovitrampas, LIRAa, bloqueio, amostras, relatórios (35a); animal, vacinação antirrábica, observação do agressor, leishmaniose canina, canil e gatil (35b).
- `apps/dashboard`: telas da coordenação de endemias (35a) e do centro de zoonoses (35b).
- `rotasaude/campo` (app do 24c): boletim, ovitrampa, LIRAa e bloqueio offline (35a); animal, vacinação e inquérito offline (35b).
- `rotasaude/contracts`: `field-sync` com uma MINOR por entrega (35a e 35b).
- `apps/maintenance`: interruptores `vector_control` (35a) e `zoonoses` (35b).
- `apps/wpda`: categoria "zoonoses" na ouvidoria do cidadão (35b).

**Módulo:** `docs/modulos/35--zoonoses-e-endemias.md` · **Pesquisa:** `docs/pesquisa/2026-11-14-zoonoses-e-endemias-35.md` · **ADR:** `docs/adr/0054.md`
- **ADRs relacionados:** 0020 (banco por cidade), 0028 (LEDI), 0040 (linhas de cuidado), 0041 (núcleo e ferramenta), 0042 (cadastro territorial), 0043 (vigilância epidemiológica), 0051 (ouvidoria), 0052 (visita e sincronização v1), 0053 (app de campo).
- **Módulos relacionados:** 11 (território), 24a/24b/24c (cadastro, visita, app), 25 (vigilância), 27a (estoque), 33 (ouvidoria).

**Fora do 35 (os dois):**
- integração com sistemas estaduais (Sisaweb, SIMAZ e outros) e exportação para o programa de computador do LIRAa;
- leitura ou envio ao Conta Ovos (card para depois; decisão 5);
- módulo de laboratório com recebimento e controle de qualidade, e GAL;
- sorteio de supervisão dos agentes;
- integração com o cadastro imobiliário da prefeitura;
- prontuário veterinário completo;
- Wolbachia, estações disseminadoras e técnica do inseto estéril;
- envio automático ao Sinan (como no 25, a ficha é digitada).

## 1. Intenção

A prefeitura que compra uma suíte de saúde quer, no mesmo sistema, o trabalho de rua contra a dengue e o centro de zoonoses. O Rota Saúde já tem o cadastro dos imóveis, a visita, o app offline, a vigilância, o estoque e a ouvidoria: o 35 junta as peças que faltam para a vigilância ambiental do município.

**Por quê:** 8 de 15 editais pedem zoonoses e endemias, e a pesquisa de mercado marcou como lacuna nossa. Não há sistema nacional vigente que receba o trabalho do agente de endemias por imóvel, nem um para a unidade de zoonoses: o Rota Saúde é o sistema municipal, e o que vai ao estado sai em relatório ou planilha.

**Sucesso (decisão do usuário):** passar na **prova de conceito dos editais** na parte de zoonoses e endemias. Cada requisito que os editais repetem tem um caminho curto na tela, demonstrável com a semente de dev.

**Antiobjetivos:**
- Agente de endemias com duas listas (a visita do 24b e um boletim à parte).
- Dado do morador ou do tutor do animal em URL, log ou evento.
- Regra técnica fixa no código (produto, carga, cadência, raio) que uma nota técnica nova obrigue a reprogramar.
- Construir concorrente do Conta Ovos ou do LIRAa estadual em vez de produzir o dado que eles pedem.

## 2. Decisões

### Decisões do usuário

| # | Decisão |
|---|---|
| 1 | **Os dois:** campo de endemias e centro de zoonoses. |
| 2 | **Sucesso = prova de conceito dos editais;** entra só o que os editais repetem. |
| 3 | **Corte:** 35a (endemias) primeiro, 35b (zoonoses) depois; fica fora o que só Belo Horizonte pede. |
| 4 | **A — boletim dentro da visita do 24b:** a visita do agente de endemias ganha o bloco de inspeção; a ficha LEDI de visita continua saindo da mesma visita. |
| 5 | **Ovitrampas próprias,** com Conta Ovos depois. |
| 6 | **35b no dashboard e no app de campo:** o animal é item novo do cadastro no app. |
| 7 | **B — o 35 usa o cadastro do 24a como está:** o agente de endemias fica numa equipe de saúde da família, com microárea atribuída pelo enfermeiro (`HomeVisits::Access` do 24b), e trabalha só nos imóveis cadastrados; todo imóvel tem microárea e equipe. O denominador do LIRAa e dos ciclos é o dos imóveis cadastrados. |

### Decisões do coordenador (desenho)

**35a — endemias**
1. **Interruptor `vector_control`** por cidade, escrito só pelo maintenance (como `home_visits`); exige `territory_registry`.
2. **Quem faz:** o agente de combate às endemias (CBO 515140) registra boletim, ovitrampa, LIRAa e bloqueio; a **coordenação de endemias** é um papel novo da cidade (planeja ciclos, sorteia LIRAa, abre e fecha bloqueio, lança resultado de amostra, vê relatórios). O agente de saúde pública (352210) entra junto com o api#171.
3. **Imóvel:** o domicílio do 24a ganha **quarteirão** (número dentro do bairro, cadastro da cidade) e **classe de risco** por cor (verde, amarelo, vermelho, cinza, roxo; DN-2025), com data da última avaliação. Tipo de imóvel e ponto estratégico já existem (tipoDeImovel 1–13, 99). Imóvel não residencial que falta é cadastrado no fluxo do 24a (tipos 2–13, sem moradores), na microárea de quem cadastra, como hoje (decisão 7).
4. **Boletim (decisão 4):** bloco opcional da visita do 24b quando quem registra é agente de endemias e a cidade tem `vector_control`:
   - atividade (LI, LI+T, PE, IE, bloqueio, LIRAa) e tipo de trabalho (rotina ou pendência);
   - situação do imóvel: trabalhado, fechado, recusado, desocupado; "recuperado" é a pendência que trabalhou imóvel antes fechado;
   - depósitos por grupo (A1, A2, B, C, D1, D2, E): existentes, com água, com larva, eliminados, tratados;
   - medidas: mecânico, focal, perifocal, nebulização, cada uma com produto e cargas;
   - amostras: um tubito por depósito positivo, numeração sequencial por imóvel, etiqueta com código;
   - hora de entrada e saída e coordenada (24c).
   A ficha LEDI tipo 8 continua derivada: o motivo de controle vetorial (34–37) sai do boletim, sem o agente marcar duas vezes.
5. **Produtos e cargas** são catálogo da cidade (nome, código, unidade, tamanho da carga, uso: focal, perifocal, espacial), não código: mudam por nota técnica.
6. **Consumo pelo estoque do 27a:** o boletim baixa as cargas do estoque da unidade de vigilância ambiental (decisão 29); larvicida e inseticida são itens do catálogo do 27a (o plano confere se o catálogo aceita item não medicamento; se não, a categoria entra no 35a).
7. **Ciclos por risco:** a coordenação define, por área (bairro ou conjunto de quarteirões) e por classe de risco, a cadência de visita em dias; o padrão de fábrica é 60 dias para tudo (ciclo clássico) e PE quinzenal. O sistema mostra, por ciclo, imóveis a visitar, visitados, pendentes e a % de pendência; a folha do dia do agente de endemias traz os imóveis vencidos da sua área.
8. **Ovitrampas (decisão 5):** cadastro da armadilha (número, imóvel, quarteirão, responsável, aceite do morador), instalação e retirada ligadas a um ciclo de leitura, contagem de ovos por palheta (inclusive negativa), avaria e perda. Índices IPO, IDO e IDV por área e semana; mapa por contagem. A contagem alta acende o alerta de intensificar num raio de 200 m (NT 3/2025), que vira tarefa da coordenação.
9. **LIRAa:** sobre os imóveis cadastrados (decisão 7), a coordenação monta os estratos (8.100–12.000 imóveis, ou 2.000–8.100 com 50%), o sistema calcula a amostra (n = 450 / (1 + 450/N)), sorteia os quarteirões por intervalo e 20% dos imóveis de cada um, com substituição do fechado pelo vizinho. Os agentes fazem a inspeção como boletim de atividade LIRAa. Índices IIP, IB e ITR por estrato e município; o relatório sai em PDF e CSV. Classificação do IIP (<1, 1–3,9, ≥4) parametrizável até a fonte oficial confirmar.
10. **Bloqueio:** a notificação de arbovirose (dengue, chikungunya, zika) da fila do 25 propõe um bloqueio com raio de 150 m em torno do domicílio do caso (em quarteirões, decisão 26); casos da mesma área em até 7 dias entram no mesmo bloqueio; até 3 ciclos de UBV, cada um com seus dias. A coordenação confirma, e os imóveis do raio entram na folha dos agentes como atividade "bloqueio". O caso no mapa nunca mostra nome nem endereço do paciente, só o ponto do raio.
11. **Amostras:** o resultado (tubitos examinados, *Ae. aegypti*, *Ae. albopictus*, recipientes positivos por espécie) é lançado pela coordenação ou pelo laboratório municipal com papel próprio; volta ao imóvel e ao índice.
12. **Relatórios (35a):** imóveis trabalhados, fechados, recusados e recuperados por tipo e por ciclo; PE por situação; produção do agente por dia e ciclo; consumo de produto por tipo; ovitrampas; LIRAa; bloqueios. Supressão de números pequenos onde houver dado de pessoa (ADR 0025).
13. **App (35a):** o boletim é parte do item `home_visit` do `field-sync` (MINOR do 35a); ovitrampa (instalação, retirada, contagem no campo se houver) e imóvel de LIRAa e de bloqueio chegam no download da área do agente. O download continua por microárea (decisão 7); quarteirão, risco, ovitrampa, LIRAa e bloqueio vêm como dados a mais dos imóveis dessas microáreas.

**35b — zoonoses**
14. **Interruptor `zoonoses`** por cidade, só pelo maintenance; exige `territory_registry`. Papéis: **médico veterinário** (CBO 223305, conselho CRMV, entra no `cbo_saude.yml`), agente de endemias e um papel "equipe de zoonoses" para técnicos e balcão.
15. **Animal:** cadastro individual (espécie, raça, sexo, pelagem, porte, nascimento estimado, microchip ou registro geral, situação), ligado a um domicílio do 24a e a um tutor (pessoa do cadastro individual) ou "sem tutor" (animal comunitário ou recolhido). Histórico por animal: vacinas, observações, sorologias, coleiras, entradas no canil. O animal vai ao app como item novo do cadastro, ao lado da família (decisão 6).
16. **Vacinação antirrábica:** casa a casa pelo app, em posto fixo no dia da campanha (pelo dashboard ou pelo app) e rotina no centro. Campanha com período e meta. Animal vacinado sem cadastro completo (posto fixo) entra com o mínimo (espécie e tutor ou "sem tutor"). Doses baixam do estoque do 27a. Resultado semanal no formato da planilha estadual (cães e gatos vacinados por semana, cobertura sobre a estimativa), em CSV. Evento adverso fica como registro e lembrete de notificar pelo formulário do MS (sem integração).
17. **Demandas da população:** a ouvidoria do 33 ganha a categoria "zoonoses" (subtipos: escorpião, morcego, roedor, carrapato, animal agressor, animal com sintoma neurológico, esporotricose, outro), que cai numa fila do centro com status, vistoria e retorno ao cidadão pelo fluxo do 33.
18. **Animal agressor:** a notificação de "acidente por animal potencialmente transmissor da raiva" na fila do 25 abre uma tarefa de observação de 10 dias para o centro (cão ou gato passível de observação). O desfecho (sadio, morreu, desapareceu, sinais de raiva, enviado a exame) volta à notificação para a ficha do Sinan, e um aviso vai à unidade que atendeu.
19. **Leishmaniose visceral canina:** inquérito por área (amostral, pela tabela de população canina × prevalência, ou censitário), sorologia TR-DPP e ELISA lançada à mão por veterinário ou equipe de zoonoses, investigação de foco com raio e mínimo de 100 cães, coleira com data de troca em 6 meses e aviso na folha. Desfecho do cão reagente: tratamento pelo tutor, eutanásia (decisão 20) ou recusa.
20. **Canil e gatil:** entrada (recolhimento, entrega voluntária, observação), saída (devolução, adoção, óbito, eutanásia), castração. Eutanásia só registra médico veterinário, com motivo e base (Resolução CFMV 1.000/2012, Lei 14.228/2021), e o registro é imutável.
21. **Relatórios (35b):** vacinação por campanha e semana; demandas por tipo e prazo; observações por desfecho; inquérito e prevalência por área; canil (lotação, entradas e saídas, castrações, eutanásias).
22. **App (35b):** item `animal` e vacinação no `field-sync` (MINOR do 35b), inquérito de leishmaniose no download da área.

**Depois da revisão de construção**
26. **Raio contado em quarteirões:** bloqueio, intensificação da ovitrampa e foco de leishmaniose usam quarteirões como unidade (prática de campo; BH RF38, cerca de 9 quarteirões para 150 m). A cidade cadastra a vizinhança dos quarteirões; o sistema sugere o quarteirão do caso mais os vizinhos até o raio, e a coordenação ajusta. Quando houver coordenada (24c), o raio geométrico entra só como sugestão. Assim o 35a não espera o 24c.
27. **Quarteirão do caso:** vem do domicílio do paciente no 24a, quando ele existe; se não, a coordenação escolhe o quarteirão pelo endereço que vê no detalhe da notificação (leitura com trilha). O mapa e as listas do bloqueio nunca mostram o paciente.
28. **Quando o bloqueio é proposto:** quando o profissional confirma a suspeita e a notificação entra na fila do 25, não na sugestão automática. Caso com residência em outro município só gera aviso, não bloqueio.
29. **Estoque:** a unidade de vigilância ambiental (decisão 30) tem um local de estoque do 27a; o consumo do boletim sai desse local por FEFO, sem lote escolhido em campo. Saldo insuficiente na sincronização **não recusa a visita**: o consumo grava e abre uma pendência de ajuste para a coordenação. A vacina antirrábica animal é insumo desse mesmo domínio do 27a, não do 29 (sala de vacina humana e RNDS).
30. **Unidades:** a Unidade de Vigilância de Zoonoses (CNES tipo 50) e a coordenação de endemias entram como unidades de saúde da cidade (`health_unit`), o que dá local de estoque, vínculo dos profissionais e trilha. Os papéis novos (coordenação de endemias, equipe de zoonoses) valem para a cidade inteira, como os da ouvidoria. O resultado de amostra é lançado pela coordenação; não usa os papéis do laboratório humano do 21.
31. **Ouvidoria (35b):** a demanda entra como manifestação do tipo "solicitação" com assunto "zoonoses" e subtipo; vai direto à fila do centro de zoonoses, sem triagem do ouvidor, e segue os prazos legais do 33. O endereço do local da vistoria é campo novo, cifrado; a manifestação anônima pode ter endereço do local (é do lugar, não da pessoa).
32. **Animal agressor (35b):** a notificação de acidente por animal potencialmente transmissor da raiva no 25 ganha o bloco do animal da ficha do Sinan (espécie, condição, passível de observação) e o endereço e o contato do tutor, cifrados, preenchidos pela unidade que atendeu. A tarefa de observação mostra ao centro só esse bloco, não o paciente.

**Os dois**
23. **LGPD:** tutor e morador seguem as regras do cadastro do 24a (cifra, nunca em URL, log ou evento); coordenada do caso de arbovirose nunca sai do api ligada à pessoa; painéis com supressão. Microchip é identificador do animal, não da pessoa, mas não vai a URL.
24. **Parâmetros técnicos** (produtos, cargas, cadência, raios, janelas, faixas de índice, tabela de amostra canina) são dados com padrão de fábrica tirado das diretrizes de 2025–2026, editáveis pela cidade e com versão.
25. **Semente de dev** para a prova de conceito: uma cidade com quarteirões, PE, ovitrampas com contagens de 4 semanas, um LIRAa sorteado, um caso de dengue com bloqueio aberto (35a); animais com tutor, uma campanha, uma observação em curso e um inquérito (35b). Fictícia, imitando o real (CBO, CNS e CPF com dígito válido, vínculos coerentes).

## 3. Dados (resumo)

| Entidade | Entrega | Notas |
|---|---|---|
| `blocks` (quarteirão) | 35a | por bairro, número, ligado aos domicílios |
| `household` + `block_id`, `risk_class`, `risk_assessed_on` | 35a | colunas novas no 24a |
| `vector_inspections` | 35a | 1:1 com a visita do 24b, depósitos e medidas como filhos |
| `vector_samples` | 35a | tubito, etiqueta, resultado |
| `vector_products` | 35a | catálogo da cidade, ligado ao item do 27a |
| `vector_cycles`, `vector_cadences` | 35a | cadência por área e risco |
| `ovitraps`, `ovitrap_readings` | 35a | instalação, retirada, contagem |
| `liraa_surveys`, `liraa_strata`, `liraa_draws` | 35a | estrato, amostra, sorteio |
| `transmission_blocks` | 35a | raio, casos agrupados, ciclos |
| `animals`, `animal_events` | 35b | histórico por animal |
| `rabies_campaigns`, `rabies_vaccinations` | 35b | |
| `bite_observations` | 35b | 10 dias, ligada à notificação do 25 |
| `leish_surveys`, `leish_tests`, `collars` | 35b | |
| `shelter_stays` | 35b | canil e gatil, castração, eutanásia |

Os nomes finais ficam com o contrato de cada entrega.

## 4. Telas

- **Dashboard (35a):** quarteirões e risco no mapa da área; planejamento de ciclos; folha do agente de endemias (do 24b, com os vencidos, LIRAa e bloqueio); boletim no detalhe da visita; ovitrampas (lista, mapa, contagem); LIRAa (estratos, sorteio, índices); bloqueios (proposta, raio no mapa, ciclos); amostras (fila de resultado); relatórios.
- **Dashboard (35b):** animais (busca por microchip, tutor, domicílio); campanha e posto fixo; fila de demandas de zoonoses; observações; inquéritos e coleiras; canil e gatil; relatórios.
- **App (24c):** boletim na visita; ovitrampa; imóveis do LIRAa e do bloqueio na folha; animal no domicílio; vacinação casa a casa; coleta do inquérito.
- **wpda (35b):** categoria "zoonoses" na ouvidoria.

## 5. Testes

- api: specs de cada regra técnica com os números das diretrizes (amostra do LIRAa, índices, raio e agrupamento do bloqueio, cadência, IPO/IDO/IDV, janela da observação, troca de coleira), inclusive bordas (estrato fora da faixa, imóvel fechado substituído, caso no limite de 7 dias, contagem zero).
- Guarda de vazamento: nenhum dado de morador ou tutor em URL, log, evento ou resposta de mapa.
- Ficha LEDI tipo 8 do agente de endemias com o motivo derivado do boletim.
- dashboard: vitest das telas; prova no navegador seguindo um **roteiro de prova de conceito** (um caminho por requisito repetido nos editais) com a semente.
- App: Jest e Maestro do 24c para os itens novos, offline.
- Suíte completa pela sessão "Testes" no fim de cada entrega.

## 6. Entrega e dependências

- **35a:** api e dashboard não dependem do 24c; o que é do app entra depois que o 24c estiver mergeado. O contrato do 35a define a MINOR do `field-sync`.
- **35b:** depois do 35a; depende do 33 (categoria nova) e do 25 (notificação de raiva). A parte do app, como no 35a.
- Interruptores desligados por padrão; ligar em cidade real exige runbook (produtos, cadência, quarteirões).

## 7. F-IDs

| ID | Funcionalidade | Superfície | Entrega |
|---|---|---|---|
| F-35.1 | Interruptor de vigilância ambiental, coordenação de endemias, quarteirão e risco do imóvel | api, dashboard, maintenance | 35a |
| F-35.2 | Boletim de campo na visita do agente de endemias, com a ficha LEDI derivada | api, dashboard | 35a |
| F-35.3 | Produtos, cargas e consumo pelo estoque | api, dashboard | 35a |
| F-35.4 | Ciclos por risco, folha do agente e produção | api, dashboard | 35a |
| F-35.5 | Ovitrampas: instalação, retirada, contagem, índices e mapa | api, dashboard | 35a |
| F-35.6 | LIRAa: estratos, sorteio, índices e relatório | api, dashboard | 35a |
| F-35.7 | Bloqueio de transmissão a partir da notificação | api, dashboard | 35a |
| F-35.8 | Amostras com etiqueta e resultado lançado | api, dashboard | 35a |
| F-35.9 | Relatórios de endemias | api, dashboard | 35a |
| F-35.10 | App de campo: boletim, ovitrampa, LIRAa e bloqueio offline | contracts, api, campo | 35a |
| F-35.11 | Interruptor de zoonoses, veterinário e cadastro do animal com tutor | api, dashboard, maintenance | 35b |
| F-35.12 | Vacinação antirrábica: campanha, posto fixo, rotina e resultado semanal | api, dashboard | 35b |
| F-35.13 | Demandas de zoonoses pela ouvidoria | api, dashboard, wpda | 35b |
| F-35.14 | Observação do animal agressor ligada à notificação de raiva | api, dashboard | 35b |
| F-35.15 | Leishmaniose canina: inquérito, sorologia e coleira | api, dashboard | 35b |
| F-35.16 | Canil e gatil: entrada, saída, castração e eutanásia | api, dashboard | 35b |
| F-35.17 | Relatórios de zoonoses | api, dashboard | 35b |
| F-35.18 | App de campo: animal, vacinação e inquérito offline | contracts, api, campo | 35b |

## 8. Riscos e pendências

- **Cobertura do cadastro (decisão 7):** onde não há agente comunitário nem equipe, não há imóvel, e o LIRAa e os ciclos ficam incompletos. Os relatórios mostram a cobertura (imóveis cadastrados por bairro) ao lado dos índices; território próprio de endemias, imóvel sem microárea e ficha LEDI do agente sem equipe ficam para depois, em card.
- Tabela oficial de códigos de tipo de imóvel do boletim (R/C/TB/PE/O) e faixas do IIP não confirmadas; ficam parametrizadas (decisão 24).
- Catálogo do 27a para item não medicamento: o plano do api do 35a confere (decisão 6).
- A parte de app das duas entregas espera o 24c.
- Gates de go-live: diretriz vigente conferida com a vigilância ambiental de uma cidade piloto; formato da planilha estadual da campanha antirrábica da UF piloto; ficha 8 do agente de endemias aceita no PEC real.
- Card para depois: Conta Ovos (leitura e envio), exportação do LIRAa para o programa estadual, integração com sistemas estaduais.
