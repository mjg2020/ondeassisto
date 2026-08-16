# TMDB API — Ficha técnica de engenharia
**Escopo:** catálogo público de filmes disponíveis em streaming no Brasil (`watch_region=BR`).
**Regra de leitura:** tudo abaixo é fato confirmado em fonte primária, salvo a última seção.

---

## Autenticação

| Item | Valor |
|---|---|
| Métodos aceitos (v3) | query param `api_key` **ou** access token como header `Authorization: Bearer <token>` |
| Método documentado como padrão | Bearer (API Read Access Token) — subtítulo literal da página: *"The default way to authenticate."* |
| Nível de acesso | Idêntico nos dois. Doc: *"Both authentication methods provide the same level of access, and which one you choose is completely up to you."* Nenhum é chamado de "recomendado"; `api_key` **não** é depreciado |
| OpenAPI oficial da TMDB | Declara **um único** security scheme: `{"type":"apiKey","in":"header","name":"Authorization","x-bearer-format":"bearer"}`. `api_key` não existe no spec machine-readable |
| Base URL | `https://api.themoviedb.org` |

Exemplo doc: `curl --request GET --url 'https://api.themoviedb.org/3/movie/11' --header 'Authorization: Bearer <<access_token>>'`

**Recomendação de engenharia (não é texto da TMDB):** usar o Read Access Token via header para backend read-only — é o padrão documentado, é o único esquema do OpenAPI, serve v3 e v4 com uma credencial, e não vaza em logs de proxy/CDN, histórico ou `Referer`. Não apresentar o argumento de vazamento como controle documentado pela TMDB.

Fonte: https://developer.themoviedb.org/docs/authentication-application · https://developer.themoviedb.org/reference/configuration-details

---

## Endpoints principais

| Endpoint | Retorno / notas |
|---|---|
| `GET /3/discover/movie` | Enumeração e filtragem do catálogo. Topo da resposta: `page`, `results`, `total_pages`, `total_results` |
| `GET /3/discover/tv` | Contrato de watch-provider **byte-idêntico** ao de filmes; `sort_by` diferente (ver Paginação) |
| `GET /3/movie/{movie_id}/watch/providers` | `{id, results}` — `results` é **objeto** chaveado por código ISO 3166-1 alpha-2 (`"BR"`, `"US"`, …), **não** array; não há campo `iso_3166_1` dentro dos buckets |
| `GET /3/watch/providers/movie` | `{ "results": [...] }` com todos os provedores de filmes. Params: `language` (default `en-US`), `watch_region` (opcional, filtra por país) |
| `GET /3/watch/providers/regions` | Países com dados de OTT. Único param: `language`. Campos por item: `iso_3166_1`, `english_name`, `native_name` — nada mais |
| `GET /3/configuration` | `images.base_url`, `images.secure_base_url`, arrays de tamanhos, `change_keys` |
| Changes API (movie/person/tv) | Listas de IDs alterados nas últimas **24 h** por default, estendível a **14 dias** |
| Daily ID Exports | `https://files.tmdb.org/p/exports/movie_ids_MM_DD_YYYY.json.gz` |

**Objeto de item em `/discover/movie` — campos exatos:** `adult`, `backdrop_path`, `genre_ids`, `id`, `original_language`, `original_title`, `overview`, `popularity`, `poster_path`, `release_date`, `title`, `video`, `vote_average`, `vote_count`. **Nenhum campo de disponibilidade.** `with_watch_providers` é filtro de entrada apenas; o dado de disponibilidade nunca é ecoado. → padrão N+1: uma chamada extra a `/movie/{id}/watch/providers` por título.

**Estrutura de `/movie/{id}/watch/providers`** — dentro de cada bucket de país:
- `link` (um por país, nível do país, irmão dos arrays): `https://www.themoviedb.org/movie/550-fight-club/watch?locale=BR` → padrão `https://www.themoviedb.org/movie/{id}-{slug}/watch?locale={COUNTRY}`. É página TMDB, **não** deep link do provedor.
- Arrays irmãos por tipo de monetização, presentes **só quando há oferta** — obrigatório null-check em cada chave.
- **Correção sobre o dossiê:** no exemplo oficial (Fight Club, id 550, 95 buckets de país) o conjunto completo de chaves é exatamente `{link, flatrate, rent, buy, ads}`. **`free` nunca aparece** como chave nesse exemplo (embora `free` seja valor válido de `with_watch_monetization_types`). `ads` aparece em **ES e HR**. O bucket **BR** do exemplo tem apenas `{link, flatrate}`.
- Objeto de provedor aqui tem **4 campos**: `logo_path`, `provider_id`, `provider_name`, `display_priority`. **`display_priorities` não existe neste endpoint.** Ex. BR: `{"logo_path":"/emthp39XA2YScoYL1p0sdbAH2WA.jpg","provider_id":119,"provider_name":"Amazon Prime Video","display_priority":2}`
- Doc: *"This is not going to return full deep links, but rather, it's just enough information to display what's available where."*

**Objeto de provedor em `/watch/providers/movie` — 5 campos:** `provider_name`, `provider_id`, `logo_path`, `display_priority`, `display_priorities`. `display_priority` = ordenação global; `display_priorities` = mapa `{país: inteiro}` com prioridade regional. Menor valor = mais alto na lista. Ex.: `{"provider_name":"Apple TV","provider_id":2,"logo_path":"/peURlLlr8jggOwK53fJ5wdQl05y.jpg","display_priority":2,"display_priorities":{"CA":6,"AE":1,"US":4,…}}`

**Correção sobre o dossiê (contagens do exemplo da doc):**
- `/3/watch/providers/movie`: **529** objetos de provedor (529 `provider_id` únicos) — não ~630. Irmão `/3/watch/providers/tv`: 474.
- `/3/watch/providers/regions`: **120** entradas, de Andorra (AD) a **Zâmbia (ZM)** — não 200+. **Zimbábue (ZW) não aparece** no exemplo.

Fontes: https://developer.themoviedb.org/reference/discover-movie · https://developer.themoviedb.org/reference/discover-tv · https://developer.themoviedb.org/reference/movie-watch-providers · https://developer.themoviedb.org/reference/watch-providers-movie-list · https://developer.themoviedb.org/reference/watch-providers-available-regions · https://developer.themoviedb.org/docs/tracking-content-changes · https://developer.themoviedb.org/docs/daily-id-exports

---

## Filtro por streaming

### Os três parâmetros

| Param | Tipo | Descrição verbatim na doc |
|---|---|---|
| `with_watch_providers` | string | *"use in conjunction with `watch_region`, can be a comma (`AND`) or pipe (`OR`) separated query"* |
| `watch_region` | string | *"use in conjunction with `with_watch_monetization_types ` or `with_watch_providers `"* (espaços residuais dentro das crases, como renderizado). É sua **única** descrição documentada |
| `with_watch_monetization_types` | string | *"possible values are: [flatrate, free, ads, rent, buy] use in conjunction with `watch_region`, can be a comma (`AND`) or pipe (`OR`) separated query"* |
| `without_watch_providers` | string | Existe como filtro de exclusão. Sem texto de descrição no schema |

- **Nenhum dos três é marcado `required`** no schema da página de referência. A obrigatoriedade mútua é semântica/comportamental, não imposta pelo schema — a API aceita `with_watch_providers` sozinho e devolve HTTP 200.
- **Enviar `with_watch_providers` sem `watch_region` retorna resultados NÃO filtrados, silenciosamente, sem erro.** Tratar `watch_region` como obrigatório de fato. (https://www.themoviedb.org/talk/61cfcb75028420001c642ce8)
- `watch_region` recebe código **ISO 3166-1**; valores válidos vêm de `/3/watch/providers/regions`.
- **Um único `watch_region` por requisição.** É possível fazer OR de vários provedores na mesma chamada (`with_watch_providers=8|119|337&watch_region=BR`), mas é necessária uma requisição por país. (https://www.themoviedb.org/talk/6049f37d1d78f200577b9274)
- Chamada canônica do catálogo de assinatura de um provedor no Brasil:
  `/3/discover/movie?watch_region=BR&with_watch_providers=8&with_watch_monetization_types=flatrate&language=pt-BR`
- **Não existe lookup reverso** nos endpoints de watch provider — `/discover/movie` é o único caminho prático para enumerar o catálogo de um provedor numa região.

### Ausência de data de disponibilidade (definitivo)

**Não existe, em nenhum ponto da API TMDB, campo indicando quando um título entrou num provedor** — sem `date_added`, sem início/fim de disponibilidade, sem timestamp. Verificado por enumeração exaustiva de campos em `/movie/{id}/watch/providers`, `/watch/providers/movie`, `/watch/providers/regions` e `/discover/movie`. Não há filtro nem `sort_by` correspondente, e `watch_providers` não é change key documentada. Toda opção de data em `sort_by` refere-se ao lançamento primário/teatral, nunca à disponibilidade em streaming.

**Consequência de arquitetura:** uma feature "novidades no streaming BR" exige snapshot próprio agendado de `/discover/movie` (ou `/movie/{id}/watch/providers`) e diff entre execuções sucessivas. A TMDB não fornece isso.

### provider_id — Brasil

IDs confirmados no exemplo oficial de `/watch/providers/movie`:

| Provedor | provider_id | logo_path |
|---|---|---|
| Apple TV | 2 | `/peURlLlr8jggOwK53fJ5wdQl05y.jpg` |
| Netflix | 8 | `/t2yyOv40HZeVlLjYsCsPHnWLk4W.jpg` |
| MUBI | 11 | `/bVR4Z1LCHY7gidXAJF5pMa4QrDS.jpg` |
| Amazon Prime Video | 119 (e também 9) | `/emthp39XA2YScoYL1p0sdbAH2WA.jpg` |
| Crunchyroll | 283 | `/8Gt1iClBlzTeQs8WQm8UrCoIxnQ.jpg` |
| Globoplay | 307 | `/oBoWstXQFHAlPApyxIQ31CIbNQk.jpg` |
| Disney Plus | 337 | `/7rwgEs15tFwyR9NPQ5vpzxTj19Q.jpg` |
| Apple TV Plus | 350 | `/6uhKBfmtzFqOcLousHwZuzcrScK.jpg` |
| HBO Max | 384 | `/Ajqyt5aNxNGjmF9uOfxArGrdf3X.jpg` |
| Paramount Plus | 531 | `/xbhHHa1YgtpwhC8lb1NQ3ACVcLd.jpg` |

- **Uma marca pode ter múltiplos `provider_id`**, atribuídos pela JustWatch e variando por país — Amazon Prime Video aparece como **9 e 119** com o mesmo `logo_path`. Travis Bell (staff TMDB): *"I've noticed there has been different providers created in some different countries and sometimes the extra ID may only have a few items added to it."* → casar por **conjunto de IDs**, nunca por ID único, e puxar a lista autoritativa via `/3/watch/providers/movie?watch_region=BR` em vez de hardcode. (https://www.themoviedb.org/talk/6066b49563d71300402c7672)
- **Correção sobre o dossiê — HBO Max:** o ID **1899** está ausente do exemplo da doc (que traz HBO Max = 384 e nenhum provedor chamado "Max"), mas **1899 está vivo e rotulado "HBO Max"** na página de browse BR da TMDB, na posição 9. O **provider_id 384 não aparece na lista BR ao vivo**. Consultar a API antes de fixar qualquer um dos dois. (fonte da alegação original — https://www.themoviedb.org/talk/638e664dede1b0007c59e2f2 — não sustenta o número 1899; é uma thread sobre "Network ID" que nunca menciona 1899 nem "Max")
- Brasil tem **~85 provedores de filmes distintos** na página de browse da TMDB (`https://www.themoviedb.org/movie?watch_region=BR`). Trabalhar com 80–90 como faixa. Cauda longa dominada por revendas tipo "X Amazon Channel" (Telecine Amazon Channel, Paramount+ Amazon Channel, HBO Max Amazon Channel, MUBI Amazon Channel), que são `provider_id` separados do serviço-mãe.
- Ordem de exibição no topo da lista BR: Netflix, Amazon Prime Video, Apple TV, Disney Plus, JustWatch TV, Claro video, Looke, Paramount Plus, HBO Max, Apple TV Store, Globoplay, Crunchyroll, MUBI, Amazon Video, Google Play Movies, NetMovies. **Telecine só existe como "Telecine Amazon Channel".** A TMDB distingue "Apple TV" (assinatura) de "Apple TV Store" (transacional).

### Demais filtros de `/discover/movie`

| Param | Tipo / notas |
|---|---|
| `with_genres` | comma (`AND`) ou pipe (`OR`), IDs de gênero TMDB. Mesma redação em `with_cast`, `with_companies`, `with_crew`, `with_keywords`, `with_people` |
| `without_genres`, `without_companies`, `without_keywords` | filtros de exclusão |
| `primary_release_date.gte` / `.lte` | date string (`YYYY-MM-DD`), sem descrição no schema |
| `release_date.gte` / `.lte` | irmãos, limitam qualquer data de lançamento |
| `primary_release_year`, `year` | inteiros |
| `vote_average.gte` / `.lte` | **float** |
| `vote_count.gte` / `.lte` | **float** |
| `with_runtime.gte` / `.lte` | inteiro, em minutos |
| `include_adult`, `include_video` | boolean, default `false` |
| `with_original_language` | string, código de idioma. Sem descrição. Distinto de `language` (default `en-US`, controla o idioma dos **metadados retornados**) e de `with_origin_country` |
| `region` | linha **separada** de `watch_region` no schema, sem descrição própria. `certification`, `certification.gte` e `certification.lte` leem *"use in conjunction with `region`"*; `with_release_type` lê *"possible values are: [1, 2, 3, 4, 5, 6] can be a comma (`AND`) or pipe (`OR`) separated query, can be used in conjunction with `region`"*. **`region` não filtra por streaming** |

Para metadados em português: `language=pt-BR`. Manter `include_video=false` mantém trailers/extras fora do catálogo.

Fontes: https://developer.themoviedb.org/reference/discover-movie · https://developer.themoviedb.org/reference/watch-providers-movie-list · https://www.themoviedb.org/movie?watch_region=BR

---

## Paginação e limites

- `page` em `/discover/movie`: tipo integer, **default 1**, sem descrição e **sem min/max no schema da página de referência**. O teto só é descobrível na página de erros.
- **Teto rígido: 500 páginas.** Erro TMDB status_code **22** / HTTP **400**: *"Invalid page: Pages start at 1 and max at 500. They are expected to be an integer."* `page=501` falha — não retorna conjunto vazio.
- **Teto efetivo ≈ 10.000 resultados por query** (500 páginas × 20 por página). O valor de 20 itens/página **não está declarado em nenhuma doc primária** — é comportamento observado. `total_pages` na resposta reporta valores muito acima de 500 (40.000+), inalcançáveis.
- **Contorno do teto de 500 páginas:** fatiar a query por janelas de data com `primary_release_date.gte` / `primary_release_date.lte` (por ano ou década), mantendo `total_results` de cada fatia abaixo da janela alcançável. Recomendado no fórum de suporte da TMDB; sem página oficial. (https://www.themoviedb.org/talk/66f6d91fb9fd27627950d0b4)

**`sort_by` em `/discover/movie` — 14 valores, default `popularity.desc`:**
`original_title.asc|desc`, `popularity.asc|desc`, `revenue.asc|desc`, `primary_release_date.asc|desc`, `title.asc|desc`, `vote_average.asc|desc`, `vote_count.asc|desc`.

**Não existe ordenação por "adicionado recentemente ao streaming".** O enum é exaustivo e cobre apenas título/popularidade/receita/data de lançamento/votos.

**`sort_by` em `/discover/tv` — enum diferente:** `first_air_date`, `name`, `original_name`, `popularity`, `vote_average`, `vote_count` (×`.asc`/`.desc`). **Sem `revenue` e sem `title`** → prateleiras de filme e série não compartilham chave de ordenação além de popularidade e votos.

**Ordenações práticas para o catálogo:**
- `popularity.desc` (default) → prateleira de tendências.
- `primary_release_date.desc` → prateleira "mais recentes"; combinar com `primary_release_date.lte=<hoje>` para excluir não lançados.
- `vote_average.desc` → inutilizável sozinho (títulos obscuros com 1–2 votos sobem ao topo). **Sempre parear com `vote_count.gte`** (usualmente 200–300). A TMDB não aplica correção bayesiana/ponderada a `vote_average` no discover. *O guard de `vote_count` é recomendação de engenharia, não texto da doc.*

**Outros limites da tabela de erros:** code 20 / HTTP 422 *"Invalid date range: Should be a range no longer than 14 days"*; code 27 / HTTP 400 *"Too many append to response objects: The maximum number of remote calls is 20."*; code 46 / HTTP 503 (manutenção); code 9 / HTTP 503 (service offline).

Fontes: https://developer.themoviedb.org/docs/errors · https://developer.themoviedb.org/reference/discover-movie · https://developer.themoviedb.org/reference/discover-tv

---

## Imagens

- Montagem da URL exige **exatamente 3 peças**: `base_url` + `file_size` + `file_path`. Doc: *"In order to generate a fully working image URL, you'll need 3 pieces of data."* As duas primeiras vêm de `/configuration`.
- `images.base_url` = `http://image.tmdb.org/t/p/` · `images.secure_base_url` = `https://image.tmdb.org/t/p/`. FAQ: *"What about SSL? It's currently available API wide… We strongly recommend you use SSL."* → **sempre `secure_base_url`**.

| Array | Valores |
|---|---|
| `backdrop_sizes` | `w300`, `w780`, `w1280`, `original` |
| `logo_sizes` | `w45`, `w92`, `w154`, `w185`, `w300`, `w500`, `original` |
| `poster_sizes` | `w92`, `w154`, `w185`, `w342`, `w500`, `w780`, `original` |
| `profile_sizes` | `w45`, `w185`, `h632`, `original` |
| `still_sizes` | `w92`, `w185`, `w300`, `original` |

- Exemplo: `https://image.tmdb.org/t/p/w500/1E5baAaEse26fej7uHcjOgEE2t2.jpg`
- **Logos de empresa/rede:** disponíveis em SVG e PNG, mas *"All of the `logo_path` fields will return a .png. This is to maintain backwards compatibility since SVG support was added after the fact."* Para SVG: *"you should call the original image size since we don't resize them."*
- **Logos de watch provider:** os `logo_path` são **`.jpg`** → retângulos opacos, sem transparência. Planejar o card considerando isso.
- Logo de provedor: `{secure_base_url}{size}{logo_path}`, ex. `https://image.tmdb.org/t/p/w92/t2yyOv40HZeVlLjYsCsPHnWLk4W.jpg` (Netflix).

Fontes: https://developer.themoviedb.org/docs/image-basics · https://developer.themoviedb.org/reference/configuration-details

---

## Rate limit e cache

**Rate limit**
- Limite legado (40 req / 10 s) **desabilitado em 16 de dezembro de 2019**.
- Limite atual, verbatim: *"we do still have some upper limits to help mitigate needlessly high bulk scraping. They sit somewhere in the 40 requests per second range. This limit could change at any time so be respectful of the service we have built and respect the `429` if you receive one."* → **teto soft, não contratual, ~40 req/s**.
- Estouro: HTTP **429**, status_code TMDB **25**, mensagem *"Your request count (#) is over the allowed limit of (40)."* O "(40)" é placeholder herdado da era do limite legado — **não** ler como valor por segundo.
- **Nenhum rate limit por IP está documentado.** A afirmação recorrente de que "o limite é por IP e não por chave" vem de posts de fórum, não da documentação. Relevante para egress NAT com IP único.
- **Nenhum header `Retry-After` ou `X-RateLimit-*` é documentado.** Implementar backoff exponencial disparado pelo próprio 429 e ler `Retry-After` oportunisticamente, sem depender dele.
- **Nenhuma cota diária ou mensal** existe para chave gratuita. O único teto documentado é o ~40 req/s, somado à cláusula de ToS contra *"an excessive amount of bandwidth"*.
- **Sem SLA.** FAQ: *"We do not currently provide an SLA. However, we do make every reasonable attempt to keep our service online and accessible."* Status: https://status.themoviedb.org

**Cache**
- **Limite contratual: 6 meses.** ToS §1.C: *"Cache, for longer than 6 months, any information obtained through or from TMDB or the TMDB APIs."* É o **único** prazo do documento; não há prazo menor específico para dados de watch provider. → job de TTL/refresh/purge chaveado por timestamp de coleta, com evidência auditável.
- **Na rescisão:** ToS §1.D — cessar uso imediatamente e *"promptly delete or otherwise purge all TMDB Content, including any cached content."*
- **Sem TTL recomendado para `/configuration`** na doc atual. A orientação "verifique a cada poucos dias" existia na doc v3 antiga e **não** está nas páginas atuais. Valores são efetivamente estáticos; cache de 24 h com refresh diário é seguro e conservador — mas é decisão de engenharia, não política documentada.
- **Nenhum suporte a ETag / Last-Modified / requisição condicional é documentado para respostas JSON** de `api.themoviedb.org`.
- **A CDN de imagens (`image.tmdb.org`) retorna ETag e Last-Modified e honra `If-None-Match` com 304** — verificado empiricamente (BunnyCDN, `Cache-Control: public, max-age=31919000` ≈ 369 dias, `CDN-Cache: HIT`). É comportamento medido, não contrato documentado; a config da CDN pode mudar. Imagens são imutáveis por path → cachear agressivamente e não proxiar pela origem própria sem necessidade.
- **Padrão de sincronização incremental sancionado:** Changes API. *"These endpoints will return a list of items that have been changed in the past 24 hours (by default but can be extended to 14 days)."* + *"It's generally a good idea to stay in sync with our changes so you can display the most up to date and accurate information."* Janela máxima de 14 dias imposta pelo erro code 20.
- **Carga em massa:** Daily ID Exports em `https://files.tmdb.org`, **sem autenticação**. Job diário inicia ~07:00 UTC, arquivos disponíveis até 08:00 UTC. Retenção: *"These files are only made available for 3 months after which they are automatically deleted."* Padrão `/p/exports/movie_ids_MM_DD_YYYY.json.gz` (também `tv_series_ids`, `person_ids`, `collection_ids`, `tv_network_ids`, `keyword_ids`, `production_company_ids`, e variantes `adult_*` desde 05/07/2023). Formato: *"These files themselves are not a valid JSON object. Instead, each line is."* (NDJSON gzipado).
- Combinação obrigatória: teto de 6 meses de cache + cláusula de "excessive bandwidth" ⇒ o refresh do catálogo BR completo tem de ser throttled e incremental (Changes API + exports), nunca re-crawl integral.

Fontes: https://developer.themoviedb.org/docs/rate-limiting · https://developer.themoviedb.org/docs/errors · https://developer.themoviedb.org/docs/faq · https://developer.themoviedb.org/docs/tracking-content-changes · https://developer.themoviedb.org/docs/daily-id-exports · https://www.themoviedb.org/api-terms-of-use

---

## Atribuição e termos

**Documento vinculante:** TMDB API Terms of Use, **última atualização 20 de outubro de 2023**. Operado por **TiVo Platform Technologies LLC**. Aceite por uso: *"BY USING THE TMDB APIS, YOU UNCONDITIONALLY CONSENT AND AGREE TO BE BOUND BY…"* Lei aplicável: Califórnia; foro exclusivo: Santa Clara County (§10.C). https://www.themoviedb.org/api-terms-of-use

### Atribuição TMDB — duas redações oficiais, divergentes

| Origem | Texto verbatim |
|---|---|
| **ToS §3 (contrato)** | *"You must place the following notice prominently in or on Your Application: 'This [website, program, service, application, product] uses TMDB and the TMDB APIs but is not endorsed, certified, or otherwise approved by TMDB.'"* |
| FAQ (documentação) | *"You shall place the following notice prominently on your application: 'This product uses the TMDB API but is not endorsed or certified by TMDB.'"* + *"the attribution must be within your application's 'About' or 'Credits' type section."* |

**Usar a redação do ToS**, adaptada: *"This website uses TMDB and the TMDB APIs but is not endorsed, certified, or otherwise approved by TMDB."* O ToS é o contrato; o FAQ é documentação.

**Logo TMDB é obrigatório**, não opcional — o texto de aviso sozinho não basta. ToS §3: *"You must use the TMDB logo to identify Your use of TMDB, the TMDB APIs, or TMDB Content. Any use of any TMDB logos in Your Application must be less prominent than the logos or marks that primarily describe or identify Your Application…"*

**Regras de marca** (https://www.themoviedb.org/about/logos-attribution): usar um dos arquivos de logo aprovados, **sem modificação** de cor, proporção, espelhamento ou rotação; apenas cores da marca, branco ou preto; apenas os nomes **"TMDB"** ou **"The Movie Database"** — *"Any other name is not acceptable"*; links de volta apontando para `https://www.themoviedb.org`. Cores aprovadas: **#0d253f** (azul escuro primário, RGB 13,37,63), **#01b4e4** (azul claro secundário, RGB 1,180,228), **#90cea1** (verde claro terciário, RGB 144,206,161). Cinco variantes SVG aprovadas. **Sem regra publicada de tamanho mínimo ou clear space.** Merchandising/embalagem exige aprovação prévia.

### Atribuição JustWatch

- Callout **"JustWatch Attribution Required"**, verbatim: *"In order to use this data you must attribute the source of the data as **JustWatch**. If we find any usage not complying with these terms we will revoke access to the API."*
- **Correção sobre o dossiê — escopo:** o callout aparece **apenas nos endpoints por título** (`/movie/{id}/watch/providers` e `/tv/{id}/watch/providers`, ambos atualizados em 01/10/2025, texto idêntico). As páginas de `/watch/providers/movie` e `/watch/providers/regions` contêm **zero ocorrências** da string "JustWatch" — a exigência não é reafirmada ali.
- **Atribuição por item de mídia, não uma vez em "Sobre".** Travis Bell (staff TMDB, 26/02/2021), respondendo se bastava um único lugar dedicado no app: *"For this data, we expect a reference or logo on each media item, just like we do here on TMDb."* — orientação de staff em fórum, **não** texto de contrato nem da referência da API. (https://www.themoviedb.org/talk/60355e30a284eb003da676f2)
- **TMDB e JustWatch são duas obrigações separadas.** Travis Bell (08/02/2021): *"If you decide to use this data you do need to make sure to attribute both JustWatch and TMDb."* Em 17/04/2021 ele forneceu dois SVGs de logo JustWatch hospedados pela TMDB (URLs com hash de conteúdo, de 2021 — podem retornar 404 hoje; verificar antes de usar). (https://www.themoviedb.org/talk/6021b54cd6c300004167d43e)
- **Não há redação obrigatória para a atribuição JustWatch.** O único texto normativo é *"you must attribute the source of the data as JustWatch"*. Nem a referência dos dois endpoints, nem o ToS, nem o FAQ, nem a página de logos prescrevem frase, arquivo de logo, tamanho ou posicionamento. `Dados de disponibilidade fornecidos por JustWatch` + referência/logo em cada card de título satisfaz o texto literal mais a orientação de staff.

### Link para a página de watch da TMDB

Doc: *"You can link to the provided TMDB URL to help support TMDB and provide the actual deep links to the content."* Staff (04/08/2022), mais forte que a doc: *"You are to link to the TMDB watch page, and attribute JustWatch."* E (27/10/2023): *"we do not provide deep links. The idea is that you push people to our watch pages. If you want the full deep link, you will have to go partner directly with a provider like JustWatch."* → usar o campo `link` do bucket de país; **não** construir deep links de provedor. (https://www.themoviedb.org/talk/62eb3ba2273648005d9c8845)

### Uso comercial

- **ToS §2.A:** *"The license in Paragraph 1.A above does not permit any commercial use of TMDB, the TMDB APIs, or TMDB Content. Selling, leasing, or sublicensing… or deriving revenues from the use or provision of TMDB, the TMDB APIs, or TMDB Content, for commercial or monetary gain, directly or indirectly, is… considered a commercial use and is only permitted under a separate written agreement between You and TMDB."* TMDB reserva discrição exclusiva para determinar e revisar essa classificação. Uso comercial sem acordo escrito = *"material breach"* que *"causes irreparable harm to TMDB"*.
- **Teste da TMDB para "comercial" é receita**, não disponibilidade pública nem número de usuários. FAQ: *"Your project is considered commercial if the primary purpose is to create revenue for the benefit of the owner."* Não há limiar publicado de downloads, MAU, volume de requisições ou distribuição pública. Chave developer (gratuita) é o encaixe documentado para site público, gratuito e sem publicidade — sujeito à ambiguidade abaixo.
- **AMBIGUIDADE DE MAIOR RISCO PARA ESTE PROJETO.** ToS §2.A, terceiro bullet dos *"Common examples (which are by no means exhaustive) of commercial uses"*, verbatim: *"Using TMDB, the TMDB APIs, or TMDB Content on or in connection with a 'destination' website, search engine, or interactive query-response system (including large language model (LLM), artificial intelligence, or any other machine learning based interactive query-response systems or chatbots) ('Chatbot(s)'), or for driving traffic or generating revenue for a website, search engine, or Chatbot (including from advertising displayed on or by the website, search engine, Chatbot)."* Quarto bullet: *"Operating a website that generates revenue through charging users for access to content, or through recommend content, such as movies, television shows and music…"* — A primeira metade do terceiro bullet **não traz qualificador de receita**; a segunda metade é enquadrada por receita. Um site público cuja finalidade inteira é "o que está no streaming no Brasil" é discutivelmente um *"destination website"* construído sobre TMDB Content. **Questão genuinamente ambígua no texto — levar a jurídico e, idealmente, obter resposta escrita de sales@themoviedb.org descrevendo o app, e arquivá-la.**
- **Tier comercial:** Travis Bell (staff, 09/03/2026): *"If your app generates revenue, it is considered commercial… You would need to subscribe to our 'commercial' API key which costs $149/mo. A developer key is for personal applications that do not generate any revenue."* Respostas de fórum anteriores que permitiam anúncios/tier pago em chave developer estão **superadas**. **O valor de US$ 149/mês aparece somente em post de fórum** — não está na página API for Business (que não publica preço, só formulário de contato). Tratar como indicativo, **não contratual**. Contato comercial: sales@themoviedb.org. (https://www.themoviedb.org/talk/65e1a595f8595801634f40ab)
- **Não existe restrição comercial específica de watch provider e não existe exigência de contatar a JustWatch para uso comercial.** A palavra "commercial" **não aparece** em nenhuma das duas páginas de referência de watch provider. As regras gerais de §2 aplicam-se a todo TMDB Content, incluindo dados de disponibilidade, mas não há regra separada ou agravada para eles. A alegação recorrente de que "uso comercial de watch providers exige licença JustWatch" **não está na documentação** — o que o staff disse é mais estreito: ir à JustWatch é a alternativa para obter **deep links**, que a TMDB não fornece.

### Demais restrições do ToS §1.C com impacto direto

- *"Make derivatives of the TMDB APIs or TMDB Content."*
- **IA/ML — correção sobre o dossiê.** O bullet real, e único, é: *"Use the TMDB APIs or TMDB Content in connection with, including for training, a machine learning (ML) or artificial intelligence (AI) based Application."* A redação citada no dossiê original ("Training or validating… including large language models and Chatbots… collecting data sets…") **não existe** no ToS. A cláusula real é **mais ampla**, não mais estreita: proíbe usar as APIs ou o Content **em conexão com** qualquer aplicação baseada em IA/ML, não apenas treinar ou validar. Consequência: corpus RAG, fine-tune **e também** um agente de IA que apenas consome TMDB Content em tempo de inferência estão plausivelmente cobertos pela proibição. **Nenhuma feature de IA/LLM sobre dados TMDB neste produto.** §1.A ainda reserva à própria TMDB o direito de fazer derivados e de treinar IA com o conteúdo.
- *"Sell, lease, or sublicense… or derive revenues… except as expressly permitted in a written agreement…"*
- *"Use the TMDB APIs in a manner that is confusing or misleading as to the source or origin of Your Application."*
- *"Use, or create any Application that, in Our sole discretion, (i) uses an excessive amount of bandwidth, (ii) degrade or impairs… access to, or use of, TMDB… or (iii) otherwise adversely impacts the stability of TMDB."*
- *"Attempt to cloak or conceal Your identity…"*
- **Correção de citação:** o bullet de hospedagem de imagens lê *"Use the TMDB APIs as an image hosting service for banner advertisements, graphics, etc."* — e não "Use TMDB as an image hosting service…". Sentido inalterado; corrigir antes de colar em seção jurídica.

### Sem garantia

FAQ: *"We do not currently provide an SLA."* ToS §6: *"THE TMDB APIS ARE PROVIDED 'AS IS' WITH NO WARRANTY… TMDB DOES NOT REPRESENT OR WARRANT THAT ANY TMDB APIS ARE FREE OF INACCURACIES, ERRORS, BUGS, OR INTERRUPTIONS, OR ARE RELIABLE, ACCURATE, COMPLETE, OR OTHERWISE USEABLE OR VALID."* §9 impõe indenização ao usuário por reclamações de terceiros. Para site consumidor no Brasil (CDC): exibir aviso de que a disponibilidade é informativa, originada da JustWatch via TMDB, e pode estar desatualizada.

### Conjunto mínimo de conformidade (síntese; cada regra tem fonte acima)

1. Logo TMDB do conjunto SVG aprovado, sem modificação, menos proeminente que a marca própria. *(ToS — vinculante)*
2. Aviso na redação do ToS, adaptado para "website", em posição proeminente e em seção Sobre/Créditos. *(ToS — vinculante)*
3. Referência ou logo JustWatch **em cada título** onde a disponibilidade é exibida, não só no rodapé. *(orientação de staff em fórum — mais forte que a doc, não é texto de contrato)*
4. Linkar disponibilidade para a página `/watch` da TMDB, não para deep links de provedor. *(doc diz "you can link"; staff diz "you are to link" — não é texto de contrato)*
5. TTL de cache abaixo de 6 meses com job de purga documentado. *(ToS §1.C — vinculante)*
6. Sem anúncios, links de afiliado, tier pago ou patrocínios enquanto estiver em chave developer. *(política declarada em fórum, US$ 149/mês; não há tabela de preços publicada)*
7. Nenhuma feature de IA/LLM sobre conteúdo TMDB. *(ToS §1.C — vinculante, e mais ampla que "treinamento")*
8. Ambiguidade em aberto para jurídico + confirmação escrita da TMDB: um site público de disponibilidade se qualifica como *"destination website"* comercial sob §2.A mesmo com receita zero?

---

## Pendente de verificação

Itens que **não** puderam ser confirmados em fonte primária. Não tratar como resolvidos; validar contra a API ao vivo com chave antes de depender deles.

1. **Semântica prática da vírgula (`AND`) em `with_watch_providers`.** A redação `comma (AND) or pipe (OR)` **está confirmada na doc**. O que **não** se sustenta é a alegação de que o filtro não honra o `AND` de forma confiável: a thread citada (https://www.themoviedb.org/talk/5ff0a2b7176a940045e88665) **não contém nenhum relato de mau funcionamento** — mostra apenas Travis Bell (staff) recomendando a forma com pipe: `…&with_watch_providers=8|119|337&watch_region=CA`, buscando *"either Netflix or Amazon Prime or Disney+ in Canada."* Nenhuma fonte primária sustenta "AND não é confiável". **Independentemente disso, para um catálogo de streaming a forma correta é o pipe (OR)** — `AND` significaria "disponível no provedor A **e** no B simultaneamente", que casa com pouquíssimos títulos. Testar empiricamente se o comportamento do `AND` importar.

2. **`append_to_response=watch/providers` em `GET /3/movie/{movie_id}`.** Prática amplamente usada, que reduziria pela metade as chamadas de uma página de detalhe. **Nenhuma página primária da TMDB declara `watch/providers` como valor aceito de `append_to_response`.** A página oficial (https://developer.themoviedb.org/docs/append-to-response) descreve anexar *"sub requests within the same namespace in a single HTTP request"* para os métodos de detalhe de movie/TV/season/episode/person, mas seus **únicos** exemplos são `?append_to_response=videos` e `?append_to_response=videos,images`; nenhum nome de sub-request com barra é documentado. A página de referência de `/movie/{id}/watch/providers` também não menciona suporte a append. **Adicionalmente:** existe thread de fórum confirmada, intitulada *"Watch providers from append_to_response does not match watch providers endpoint (TV Shows)"*, relatando divergência entre os dois caminhos (https://www.themoviedb.org/talk/6543fe1941a561336b766e87). **Postura:** verificar contra a API ao vivo; preferir o endpoint standalone quando a exatidão importar. Lembrar do limite documentado de 20 sub-requests (erro code 27 / HTTP 400).

3. **Semântica do parâmetro `region` em `/discover/movie`.** Está **confirmado** que `region` é linha separada de `watch_region` no schema, e que `certification`, `certification.gte`, `certification.lte` e `with_release_type` referenciam `region` verbatim. **Não está documentado** que `region` "determina qual data de lançamento é usada" — `region` não tem descrição alguma na página de referência e nenhuma página primária declara sua semântica para `/discover/movie`. Tratar como inferência.

4. **Quotes de staff não localizadas nas fontes citadas** (não são load-bearing para as conclusões, mas não devem ser citadas):
   - Travis Bell, 29/01/2023, *"You can use our watch pages, as outlined in the docs for free. If you don't want to do that, you can contact them directly about licensing the data yourself."* — atribuída à URL da referência de `movie-watch-providers`, que não hospeda conteúdo de fórum. A conclusão negativa que ela apoiava (não há restrição comercial específica de watch provider) **está confirmada** por outra via.
   - Travis Bell, 04/07/2023, *"A developer key is fine"* para app com anúncios AdMob + tier pago sem anúncios — não localizada na thread citada.
   - Travis Bell, 01/03/2024 — a continuação real do texto localizado é *"just make sure you're attributing TMDB."*, **não** *"Later this year we'll be introducing a new offering designed for apps like yours."*

5. **Preço do tier comercial (US$ 149/mês).** Consta apenas de post de fórum de staff (09/03/2026). A página https://www.themoviedb.org/api-for-business **não publica preço** — só formulário de contato, com a nota de que acordos *"may be subject to, among other things, payment of fees."* **Não citar o valor em nenhum documento comercial sem confirmação escrita de sales@themoviedb.org.**

6. **Comportamento de ETag / 304 na CDN de imagens.** Verificado empiricamente (200 com `ETag: "67ef8825-3a64"`, `Last-Modified: Fri, 04 Apr 2025 07:20:05 GMT`, e 304 ao reenviar com `If-None-Match`), mas **não é contrato documentado** — a configuração da BunnyCDN pode mudar sem aviso.

7. **Rate limiting por IP.** Amplamente repetido em fóruns, **ausente de toda a documentação oficial** (índice completo de docs, rate-limiting, errors, getting-started, FAQ e ToS). Plausível, não verificado. Importa em cenário de egress NAT com IP público único: o orçamento seria compartilhado entre todos os chamadores internos.

8. **20 resultados por página.** Comportamento universal observado nos endpoints de lista da TMDB, **não declarado em nenhuma doc primária**. O teto de 10.000 resultados por query é derivado dele. Medir empiricamente.

9. **Contagem de ~85 provedores para BR.** Lida da página de browse `https://www.themoviedb.org/movie?watch_region=BR` (painel "Onde Assistir"), **não** de chamada direta a `/3/watch/providers/movie?watch_region=BR`. O número oscila conforme provedores entram e saem. Confirmar via API.