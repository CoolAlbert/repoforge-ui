# RepoForge UI

Svelte dashboard for the RepoForge PM workflow.

The UI starts new Pi-agent runs, lists run status, and subscribes to live Server-Sent Event logs from `repoforge-control`.

## Development

```bash
npm install
npm run dev
```

Set `VITE_REPOFORGE_API` if the control API is not running at `http://127.0.0.1:4077`.
