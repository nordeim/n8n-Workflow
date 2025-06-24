<!-- Banner / Hero -->
<h1 align="center">
  🚀 n8n Power-Playbook
</h1>

<p align="center">
  <strong>Production-ready recipes, JSON exports & deep-dive docs for automating absolutely everything with <a href="https://n8n.io/">n8n</a>.</strong>
  <br />
  <sub>Because clicking boxes is nice, but <em>understanding</em> the JSON is 🔥.</sub>
</p>

<p align="center">
  <a href="https://github.com/your-org/n8n-power-playbook/actions"><img src="https://github.com/your-org/n8n-power-playbook/workflows/lint/badge.svg" alt="CI" /></a>
  <a href="LICENSE"><img src="https://img.shields.io/github/license/your-org/n8n-power-playbook" alt="license" /></a>
  <a href="https://twitter.com/n8n_io"><img src="https://img.shields.io/badge/PRs-welcome-brightgreen.svg" alt="PRs Welcome" /></a>
</p>

---

## 🗂️ What’s Inside?

| Guide | TL;DR | JSON Demo |
|-------|-------|-----------|
| `guides/01-complex-workflow.md` | 6 000-word deep dive that dissects a multi-service automation: GitHub → OpenAI → Trello → Slack with full error-workflow. | `workflows/github-bug-trello-slack.json` |
| `guides/02-onboarding-workflow.md` | End-to-end client-onboarding flow fuelled by GPT-4o: Form ➜ PDF parse ➜ Drive, ClickUp, Slack, Gmail. | `workflows/onboarding-workflow.json` |

> Both guides are **copy-paste ready** and fully annotated—perfect for learning or forking straight into production.

---

## ✨ Quick Glance

```bash
git clone https://github.com/your-org/n8n-power-playbook.git
cd n8n-power-playbook

# Import a workflow into your local/self-hosted n8n
# (UI → Workflows → Import from file)
```

<p align="center">
  <img src="assets/screenshot-canvas.png" width="650" alt="Workflow canvas preview" />
</p>

---

## 🚀 Quick-Start (5 mins)

1. **Spin up n8n**  
   ```bash
   docker run -it --rm \
     -p 5678:5678 \
     -e N8N_BASIC_AUTH_USER=admin \
     -e N8N_BASIC_AUTH_PASSWORD=supersecret \
     n8nio/n8n
   ```
2. **Import JSON**  
   *Open n8n → Workflows → “Import from file” → select any file from `/workflows`.*

3. **Create required credentials**  
   The guides call out every scope/token you’ll need (GitHub PAT, Slack bot, Google OAuth, …).

4. **Run in Manual mode**  
   Pin data, inspect outputs, then flip the `active` switch when you’re happy.

---

## 🧩 Repository Structure

```
.
├─ guides/
│  ├─ 01-complex-workflow.md
│  └─ 02-onboarding-workflow.md
├─ workflows/
│  ├─ github-bug-trello-slack.json
│  └─ onboarding-workflow.json
├─ assets/
│  └─ screenshot-canvas.png
├─ .github/
│  ├─ ISSUE_TEMPLATE/
│  └─ workflows/
├─ LICENSE
└─ README.md  ← you are here
```

---

## 🛡️ Requirements

| Service | Minimum Scope / Permission |
|---------|----------------------------|
| **Slack** | `channels:write`, `chat:write`, `users:read.email` |
| **Google Drive & Gmail** | OAuth2 (`drive.file`, `gmail.send`) |
| **ClickUp** | Personal API token or OAuth app with `task:create` |
| **OpenAI** | API key (gpt-3.5-turbo or gpt-4o mini) |

> Don’t worry—each guide has a dedicated “Credentials” section with screenshots and links.

---

## 🏗️ Extending / Customising

* Fork → adjust `.env.example` for your secrets  
* Add or swap nodes (e.g., replace Trello with Linear)  
* Create PRs for **new guides, bug fixes, or improved prompts**—the community will ❤️ you.

---

## 🤝 Contributing

1. Fork the repo & create your branch: `git checkout -b feat/your-feature`
2. Commit your changes: `git commit -m 'feat: amazing thing'`
3. Push to the branch: `git push origin feat/your-feature`
4. Open a Pull Request—follow the PR template.  
   *Lint & Markdown-link-check run via GitHub Actions.*

---

## 📜 License

MIT © Your Name or Org.  
Feel free to copy, modify, merge, publish, distribute, sublicense, and/or sell copies of these docs and example workflows. A star ⭐ is always appreciated.

---

## 🙌 Acknowledgements

* The amazing **n8n** community for constant inspiration  
* Josh Burns Tech, Coding in Public & the LangChain team for stellar content  
* Everyone submitting PRs—automation is a team sport!

---

<p align="center">
  <strong>Happy automating &lt;3</strong><br />
  <!-- feel free to swap the gif -->
  <img src="https://media.giphy.com/media/v1.Y2lkPTc5MGI3NjExOWI4NmU1YjAyOGI3OGRiMzlmMjc3ZDZmODRiNjY1YzM4ZGZiNmNkZiZjdD1n/tXL4FHPSnVJ0A/giphy.gif" width="270" alt="automation gif" />
</p>
