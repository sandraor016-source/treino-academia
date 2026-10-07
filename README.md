# Treino Full Body

React + TypeScript + Vite. Dados em `src/data/*.json` (extraídos do app original), persistência atrás da interface `Repository`.

    npm install
    npm run dev       # desenvolvimento
    npm run build     # produção (dist/)

## Arquitetura
- `src/repo/` – **único** lugar que sabe onde os dados vivem. `localRepo` (localStorage) é o padrão; com `VITE_SUPABASE_URL`/`VITE_SUPABASE_ANON_KEY` no `.env`, o `index.ts` passa a gravar local + sincronizar com o Supabase.
- `src/useAppState.ts` – carrega/salva o estado (com debounce); `src/lib/sets.ts` – regras de séries/repetições (puras, testáveis).
- `src/components/` – UI. `supabase/schema.sql` – tabelas + RLS.

## Próximos passos para o backend
1. Rodar `supabase/schema.sql`, criar `.env`. 2. Adicionar tela de login (`sb.auth`) — sem usuário o sync fica inativo. 3. Fase 2: tabelas normalizadas + histórico de treinos (sessões por data) e gráficos de evolução de carga.

## Testes e offline
`npm test` (vitest) cobre as regras de séries e a migração do app antigo. Em produção o app registra um service worker (`public/sw.js`) e funciona offline após a primeira visita; pode ser instalado na tela inicial do celular.

## Páginas
`/` abre a landing page (`src/pages/Landing.tsx`); o botão **Entrar** leva a `/app` (`src/App.tsx`). O app instalado no celular abre direto em `/app`. Em hospedagem estática, configure fallback para `index.html` (Netlify/Vercel/Cloudflare Pages: regra SPA).

## Imagens dos exercícios
Imagens em `public/exercises/` (RepDB free, WebP 512×512); o mapeamento nome → arquivos está em `src/data/exerciseImages.json`. Prioridade no card: foto do usuário > imagem RepDB (alterna início/pico) > desenho SVG. Atribuição obrigatória: Exercise data by [RepDB](https://repdb.co) (já no rodapé da landing).

## Catálogo ampliado
O catálogo tem 260 exercícios (133 originais + 127 de academia do RepDB Free: halteres, barra, barra W, polia, Smith, barra fixa e máquinas). Cada um tem 2 imagens (início/pico) em `public/exercises/`. Para importar mais, siga o mesmo padrão: arquivos `<id>-start.webp` e `<id>-peak.webp`, uma entrada em `src/data/catalog.json` e outra em `src/data/exerciseImages.json` (nome em português igual nas duas).

## Editar treinos (prontos ou criados)
- **Na tela de treino:** `✕` no card remove o exercício (o progresso dos demais é mantido); `＋ Adicionar exercício` no fim da lista inclui qualquer exercício do catálogo; `✎ Editar treino` abre o editor completo (nome, cor, dias, ordem).
- **Treinos prontos** (Full Body, PPL, Superior/Inferior, ABC) também são editáveis. A versão do usuário fica em `state.edits[id]` (só no aparelho/Supabase) e o botão **Restaurar**, em *Meus treinos → Treinos prontos*, volta ao modelo original. Os arquivos `src/data/workouts.json` nunca são alterados pelo app.
- Lógica de reindexação do progresso: `src/lib/edit.ts` (com testes em `edit.test.ts`).

## Reordenar exercícios (arrastar)
- **Celular/tablet (toque):** segure o dedo ~0,2 s em qualquer parte do card (ou no ícone ⠿) e arraste; o aparelho vibra ao começar. Deslizar rápido continua **rolando** a tela, e botões, campos e listas do card funcionam normalmente (segurar neles não arrasta).
- **PC:** clique e arraste pelo ⠿. **Teclado:** foque o ⠿, espaço + setas.
- Usa `@dnd-kit` (`src/components/SortableList.tsx`). A nova ordem fica salva no treino (pronto ou criado); progresso, cargas e trocas acompanham o card (`afterMove` em `src/lib/edit.ts`, com testes).
- PWA: `public/sw.js` agora busca a página **da rede primeiro** (a versão nova chega na próxima abertura) e usa cache só offline; a versão do cache subiu para `ft-v4`.

## Segurança e dados (revisão)
- **Tudo que entra é validado** em `src/lib/sanitize.ts` (`sanitizeState`): localStorage, Supabase e importação do app antigo. Faixas numéricas, tamanhos, cores (`#rrggbb`), ids e chaves; descarta `__proto__`, `NaN/Infinity` e campos desconhecidos. Dado ilegível do aparelho é copiado para `ft3-backup` antes de ser sobrescrito. Testes em `sanitize.test.ts`.
- **Sincronização segura** (`src/repo/`): o app só grava no Supabase depois de ter conseguido **ler** de lá na sessão (rede ruim não sobrescreve dados novos com cópia velha); erro de consulta é erro, não "vazio"; ao abrir vale a cópia mais nova entre local e servidor (`merge.ts`, testado).
- **Salva ao minimizar/fechar** (`visibilitychange`/`pagehide`) além do debounce de 400 ms; cronômetro de descanso usa o relógio (não para com a tela bloqueada).
- **Banco**: `supabase/schema.sql` (idempotente) — RLS só para `authenticated`, `(select auth.uid())`, teto de ~1 MB por usuário, `anon` sem acesso. A chave `anon` no front é pública por desenho; quem protege é a RLS. **Nunca** use a `service_role` no front.
- **Cabeçalhos**: `public/_headers` (CSP, nosniff, Referrer-Policy…) para Netlify/Cloudflare Pages; na Vercel copie para `vercel.json`. CSP testado sem violações.
- `ErrorBoundary` evita tela branca; o service worker só guarda respostas 200 da própria origem (Supabase nunca é cacheado).

## Testar no celular
1. Rodar `supabase/schema.sql` no SQL Editor; criar `.env` (copie `.env.example`) com URL e anon key.
2. **Login ainda não existe na interface**: sem usuário logado o app funciona só local (nada é enviado). Falta uma tela de login (`sb().auth`) para o sync ativar.
3. Mesma rede Wi-Fi: `npm run dev -- --host` e abra `http://IP-do-PC:5173/app` no celular. PWA/service worker exigem **HTTPS** (use um deploy ou túnel); em HTTP o app funciona, só não instala/offline.
4. Fotos de exercícios ficam só no aparelho (localStorage), não sincronizam.
