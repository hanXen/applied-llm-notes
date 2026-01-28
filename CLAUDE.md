# Document Workflow

When adding a new document to this repo:

## 1. Review
- Check title clarity
- Remove exaggerated tone ("always", "must", "critical" → "recommended", "consider")
- Distinguish original content vs. author's thoughts

## 2. Translate
- Korean doc → Create English version
- English doc → Create Korean version
- Keep structure identical for easy switching

## 3. Create Folder
- Use descriptive name: `topic-name-analysis`, `concept-name-overview`
- Avoid generic names: ~~`context-engineering`~~ (too broad)

## 4. Structure
```
topic-name/
├── README.md      # Index (Language links + TL;DR + ToC)
├── ko/
│   └── README.md  # Full content in Korean
└── en/
    └── README.md  # Full content in English
```

## 5. Fill README
Use template: [templates/topic-readme.md](templates/topic-readme.md)
