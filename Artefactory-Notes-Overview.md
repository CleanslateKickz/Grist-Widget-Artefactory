# Artefactory: a mini-IDE for creating Grist widgets from a Grist document

[Hello everyone](https://forum.grist.libre.sh/t/artefactory-un-mini-ide-pour-creer-ses-widgets-grist-depuis-un-doc-grist/3712),

I'm sharing a widget I've just stabilized: **Artefactory** , a mini-IDE that runs **as a custom Grist widget** and allows you to create, edit and preview your widgets directly from a Grist document — code stored in a table in the document, no backend, no repository to maintain.

![:link:](https://forum.grist.libre.sh/images/emoji/twitter/link.png?v=12 ":link:") **Live demo (with built-in guide)** : [🏭Artefactory v1 - Grist](https://grist.numerique.gouv.fr/o/docs/7YYPiF5XADvc/Artefactory-v1)   
![:link](https://forum.grist.libre.sh/images/emoji/twitter/link.png?v=12 ":link:") **Widget** : [Artefactory AI v11](https://nic01asfr.github.io/Widgets-Grist/artefactory/)

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

① What it does

Artefactory transforms a Grist table into a mini-IDE:

● **Ace editor** (coloring, autocomplete) with live preview (~800 ms hot-reload)  
● **6 artifact types** : React/JSX, HTML, Grist widget, SVG, Mermaid, Markdown  
● **Grist bridge** automatically injected into artifacts → a widget written in Artefactory can read/write to the documentation like a native widget ( `grist.docApi.fetchTable`, `applyUserActions`, `onRecord`…)  
● **100% Grist storage** : each artifact is a row in a table `Artefacts`automatically created on first launch  
● **Contextual documentation** : tagged Markdown artifacts `IsDoc=true`appear as selectable chips to enrich the AI ​​assistant's context

The demo doc contains the Artefactory widget itself + a complete guide to artifacts (one per type) — duplicate it to test at home.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

② Architecture - bridge Grist + iframe sandbox

Each artifact is rendered in a `srcdoc`sandboxed iframe. The widget automatically replaces it `<script src="grist-plugin-api.js">`with a fake one `window.grist`that proxies via `postMessage`the host widget, which then calls the real Grist.

In concrete terms, the code written in Artefactory uses the same APIs as a classically deployed Grist widget — but without having to deploy it.

For React artifacts, the environment includes React 18, Tailwind CDN and about ten inlined UI components (Card, Button, Input, Badge…) ready to use without imports.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

③ Connect an AI assistant (optional)

The widget includes an AI panel, but **Artefactory doesn't include any models** . It sends a JSON POST request to a webhook that you configure. **You decide which agent responds** .

● Claude / GPT via a Cloudflare Workers or Vercel proxy (~30 Node lines)  
● **Albert API** via n8n ● A **local Ollama**  
agent ● A homemade "anything goes" return  
`{ code, message }`

The HTTP contract is documented on the repo side, and a **copyable system prompt** is provided so that the agent knows the constraints of the widget (available Grist APIs, React environment, anti-patterns) — this prevents it from generating crashing code by inventing APIs.

The AI ​​panel is fully hideable. You can use Artefactory **without AI** as a simple editor.

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

4 Returns

● Use cases where you would use it?  
● Webhook patterns you have already implemented on the Albert/n8n/Ollama side?  
● Feedback on other attempts at in-Grist editors?

Issues and MRs are welcome on the repository.

![:link:](https://forum.grist.libre.sh/images/emoji/twitter/link.png?v=12 ":link:") **Live demo** : [🏭Artefactory v1 - Grist](https://grist.numerique.gouv.fr/o/docs/7YYPiF5XADvc/Artefactory-v1)   
![:link:](https://forum.grist.libre.sh/images/emoji/twitter/link.png?v=12 ":link:") **Code and docs** : [GitHub - nic01asFr/Widgets-Grist · GitHub](https://github.com/nic01asFr/Widgets-Grist)


---

# Newsletter:

One very ambitious example (from frequent contributor nic01asFr) is called [Artefactory](https://forum.grist.libre.sh/t/artefactory-un-mini-ide-pour-creer-ses-widgets-grist-depuis-un-doc-grist/3712). It’s a custom widget (in French only at the moment) that lets you build and preview custom widgets and other artifacts (React/JSX, HTML, SVG, Mermaid, Markdown) with Grist as the backend – each artifact is stored as a simple record. Kind of like a mini-IDE, powered by whatever AI model you supply (using webhooks).


![Artefactory](https://support.getgrist.com/images/newsletters/2026-05/artefactory.gif)
