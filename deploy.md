# deploy.md — деплой kisa-card

## Платформа
GitHub Pages через GitHub Actions.

## Конфиг
- `vite.config.js`: `base: '/kisa-card/'`
- Workflow: `.github/workflows/deploy.yml` — триггер на push в `main`, деплоит `dist/`

## Первый деплой (инструкция)

```bash
cd kisa-card
git init
git add .
git commit -m "initial"
git remote add origin https://github.com/ТВОЙ_ЮЗЕР/kisa-card.git
git push -u origin main
```

Затем в репозитории:
**Settings → Pages → Source: GitHub Actions**

После этого каждый `git push` в `main` автоматически деплоит на:
`https://ТВОЙ_ЮЗЕР.github.io/kisa-card/`

## Статус
- [ ] Репозиторий создан
- [ ] Первый push
- [ ] Pages включён
