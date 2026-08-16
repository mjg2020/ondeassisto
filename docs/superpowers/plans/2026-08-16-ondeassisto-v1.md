# ondeassisto.com.br — Plano de Implementação

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Construir um site público que mostra quais filmes estão disponíveis agora nos serviços de streaming que o usuário assina, no Brasil.

**Architecture:** Uma aplicação Next.js (App Router) na Vercel. Sem banco, sem autenticação. Páginas são Server Components que chamam o TMDB na renderização, de modo que o token nunca sai do servidor. O estado de filtro mora na URL, tornando cada combinação uma página renderizada no servidor e indexável. A watchlist vive em `localStorage`.

**Tech Stack:** Next.js 15 (App Router) · TypeScript · Tailwind CSS 4 · Vitest + Testing Library · Playwright · Node 20+

**Spec:** [`docs/superpowers/specs/2026-08-16-catalogo-streaming-design.md`](../specs/2026-08-16-catalogo-streaming-design.md)
**Referência da API:** [`docs/tmdb-api-ficha-tecnica.md`](../../tmdb-api-ficha-tecnica.md)

## Global Constraints

Estas regras valem para **todas** as tarefas. Violá-las é defeito, não escolha de estilo.

- **`watch_region=BR` acompanha obrigatoriamente todo uso de `with_watch_providers`.** Sem ele a API devolve HTTP 200 com o catálogo global — 1.169.919 resultados contra 4.891. Não falha, entrega dado errado parecendo sucesso
- **Provedores usam pipe (`|`) = união.** Vírgula é interseção (Netflix ∩ Prime = 124 títulos), quase sempre errado para este produto
- **Nenhum `provider_id` fixado em código.** A lista vem de `/watch/providers/movie?watch_region=BR`. HBO Max é `1899` em BR e `384` não existe; Apple TV assinatura é `350` e a loja é `2`
- **`vote_count.gte=200` acompanha toda requisição em que `vote_average` participe**, seja como ordenação, seja como filtro
- **`page` limitado a 500** antes de qualquer chamada. `page=501` retorna HTTP 400 / `status_code 22`
- **Valor inválido na URL é descartado e cai no default.** Nunca gera erro para o usuário
- **O token só existe no servidor.** Nenhuma variável com prefixo `NEXT_PUBLIC_` toca credencial. Variável: `TMDB_READ_TOKEN`, header `Authorization: Bearer`
- **Contraste medido, não estimado.** Fundo `#0b0b0f`; texto `#f2f2f4` (17,6:1); secundário `#9a9aa6` (7,1:1); âmbar `#f5c518` (12,1:1). Alterar qualquer hex obriga a medir de novo
- **Texto nunca sobre o pôster.** Título e ano abaixo da imagem. Única sobreposição: a pílula de nota, com fundo âmbar sólido
- **Nenhum recurso de IA/LLM sobre conteúdo do TMDB** — proibido pelo ToS §1.C
- **Atribuição obrigatória:** logo TMDB + aviso do ToS §3, e referência à JustWatch **em cada item** onde a disponibilidade aparece
- **Idioma da interface e dos metadados: `pt-BR`.** Nomes de função, variável e arquivo em português, seguindo o vocabulário do spec

## Desvio consciente do spec

O spec (§4.3) prevê chamar `/configuration` com TTL de 24 h para obter `secure_base_url` e as listas de tamanhos de imagem. **Este plano fixa esses valores em código** (`lib/tmdb/images.ts`), por três razões: são públicos e estáveis há anos; a alternativa coloca uma chamada de rede no caminho crítico de toda renderização, inclusive a primeira; e um erro ali quebraria todas as imagens de uma vez, de forma óbvia, não sutil.

O custo é que uma mudança de infraestrutura do TMDB passaria despercebida até as imagens sumirem. Aceito, com uma proteção: a suíte de contrato (Task 14) pode ganhar um teste que compara os valores fixados com `/configuration` — acrescente se algum dia isso mudar.

---

## Estrutura de arquivos

```
app/
  layout.tsx                    tema escuro, cabeçalho com SearchBar, rodapé de atribuição
  page.tsx                      grade (Server Component, lê searchParams)
  loading.tsx                   skeleton da grade
  busca/page.tsx                resultados de busca particionados
  filme/[id]/page.tsx           detalhe + generateMetadata
  filme/[id]/loading.tsx        skeleton do detalhe
  minha-lista/page.tsx          watchlist (Client Component)
  sobre/page.tsx                atribuição e aviso legal
  api/discover/route.ts         paginação do scroll infinito
  globals.css                   tokens de tema (Tailwind 4 @theme)

lib/
  tmdb/client.ts                fetchTmdb: Bearer, timeout, backoff no 429, erros tipados
  tmdb/types.ts                 tipos de domínio (Filme, Oferta, Provedor...)
  tmdb/discover.ts              discoverMovies
  tmdb/movie.ts                 getMovie (append), getMovieProviders
  tmdb/search.ts                searchMovies com anotação e particionamento
  tmdb/providers.ts             getBrProviders
  tmdb/images.ts                posterUrl / backdropUrl / logoUrl
  filtros/tipos.ts              Filtros, Ordem, ORDENS
  filtros/parse.ts              parseSearchParams, toSearchParams
  filtros/discover-params.ts    construirParamsDiscover
  lista/armazenamento.ts        localStorage versionado
  lista/useLista.ts             hook de watchlist

componentes/
  CardFilme.tsx  GradeFilmes.tsx  BarraFiltros.tsx  CampoBusca.tsx
  BotaoLista.tsx  OfertasProvedor.tsx  ListaElenco.tsx  Trailer.tsx
  Atribuicao.tsx  EstadoVazio.tsx  EstadoErro.tsx  SkeletonGrade.tsx  SkeletonDetalhe.tsx
  CarregadorInfinito.tsx

testes/
  contrato/tmdb.contrato.test.ts   suíte contra a API real (npm run test:contract)
e2e/
  fluxo.spec.ts                    Playwright
```

---

## Task 1: Bootstrap, tema e tokens de contraste

**Files:**
- Create: `package.json`, `tsconfig.json`, `next.config.ts`, `vitest.config.ts`, `app/globals.css`, `app/layout.tsx`, `app/page.tsx`
- Create: `.env.example` (já existe — conferir), `.nvmrc`
- Test: `testes/tema.test.ts`

**Interfaces:**
- Consumes: nada
- Produces: projeto executável com `npm run dev`, `npm test`; tokens CSS `--color-fundo`, `--color-texto`, `--color-secundario`, `--color-ambar`

- [ ] **Step 1: Criar o projeto Next.js**

```bash
cd /c/AI/Curso_CLAUDE
npx create-next-app@latest . --typescript --tailwind --app --no-src-dir --import-alias "@/*" --eslint
```

Quando perguntar sobre sobrescrever arquivos existentes, **preserve** `.gitignore`, `.env.example` e `docs/`.

- [ ] **Step 2: Fixar a versão do Node e instalar dependências de teste**

```bash
echo "20" > .nvmrc
npm install -D vitest @vitejs/plugin-react jsdom @testing-library/react @testing-library/jest-dom @playwright/test
```

- [ ] **Step 3: Configurar o Vitest**

`vitest.config.ts`:

```ts
import { defineConfig } from 'vitest/config'
import react from '@vitejs/plugin-react'
import path from 'node:path'

export default defineConfig({
  plugins: [react()],
  test: {
    environment: 'jsdom',
    globals: true,
    setupFiles: ['./testes/setup.ts'],
    exclude: ['**/node_modules/**', '**/e2e/**', '**/testes/contrato/**'],
  },
  resolve: { alias: { '@': path.resolve(__dirname, '.') } },
})
```

`testes/setup.ts`:

```ts
import '@testing-library/jest-dom/vitest'
```

Em `package.json`, acrescente aos scripts:

```json
"test": "vitest run",
"test:watch": "vitest",
"test:contract": "vitest run --config vitest.contrato.config.ts",
"test:e2e": "playwright test"
```

- [ ] **Step 4: Escrever o teste de contraste da paleta**

`testes/tema.test.ts`:

```ts
import { describe, it, expect } from 'vitest'
import fs from 'node:fs'

// Contraste WCAG 2.1 — retorna a razão entre duas cores hex
function luminancia(hex: string): number {
  const n = parseInt(hex.replace('#', ''), 16)
  const canais = [(n >> 16) & 255, (n >> 8) & 255, n & 255].map((v) => {
    const c = v / 255
    return c <= 0.03928 ? c / 12.92 : Math.pow((c + 0.055) / 1.055, 2.4)
  })
  return 0.2126 * canais[0] + 0.7152 * canais[1] + 0.0722 * canais[2]
}

function contraste(a: string, b: string): number {
  const [l1, l2] = [luminancia(a), luminancia(b)].sort((x, y) => y - x)
  return (l1 + 0.05) / (l2 + 0.05)
}

const FUNDO = '#0b0b0f'

describe('paleta do tema escuro', () => {
  it('texto primário passa em AA (4.5:1)', () => {
    expect(contraste('#f2f2f4', FUNDO)).toBeGreaterThanOrEqual(4.5)
  })

  it('texto secundário passa em AA (4.5:1)', () => {
    expect(contraste('#9a9aa6', FUNDO)).toBeGreaterThanOrEqual(4.5)
  })

  it('âmbar passa em AA (4.5:1)', () => {
    expect(contraste('#f5c518', FUNDO)).toBeGreaterThanOrEqual(4.5)
  })

  it('texto escuro sobre âmbar passa em AA', () => {
    expect(contraste('#0b0b0f', '#f5c518')).toBeGreaterThanOrEqual(4.5)
  })

  it('globals.css declara exatamente estes hexadecimais', () => {
    const css = fs.readFileSync('app/globals.css', 'utf8')
    for (const hex of ['#0b0b0f', '#f2f2f4', '#9a9aa6', '#f5c518']) {
      expect(css).toContain(hex)
    }
  })
})
```

- [ ] **Step 5: Rodar o teste e ver falhar**

Run: `npm test -- testes/tema.test.ts`
Expected: FAIL — `globals.css` ainda não contém os hexadecimais.

- [ ] **Step 6: Escrever os tokens de tema**

`app/globals.css`:

```css
@import "tailwindcss";

@theme {
  --color-fundo: #0b0b0f;
  --color-superficie: #16161c;
  --color-borda: #2a2a33;
  --color-texto: #f2f2f4;
  --color-secundario: #9a9aa6;
  --color-ambar: #f5c518;
}

html { color-scheme: dark; }

body {
  background-color: var(--color-fundo);
  color: var(--color-texto);
}

:focus-visible {
  outline: 2px solid var(--color-ambar);
  outline-offset: 2px;
}

@media (prefers-reduced-motion: reduce) {
  *, *::before, *::after {
    animation-duration: 0.01ms !important;
    transition-duration: 0.01ms !important;
  }
}
```

- [ ] **Step 7: Rodar o teste e ver passar**

Run: `npm test -- testes/tema.test.ts`
Expected: PASS, 5 testes.

- [ ] **Step 8: Layout raiz**

`app/layout.tsx`:

```tsx
import type { Metadata } from 'next'
import './globals.css'

export const metadata: Metadata = {
  title: 'ondeassisto — o que está nos seus streamings',
  description: 'Descubra quais filmes estão disponíveis agora nos serviços de streaming que você assina, no Brasil.',
}

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="pt-BR">
      <body className="min-h-screen bg-fundo text-texto antialiased">
        {children}
      </body>
    </html>
  )
}
```

- [ ] **Step 9: Commit**

```bash
git add -A
git commit -m "feat: bootstrap Next.js com tema escuro e contraste testado"
```

---

## Task 2: Cliente TMDB com backoff e erros tipados

**Files:**
- Create: `lib/tmdb/client.ts`, `lib/tmdb/erros.ts`
- Test: `testes/tmdb/client.test.ts`

**Interfaces:**
- Consumes: variável de ambiente `TMDB_READ_TOKEN`
- Produces:
  - `fetchTmdb<T>(caminho: string, params?: Params, opts?: { revalidate?: number }): Promise<T>`
  - `type Params = Record<string, string | number | boolean | undefined>`
  - `class ErroTmdb extends Error { status: number; codigoTmdb?: number }`

- [ ] **Step 1: Escrever os testes**

`testes/tmdb/client.test.ts`:

```ts
import { describe, it, expect, vi, beforeEach, afterEach } from 'vitest'
import { fetchTmdb } from '@/lib/tmdb/client'
import { ErroTmdb } from '@/lib/tmdb/erros'

beforeEach(() => {
  process.env.TMDB_READ_TOKEN = 'token-de-teste'
  vi.restoreAllMocks()
})
afterEach(() => vi.useRealTimers())

function respostaOk(corpo: unknown) {
  return { ok: true, status: 200, json: async () => corpo } as Response
}

describe('fetchTmdb', () => {
  it('envia o token no header Authorization, nunca na query string', async () => {
    const espiao = vi.spyOn(global, 'fetch').mockResolvedValue(respostaOk({ ok: 1 }))
    await fetchTmdb('/movie/550')
    const [url, init] = espiao.mock.calls[0]
    expect(String(url)).not.toContain('token-de-teste')
    expect((init?.headers as Record<string, string>).Authorization).toBe('Bearer token-de-teste')
  })

  it('inclui language=pt-BR por padrão', async () => {
    const espiao = vi.spyOn(global, 'fetch').mockResolvedValue(respostaOk({}))
    await fetchTmdb('/movie/550')
    expect(String(espiao.mock.calls[0][0])).toContain('language=pt-BR')
  })

  it('omite parâmetros undefined', async () => {
    const espiao = vi.spyOn(global, 'fetch').mockResolvedValue(respostaOk({}))
    await fetchTmdb('/discover/movie', { with_genres: undefined, page: 2 })
    const url = String(espiao.mock.calls[0][0])
    expect(url).not.toContain('with_genres')
    expect(url).toContain('page=2')
  })

  it('tenta de novo no 429 e vence na terceira', async () => {
    const r429 = { ok: false, status: 429, json: async () => ({ status_code: 25 }) } as Response
    const espiao = vi
      .spyOn(global, 'fetch')
      .mockResolvedValueOnce(r429)
      .mockResolvedValueOnce(r429)
      .mockResolvedValueOnce(respostaOk({ id: 550 }))
    const r = await fetchTmdb<{ id: number }>('/movie/550', {}, { esperar: async () => {} })
    expect(r.id).toBe(550)
    expect(espiao).toHaveBeenCalledTimes(3)
  })

  it('desiste após 3 tentativas no 429 e lança ErroTmdb', async () => {
    const r429 = { ok: false, status: 429, json: async () => ({ status_code: 25 }) } as Response
    vi.spyOn(global, 'fetch').mockResolvedValue(r429)
    await expect(
      fetchTmdb('/movie/550', {}, { esperar: async () => {} })
    ).rejects.toBeInstanceOf(ErroTmdb)
  })

  it('não tenta de novo em 404', async () => {
    const espiao = vi.spyOn(global, 'fetch').mockResolvedValue({
      ok: false, status: 404, json: async () => ({ status_code: 34 }),
    } as Response)
    await expect(fetchTmdb('/movie/0')).rejects.toMatchObject({ status: 404 })
    expect(espiao).toHaveBeenCalledTimes(1)
  })
})
```

- [ ] **Step 2: Rodar e ver falhar**

Run: `npm test -- testes/tmdb/client.test.ts`
Expected: FAIL — módulo `@/lib/tmdb/client` não existe.

- [ ] **Step 3: Implementar os erros**

`lib/tmdb/erros.ts`:

```ts
export class ErroTmdb extends Error {
  constructor(
    mensagem: string,
    readonly status: number,
    readonly codigoTmdb?: number
  ) {
    super(mensagem)
    this.name = 'ErroTmdb'
  }

  /** 503 com status_code 9 ou 46 = TMDB em manutenção. */
  get emManutencao(): boolean {
    return this.status === 503 && (this.codigoTmdb === 9 || this.codigoTmdb === 46)
  }
}
```

- [ ] **Step 4: Implementar o cliente**

`lib/tmdb/client.ts`:

```ts
import { ErroTmdb } from './erros'

const BASE = 'https://api.themoviedb.org/3'
const TENTATIVAS = 3

export type Params = Record<string, string | number | boolean | undefined>

type Opcoes = {
  revalidate?: number
  /** injetável para teste; produção usa espera real */
  esperar?: (ms: number) => Promise<void>
}

const dormir = (ms: number) => new Promise<void>((r) => setTimeout(r, ms))

export async function fetchTmdb<T>(
  caminho: string,
  params: Params = {},
  opcoes: Opcoes = {}
): Promise<T> {
  const token = process.env.TMDB_READ_TOKEN
  if (!token) throw new ErroTmdb('TMDB_READ_TOKEN não configurado', 500)

  const url = new URL(BASE + caminho)
  url.searchParams.set('language', 'pt-BR')
  for (const [chave, valor] of Object.entries(params)) {
    if (valor !== undefined) url.searchParams.set(chave, String(valor))
  }

  const esperar = opcoes.esperar ?? dormir
  let ultimo: ErroTmdb | null = null

  for (let tentativa = 1; tentativa <= TENTATIVAS; tentativa++) {
    const resposta = await fetch(url, {
      headers: { Authorization: `Bearer ${token}`, Accept: 'application/json' },
      next: { revalidate: opcoes.revalidate ?? 3600 },
    })

    if (resposta.ok) return (await resposta.json()) as T

    const corpo = await resposta.json().catch(() => ({}) as { status_code?: number })
    ultimo = new ErroTmdb(
      `TMDB respondeu ${resposta.status} em ${caminho}`,
      resposta.status,
      corpo.status_code
    )

    // Só 429 e 5xx merecem nova tentativa. 4xx é erro nosso.
    if (resposta.status !== 429 && resposta.status < 500) throw ultimo

    if (tentativa < TENTATIVAS) {
      const base = 500 * 2 ** (tentativa - 1)
      await esperar(base + Math.floor(Math.random() * 250))
    }
  }

  throw ultimo!
}
```

- [ ] **Step 5: Rodar e ver passar**

Run: `npm test -- testes/tmdb/client.test.ts`
Expected: PASS, 6 testes.

- [ ] **Step 6: Commit**

```bash
git add lib/tmdb/client.ts lib/tmdb/erros.ts testes/tmdb/client.test.ts
git commit -m "feat: cliente TMDB com backoff no 429 e erros tipados"
```

---

## Task 3: Tipos de domínio e montagem de URL de imagem

**Files:**
- Create: `lib/tmdb/types.ts`, `lib/tmdb/images.ts`
- Test: `testes/tmdb/images.test.ts`

**Interfaces:**
- Consumes: `fetchTmdb`
- Produces:
  - `type Filme`, `type DetalheFilme`, `type Provedor`, `type Oferta`, `type TipoOferta`, `type Disponibilidade`
  - `posterUrl(path: string | null, tamanho: TamanhoPoster): string | null`
  - `backdropUrl(path: string | null, tamanho: TamanhoBackdrop): string | null`
  - `logoUrl(path: string | null, tamanho: TamanhoLogo): string | null`

- [ ] **Step 1: Escrever os testes**

`testes/tmdb/images.test.ts`:

```ts
import { describe, it, expect } from 'vitest'
import { posterUrl, backdropUrl, logoUrl } from '@/lib/tmdb/images'

describe('montagem de URL de imagem', () => {
  it('monta URL de pôster com as três peças', () => {
    expect(posterUrl('/abc.jpg', 'w342')).toBe('https://image.tmdb.org/t/p/w342/abc.jpg')
  })

  it('sempre usa https', () => {
    expect(posterUrl('/abc.jpg', 'w342')!.startsWith('https://')).toBe(true)
  })

  it('devolve null quando o path é null', () => {
    expect(posterUrl(null, 'w342')).toBeNull()
    expect(backdropUrl(null, 'w1280')).toBeNull()
    expect(logoUrl(null, 'w92')).toBeNull()
  })

  it('monta backdrop com tamanho da família correta', () => {
    expect(backdropUrl('/bg.jpg', 'w1280')).toBe('https://image.tmdb.org/t/p/w1280/bg.jpg')
  })

  it('monta logo de provedor', () => {
    expect(logoUrl('/n.jpg', 'w92')).toBe('https://image.tmdb.org/t/p/w92/n.jpg')
  })
})
```

Um teste que não cabe em runtime: **`w500` não existe em `backdrop_sizes`.** Isso é garantido pelo tipo `TamanhoBackdrop`, e o compilador recusa `backdropUrl(x, 'w500')`. Verifique com `npx tsc --noEmit` no Step 5.

- [ ] **Step 2: Rodar e ver falhar**

Run: `npm test -- testes/tmdb/images.test.ts`
Expected: FAIL — módulo inexistente.

- [ ] **Step 3: Escrever os tipos**

`lib/tmdb/types.ts`:

```ts
export type Filme = {
  id: number
  titulo: string
  ano: number | null
  posterPath: string | null
  backdropPath: string | null
  nota: number
  votos: number
  sinopse: string
}

export type TipoOferta = 'flatrate' | 'free' | 'ads' | 'rent' | 'buy'

/** Ordem de exibição: assinatura primeiro, transacional por último. */
export const TIPOS_OFERTA: readonly TipoOferta[] = ['flatrate', 'free', 'ads', 'rent', 'buy']

/** Tipos que contam como "incluído no que eu assino ou é grátis". */
export const TIPOS_SEM_CUSTO_EXTRA: readonly TipoOferta[] = ['flatrate', 'free', 'ads']

export type Provedor = {
  id: number
  nome: string
  logoPath: string | null
  prioridadeBR: number
}

export type Oferta = { tipo: TipoOferta; provedores: Provedor[] }

export type Disponibilidade = {
  /** página /watch do TMDB. Nunca deep link de provedor — exigência do TMDB. */
  link: string | null
  ofertas: Oferta[]
}

export type Pessoa = { nome: string; personagem: string; fotoPath: string | null }

export type DetalheFilme = Filme & {
  duracaoMin: number | null
  generos: string[]
  disponibilidade: Disponibilidade
  elenco: Pessoa[]
  trailerYoutubeKey: string | null
}
```

- [ ] **Step 4: Implementar images.ts**

`lib/tmdb/images.ts`:

```ts
// base_url vem de /configuration, mas é estável há anos e o valor é público.
// Fixar evita uma chamada de rede no caminho crítico de toda renderização.
const BASE = 'https://image.tmdb.org/t/p'

export type TamanhoPoster = 'w92' | 'w154' | 'w185' | 'w342' | 'w500' | 'w780' | 'original'
export type TamanhoBackdrop = 'w300' | 'w780' | 'w1280' | 'original'
export type TamanhoLogo = 'w45' | 'w92' | 'w154' | 'w185' | 'w300' | 'w500' | 'original'

function montar(path: string | null, tamanho: string): string | null {
  return path ? `${BASE}/${tamanho}${path}` : null
}

export const posterUrl = (p: string | null, t: TamanhoPoster) => montar(p, t)
export const backdropUrl = (p: string | null, t: TamanhoBackdrop) => montar(p, t)
export const logoUrl = (p: string | null, t: TamanhoLogo) => montar(p, t)
```

- [ ] **Step 5: Rodar testes e checagem de tipos**

Run: `npm test -- testes/tmdb/images.test.ts && npx tsc --noEmit`
Expected: PASS, 5 testes; `tsc` sem erros.

- [ ] **Step 6: Commit**

```bash
git add lib/tmdb/types.ts lib/tmdb/images.ts testes/tmdb/images.test.ts
git commit -m "feat: tipos de domínio e montagem de URL de imagem por família de tamanho"
```

---

## Task 4: Camada de filtros — o tradutor URL ↔ TMDB

Esta é a camada de maior risco de regressão silenciosa do projeto. É pura, sem I/O, e concentra as invariantes que a §9.3 mediu.

**Files:**
- Create: `lib/filtros/tipos.ts`, `lib/filtros/parse.ts`, `lib/filtros/discover-params.ts`
- Test: `testes/filtros/parse.test.ts`, `testes/filtros/discover-params.test.ts`

**Interfaces:**
- Consumes: nada
- Produces:
  - `type Ordem = 'popularidade' | 'nota' | 'recentes'`
  - `type Filtros = { servicos: number[]; genero: number | null; ano: number | null; notaMinima: number | null; ordem: Ordem; incluirGratis: boolean }`
  - `FILTROS_PADRAO: Filtros`
  - `parseSearchParams(sp: URLSearchParams | Record<string, string | string[] | undefined>): Filtros`
  - `toSearchParams(f: Filtros): URLSearchParams`
  - `construirParamsDiscover(f: Filtros, pagina: number, hoje?: Date): Params`

- [ ] **Step 1: Escrever os testes de parse**

`testes/filtros/parse.test.ts`:

```ts
import { describe, it, expect } from 'vitest'
import { parseSearchParams, toSearchParams, FILTROS_PADRAO } from '@/lib/filtros/parse'

const p = (s: string) => parseSearchParams(new URLSearchParams(s))

describe('parseSearchParams', () => {
  it('URL vazia devolve os padrões', () => {
    expect(p('')).toEqual(FILTROS_PADRAO)
  })

  it('lê múltiplos serviços separados por vírgula', () => {
    expect(p('servicos=8,119').servicos).toEqual([8, 119])
  })

  it('descarta id de serviço não numérico, preservando os válidos', () => {
    expect(p('servicos=8,abc,119').servicos).toEqual([8, 119])
  })

  it('ordem desconhecida cai no padrão, sem erro', () => {
    expect(p('ordem=inexistente').ordem).toBe('popularidade')
  })

  it('aceita as três ordens válidas', () => {
    expect(p('ordem=nota').ordem).toBe('nota')
    expect(p('ordem=recentes').ordem).toBe('recentes')
    expect(p('ordem=popularidade').ordem).toBe('popularidade')
  })

  it('ano fora de faixa é ignorado', () => {
    expect(p('ano=1200').ano).toBeNull()
    expect(p('ano=3000').ano).toBeNull()
    expect(p('ano=2024').ano).toBe(2024)
  })

  it('nota fora de 0-10 é ignorada', () => {
    expect(p('nota=99').notaMinima).toBeNull()
    expect(p('nota=7').notaMinima).toBe(7)
  })

  it('incluirGratis só é verdadeiro com o valor 1', () => {
    expect(p('incluirGratis=1').incluirGratis).toBe(true)
    expect(p('incluirGratis=0').incluirGratis).toBe(false)
    expect(p('').incluirGratis).toBe(false)
  })

  it('entrada hostil não lança exceção', () => {
    expect(() => p('servicos=<script>&ano=NaN&nota=-5&ordem=%00')).not.toThrow()
  })
})

describe('toSearchParams', () => {
  it('faz ida e volta preservando os filtros', () => {
    const f = { ...FILTROS_PADRAO, servicos: [8, 119], genero: 28, ordem: 'nota' as const }
    expect(parseSearchParams(toSearchParams(f))).toEqual(f)
  })

  it('omite valores padrão para manter a URL curta', () => {
    expect(toSearchParams(FILTROS_PADRAO).toString()).toBe('')
  })
})
```

- [ ] **Step 2: Rodar e ver falhar**

Run: `npm test -- testes/filtros/parse.test.ts`
Expected: FAIL — módulo inexistente.

- [ ] **Step 3: Implementar tipos e parse**

`lib/filtros/tipos.ts`:

```ts
export type Ordem = 'popularidade' | 'nota' | 'recentes'

export type Filtros = {
  servicos: number[]
  genero: number | null
  ano: number | null
  notaMinima: number | null
  ordem: Ordem
  incluirGratis: boolean
}

/** Vocabulário fechado. Cada ordem arrasta seu acompanhamento obrigatório. */
export const ORDENS: Record<Ordem, { sortBy: string; rotulo: string }> = {
  popularidade: { sortBy: 'popularity.desc', rotulo: 'Populares' },
  nota: { sortBy: 'vote_average.desc', rotulo: 'Melhor avaliados' },
  recentes: { sortBy: 'primary_release_date.desc', rotulo: 'Mais recentes' },
}

/** Piso de votos: obrigatório sempre que vote_average participa. */
export const PISO_VOTOS = 200
```

`lib/filtros/parse.ts`:

```ts
import { type Filtros, type Ordem, ORDENS } from './tipos'

export const FILTROS_PADRAO: Filtros = {
  servicos: [],
  genero: null,
  ano: null,
  notaMinima: null,
  ordem: 'popularidade',
  incluirGratis: false,
}

type Entrada = URLSearchParams | Record<string, string | string[] | undefined>

function ler(entrada: Entrada, chave: string): string | null {
  if (entrada instanceof URLSearchParams) return entrada.get(chave)
  const v = entrada[chave]
  return Array.isArray(v) ? (v[0] ?? null) : (v ?? null)
}

function inteiroNaFaixa(bruto: string | null, min: number, max: number): number | null {
  if (bruto === null) return null
  const n = Number(bruto)
  return Number.isInteger(n) && n >= min && n <= max ? n : null
}

export function parseSearchParams(entrada: Entrada): Filtros {
  const servicosBruto = ler(entrada, 'servicos') ?? ''
  const servicos = servicosBruto
    .split(',')
    .map((s) => Number(s.trim()))
    .filter((n) => Number.isInteger(n) && n > 0)

  const ordemBruta = ler(entrada, 'ordem')
  const ordem: Ordem =
    ordemBruta && ordemBruta in ORDENS ? (ordemBruta as Ordem) : FILTROS_PADRAO.ordem

  const notaBruta = ler(entrada, 'nota')
  const nota = notaBruta === null ? null : Number(notaBruta)

  return {
    servicos,
    genero: inteiroNaFaixa(ler(entrada, 'genero'), 1, 999_999),
    ano: inteiroNaFaixa(ler(entrada, 'ano'), 1874, new Date().getFullYear() + 2),
    notaMinima: nota !== null && Number.isFinite(nota) && nota >= 0 && nota <= 10 ? nota : null,
    ordem,
    incluirGratis: ler(entrada, 'incluirGratis') === '1',
  }
}

export function toSearchParams(f: Filtros): URLSearchParams {
  const sp = new URLSearchParams()
  if (f.servicos.length) sp.set('servicos', f.servicos.join(','))
  if (f.genero !== null) sp.set('genero', String(f.genero))
  if (f.ano !== null) sp.set('ano', String(f.ano))
  if (f.notaMinima !== null) sp.set('nota', String(f.notaMinima))
  if (f.ordem !== FILTROS_PADRAO.ordem) sp.set('ordem', f.ordem)
  if (f.incluirGratis) sp.set('incluirGratis', '1')
  return sp
}
```

- [ ] **Step 4: Rodar e ver passar**

Run: `npm test -- testes/filtros/parse.test.ts`
Expected: PASS, 11 testes.

- [ ] **Step 5: Escrever os testes de construção de parâmetros**

`testes/filtros/discover-params.test.ts`:

```ts
import { describe, it, expect } from 'vitest'
import { construirParamsDiscover } from '@/lib/filtros/discover-params'
import { FILTROS_PADRAO } from '@/lib/filtros/parse'

const HOJE = new Date('2026-08-16T12:00:00Z')

describe('construirParamsDiscover', () => {
  it('SEMPRE inclui watch_region=BR — invariante de segurança', () => {
    const p = construirParamsDiscover(FILTROS_PADRAO, 1, HOJE)
    expect(p.watch_region).toBe('BR')
  })

  it('usa pipe (união) e nunca vírgula (interseção) para serviços', () => {
    const p = construirParamsDiscover({ ...FILTROS_PADRAO, servicos: [8, 119] }, 1, HOJE)
    expect(p.with_watch_providers).toBe('8|119')
    expect(String(p.with_watch_providers)).not.toContain(',')
  })

  it('omite with_watch_providers quando nenhum serviço está marcado', () => {
    expect(construirParamsDiscover(FILTROS_PADRAO, 1, HOJE).with_watch_providers).toBeUndefined()
  })

  it('monetização padrão é só assinatura', () => {
    expect(construirParamsDiscover(FILTROS_PADRAO, 1, HOJE).with_watch_monetization_types)
      .toBe('flatrate')
  })

  it('incluirGratis soma free e ads em união', () => {
    const p = construirParamsDiscover({ ...FILTROS_PADRAO, incluirGratis: true }, 1, HOJE)
    expect(p.with_watch_monetization_types).toBe('flatrate|free|ads')
  })

  it('ordenar por nota arrasta o piso de 200 votos', () => {
    const p = construirParamsDiscover({ ...FILTROS_PADRAO, ordem: 'nota' }, 1, HOJE)
    expect(p.sort_by).toBe('vote_average.desc')
    expect(p['vote_count.gte']).toBe(200)
  })

  it('filtrar por nota mínima também arrasta o piso, mesmo sem ordenar por nota', () => {
    const p = construirParamsDiscover({ ...FILTROS_PADRAO, notaMinima: 7 }, 1, HOJE)
    expect(p['vote_average.gte']).toBe(7)
    expect(p['vote_count.gte']).toBe(200)
  })

  it('sem vote_average em jogo, não há piso de votos', () => {
    expect(construirParamsDiscover(FILTROS_PADRAO, 1, HOJE)['vote_count.gte']).toBeUndefined()
  })

  it('ordem recentes exclui filmes ainda não lançados', () => {
    const p = construirParamsDiscover({ ...FILTROS_PADRAO, ordem: 'recentes' }, 1, HOJE)
    expect(p.sort_by).toBe('primary_release_date.desc')
    expect(p['primary_release_date.lte']).toBe('2026-08-16')
  })

  it('limita a página a 500', () => {
    expect(construirParamsDiscover(FILTROS_PADRAO, 9999, HOJE).page).toBe(500)
    expect(construirParamsDiscover(FILTROS_PADRAO, 0, HOJE).page).toBe(1)
  })

  it('desliga conteúdo adulto e itens de vídeo', () => {
    const p = construirParamsDiscover(FILTROS_PADRAO, 1, HOJE)
    expect(p.include_adult).toBe(false)
    expect(p.include_video).toBe(false)
  })
})
```

- [ ] **Step 6: Rodar e ver falhar**

Run: `npm test -- testes/filtros/discover-params.test.ts`
Expected: FAIL — módulo inexistente.

- [ ] **Step 7: Implementar**

`lib/filtros/discover-params.ts`:

```ts
import type { Params } from '@/lib/tmdb/client'
import { type Filtros, ORDENS, PISO_VOTOS } from './tipos'

export const PAGINA_MAXIMA = 500

function iso(data: Date): string {
  return data.toISOString().slice(0, 10)
}

export function construirParamsDiscover(
  filtros: Filtros,
  pagina: number,
  hoje: Date = new Date()
): Params {
  const usaVoteAverage = filtros.ordem === 'nota' || filtros.notaMinima !== null

  const params: Params = {
    // INVARIANTE: sem watch_region a API devolve o catálogo global com HTTP 200.
    watch_region: 'BR',
    with_watch_monetization_types: filtros.incluirGratis ? 'flatrate|free|ads' : 'flatrate',
    sort_by: ORDENS[filtros.ordem].sortBy,
    page: Math.min(Math.max(Math.trunc(pagina) || 1, 1), PAGINA_MAXIMA),
    include_adult: false,
    include_video: false,
    with_genres: filtros.genero ?? undefined,
    primary_release_year: filtros.ano ?? undefined,
    'vote_average.gte': filtros.notaMinima ?? undefined,
    'vote_count.gte': usaVoteAverage ? PISO_VOTOS : undefined,
    // Pipe = união. Vírgula seria interseção: Netflix ∩ Prime = 124 títulos.
    with_watch_providers: filtros.servicos.length ? filtros.servicos.join('|') : undefined,
    'primary_release_date.lte': filtros.ordem === 'recentes' ? iso(hoje) : undefined,
  }

  return params
}
```

- [ ] **Step 8: Rodar e ver passar**

Run: `npm test -- testes/filtros/`
Expected: PASS, 22 testes no total.

- [ ] **Step 9: Commit**

```bash
git add lib/filtros testes/filtros
git commit -m "feat: camada de filtros com as invariantes de watch_region, pipe e piso de votos"
```

---

## Task 5: Discover e provedores

**Files:**
- Create: `lib/tmdb/discover.ts`, `lib/tmdb/providers.ts`
- Test: `testes/tmdb/discover.test.ts`, `testes/tmdb/providers.test.ts`

**Interfaces:**
- Consumes: `fetchTmdb`, `construirParamsDiscover`, tipos de `lib/tmdb/types`
- Produces:
  - `discoverMovies(f: Filtros, pagina: number): Promise<PaginaDeFilmes>`
  - `type PaginaDeFilmes = { filmes: Filme[]; pagina: number; totalPaginas: number; totalResultados: number; chegouAoFim: boolean }`
  - `getBrProviders(): Promise<Provedor[]>`
  - `normalizarFilme(bruto: unknown): Filme` (exportado para reuso em search)

- [ ] **Step 1: Escrever os testes**

`testes/tmdb/discover.test.ts`:

```ts
import { describe, it, expect, vi, beforeEach } from 'vitest'
import { discoverMovies, normalizarFilme } from '@/lib/tmdb/discover'
import { FILTROS_PADRAO } from '@/lib/filtros/parse'
import * as cliente from '@/lib/tmdb/client'

const BRUTO = {
  id: 27205,
  title: 'A Origem',
  release_date: '2010-07-15',
  poster_path: '/p.jpg',
  backdrop_path: '/b.jpg',
  vote_average: 8.369,
  vote_count: 36000,
  overview: 'Dom Cobb é um ladrão...',
}

beforeEach(() => vi.restoreAllMocks())

describe('normalizarFilme', () => {
  it('extrai o ano da data de lançamento', () => {
    expect(normalizarFilme(BRUTO).ano).toBe(2010)
  })

  it('arredonda a nota para uma casa', () => {
    expect(normalizarFilme(BRUTO).nota).toBe(8.4)
  })

  it('tolera release_date vazio', () => {
    expect(normalizarFilme({ ...BRUTO, release_date: '' }).ano).toBeNull()
  })

  it('tolera poster_path nulo', () => {
    expect(normalizarFilme({ ...BRUTO, poster_path: null }).posterPath).toBeNull()
  })
})

describe('discoverMovies', () => {
  it('limita totalPaginas a 500 mesmo quando a API reporta mais', async () => {
    vi.spyOn(cliente, 'fetchTmdb').mockResolvedValue({
      page: 1, results: [BRUTO], total_pages: 2431, total_results: 48_600,
    })
    const r = await discoverMovies(FILTROS_PADRAO, 1)
    expect(r.totalPaginas).toBe(500)
    expect(r.totalResultados).toBe(48_600)
  })

  it('sinaliza chegouAoFim na última página alcançável', async () => {
    vi.spyOn(cliente, 'fetchTmdb').mockResolvedValue({
      page: 500, results: [BRUTO], total_pages: 2431, total_results: 48_600,
    })
    expect((await discoverMovies(FILTROS_PADRAO, 500)).chegouAoFim).toBe(true)
  })

  it('não sinaliza fim no meio do catálogo', async () => {
    vi.spyOn(cliente, 'fetchTmdb').mockResolvedValue({
      page: 3, results: [BRUTO], total_pages: 245, total_results: 4891,
    })
    expect((await discoverMovies(FILTROS_PADRAO, 3)).chegouAoFim).toBe(false)
  })
})
```

- [ ] **Step 2: Rodar e ver falhar**

Run: `npm test -- testes/tmdb/discover.test.ts`
Expected: FAIL — módulo inexistente.

- [ ] **Step 3: Implementar discover**

`lib/tmdb/discover.ts`:

```ts
import { fetchTmdb } from './client'
import type { Filme } from './types'
import type { Filtros } from '@/lib/filtros/tipos'
import { construirParamsDiscover, PAGINA_MAXIMA } from '@/lib/filtros/discover-params'

export type PaginaDeFilmes = {
  filmes: Filme[]
  pagina: number
  totalPaginas: number
  totalResultados: number
  /** true quando não há mais páginas alcançáveis (limite do TMDB, não fim real do catálogo) */
  chegouAoFim: boolean
}

type RespostaLista = {
  page: number
  results: unknown[]
  total_pages: number
  total_results: number
}

export function normalizarFilme(bruto: unknown): Filme {
  const b = bruto as Record<string, unknown>
  const data = typeof b.release_date === 'string' ? b.release_date : ''
  const ano = data.length >= 4 ? Number(data.slice(0, 4)) : NaN

  return {
    id: Number(b.id),
    titulo: String(b.title ?? ''),
    ano: Number.isInteger(ano) ? ano : null,
    posterPath: (b.poster_path as string | null) ?? null,
    backdropPath: (b.backdrop_path as string | null) ?? null,
    nota: Math.round(Number(b.vote_average ?? 0) * 10) / 10,
    votos: Number(b.vote_count ?? 0),
    sinopse: String(b.overview ?? ''),
  }
}

export async function discoverMovies(filtros: Filtros, pagina: number): Promise<PaginaDeFilmes> {
  const resposta = await fetchTmdb<RespostaLista>(
    '/discover/movie',
    construirParamsDiscover(filtros, pagina),
    { revalidate: 3600 }
  )

  // total_pages da API chega a reportar milhares, mas nada acima de 500 é buscável.
  const totalPaginas = Math.min(resposta.total_pages, PAGINA_MAXIMA)

  return {
    filmes: resposta.results.map(normalizarFilme),
    pagina: resposta.page,
    totalPaginas,
    totalResultados: resposta.total_results,
    chegouAoFim: resposta.page >= totalPaginas,
  }
}
```

- [ ] **Step 4: Escrever os testes de provedores**

`testes/tmdb/providers.test.ts`:

```ts
import { describe, it, expect, vi, beforeEach } from 'vitest'
import { getBrProviders } from '@/lib/tmdb/providers'
import * as cliente from '@/lib/tmdb/client'

beforeEach(() => vi.restoreAllMocks())

const RESPOSTA = {
  results: [
    { provider_id: 1899, provider_name: 'HBO Max', logo_path: '/h.jpg', display_priorities: { BR: 8, US: 3 } },
    { provider_id: 8, provider_name: 'Netflix', logo_path: '/n.jpg', display_priorities: { BR: 0 } },
    { provider_id: 119, provider_name: 'Amazon Prime Video', logo_path: '/a.jpg', display_priorities: { BR: 1 } },
    { provider_id: 999, provider_name: 'Serviço Sem Prioridade BR', logo_path: '/x.jpg', display_priorities: { US: 4 } },
  ],
}

describe('getBrProviders', () => {
  it('ordena ascendente por display_priorities.BR — menor valor primeiro', async () => {
    vi.spyOn(cliente, 'fetchTmdb').mockResolvedValue(RESPOSTA)
    const p = await getBrProviders()
    expect(p.map((x) => x.nome).slice(0, 3)).toEqual(['Netflix', 'Amazon Prime Video', 'HBO Max'])
  })

  it('joga provedores sem prioridade BR para o fim, sem descartá-los', async () => {
    vi.spyOn(cliente, 'fetchTmdb').mockResolvedValue(RESPOSTA)
    const p = await getBrProviders()
    expect(p[p.length - 1].nome).toBe('Serviço Sem Prioridade BR')
    expect(p).toHaveLength(4)
  })

  it('pede a lista com watch_region=BR', async () => {
    const espiao = vi.spyOn(cliente, 'fetchTmdb').mockResolvedValue(RESPOSTA)
    await getBrProviders()
    expect(espiao.mock.calls[0][1]).toMatchObject({ watch_region: 'BR' })
  })
})
```

- [ ] **Step 5: Implementar provedores**

`lib/tmdb/providers.ts`:

```ts
import { fetchTmdb } from './client'
import type { Provedor } from './types'

const SEM_PRIORIDADE = 9999

type ProvedorBruto = {
  provider_id: number
  provider_name: string
  logo_path: string | null
  display_priorities?: Record<string, number>
}

/**
 * Lista viva dos provedores do Brasil. Nunca fixe IDs em código: HBO Max é 1899
 * em BR (o 384 da documentação não existe), Prime Video é 119, e "Apple TV" (350)
 * é a assinatura enquanto "Apple TV Store" (2) é a loja transacional.
 */
export async function getBrProviders(): Promise<Provedor[]> {
  const resposta = await fetchTmdb<{ results: ProvedorBruto[] }>(
    '/watch/providers/movie',
    { watch_region: 'BR' },
    { revalidate: 86_400 }
  )

  return resposta.results
    .map((p) => ({
      id: p.provider_id,
      nome: p.provider_name,
      logoPath: p.logo_path,
      prioridadeBR: p.display_priorities?.BR ?? SEM_PRIORIDADE,
    }))
    .sort((a, b) => a.prioridadeBR - b.prioridadeBR || a.nome.localeCompare(b.nome, 'pt-BR'))
}
```

- [ ] **Step 6: Rodar e ver passar**

Run: `npm test -- testes/tmdb/`
Expected: PASS — 7 de discover, 3 de providers, mais os de client e images.

- [ ] **Step 7: Commit**

```bash
git add lib/tmdb/discover.ts lib/tmdb/providers.ts testes/tmdb/discover.test.ts testes/tmdb/providers.test.ts
git commit -m "feat: discover com limite de 500 páginas e lista viva de provedores BR"
```

---

## Task 6: Detalhe do filme em uma chamada

Verificado em 16/08/2026: `append_to_response=credits,videos,watch/providers` funciona e o bucket `BR` é idêntico ao do endpoint direto.

**Files:**
- Create: `lib/tmdb/movie.ts`
- Test: `testes/tmdb/movie.test.ts`

**Interfaces:**
- Consumes: `fetchTmdb`, `normalizarFilme`, tipos
- Produces:
  - `getMovie(id: number): Promise<DetalheFilme | null>`
  - `getMovieProviders(id: number): Promise<Disponibilidade>`
  - `normalizarDisponibilidade(bucketBR: unknown): Disponibilidade`

- [ ] **Step 1: Escrever os testes**

`testes/tmdb/movie.test.ts`:

```ts
import { describe, it, expect, vi, beforeEach } from 'vitest'
import { getMovie, normalizarDisponibilidade } from '@/lib/tmdb/movie'
import * as cliente from '@/lib/tmdb/client'
import { ErroTmdb } from '@/lib/tmdb/erros'

beforeEach(() => vi.restoreAllMocks())

const DETALHE = {
  id: 550, title: 'Clube da Luta', release_date: '1999-10-15',
  poster_path: '/p.jpg', backdrop_path: '/b.jpg',
  vote_average: 8.4, vote_count: 30000, overview: 'Um funcionário...',
  runtime: 139, genres: [{ id: 18, name: 'Drama' }],
  credits: { cast: [{ name: 'Brad Pitt', character: 'Tyler Durden', profile_path: '/bp.jpg' }] },
  videos: { results: [
    { key: 'abc', type: 'Teaser', site: 'YouTube' },
    { key: 'xyz', type: 'Trailer', site: 'YouTube' },
    { key: 'vim', type: 'Trailer', site: 'Vimeo' },
  ] },
  'watch/providers': { results: { BR: {
    link: 'https://www.themoviedb.org/movie/550/watch?locale=BR',
    flatrate: [{ provider_id: 1899, provider_name: 'HBO Max', logo_path: '/h.jpg', display_priority: 8 }],
    rent: [{ provider_id: 2, provider_name: 'Apple TV Store', logo_path: '/a.jpg', display_priority: 9 }],
  } } },
}

describe('getMovie', () => {
  it('pede tudo numa única chamada com append', async () => {
    const espiao = vi.spyOn(cliente, 'fetchTmdb').mockResolvedValue(DETALHE)
    await getMovie(550)
    expect(espiao).toHaveBeenCalledTimes(1)
    expect(espiao.mock.calls[0][1]).toMatchObject({
      append_to_response: 'credits,videos,watch/providers',
    })
  })

  it('inclui include_video_language para não perder trailer sem versão pt', async () => {
    const espiao = vi.spyOn(cliente, 'fetchTmdb').mockResolvedValue(DETALHE)
    await getMovie(550)
    expect(espiao.mock.calls[0][1]).toMatchObject({ include_video_language: 'pt,en,null' })
  })

  it('escolhe o primeiro Trailer do YouTube, ignorando teaser e outros sites', async () => {
    vi.spyOn(cliente, 'fetchTmdb').mockResolvedValue(DETALHE)
    expect((await getMovie(550))!.trailerYoutubeKey).toBe('xyz')
  })

  it('devolve null quando o filme não existe (404)', async () => {
    vi.spyOn(cliente, 'fetchTmdb').mockRejectedValue(new ErroTmdb('nao encontrado', 404, 34))
    expect(await getMovie(0)).toBeNull()
  })

  it('propaga erros que não são 404', async () => {
    vi.spyOn(cliente, 'fetchTmdb').mockRejectedValue(new ErroTmdb('indisponivel', 503, 46))
    await expect(getMovie(550)).rejects.toBeInstanceOf(ErroTmdb)
  })

  it('sobrevive à ausência total do bucket BR', async () => {
    vi.spyOn(cliente, 'fetchTmdb').mockResolvedValue({
      ...DETALHE, 'watch/providers': { results: {} },
    })
    const d = await getMovie(550)
    expect(d!.disponibilidade.ofertas).toEqual([])
    expect(d!.disponibilidade.link).toBeNull()
  })
})

describe('normalizarDisponibilidade', () => {
  it('ordena assinatura antes de transacional', () => {
    const d = normalizarDisponibilidade(DETALHE['watch/providers'].results.BR)
    expect(d.ofertas.map((o) => o.tipo)).toEqual(['flatrate', 'rent'])
  })

  it('trata free como chave de primeira classe', () => {
    const d = normalizarDisponibilidade({
      link: 'x',
      free: [{ provider_id: 1, provider_name: 'Pluto', logo_path: '/p.jpg', display_priority: 1 }],
    })
    expect(d.ofertas[0].tipo).toBe('free')
  })

  it('ignora chaves ausentes sem lançar', () => {
    expect(() => normalizarDisponibilidade({ link: 'x' })).not.toThrow()
    expect(normalizarDisponibilidade({ link: 'x' }).ofertas).toEqual([])
  })
})
```

- [ ] **Step 2: Rodar e ver falhar**

Run: `npm test -- testes/tmdb/movie.test.ts`
Expected: FAIL — módulo inexistente.

- [ ] **Step 3: Implementar**

`lib/tmdb/movie.ts`:

```ts
import { fetchTmdb } from './client'
import { ErroTmdb } from './erros'
import { normalizarFilme } from './discover'
import {
  type DetalheFilme, type Disponibilidade, type Oferta,
  type Pessoa, type Provedor, type TipoOferta, TIPOS_OFERTA,
} from './types'

const TTL_DISPONIBILIDADE = 21_600 // 6 h — o dado mais volátil manda no TTL combinado

type ProvedorBruto = {
  provider_id: number
  provider_name: string
  logo_path: string | null
  display_priority?: number
}

export function normalizarDisponibilidade(bucket: unknown): Disponibilidade {
  const b = (bucket ?? {}) as Record<string, unknown>

  const ofertas: Oferta[] = []
  for (const tipo of TIPOS_OFERTA) {
    const lista = b[tipo] as ProvedorBruto[] | undefined
    // A API só inclui a chave quando existe oferta daquele tipo.
    if (!Array.isArray(lista) || lista.length === 0) continue
    const provedores: Provedor[] = lista
      .map((p) => ({
        id: p.provider_id,
        nome: p.provider_name,
        logoPath: p.logo_path,
        prioridadeBR: p.display_priority ?? 9999,
      }))
      .sort((x, y) => x.prioridadeBR - y.prioridadeBR)
    ofertas.push({ tipo: tipo as TipoOferta, provedores })
  }

  return { link: typeof b.link === 'string' ? b.link : null, ofertas }
}

export async function getMovie(id: number): Promise<DetalheFilme | null> {
  let bruto: Record<string, unknown>
  try {
    bruto = await fetchTmdb<Record<string, unknown>>(
      `/movie/${id}`,
      {
        append_to_response: 'credits,videos,watch/providers',
        include_video_language: 'pt,en,null',
      },
      { revalidate: TTL_DISPONIBILIDADE }
    )
  } catch (erro) {
    if (erro instanceof ErroTmdb && erro.status === 404) return null
    throw erro
  }

  const videos = (bruto.videos as { results?: { key: string; type: string; site: string }[] })?.results ?? []
  const trailer = videos.find((v) => v.type === 'Trailer' && v.site === 'YouTube')

  const elencoBruto =
    (bruto.credits as { cast?: { name: string; character: string; profile_path: string | null }[] })
      ?.cast ?? []

  const bucketBR = (bruto['watch/providers'] as { results?: Record<string, unknown> })?.results?.BR

  const elenco: Pessoa[] = elencoBruto.slice(0, 12).map((p) => ({
    nome: p.name,
    personagem: p.character,
    fotoPath: p.profile_path,
  }))

  return {
    ...normalizarFilme(bruto),
    duracaoMin: typeof bruto.runtime === 'number' && bruto.runtime > 0 ? bruto.runtime : null,
    generos: ((bruto.genres as { name: string }[]) ?? []).map((g) => g.name),
    disponibilidade: normalizarDisponibilidade(bucketBR),
    elenco,
    trailerYoutubeKey: trailer?.key ?? null,
  }
}

/** Chamada isolada — usada apenas pela anotação da busca (§3.7). */
export async function getMovieProviders(id: number): Promise<Disponibilidade> {
  const r = await fetchTmdb<{ results?: Record<string, unknown> }>(
    `/movie/${id}/watch/providers`,
    {},
    { revalidate: TTL_DISPONIBILIDADE }
  )
  return normalizarDisponibilidade(r.results?.BR)
}
```

- [ ] **Step 4: Rodar e ver passar**

Run: `npm test -- testes/tmdb/movie.test.ts`
Expected: PASS, 9 testes.

- [ ] **Step 5: Commit**

```bash
git add lib/tmdb/movie.ts testes/tmdb/movie.test.ts
git commit -m "feat: detalhe do filme em uma chamada com append_to_response"
```

---

## Task 7: Busca com anotação e particionamento

**Files:**
- Create: `lib/tmdb/search.ts`
- Test: `testes/tmdb/search.test.ts`

**Interfaces:**
- Consumes: `fetchTmdb`, `normalizarFilme`, `getMovieProviders`
- Produces:
  - `LIMITE_ANOTACAO = 10`
  - `type ResultadoBusca = Filme & { disponibilidade: Disponibilidade | null }`
  - `type ResultadoParticionado = { nosServicos: ResultadoBusca[]; fora: ResultadoBusca[]; totalResultados: number; anotados: number }`
  - `searchMovies(termo: string, servicos: number[]): Promise<ResultadoParticionado>`

- [ ] **Step 1: Escrever os testes**

`testes/tmdb/search.test.ts`:

```ts
import { describe, it, expect, vi, beforeEach } from 'vitest'
import { searchMovies, LIMITE_ANOTACAO } from '@/lib/tmdb/search'
import * as cliente from '@/lib/tmdb/client'
import * as filme from '@/lib/tmdb/movie'

beforeEach(() => vi.restoreAllMocks())

const item = (id: number) => ({
  id, title: `Filme ${id}`, release_date: '2020-01-01',
  poster_path: '/p.jpg', backdrop_path: null,
  vote_average: 7, vote_count: 1000, overview: '',
})

const disp = (idProvedor: number) => ({
  link: 'https://tmdb/watch',
  ofertas: [{
    tipo: 'flatrate' as const,
    provedores: [{ id: idProvedor, nome: 'X', logoPath: '/x.jpg', prioridadeBR: 1 }],
  }],
})

describe('searchMovies', () => {
  it('NÃO envia parâmetros de provedor — a API os ignora silenciosamente', async () => {
    const espiao = vi.spyOn(cliente, 'fetchTmdb')
      .mockResolvedValue({ results: [], total_results: 0 })
    await searchMovies('matrix', [8, 119])
    const params = espiao.mock.calls[0][1] as Record<string, unknown>
    expect(params.with_watch_providers).toBeUndefined()
    expect(params.watch_region).toBeUndefined()
    expect(params.query).toBe('matrix')
  })

  it('separa quem está nos serviços marcados de quem não está', async () => {
    vi.spyOn(cliente, 'fetchTmdb')
      .mockResolvedValue({ results: [item(1), item(2)], total_results: 2 })
    vi.spyOn(filme, 'getMovieProviders')
      .mockImplementation(async (id) => (id === 1 ? disp(8) : disp(555)))

    const r = await searchMovies('x', [8])
    expect(r.nosServicos.map((f) => f.id)).toEqual([1])
    expect(r.fora.map((f) => f.id)).toEqual([2])
  })

  it(`anota no máximo ${LIMITE_ANOTACAO} resultados`, async () => {
    const muitos = Array.from({ length: 20 }, (_, i) => item(i + 1))
    vi.spyOn(cliente, 'fetchTmdb').mockResolvedValue({ results: muitos, total_results: 20 })
    const espiao = vi.spyOn(filme, 'getMovieProviders').mockResolvedValue(disp(8))

    const r = await searchMovies('x', [8])
    expect(espiao).toHaveBeenCalledTimes(LIMITE_ANOTACAO)
    expect(r.anotados).toBe(LIMITE_ANOTACAO)
  })

  it('resultado não anotado tem disponibilidade null e cai no grupo "fora"', async () => {
    const muitos = Array.from({ length: 12 }, (_, i) => item(i + 1))
    vi.spyOn(cliente, 'fetchTmdb').mockResolvedValue({ results: muitos, total_results: 12 })
    vi.spyOn(filme, 'getMovieProviders').mockResolvedValue(disp(8))

    const r = await searchMovies('x', [8])
    const naoAnotado = r.fora.find((f) => f.id === 12)
    expect(naoAnotado?.disponibilidade).toBeNull()
  })

  it('falha de anotação não derruba a busca — o item cai em "fora"', async () => {
    vi.spyOn(cliente, 'fetchTmdb')
      .mockResolvedValue({ results: [item(1), item(2)], total_results: 2 })
    vi.spyOn(filme, 'getMovieProviders').mockImplementation(async (id) => {
      if (id === 1) throw new Error('rede caiu')
      return disp(8)
    })

    const r = await searchMovies('x', [8])
    expect(r.nosServicos.map((f) => f.id)).toEqual([2])
    expect(r.fora.find((f) => f.id === 1)?.disponibilidade).toBeNull()
  })

  it('sem serviços marcados, tudo cai em "fora" e nada quebra', async () => {
    vi.spyOn(cliente, 'fetchTmdb')
      .mockResolvedValue({ results: [item(1)], total_results: 1 })
    vi.spyOn(filme, 'getMovieProviders').mockResolvedValue(disp(8))

    const r = await searchMovies('x', [])
    expect(r.nosServicos).toEqual([])
    expect(r.fora).toHaveLength(1)
  })

  it('termo vazio não chama a API', async () => {
    const espiao = vi.spyOn(cliente, 'fetchTmdb')
    const r = await searchMovies('   ', [8])
    expect(espiao).not.toHaveBeenCalled()
    expect(r.totalResultados).toBe(0)
  })
})
```

- [ ] **Step 2: Rodar e ver falhar**

Run: `npm test -- testes/tmdb/search.test.ts`
Expected: FAIL — módulo inexistente.

- [ ] **Step 3: Implementar**

`lib/tmdb/search.ts`:

```ts
import { fetchTmdb } from './client'
import { normalizarFilme } from './discover'
import { getMovieProviders } from './movie'
import { type Disponibilidade, type Filme, TIPOS_SEM_CUSTO_EXTRA } from './types'

/** Teto de anotação: o N+1 é aceitável aqui porque o conjunto é pequeno e a ação é deliberada. */
export const LIMITE_ANOTACAO = 10

export type ResultadoBusca = Filme & { disponibilidade: Disponibilidade | null }

export type ResultadoParticionado = {
  nosServicos: ResultadoBusca[]
  fora: ResultadoBusca[]
  totalResultados: number
  /** quantos itens receberam anotação — o resto exibe "disponibilidade não verificada" */
  anotados: number
}

function estaNosServicos(d: Disponibilidade | null, servicos: number[]): boolean {
  if (!d || servicos.length === 0) return false
  const marcados = new Set(servicos)
  return d.ofertas
    .filter((o) => TIPOS_SEM_CUSTO_EXTRA.includes(o.tipo))
    .some((o) => o.provedores.some((p) => marcados.has(p.id)))
}

export async function searchMovies(
  termo: string,
  servicos: number[]
): Promise<ResultadoParticionado> {
  const query = termo.trim()
  if (!query) return { nosServicos: [], fora: [], totalResultados: 0, anotados: 0 }

  // A API ignora with_watch_providers e watch_region aqui — enviá-los daria falsa
  // impressão de filtro. Verificado: 92 resultados com e sem os parâmetros.
  const resposta = await fetchTmdb<{ results: unknown[]; total_results: number }>(
    '/search/movie',
    { query, include_adult: false },
    { revalidate: 3600 }
  )

  const filmes = resposta.results.map(normalizarFilme)
  const paraAnotar = filmes.slice(0, LIMITE_ANOTACAO)

  const disponibilidades = await Promise.all(
    paraAnotar.map((f) => getMovieProviders(f.id).catch(() => null))
  )

  const resultados: ResultadoBusca[] = filmes.map((f, i) => ({
    ...f,
    disponibilidade: i < LIMITE_ANOTACAO ? disponibilidades[i] : null,
  }))

  const nosServicos: ResultadoBusca[] = []
  const fora: ResultadoBusca[] = []
  for (const r of resultados) {
    ;(estaNosServicos(r.disponibilidade, servicos) ? nosServicos : fora).push(r)
  }

  return {
    nosServicos,
    fora,
    totalResultados: resposta.total_results,
    anotados: paraAnotar.length,
  }
}
```

- [ ] **Step 4: Rodar e ver passar**

Run: `npm test -- testes/tmdb/search.test.ts`
Expected: PASS, 7 testes.

- [ ] **Step 5: Commit**

```bash
git add lib/tmdb/search.ts testes/tmdb/search.test.ts
git commit -m "feat: busca com anotação limitada a 10 e particionamento por serviço"
```

---

## Task 8: Watchlist em localStorage

**Files:**
- Create: `lib/lista/armazenamento.ts`, `lib/lista/useLista.ts`
- Test: `testes/lista/armazenamento.test.ts`

**Interfaces:**
- Consumes: nada
- Produces:
  - `type ItemLista = { id: number; titulo: string; posterPath: string | null; ano: number | null; salvoEm: string }`
  - `lerLista(): ItemLista[]` · `salvarItem(item: Omit<ItemLista,'salvoEm'>): ItemLista[]` · `removerItem(id: number): ItemLista[]` · `estaNaLista(id: number): boolean`
  - `useLista(): { itens: ItemLista[]; alternar(item: Omit<ItemLista,'salvoEm'>): void; contem(id: number): boolean; montado: boolean }`

`montado` é `false` até a hidratação terminar. Componentes devem tratá-lo como "ainda não sei" — nunca renderizar "não está na lista" antes dele virar `true`, senão o botão pisca no estado errado.

- [ ] **Step 1: Escrever os testes**

`testes/lista/armazenamento.test.ts`:

```ts
import { describe, it, expect, beforeEach, vi } from 'vitest'
import { lerLista, salvarItem, removerItem, CHAVE, VERSAO } from '@/lib/lista/armazenamento'

const item = { id: 550, titulo: 'Clube da Luta', posterPath: '/p.jpg', ano: 1999 }

beforeEach(() => {
  localStorage.clear()
  vi.restoreAllMocks()
})

describe('armazenamento da watchlist', () => {
  it('começa vazia', () => {
    expect(lerLista()).toEqual([])
  })

  it('salva e relê', () => {
    salvarItem(item)
    expect(lerLista()).toHaveLength(1)
    expect(lerLista()[0].titulo).toBe('Clube da Luta')
  })

  it('guarda snapshot completo, não só o id — /minha-lista renderiza sem chamar a API', () => {
    salvarItem(item)
    const guardado = lerLista()[0]
    expect(guardado.posterPath).toBe('/p.jpg')
    expect(guardado.ano).toBe(1999)
    expect(guardado.salvoEm).toBeTruthy()
  })

  it('não duplica o mesmo filme', () => {
    salvarItem(item)
    salvarItem(item)
    expect(lerLista()).toHaveLength(1)
  })

  it('remove por id', () => {
    salvarItem(item)
    expect(removerItem(550)).toEqual([])
  })

  it('grava com o número de versão do schema', () => {
    salvarItem(item)
    expect(JSON.parse(localStorage.getItem(CHAVE)!).v).toBe(VERSAO)
  })

  it('JSON corrompido devolve lista vazia em vez de explodir', () => {
    localStorage.setItem(CHAVE, '{isso não é json')
    expect(() => lerLista()).not.toThrow()
    expect(lerLista()).toEqual([])
  })

  it('versão desconhecida é descartada em vez de interpretada errado', () => {
    localStorage.setItem(CHAVE, JSON.stringify({ v: 99, items: [{ id: 1 }] }))
    expect(lerLista()).toEqual([])
  })

  it('cota estourada não derruba a aplicação', () => {
    vi.spyOn(Storage.prototype, 'setItem').mockImplementation(() => {
      throw new DOMException('cheio', 'QuotaExceededError')
    })
    expect(() => salvarItem(item)).not.toThrow()
  })

  it('mais recentes primeiro', () => {
    salvarItem({ ...item, id: 1, titulo: 'Primeiro' })
    salvarItem({ ...item, id: 2, titulo: 'Segundo' })
    expect(lerLista()[0].titulo).toBe('Segundo')
  })
})
```

- [ ] **Step 2: Rodar e ver falhar**

Run: `npm test -- testes/lista/armazenamento.test.ts`
Expected: FAIL — módulo inexistente.

- [ ] **Step 3: Implementar o armazenamento**

`lib/lista/armazenamento.ts`:

```ts
export const CHAVE = 'ondeassisto:lista'
export const VERSAO = 1

export type ItemLista = {
  id: number
  titulo: string
  posterPath: string | null
  ano: number | null
  salvoEm: string
}

type Envelope = { v: number; items: ItemLista[] }

function disponivel(): boolean {
  try {
    return typeof window !== 'undefined' && !!window.localStorage
  } catch {
    return false
  }
}

export function lerLista(): ItemLista[] {
  if (!disponivel()) return []
  try {
    const cru = localStorage.getItem(CHAVE)
    if (!cru) return []
    const env = JSON.parse(cru) as Envelope
    // Versão desconhecida: descartar é mais seguro que interpretar errado.
    if (env?.v !== VERSAO || !Array.isArray(env.items)) return []
    return env.items.filter((i) => typeof i?.id === 'number')
  } catch {
    return []
  }
}

function gravar(items: ItemLista[]): ItemLista[] {
  if (!disponivel()) return items
  try {
    localStorage.setItem(CHAVE, JSON.stringify({ v: VERSAO, items } satisfies Envelope))
  } catch {
    // Cota estourada ou modo restrito: a lista vale para esta sessão e o app segue.
  }
  return items
}

export function salvarItem(item: Omit<ItemLista, 'salvoEm'>): ItemLista[] {
  const atual = lerLista().filter((i) => i.id !== item.id)
  return gravar([{ ...item, salvoEm: new Date().toISOString() }, ...atual])
}

export function removerItem(id: number): ItemLista[] {
  return gravar(lerLista().filter((i) => i.id !== id))
}

export function estaNaLista(id: number): boolean {
  return lerLista().some((i) => i.id === id)
}
```

- [ ] **Step 4: Implementar o hook**

`lib/lista/useLista.ts`:

```ts
'use client'

import { useCallback, useEffect, useState } from 'react'
import { type ItemLista, lerLista, removerItem, salvarItem } from './armazenamento'

export function useLista() {
  const [itens, setItens] = useState<ItemLista[]>([])
  const [montado, setMontado] = useState(false)

  // localStorage não existe no servidor: só ler depois da hidratação.
  useEffect(() => {
    setItens(lerLista())
    setMontado(true)
  }, [])

  const alternar = useCallback((item: Omit<ItemLista, 'salvoEm'>) => {
    setItens((atuais) =>
      atuais.some((i) => i.id === item.id) ? removerItem(item.id) : salvarItem(item)
    )
  }, [])

  const contem = useCallback((id: number) => itens.some((i) => i.id === id), [itens])

  return { itens, alternar, contem, montado }
}
```

- [ ] **Step 5: Rodar e ver passar**

Run: `npm test -- testes/lista/`
Expected: PASS, 10 testes.

- [ ] **Step 6: Commit**

```bash
git add lib/lista testes/lista
git commit -m "feat: watchlist em localStorage com schema versionado e degradação segura"
```

---

## Task 9: Componentes de card, grade e skeleton

**Files:**
- Create: `componentes/CardFilme.tsx`, `componentes/GradeFilmes.tsx`, `componentes/SkeletonGrade.tsx`, `componentes/EstadoVazio.tsx`, `componentes/BotaoLista.tsx`
- Test: `testes/componentes/CardFilme.test.tsx`

**Interfaces:**
- Consumes: `Filme`, `posterUrl`, `useLista`
- Produces: `<CardFilme filme={Filme} />`, `<GradeFilmes filmes={Filme[]} />`, `<SkeletonGrade quantidade={number} />`, `<EstadoVazio titulo mensagem />`, `<BotaoLista filme={Filme} />`

- [ ] **Step 1: Escrever os testes**

`testes/componentes/CardFilme.test.tsx`:

```tsx
import { describe, it, expect } from 'vitest'
import { render, screen } from '@testing-library/react'
import { CardFilme } from '@/componentes/CardFilme'
import type { Filme } from '@/lib/tmdb/types'

const filme: Filme = {
  id: 550, titulo: 'Clube da Luta', ano: 1999,
  posterPath: '/p.jpg', backdropPath: null,
  nota: 8.4, votos: 30000, sinopse: '',
}

describe('CardFilme', () => {
  it('usa o título do filme como texto alternativo do pôster', () => {
    render(<CardFilme filme={filme} />)
    expect(screen.getByAltText('Clube da Luta')).toBeInTheDocument()
  })

  it('mostra título e ano como texto, fora da imagem', () => {
    render(<CardFilme filme={filme} />)
    expect(screen.getByText('Clube da Luta')).toBeInTheDocument()
    expect(screen.getByText('1999')).toBeInTheDocument()
  })

  it('leva para a página do filme', () => {
    render(<CardFilme filme={filme} />)
    expect(screen.getByRole('link')).toHaveAttribute('href', '/filme/550')
  })

  it('exibe a nota com uma casa decimal', () => {
    render(<CardFilme filme={filme} />)
    expect(screen.getByText('8.4')).toBeInTheDocument()
  })

  it('sem pôster, mostra placeholder com alt próprio — nunca alt vazio', () => {
    render(<CardFilme filme={{ ...filme, posterPath: null }} />)
    expect(screen.getByAltText('Clube da Luta — pôster indisponível')).toBeInTheDocument()
  })

  it('omite o ano quando não há data de lançamento', () => {
    render(<CardFilme filme={{ ...filme, ano: null }} />)
    expect(screen.queryByText('1999')).not.toBeInTheDocument()
  })
})
```

- [ ] **Step 2: Rodar e ver falhar**

Run: `npm test -- testes/componentes/CardFilme.test.tsx`
Expected: FAIL — componente inexistente.

- [ ] **Step 3: Implementar o card**

`componentes/CardFilme.tsx`:

```tsx
import Image from 'next/image'
import Link from 'next/link'
import { posterUrl } from '@/lib/tmdb/images'
import type { Filme } from '@/lib/tmdb/types'

export function CardFilme({ filme }: { filme: Filme }) {
  const url = posterUrl(filme.posterPath, 'w342')

  return (
    <Link href={`/filme/${filme.id}`} className="group block rounded-md">
      <div className="relative aspect-[2/3] overflow-hidden rounded-md bg-superficie">
        {url ? (
          <Image
            src={url}
            alt={filme.titulo}
            fill
            sizes="(max-width: 640px) 45vw, (max-width: 1024px) 22vw, 15vw"
            className="object-cover transition-transform group-hover:scale-105"
          />
        ) : (
          <div
            role="img"
            aria-label={`${filme.titulo} — pôster indisponível`}
            className="flex h-full items-center justify-center px-2 text-center text-xs text-secundario"
          >
            sem pôster
          </div>
        )}

        {filme.nota > 0 && (
          // Pílula opaca: única sobreposição permitida sobre a capa.
          <span className="absolute right-1.5 top-1.5 rounded bg-ambar px-1.5 py-0.5 text-xs font-bold text-fundo">
            {filme.nota.toFixed(1)}
          </span>
        )}
      </div>

      {/* Texto FORA da imagem — contraste sobre capa arbitrária é indeterminado. */}
      <div className="mt-2">
        <h3 className="line-clamp-2 text-sm font-medium leading-tight text-texto">{filme.titulo}</h3>
        {filme.ano !== null && <p className="mt-0.5 text-xs text-secundario">{filme.ano}</p>}
      </div>
    </Link>
  )
}
```

Nota: o placeholder usa `role="img"` + `aria-label` porque não há elemento `<img>` — é a forma correta de dar nome acessível a um substituto visual.

- [ ] **Step 4: Implementar grade, skeleton e estado vazio**

`componentes/GradeFilmes.tsx`:

```tsx
import { CardFilme } from './CardFilme'
import type { Filme } from '@/lib/tmdb/types'

export function GradeFilmes({ filmes }: { filmes: Filme[] }) {
  return (
    <ul className="grid grid-cols-2 gap-4 sm:grid-cols-3 md:grid-cols-4 lg:grid-cols-6">
      {filmes.map((filme) => (
        <li key={filme.id}>
          <CardFilme filme={filme} />
        </li>
      ))}
    </ul>
  )
}
```

`componentes/SkeletonGrade.tsx`:

```tsx
export function SkeletonGrade({ quantidade = 18 }: { quantidade?: number }) {
  return (
    <div
      className="grid grid-cols-2 gap-4 sm:grid-cols-3 md:grid-cols-4 lg:grid-cols-6"
      aria-busy="true"
      aria-label="Carregando filmes"
    >
      {Array.from({ length: quantidade }, (_, i) => (
        <div key={i}>
          <div className="aspect-[2/3] animate-pulse rounded-md bg-superficie" />
          <div className="mt-2 h-3 w-3/4 animate-pulse rounded bg-superficie" />
        </div>
      ))}
    </div>
  )
}
```

`componentes/EstadoVazio.tsx`:

```tsx
export function EstadoVazio({ titulo, mensagem }: { titulo: string; mensagem: string }) {
  return (
    <div className="py-16 text-center">
      <p className="text-lg font-medium text-texto">{titulo}</p>
      <p className="mt-2 text-sm text-secundario">{mensagem}</p>
    </div>
  )
}
```

- [ ] **Step 5: Implementar o botão de lista**

`componentes/BotaoLista.tsx`:

```tsx
'use client'

import { useLista } from '@/lib/lista/useLista'
import type { Filme } from '@/lib/tmdb/types'

export function BotaoLista({ filme }: { filme: Filme }) {
  const { alternar, contem, montado } = useLista()
  const salvo = montado && contem(filme.id)

  return (
    <button
      type="button"
      aria-pressed={salvo}
      onClick={() =>
        alternar({
          id: filme.id, titulo: filme.titulo,
          posterPath: filme.posterPath, ano: filme.ano,
        })
      }
      className={`rounded-md px-3 py-1.5 text-sm font-medium transition-colors ${
        salvo ? 'bg-ambar text-fundo' : 'border border-borda text-texto hover:border-ambar'
      }`}
    >
      {salvo ? '✓ Na minha lista' : '+ Minha lista'}
    </button>
  )
}
```

- [ ] **Step 6: Rodar e ver passar**

Run: `npm test -- testes/componentes/`
Expected: PASS, 6 testes.

- [ ] **Step 7: Commit**

```bash
git add componentes testes/componentes
git commit -m "feat: card, grade, skeleton e botão de lista com texto fora do pôster"
```

---

## Task 10: Grade — rota `/` com barra de filtros

**Files:**
- Create: `app/page.tsx`, `app/loading.tsx`, `componentes/BarraFiltros.tsx`, `componentes/CampoBusca.tsx`
- Modify: `app/layout.tsx`

**Interfaces:**
- Consumes: `discoverMovies`, `getBrProviders`, `parseSearchParams`, `toSearchParams`, `GradeFilmes`, `SkeletonGrade`
- Produces: rota `/` funcional com filtros na URL

- [ ] **Step 1: Implementar a barra de filtros**

`componentes/BarraFiltros.tsx`:

```tsx
'use client'

import { useRouter, useSearchParams } from 'next/navigation'
import { useTransition } from 'react'
import { parseSearchParams, toSearchParams } from '@/lib/filtros/parse'
import { ORDENS } from '@/lib/filtros/tipos'
import type { Provedor } from '@/lib/tmdb/types'

export function BarraFiltros({ provedores }: { provedores: Provedor[] }) {
  const router = useRouter()
  const sp = useSearchParams()
  const [pendente, iniciar] = useTransition()
  const filtros = parseSearchParams(sp)

  function navegar(novos: Partial<typeof filtros>) {
    const params = toSearchParams({ ...filtros, ...novos })
    iniciar(() => router.push(params.toString() ? `/?${params}` : '/'))
  }

  function alternarServico(id: number) {
    const ativos = filtros.servicos.includes(id)
      ? filtros.servicos.filter((s) => s !== id)
      : [...filtros.servicos, id]
    navegar({ servicos: ativos })
  }

  return (
    <div className="flex flex-wrap items-center gap-2" data-pendente={pendente}>
      {provedores.map((p) => {
        const ativo = filtros.servicos.includes(p.id)
        return (
          <button
            key={p.id}
            type="button"
            aria-pressed={ativo}
            onClick={() => alternarServico(p.id)}
            className={`min-h-11 rounded-full border px-3 text-sm transition-colors ${
              ativo
                ? 'border-ambar bg-ambar font-semibold text-fundo'
                : 'border-borda bg-superficie text-secundario hover:border-ambar'
            }`}
          >
            {p.nome}
          </button>
        )
      })}

      <label className="ml-auto flex items-center gap-2 text-sm text-secundario">
        Ordenar
        <select
          value={filtros.ordem}
          onChange={(e) => navegar({ ordem: e.target.value as typeof filtros.ordem })}
          className="min-h-11 rounded-md border border-borda bg-superficie px-2 text-texto"
        >
          {Object.entries(ORDENS).map(([chave, { rotulo }]) => (
            <option key={chave} value={chave}>{rotulo}</option>
          ))}
        </select>
      </label>
    </div>
  )
}
```

- [ ] **Step 2: Implementar o campo de busca**

`componentes/CampoBusca.tsx`:

```tsx
'use client'

import { useRouter } from 'next/navigation'
import { useState } from 'react'

export function CampoBusca() {
  const router = useRouter()
  const [termo, setTermo] = useState('')

  return (
    <form
      role="search"
      onSubmit={(e) => {
        e.preventDefault()
        if (termo.trim()) router.push(`/busca?q=${encodeURIComponent(termo.trim())}`)
      }}
    >
      <input
        type="search"
        value={termo}
        onChange={(e) => setTermo(e.target.value)}
        placeholder="Buscar um filme"
        aria-label="Buscar um filme pelo título"
        className="min-h-11 w-full rounded-full border border-borda bg-superficie px-4 text-sm text-texto placeholder:text-secundario sm:w-64"
      />
    </form>
  )
}
```

- [ ] **Step 3: Acrescentar cabeçalho e rodapé ao layout**

Em `app/layout.tsx`, substitua `{children}` por:

```tsx
<header className="sticky top-0 z-10 border-b border-borda bg-fundo/95 backdrop-blur">
  <div className="mx-auto flex max-w-7xl flex-wrap items-center gap-4 px-4 py-3">
    <a href="/" className="text-lg font-extrabold tracking-tight text-texto">ondeassisto</a>
    <nav className="text-sm text-secundario">
      <a href="/minha-lista" className="hover:text-texto">Minha lista</a>
    </nav>
    <div className="ml-auto"><CampoBusca /></div>
  </div>
</header>

<main className="mx-auto max-w-7xl px-4 py-6">{children}</main>

<Atribuicao />
```

Importe `CampoBusca` e `Atribuicao` no topo. `Atribuicao` é criado na Task 13 — até lá, comente a linha.

- [ ] **Step 4: Implementar a página da grade**

`app/page.tsx`:

```tsx
import { BarraFiltros } from '@/componentes/BarraFiltros'
import { GradeFilmes } from '@/componentes/GradeFilmes'
import { EstadoVazio } from '@/componentes/EstadoVazio'
import { discoverMovies } from '@/lib/tmdb/discover'
import { getBrProviders } from '@/lib/tmdb/providers'
import { parseSearchParams } from '@/lib/filtros/parse'

export default async function Home({
  searchParams,
}: {
  searchParams: Promise<Record<string, string | string[] | undefined>>
}) {
  const sp = await searchParams
  const filtros = parseSearchParams(sp)

  const [provedores, pagina] = await Promise.all([
    getBrProviders(),
    discoverMovies(filtros, 1),
  ])

  return (
    <div className="space-y-6">
      <BarraFiltros provedores={provedores.slice(0, 12)} />

      {pagina.filmes.length === 0 ? (
        <EstadoVazio
          titulo="Nenhum filme com esses filtros"
          mensagem="Tente remover um filtro ou marcar mais serviços."
        />
      ) : (
        <>
          <p className="text-sm text-secundario">
            {pagina.totalResultados.toLocaleString('pt-BR')} filmes
          </p>
          <GradeFilmes filmes={pagina.filmes} />
        </>
      )}
    </div>
  )
}
```

- [ ] **Step 5: Skeleton de carregamento**

`app/loading.tsx`:

```tsx
import { SkeletonGrade } from '@/componentes/SkeletonGrade'

export default function Loading() {
  return <SkeletonGrade />
}
```

- [ ] **Step 6: Configurar o domínio de imagens**

`next.config.ts`:

```ts
import type { NextConfig } from 'next'

const config: NextConfig = {
  images: {
    remotePatterns: [{ protocol: 'https', hostname: 'image.tmdb.org', pathname: '/t/p/**' }],
  },
}

export default config
```

- [ ] **Step 7: Verificar no navegador**

Run: `npm run dev` e abra `http://localhost:3000`
Expected: grade carrega, chips de provedor aparecem, clicar num chip muda a URL para `/?servicos=8` e a grade recarrega.

- [ ] **Step 8: Commit**

```bash
git add app componentes/BarraFiltros.tsx componentes/CampoBusca.tsx next.config.ts
git commit -m "feat: grade com filtros na URL e skeleton de carregamento"
```

---

## Task 11: Scroll infinito

**Files:**
- Create: `app/api/discover/route.ts`, `componentes/CarregadorInfinito.tsx`
- Modify: `app/page.tsx`
- Test: `testes/api/discover.test.ts`

**Interfaces:**
- Consumes: `discoverMovies`, `parseSearchParams`
- Produces: `GET /api/discover?page=N&<filtros>` → `PaginaDeFilmes` em JSON

- [ ] **Step 1: Escrever o teste da rota**

`testes/api/discover.test.ts`:

```ts
import { describe, it, expect, vi, beforeEach } from 'vitest'
import { GET } from '@/app/api/discover/route'
import * as descoberta from '@/lib/tmdb/discover'

beforeEach(() => vi.restoreAllMocks())

const pagina = {
  filmes: [], pagina: 2, totalPaginas: 245, totalResultados: 4891, chegouAoFim: false,
}

function req(qs: string) {
  return new Request(`http://localhost/api/discover?${qs}`)
}

describe('GET /api/discover', () => {
  it('repassa os filtros da query string', async () => {
    const espiao = vi.spyOn(descoberta, 'discoverMovies').mockResolvedValue(pagina)
    await GET(req('servicos=8,119&page=2'))
    expect(espiao.mock.calls[0][0].servicos).toEqual([8, 119])
    expect(espiao.mock.calls[0][1]).toBe(2)
  })

  it('página ausente vira 1', async () => {
    const espiao = vi.spyOn(descoberta, 'discoverMovies').mockResolvedValue(pagina)
    await GET(req('servicos=8'))
    expect(espiao.mock.calls[0][1]).toBe(1)
  })

  it('devolve JSON com o formato de PaginaDeFilmes', async () => {
    vi.spyOn(descoberta, 'discoverMovies').mockResolvedValue(pagina)
    const corpo = await (await GET(req('page=2'))).json()
    expect(corpo).toMatchObject({ pagina: 2, chegouAoFim: false })
  })

  it('erro na API vira 502, não exceção não tratada', async () => {
    vi.spyOn(descoberta, 'discoverMovies').mockRejectedValue(new Error('caiu'))
    expect((await GET(req('page=2'))).status).toBe(502)
  })
})
```

- [ ] **Step 2: Rodar e ver falhar**

Run: `npm test -- testes/api/discover.test.ts`
Expected: FAIL — rota inexistente.

- [ ] **Step 3: Implementar a rota**

`app/api/discover/route.ts`:

```ts
import { NextResponse } from 'next/server'
import { discoverMovies } from '@/lib/tmdb/discover'
import { parseSearchParams } from '@/lib/filtros/parse'

export async function GET(request: Request) {
  const url = new URL(request.url)
  const filtros = parseSearchParams(url.searchParams)
  const pagina = Number(url.searchParams.get('page')) || 1

  try {
    // O limite de 500 é aplicado dentro de construirParamsDiscover.
    return NextResponse.json(await discoverMovies(filtros, pagina))
  } catch {
    return NextResponse.json({ erro: 'Não foi possível carregar mais filmes.' }, { status: 502 })
  }
}
```

- [ ] **Step 4: Implementar o carregador**

`componentes/CarregadorInfinito.tsx`:

```tsx
'use client'

import { useEffect, useRef, useState } from 'react'
import { useSearchParams } from 'next/navigation'
import { GradeFilmes } from './GradeFilmes'
import type { Filme } from '@/lib/tmdb/types'

export function CarregadorInfinito({
  paginaInicial,
  chegouAoFimInicial,
}: {
  paginaInicial: number
  chegouAoFimInicial: boolean
}) {
  const sp = useSearchParams()
  const [filmes, setFilmes] = useState<Filme[]>([])
  const [pagina, setPagina] = useState(paginaInicial)
  const [fim, setFim] = useState(chegouAoFimInicial)
  const [carregando, setCarregando] = useState(false)
  const sentinela = useRef<HTMLDivElement>(null)

  // Troca de filtro reinicia a acumulação — senão a grade mistura consultas.
  useEffect(() => {
    setFilmes([])
    setPagina(paginaInicial)
    setFim(chegouAoFimInicial)
  }, [sp, paginaInicial, chegouAoFimInicial])

  useEffect(() => {
    if (fim || !sentinela.current) return

    const obs = new IntersectionObserver(async ([entrada]) => {
      if (!entrada.isIntersecting || carregando) return
      setCarregando(true)
      try {
        const params = new URLSearchParams(sp.toString())
        params.set('page', String(pagina + 1))
        const r = await fetch(`/api/discover?${params}`)
        if (!r.ok) throw new Error('falhou')
        const dados = await r.json()
        setFilmes((atuais) => [...atuais, ...dados.filmes])
        setPagina(dados.pagina)
        setFim(dados.chegouAoFim)
      } catch {
        setFim(true)
      } finally {
        setCarregando(false)
      }
    })

    obs.observe(sentinela.current)
    return () => obs.disconnect()
  }, [pagina, fim, carregando, sp])

  return (
    <>
      {filmes.length > 0 && <GradeFilmes filmes={filmes} />}
      <div ref={sentinela} className="h-10" />
      {carregando && <p className="py-4 text-center text-sm text-secundario">Carregando…</p>}
      {fim && (
        <p className="py-6 text-center text-sm text-secundario">
          Fim dos resultados. O TMDB entrega no máximo 500 páginas por consulta —
          refine os filtros para ver outros títulos.
        </p>
      )}
    </>
  )
}
```

- [ ] **Step 5: Ligar na página**

No topo de `app/page.tsx`, acrescente o import:

```tsx
import { CarregadorInfinito } from '@/componentes/CarregadorInfinito'
```

E logo depois de `<GradeFilmes filmes={pagina.filmes} />`, dentro do mesmo fragmento:

```tsx
<CarregadorInfinito paginaInicial={pagina.pagina} chegouAoFimInicial={pagina.chegouAoFim} />
```

- [ ] **Step 6: Rodar e ver passar**

Run: `npm test -- testes/api/`
Expected: PASS, 4 testes.

- [ ] **Step 7: Commit**

```bash
git add app/api componentes/CarregadorInfinito.tsx app/page.tsx testes/api
git commit -m "feat: scroll infinito com aviso honesto do limite de 500 páginas"
```

---

## Task 12: Página do filme

**Files:**
- Create: `app/filme/[id]/page.tsx`, `app/filme/[id]/loading.tsx`, `componentes/OfertasProvedor.tsx`, `componentes/ListaElenco.tsx`, `componentes/Trailer.tsx`, `componentes/SkeletonDetalhe.tsx`
- Test: `testes/componentes/OfertasProvedor.test.tsx`

**Interfaces:**
- Consumes: `getMovie`, `logoUrl`, `backdropUrl`, `BotaoLista`
- Produces: rota `/filme/[id]` com `generateMetadata`

- [ ] **Step 1: Escrever o teste de ofertas**

`testes/componentes/OfertasProvedor.test.tsx`:

```tsx
import { describe, it, expect } from 'vitest'
import { render, screen } from '@testing-library/react'
import { OfertasProvedor } from '@/componentes/OfertasProvedor'
import type { Disponibilidade } from '@/lib/tmdb/types'

const prov = (nome: string) => ({ id: 1, nome, logoPath: '/l.jpg', prioridadeBR: 1 })

const disp: Disponibilidade = {
  link: 'https://www.themoviedb.org/movie/550/watch?locale=BR',
  ofertas: [
    { tipo: 'flatrate', provedores: [prov('HBO Max')] },
    { tipo: 'rent', provedores: [prov('Apple TV Store')] },
  ],
}

describe('OfertasProvedor', () => {
  it('separa assinatura de transacional com rótulos distintos', () => {
    render(<OfertasProvedor disponibilidade={disp} />)
    expect(screen.getByText(/Incluído na assinatura/i)).toBeInTheDocument()
    expect(screen.getByText(/Alugar/i)).toBeInTheDocument()
  })

  it('linka para a página do TMDB, nunca para deep link de provedor', () => {
    render(<OfertasProvedor disponibilidade={disp} />)
    const link = screen.getByRole('link', { name: /ver onde assistir/i })
    expect(link).toHaveAttribute('href', disp.link)
  })

  it('exibe a atribuição da JustWatch junto da disponibilidade', () => {
    render(<OfertasProvedor disponibilidade={disp} />)
    expect(screen.getByText(/JustWatch/)).toBeInTheDocument()
  })

  it('sem ofertas, informa em vez de sumir', () => {
    render(<OfertasProvedor disponibilidade={{ link: null, ofertas: [] }} />)
    expect(screen.getByText(/Sem disponibilidade em streaming no Brasil/i)).toBeInTheDocument()
  })
})
```

- [ ] **Step 2: Rodar e ver falhar**

Run: `npm test -- testes/componentes/OfertasProvedor.test.tsx`
Expected: FAIL — componente inexistente.

- [ ] **Step 3: Implementar as ofertas**

`componentes/OfertasProvedor.tsx`:

```tsx
import Image from 'next/image'
import { logoUrl } from '@/lib/tmdb/images'
import type { Disponibilidade, TipoOferta } from '@/lib/tmdb/types'

const ROTULOS: Record<TipoOferta, string> = {
  flatrate: 'Incluído na assinatura',
  free: 'Grátis',
  ads: 'Grátis com anúncios',
  rent: 'Alugar',
  buy: 'Comprar',
}

export function OfertasProvedor({ disponibilidade }: { disponibilidade: Disponibilidade }) {
  if (disponibilidade.ofertas.length === 0) {
    return (
      <p className="text-sm text-secundario">
        Sem disponibilidade em streaming no Brasil no momento.
      </p>
    )
  }

  return (
    <section className="space-y-4">
      {disponibilidade.ofertas.map((oferta) => (
        <div key={oferta.tipo}>
          <h3 className="mb-2 text-xs uppercase tracking-wide text-secundario">
            {ROTULOS[oferta.tipo]}
          </h3>
          <ul className="flex flex-wrap gap-2">
            {oferta.provedores.map((p) => {
              const url = logoUrl(p.logoPath, 'w92')
              return (
                <li key={`${oferta.tipo}-${p.id}`} className="flex items-center gap-2 rounded-md border border-borda bg-superficie p-1.5">
                  {url && (
                    <Image src={url} alt="" width={28} height={28} className="rounded" />
                  )}
                  <span className="pr-1 text-sm text-texto">{p.nome}</span>
                </li>
              )
            })}
          </ul>
        </div>
      ))}

      {disponibilidade.link && (
        <a
          href={disponibilidade.link}
          target="_blank"
          rel="noopener noreferrer"
          className="inline-block rounded-md bg-ambar px-3 py-1.5 text-sm font-semibold text-fundo"
        >
          Ver onde assistir
        </a>
      )}

      {/* Exigência do TMDB: atribuir a JustWatch em cada item que exibe disponibilidade. */}
      <p className="text-xs text-secundario">Dados de disponibilidade fornecidos por JustWatch.</p>
    </section>
  )
}
```

- [ ] **Step 4: Implementar elenco, trailer e skeleton**

`componentes/ListaElenco.tsx`:

```tsx
import type { Pessoa } from '@/lib/tmdb/types'

export function ListaElenco({ elenco }: { elenco: Pessoa[] }) {
  if (elenco.length === 0) return null
  return (
    <section>
      <h2 className="mb-2 text-sm font-semibold text-texto">Elenco</h2>
      <ul className="flex flex-wrap gap-x-4 gap-y-1 text-sm text-secundario">
        {elenco.map((p, i) => (
          <li key={`${p.nome}-${i}`}>
            <span className="text-texto">{p.nome}</span>
            {p.personagem && <span> como {p.personagem}</span>}
          </li>
        ))}
      </ul>
    </section>
  )
}
```

`componentes/Trailer.tsx`:

```tsx
export function Trailer({ youtubeKey, titulo }: { youtubeKey: string | null; titulo: string }) {
  if (!youtubeKey) return null
  return (
    <section>
      <h2 className="mb-2 text-sm font-semibold text-texto">Trailer</h2>
      <div className="aspect-video overflow-hidden rounded-md border border-borda">
        <iframe
          src={`https://www.youtube-nocookie.com/embed/${youtubeKey}`}
          title={`Trailer de ${titulo}`}
          allow="accelerometer; clipboard-write; encrypted-media; picture-in-picture"
          allowFullScreen
          className="h-full w-full"
        />
      </div>
    </section>
  )
}
```

`componentes/SkeletonDetalhe.tsx`:

```tsx
export function SkeletonDetalhe() {
  return (
    <div className="space-y-4" aria-busy="true" aria-label="Carregando filme">
      <div className="h-48 animate-pulse rounded-md bg-superficie sm:h-64" />
      <div className="h-6 w-1/2 animate-pulse rounded bg-superficie" />
      <div className="h-4 w-3/4 animate-pulse rounded bg-superficie" />
      <div className="h-4 w-2/3 animate-pulse rounded bg-superficie" />
    </div>
  )
}
```

- [ ] **Step 5: Implementar a página**

`app/filme/[id]/page.tsx`:

```tsx
import Image from 'next/image'
import { notFound } from 'next/navigation'
import type { Metadata } from 'next'
import { getMovie } from '@/lib/tmdb/movie'
import { backdropUrl, posterUrl } from '@/lib/tmdb/images'
import { OfertasProvedor } from '@/componentes/OfertasProvedor'
import { ListaElenco } from '@/componentes/ListaElenco'
import { Trailer } from '@/componentes/Trailer'
import { BotaoLista } from '@/componentes/BotaoLista'

export const revalidate = 21600 // 6 h — a resposta carrega disponibilidade junto

type Props = { params: Promise<{ id: string }> }

export async function generateMetadata({ params }: Props): Promise<Metadata> {
  const { id } = await params
  const filme = await getMovie(Number(id))
  if (!filme) return { title: 'Filme não encontrado — ondeassisto' }

  const imagem = backdropUrl(filme.backdropPath, 'w1280') ?? posterUrl(filme.posterPath, 'w780')

  return {
    title: `${filme.titulo}${filme.ano ? ` (${filme.ano})` : ''} — onde assistir`,
    description: filme.sinopse.slice(0, 160),
    openGraph: {
      title: filme.titulo,
      description: filme.sinopse.slice(0, 160),
      images: imagem ? [imagem] : [],
      type: 'video.movie',
    },
  }
}

export default async function PaginaFilme({ params }: Props) {
  const { id } = await params
  const filme = await getMovie(Number(id))
  if (!filme) notFound()

  const backdrop = backdropUrl(filme.backdropPath, 'w1280')
  const poster = posterUrl(filme.posterPath, 'w342')

  return (
    <article className="space-y-6">
      {backdrop && (
        <div className="relative -mx-4 h-48 overflow-hidden sm:h-72">
          <Image src={backdrop} alt="" fill priority className="object-cover opacity-40" />
        </div>
      )}

      <div className="flex flex-col gap-6 sm:flex-row">
        {poster && (
          <Image
            src={poster}
            alt={filme.titulo}
            width={180}
            height={270}
            className="w-32 flex-none rounded-md sm:w-44"
          />
        )}

        <div className="space-y-3">
          <h1 className="text-2xl font-bold text-texto">
            {filme.titulo} {filme.ano && <span className="text-secundario">({filme.ano})</span>}
          </h1>

          <p className="text-sm text-secundario">
            {[
              filme.nota > 0 && `★ ${filme.nota.toFixed(1)}`,
              filme.duracaoMin && `${filme.duracaoMin} min`,
              filme.generos.join(', ') || null,
            ].filter(Boolean).join(' · ')}
          </p>

          {filme.sinopse && <p className="max-w-prose text-sm text-texto">{filme.sinopse}</p>}

          <BotaoLista filme={filme} />
        </div>
      </div>

      <OfertasProvedor disponibilidade={filme.disponibilidade} />
      <Trailer youtubeKey={filme.trailerYoutubeKey} titulo={filme.titulo} />
      <ListaElenco elenco={filme.elenco} />
    </article>
  )
}
```

`app/filme/[id]/loading.tsx`:

```tsx
import { SkeletonDetalhe } from '@/componentes/SkeletonDetalhe'
export default function Loading() { return <SkeletonDetalhe /> }
```

- [ ] **Step 6: Rodar e verificar**

Run: `npm test -- testes/componentes/ && npm run dev`
Expected: 4 testes passam; `http://localhost:3000/filme/550` mostra Clube da Luta com ofertas.

- [ ] **Step 7: Commit**

```bash
git add app/filme componentes/OfertasProvedor.tsx componentes/ListaElenco.tsx componentes/Trailer.tsx componentes/SkeletonDetalhe.tsx testes/componentes/OfertasProvedor.test.tsx
git commit -m "feat: página do filme com ofertas, trailer, elenco e OG tags"
```

---

## Task 13: Busca, minha lista, sobre e atribuição

**Files:**
- Create: `app/busca/page.tsx`, `app/minha-lista/page.tsx`, `app/sobre/page.tsx`, `componentes/Atribuicao.tsx`
- Modify: `app/layout.tsx` (descomentar `<Atribuicao />`)

**Interfaces:**
- Consumes: `searchMovies`, `useLista`, `GradeFilmes`, `EstadoVazio`
- Produces: rotas `/busca`, `/minha-lista`, `/sobre`

- [ ] **Step 1: Implementar a página de busca**

`app/busca/page.tsx`:

```tsx
import { searchMovies, LIMITE_ANOTACAO } from '@/lib/tmdb/search'
import { parseSearchParams } from '@/lib/filtros/parse'
import { GradeFilmes } from '@/componentes/GradeFilmes'
import { EstadoVazio } from '@/componentes/EstadoVazio'

export default async function Busca({
  searchParams,
}: {
  searchParams: Promise<Record<string, string | string[] | undefined>>
}) {
  const sp = await searchParams
  const termo = typeof sp.q === 'string' ? sp.q : ''
  const { servicos } = parseSearchParams(sp)
  const r = await searchMovies(termo, servicos)

  if (r.totalResultados === 0) {
    return (
      <EstadoVazio
        titulo={`Nada encontrado para "${termo}"`}
        mensagem="Confira a grafia ou tente outro título."
      />
    )
  }

  return (
    <div className="space-y-8">
      <h1 className="text-lg text-secundario">
        {r.totalResultados} resultado(s) para <span className="text-texto">“{termo}”</span>
      </h1>

      {r.nosServicos.length > 0 && (
        <section className="space-y-3">
          <h2 className="text-sm font-semibold text-ambar">Nos seus serviços</h2>
          <GradeFilmes filmes={r.nosServicos} />
        </section>
      )}

      {r.fora.length > 0 && (
        <section className="space-y-3">
          <h2 className="text-sm font-semibold text-secundario">
            {servicos.length > 0 ? 'Fora dos seus serviços' : 'Resultados'}
          </h2>
          <GradeFilmes filmes={r.fora} />
        </section>
      )}

      {r.totalResultados > LIMITE_ANOTACAO && (
        <p className="text-xs text-secundario">
          A disponibilidade foi verificada nos {LIMITE_ANOTACAO} primeiros resultados.
          Nos demais, abra a página do filme para ver onde assistir.
        </p>
      )}

      <p className="text-xs text-secundario">Dados de disponibilidade fornecidos por JustWatch.</p>
    </div>
  )
}
```

- [ ] **Step 2: Implementar minha lista**

`app/minha-lista/page.tsx`:

```tsx
'use client'

import { useLista } from '@/lib/lista/useLista'
import { GradeFilmes } from '@/componentes/GradeFilmes'
import { EstadoVazio } from '@/componentes/EstadoVazio'
import { SkeletonGrade } from '@/componentes/SkeletonGrade'

export default function MinhaLista() {
  const { itens, montado } = useLista()

  if (!montado) return <SkeletonGrade quantidade={6} />

  if (itens.length === 0) {
    return (
      <EstadoVazio
        titulo="Sua lista está vazia"
        mensagem="Abra um filme e toque em “+ Minha lista” para salvá-lo aqui."
      />
    )
  }

  // O snapshot guardado dispensa chamar a API por item salvo.
  const filmes = itens.map((i) => ({
    id: i.id, titulo: i.titulo, ano: i.ano, posterPath: i.posterPath,
    backdropPath: null, nota: 0, votos: 0, sinopse: '',
  }))

  return (
    <div className="space-y-4">
      <h1 className="text-lg font-semibold text-texto">Minha lista ({itens.length})</h1>
      <GradeFilmes filmes={filmes} />
      <p className="text-xs text-secundario">
        Esta lista fica salva apenas neste navegador. Limpar os dados do site a apaga.
      </p>
    </div>
  )
}
```

- [ ] **Step 3: Implementar atribuição e sobre**

`componentes/Atribuicao.tsx`:

```tsx
export function Atribuicao() {
  return (
    <footer className="mt-12 border-t border-borda">
      <div className="mx-auto flex max-w-7xl flex-col gap-3 px-4 py-6 text-xs text-secundario">
        <p>
          This website uses TMDB and the TMDB APIs but is not endorsed, certified,
          or otherwise approved by TMDB.
        </p>
        <p>Dados de disponibilidade fornecidos por JustWatch.</p>
        <p>
          <a href="https://www.themoviedb.org" target="_blank" rel="noopener noreferrer" className="underline hover:text-texto">
            The Movie Database
          </a>
          {' · '}
          <a href="/sobre" className="underline hover:text-texto">Sobre</a>
        </p>
      </div>
    </footer>
  )
}
```

O logo oficial do TMDB deve ser baixado de https://www.themoviedb.org/about/logos-attribution, salvo em `public/tmdb.svg` sem qualquer modificação de cor ou proporção, e exibido neste rodapé em tamanho menor que a marca "ondeassisto" do cabeçalho.

`app/sobre/page.tsx`:

```tsx
export const metadata = { title: 'Sobre — ondeassisto' }

export default function Sobre() {
  return (
    <div className="max-w-prose space-y-4 text-sm text-texto">
      <h1 className="text-xl font-bold">Sobre o ondeassisto</h1>
      <p>
        Projeto pessoal, sem fins comerciais, que mostra quais filmes estão disponíveis
        nos serviços de streaming no Brasil.
      </p>
      <h2 className="pt-2 font-semibold">Créditos e atribuição</h2>
      <p>
        This website uses TMDB and the TMDB APIs but is not endorsed, certified,
        or otherwise approved by TMDB.
      </p>
      <p>Os dados de disponibilidade em streaming são fornecidos pela JustWatch.</p>
      <h2 className="pt-2 font-semibold">Aviso</h2>
      <p>
        A disponibilidade é informativa e pode estar desatualizada. Confirme no serviço
        de streaming antes de assinar ou alugar.
      </p>
    </div>
  )
}
```

- [ ] **Step 4: Implementar a fronteira de erro e o 404**

Cobre a linha "503 / manutenção do TMDB" da §8 do spec. Sem isso, uma indisponibilidade da API vira tela em branco.

`componentes/EstadoErro.tsx`:

```tsx
'use client'

export function EstadoErro({
  titulo = 'Não conseguimos carregar agora',
  mensagem = 'O catálogo do TMDB pode estar temporariamente indisponível. Tente novamente em instantes.',
  aoTentarDeNovo,
}: {
  titulo?: string
  mensagem?: string
  aoTentarDeNovo?: () => void
}) {
  return (
    <div className="py-16 text-center">
      <p className="text-lg font-medium text-texto">{titulo}</p>
      <p className="mx-auto mt-2 max-w-prose text-sm text-secundario">{mensagem}</p>
      {aoTentarDeNovo && (
        <button
          type="button"
          onClick={aoTentarDeNovo}
          className="mt-4 min-h-11 rounded-md bg-ambar px-4 text-sm font-semibold text-fundo"
        >
          Tentar de novo
        </button>
      )}
    </div>
  )
}
```

`app/error.tsx`:

```tsx
'use client'

import { EstadoErro } from '@/componentes/EstadoErro'

export default function Erro({ reset }: { error: Error; reset: () => void }) {
  return <EstadoErro aoTentarDeNovo={reset} />
}
```

`app/not-found.tsx`:

```tsx
import Link from 'next/link'

export default function NaoEncontrado() {
  return (
    <div className="py-16 text-center">
      <p className="text-lg font-medium text-texto">Filme não encontrado</p>
      <p className="mt-2 text-sm text-secundario">
        O título que você procurou não existe no catálogo do TMDB.
      </p>
      <Link href="/" className="mt-4 inline-block text-sm text-ambar underline">
        Voltar ao catálogo
      </Link>
    </div>
  )
}
```

- [ ] **Step 5: Descomentar `<Atribuicao />` no layout e verificar**

Run: `npm run dev`
Expected: `/busca?q=matrix` particiona resultados; `/minha-lista` mostra o vazio; `/filme/999999999` mostra o 404; rodapé com atribuição em todas as páginas.

- [ ] **Step 6: Verificar a fronteira de erro de verdade**

Sabote temporariamente o token para forçar a falha:

```bash
TMDB_READ_TOKEN=invalido npm run dev
```

Expected: a home mostra o `EstadoErro` com botão "Tentar de novo", **não** uma tela em branco nem um stack trace. Restaure o token depois.

- [ ] **Step 7: Commit**

```bash
git add app/busca app/minha-lista app/sobre app/error.tsx app/not-found.tsx componentes/Atribuicao.tsx componentes/EstadoErro.tsx app/layout.tsx
git commit -m "feat: busca particionada, minha lista, sobre, atribuição e fronteira de erro"
```

---

## Task 14: Suíte de contrato contra a API real

Protege as **invariantes** da API, não os valores — o catálogo muda todo dia.

**Files:**
- Create: `testes/contrato/tmdb.contrato.test.ts`, `vitest.contrato.config.ts`

**Interfaces:**
- Consumes: `TMDB_READ_TOKEN` real
- Produces: `npm run test:contract`

- [ ] **Step 1: Configurar a suíte separada**

`vitest.contrato.config.ts`:

```ts
import { defineConfig } from 'vitest/config'
import path from 'node:path'

export default defineConfig({
  test: {
    include: ['testes/contrato/**/*.test.ts'],
    environment: 'node',
    globals: true,
    testTimeout: 30_000,
    setupFiles: ['./testes/contrato/env.ts'],
  },
  resolve: { alias: { '@': path.resolve(__dirname, '.') } },
})
```

`testes/contrato/env.ts`:

```ts
import fs from 'node:fs'

if (!process.env.TMDB_READ_TOKEN && fs.existsSync('.env.local')) {
  const m = fs.readFileSync('.env.local', 'utf8').match(/^TMDB_READ_TOKEN=(.+)$/m)
  if (m) process.env.TMDB_READ_TOKEN = m[1].trim()
}
```

- [ ] **Step 2: Escrever a suíte**

`testes/contrato/tmdb.contrato.test.ts`:

```ts
import { describe, it, expect } from 'vitest'
import { fetchTmdb } from '@/lib/tmdb/client'
import { getBrProviders } from '@/lib/tmdb/providers'
import { getMovie } from '@/lib/tmdb/movie'

type Lista = { results: unknown[]; total_results: number; total_pages: number }

const BASE_BR = { watch_region: 'BR', with_watch_monetization_types: 'flatrate' } as const

describe('contrato TMDB — invariantes', () => {
  it('o token configurado autentica', async () => {
    await expect(fetchTmdb('/authentication')).resolves.toBeTruthy()
  })

  it('INVARIANTE: sem watch_region o filtro por provedor é ignorado', async () => {
    const com = await fetchTmdb<Lista>('/discover/movie', { ...BASE_BR, with_watch_providers: '8' })
    const sem = await fetchTmdb<Lista>('/discover/movie', {
      with_watch_monetization_types: 'flatrate', with_watch_providers: '8',
    })
    // Sem a região, a API devolve o catálogo global — ordens de grandeza a mais.
    expect(sem.total_results).toBeGreaterThan(com.total_results * 10)
  })

  it('INVARIANTE: pipe é união, vírgula é interseção', async () => {
    const [n, p, uniao, inter] = await Promise.all([
      fetchTmdb<Lista>('/discover/movie', { ...BASE_BR, with_watch_providers: '8' }),
      fetchTmdb<Lista>('/discover/movie', { ...BASE_BR, with_watch_providers: '119' }),
      fetchTmdb<Lista>('/discover/movie', { ...BASE_BR, with_watch_providers: '8|119' }),
      fetchTmdb<Lista>('/discover/movie', { ...BASE_BR, with_watch_providers: '8,119' }),
    ])
    expect(uniao.total_results).toBeGreaterThan(Math.max(n.total_results, p.total_results))
    expect(inter.total_results).toBeLessThan(Math.min(n.total_results, p.total_results))
  })

  it('INVARIANTE: 20 resultados por página', async () => {
    const r = await fetchTmdb<Lista>('/discover/movie', { ...BASE_BR, with_watch_providers: '8' })
    expect(r.results).toHaveLength(20)
  })

  it('INVARIANTE: page=501 é rejeitado com HTTP 400', async () => {
    await expect(
      fetchTmdb('/discover/movie', { ...BASE_BR, page: 501 })
    ).rejects.toMatchObject({ status: 400, codigoTmdb: 22 })
  })

  it('INVARIANTE: o append traz credits, videos e watch/providers juntos', async () => {
    const filme = await getMovie(550)
    expect(filme).not.toBeNull()
    expect(filme!.elenco.length).toBeGreaterThan(0)
    expect(filme!.disponibilidade).toBeDefined()
  })

  it('INVARIANTE: /search/movie ignora filtro por provedor', async () => {
    const puro = await fetchTmdb<Lista>('/search/movie', { query: 'matrix' })
    const comFiltro = await fetchTmdb<Lista>('/search/movie', {
      query: 'matrix', watch_region: 'BR', with_watch_providers: '8',
    })
    expect(comFiltro.total_results).toBe(puro.total_results)
  })

  it('INVARIANTE: os provedores marcados na UI existem na lista viva do BR', async () => {
    const provedores = await getBrProviders()
    const nomes = provedores.map((p) => p.nome)
    // Se algum destes sumir, o chip correspondente vira prateleira vazia.
    for (const esperado of ['Netflix', 'Amazon Prime Video', 'Disney Plus', 'HBO Max']) {
      expect(nomes).toContain(esperado)
    }
    expect(provedores[0].prioridadeBR).toBeLessThanOrEqual(provedores[1].prioridadeBR)
  })
})
```

- [ ] **Step 3: Rodar contra a API real**

Run: `npm run test:contract`
Expected: PASS, 8 testes. Se algum falhar, a API mudou — investigue antes de alterar o código.

- [ ] **Step 4: Commit**

```bash
git add testes/contrato vitest.contrato.config.ts package.json
git commit -m "test: suíte de contrato protegendo as invariantes da API do TMDB"
```

---

## Task 15: E2E e implantação

**Files:**
- Create: `e2e/fluxo.spec.ts`, `playwright.config.ts`
- Modify: `README.md`

**Interfaces:**
- Consumes: aplicação completa
- Produces: `npm run test:e2e`, site em produção

- [ ] **Step 1: Configurar o Playwright**

`playwright.config.ts`:

```ts
import { defineConfig, devices } from '@playwright/test'

export default defineConfig({
  testDir: './e2e',
  use: { baseURL: 'http://localhost:3000' },
  projects: [{ name: 'chromium', use: { ...devices['Desktop Chrome'] } }],
  webServer: {
    command: 'npm run build && npm start',
    url: 'http://localhost:3000',
    reuseExistingServer: !process.env.CI,
    timeout: 180_000,
  },
})
```

- [ ] **Step 2: Escrever o teste de fluxo**

`e2e/fluxo.spec.ts`:

```ts
import { test, expect } from '@playwright/test'

test('filtrar, abrir detalhe, salvar na lista e persistir após recarregar', async ({ page }) => {
  await page.goto('/')
  await expect(page.getByRole('link', { name: 'ondeassisto' })).toBeVisible()

  // Marcar um serviço deve mudar a URL — o filtro mora na URL, não no estado do React.
  await page.getByRole('button', { name: 'Netflix' }).click()
  await expect(page).toHaveURL(/servicos=/)

  // Abrir o primeiro filme da grade
  await page.locator('a[href^="/filme/"]').first().click()
  await expect(page).toHaveURL(/\/filme\/\d+/)
  await expect(page.getByRole('heading', { level: 1 })).toBeVisible()

  // Salvar na lista
  const botao = page.getByRole('button', { name: /Minha lista/ })
  await botao.click()
  await expect(botao).toHaveAttribute('aria-pressed', 'true')

  // Persistir após recarregar
  await page.reload()
  await expect(page.getByRole('button', { name: /Na minha lista/ })).toBeVisible()

  // Aparecer em /minha-lista
  await page.goto('/minha-lista')
  await expect(page.locator('a[href^="/filme/"]')).toHaveCount(1)
})

test('busca separa o que está nos serviços marcados', async ({ page }) => {
  await page.goto('/?servicos=8')
  await page.getByRole('searchbox').fill('matrix')
  await page.getByRole('searchbox').press('Enter')
  await expect(page).toHaveURL(/\/busca\?q=matrix/)
  await expect(page.getByText(/resultado\(s\) para/)).toBeVisible()
})

test('filme inexistente devolve 404', async ({ page }) => {
  const resposta = await page.goto('/filme/999999999')
  expect(resposta?.status()).toBe(404)
})
```

- [ ] **Step 3: Rodar o E2E**

Run: `npx playwright install chromium && npm run test:e2e`
Expected: PASS, 3 testes.

- [ ] **Step 4: Rodar a verificação completa**

Run: `npm test && npx tsc --noEmit && npm run build`
Expected: todos os testes passam, sem erro de tipo, build conclui.

- [ ] **Step 5: Escrever o README**

`README.md`:

```markdown
# ondeassisto.com.br

Catálogo dos filmes disponíveis nos serviços de streaming no Brasil, usando a API do TMDB.

## Desenvolvimento

    cp .env.example .env.local     # e preencha TMDB_READ_TOKEN
    npm install
    npm run dev

## Testes

    npm test              # unitários e de integração
    npm run test:contract # contra a API real — exige TMDB_READ_TOKEN
    npm run test:e2e      # Playwright

## Documentação

- Design: `docs/superpowers/specs/2026-08-16-catalogo-streaming-design.md`
- API do TMDB: `docs/tmdb-api-ficha-tecnica.md`

## Atribuição

This website uses TMDB and the TMDB APIs but is not endorsed, certified, or otherwise
approved by TMDB. Dados de disponibilidade fornecidos por JustWatch.
```

- [ ] **Step 6: Commit e publicar**

```bash
git add -A
git commit -m "test: E2E do fluxo principal e README"
git push origin main
```

- [ ] **Step 7: Implantar na Vercel**

Passos manuais no navegador — nenhum token da Vercel é necessário:

1. vercel.com → **Add New Project** → importar `mjg2020/ondeassisto`
2. **Environment Variables** → `TMDB_READ_TOKEN` com o valor do `.env.local`, marcando Production e Preview
3. **Deploy**
4. **Settings → Domains** → adicionar `ondeassisto.com.br` e seguir as instruções de DNS

- [ ] **Step 8: Verificar em produção**

Confirme, no site publicado:
- A grade carrega e os chips filtram
- `/filme/550` abre e mostra ofertas
- Compartilhar um link de filme no WhatsApp gera preview com pôster
- O rodapé exibe o aviso do TMDB e a atribuição da JustWatch

---

## Verificação final

- [ ] `npm test` — todos os unitários passam
- [ ] `npm run test:contract` — invariantes da API confirmadas
- [ ] `npm run test:e2e` — fluxo principal íntegro
- [ ] `npx tsc --noEmit` — sem erro de tipo
- [ ] `npm run build` — build de produção conclui
- [ ] `git status` — nenhum `.env.local` rastreado
- [ ] Logo do TMDB presente no rodapé, sem modificação
- [ ] Atribuição da JustWatch em toda tela que exibe disponibilidade
- [ ] Nenhum recurso de IA/LLM sobre conteúdo do TMDB
