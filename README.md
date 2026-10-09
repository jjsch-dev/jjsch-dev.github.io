# Firmware + AI — Blog Source

This repository powers the GitHub Pages site for the ES242F Smart Lock case study series.

## How to Publish

1. Fork or create this repository as `jjsch-dev.github.io`
2. Go to Settings → Pages → Source: Deploy from a branch → `main` / `root`
3. The site will be live at `https://jjsch-dev.github.io`

## Structure

```
.
├── _config.yml              # Jekyll configuration
├── index.md                 # Landing page
├── articles/                # Article markdown files
│   └── arco-0-old-workshop.md
├── assets/                  # Images and diagrams
│   ├── bench_oscilloscope_probe.png
│   ├── bench_overview_logic_analyzer.png
│   ├── pcba_tuya_annotated.png
│   └── keypad_capacitive_mapped.png
└── _diagrams/               # Mermaid source files
    └── method_diagram.mmd
```

## Adding a New Article

1. Create `articles/arco-N-title.md`
2. Add front matter:
   ```yaml
   ---
   layout: post
   title: "Article Title"
   date: YYYY-MM-DD
   ---
   ```
3. Link it from `index.md`
4. Commit and push

## License

Article text: CC BY-SA 4.0  
Code snippets: MIT (where applicable)  
Images: All rights reserved unless otherwise noted.
