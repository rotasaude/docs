# Pesquisa — App de campo offline (24c)

**Data:** 2026-11-13 · **Para:** `superpowers/specs/2026-11-13-module-24c-field-app-design.md` · **ADR:** 0053

Consulta técnica feita no brainstorm do 24c. "Não confirmado" marca o que precisa de prova.

## 1. Coordenadas no LEDI 8.7

- **Cadastro domiciliar (tipo 3, `CadastroDomiciliarThrift`):** `17:optional double latitude`, `18:optional double longitude` (IDL vendorizada em `apps/api/vendor/ledi/8.7.0/idl/ras`; o dicionário numera 15/16, vale a IDL). A numeração mudou em 2024-07.
- **Visita (tipo 8, `FichaVisitaDomiciliarChildThrift`):** `20:optional double latitude`, `21:optional double longitude` ("latitude do imóvel onde a visita foi realizada").
- **Regras:** `xs:double`, `minOccurs=0`, um exige o outro, faixas −90..90 e −180..180, até 8 casas.
- Os campos entraram no repositório oficial em 2021-05-04 (commit `c12d6d904f`); a versão LEDI correspondente não está confirmada.
- Fontes: https://github.com/laboratoriobridge/esusaps-integracao · https://integracao.esusaps.bridge.ufsc.tech/ledi/documentacao/

## 2. e-SUS Território (precedente)

- App oficial do Laboratório Bridge/UFSC: banco local completo, tudo offline, rede só para sincronizar; mapa com bibliotecas abertas, segmentado por UF; coordenada do imóvel desde a 3.3.0; imóveis agrupados no mapa; pensado para GPS de baixa precisão; gera as fichas de cadastro domiciliar, individual e de visita.
- Fontes: Celuppi et al., WebMedia 2022 (https://sol.sbc.org.br/index.php/webmedia_estendido/article/view/21997) · guia de bolso do DF, versão 5.0.9 (2025).

## 3. Mapa offline a partir do OpenStreetMap

- **Protomaps:** build diária do planeta em PMTiles (z0–15, ~120 GB), guardada por uma semana; `pmtiles extract --bbox|--region --maxzoom` recorta de fonte remota; a Protomaps pede cópia em infraestrutura própria, sem hotlink. Planetiler e tilemaker são alternativas de geração.
- **Geofabrik (2026-11-12):** Brasil 2,0 GB, Sul 410 MB, Sudeste 823 MB.
- **Tamanho por município:** não confirmado; referências de 2 MB (cidade pequena) a ~68 MB (Berlim). Medir com `--dry-run` para Curitiba e Maringá.
- **Estilo, fontes e ícones:** `protomaps/basemaps-assets` (glyphs PBF e sprites) para hospedar localmente; carga por `asset://` no MapLibre Android não confirmada em documentação.
- **Licença ODbL:** mapa é Produced Work; atribuição "© OpenStreetMap" visível sem interação (pode recolher depois de interação ou 5 s, com "(i)"). https://osmfoundation.org/wiki/Licence/Attribution_Guidelines
- **MapLibre:** PMTiles no MapLibre Android desde 11.7/11.8; exige caminho absoluto `pmtiles://file:///…`; não funciona como `asset://` nem em offline pack. `@maplibre/maplibre-react-native` v11.5.1 (2026-10-29), só nova arquitetura, config plugin, sem Expo Go. `pmtiles://file://` no wrapper React Native não confirmado. Reserva: `OfflineManager.createPack`, que exige servidor z/x/y.

## 4. Banco cifrado

- `expo-sqlite` com `useSQLCipher` desde o SDK 52; chave por `PRAGMA key`. Sem SQLCipher no binário, o `PRAGMA key` é ignorado em silêncio: conferir `PRAGMA cipher_version`.
- `op-sqlite` com SQLCipher conflita com `expo-sqlite` no mesmo app.
- Chave bruta (`x'hex'`) evita o custo do PBKDF2; uma conexão aberta.
- `expo-secure-store`: SharedPreferences cifradas pelo Keystore; perde-se ao desinstalar; com `requireAuthentication` é invalidada quando a biometria muda.

## 5. Versões e testes

- Expo SDK 58 (2026-10-15), React Native 0.88; `jest-expo` exige Jest 30; React Native Testing Library.
- Maestro CLI 2.12.1; Android API 29–34 confirmados (35/36 não conferidos). CI com `reactivecircus/android-emulator-runner@v2` em runner com KVM, contra APK release.

## 6. Build e distribuição

- Build local: `expo run:android --variant release` (sem assinatura) ou prebuild + Gradle com `signingConfigs`; guardar a chave fora do `android/` (o `prebuild --clean` apaga). R8 ligado por padrão no SDK 58.
- Google Play gerenciado: app privado pela EMM, aprovação em ~10 min; AAB aceito desde 2025-03.
- **Verificação de desenvolvedor Android obrigatória no Brasil desde 2026-09-30**, em aparelhos certificados, qualquer origem; isenção por AMAPI/DPC não confirmada em documentação oficial. https://developer.android.com/developer-verification/guides

## 7. Localização

- Android 12+: o usuário escolhe precisa ou aproximada; pedir FINE e COARSE juntas. `expo-location` informa `accuracy` em metros; `Accuracy.High` ~10 m; sem segundo plano por padrão. GPS de celular ~4,9 m a céu aberto.
- Concessão da permissão de localização pelo DPC em aparelho totalmente gerenciado: não confirmada.
