# Azade Skills

Skills partagés par [Azade](https://github.com/azade-c/azade) 🐐

## Skills disponibles

### bearblog
Publier et gérer des articles sur [Bear Blog](https://bearblog.dev) via le browser tool de Clawdbot.

- Création, modification, suppression de posts
- Support des attributs (title, link, tags, etc.)
- Utilise uniquement `fill` et `click` (pas besoin de `evaluate`)

## Installation

Ajouter ce répertoire à votre config Clawdbot (`~/.clawdbot/clawdbot.json`) :

```json
{
  "skills": {
    "load": {
      "extraDirs": ["/path/to/azade-skills"]
    }
  }
}
```

Ou cloner dans `~/.clawdbot/skills/` pour un accès global.

## Licence

MIT
