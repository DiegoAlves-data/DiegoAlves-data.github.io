# DiegoAlves-data.github.io

Site pessoal de Diego Rodrigues — https://diegoalves-data.github.io/

Jekyll + [Minimal Mistakes](https://mmistakes.github.io/minimal-mistakes/) (via `remote_theme`), publicado no GitHub Pages.

## Estrutura

- `index.md` — página inicial (bio).
- `_pages/blog.md` — listagem do desafio "14 Dias para FDE / A.I. Deployment" (`/blog/`).
- `_pages/estudos-data-science.md` — arquivo dos primeiros estudos de ciência de dados (`/estudos-data-science/`).
- `_posts/` — artigos publicados, um por dia do desafio.
- `_drafts/` — rascunhos com roteiro de cada dia do desafio (14 arquivos). Preencher e mover para `_posts/` com o prefixo de data (`AAAA-MM-DD-slug.md`) ao publicar.

## Publicar um dia do desafio

1. Abrir o rascunho correspondente em `_drafts/`.
2. Escrever o artigo seguindo o roteiro (que fica em comentário HTML, não aparece publicado).
3. Mover o arquivo para `_posts/`, renomeando com a data do dia (ex.: `2026-11-03-dia-05-loop-e-criterio-de-parada.md`).
4. Commit + push. O GitHub Pages rebuilda sozinho.

Para pré-visualizar localmente: `bundle exec jekyll serve --drafts`.
