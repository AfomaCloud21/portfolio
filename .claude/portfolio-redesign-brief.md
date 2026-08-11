Portfolio Redesign Brief — "Deploy Log"
Context
This is a Hugo static site using the hugo-profile theme, currently deployed via GitHub Actions to GitHub Pages. All existing content (About, Experience, Project case studies in content/) stays exactly as is — this is a visual restyle, not a rebuild or content change. Preserve all existing Markdown content and site structure.
Concept
The subject is a DevOps engineer's real, working CI/CD pipeline — this site itself deploys via that same pipeline. Instead of generic portfolio decoration, the visual language should evoke a terminal / deployment log / monitoring dashboard, grounded in how the owner actually documents her own work (her case studies already read like structured incident reports and deploy logs).
Design Tokens
Color

Background: #161B22 (deep slate, near-black but warmer than pure black)
Primary text: #E6EDF3 (warm off-white)
Secondary/muted text: #8B949E
Accent: #D4A73B (restrained amber — used sparingly: links, status indicators, one highlight per section, NOT as a background wash)
Success/status green (for "deployed"/"live" indicators only): #3FB950
Border/divider: #30363D

Typography

Display/headings: Space Grotesk (geometric sans, distinct personality, used with restraint — not oversized)
Body: Inter or IBM Plex Sans (clean, readable)
Utility/metadata/tags/dates: JetBrains Mono or IBM Plex Mono — this is the deliberate signature choice, since monospace is authentically the owner's working font, not decoration. Use it for: project tags, dates, the "status line" element below, and any code snippets.

Layout

Hero: name/title treated like a terminal status line — e.g. a big, type-out Afoma Egbuonu (no `$` prompt prefix on the name itself) with a much smaller, muted DevOps Engineer & Cloud Infrastructure Specialist subtitle underneath, prompt-style and understated, not gimmicky. Avoid a big centered photo-and-tagline template layout.
Project cards: each includes a small monospace "status line" summarizing the deploy, e.g.: $ deployed · Docker + Terraform + Ansible · 3 services · 89% image size reduction This is the signature element — it should look like real terminal output (subtle blinking cursor optional, tasteful, not cheesy), and the specific stats should be pulled from that project's actual case study content (image size %, VM count, uptime stats, etc.) — never generic filler stats.
Keep spacing disciplined and generous — this is a quiet, precise layout, not a maximalist one. The amber accent and the status-line device are the only "loud" elements; everything else should be calm.
Motion
One deliberate touch: a brief page-load sequence where the hero's status line "types out" once on first load (respecting prefers-reduced-motion — fall back to static text if set)
Subtle hover state on project cards (border color shifts to accent, no scale/shadow gimmicks)
No scroll-triggered animation spam — restraint over decoration
Quality floor
Fully responsive down to mobile
Visible keyboard focus states on all interactive elements
Respect prefers-reduced-motion
Don't break any existing Hugo shortcodes or content structure — this is CSS/template-layer work
What NOT to do
No cream background + terracotta accent (generic AI-portfolio default)
No pure-black + neon-green "hacker" cliché — the amber is warmer and more restrained than that
No numbered-marker sections (01/02/03) unless content is genuinely sequential
No stock icon sets or generic gradient accents

