<script lang="ts">
  type RunStatus = "queued" | "running" | "completed" | "failed" | "cancelled";

  interface RunRecord {
    id: string;
    projectId: string;
    repoUrl: string;
    branch?: string;
    prompt: string;
    model?: string;
    status: RunStatus;
    createdAt: string;
    updatedAt: string;
    exitCode?: number;
    error?: string;
  }

  interface LogEvent {
    ts: string;
    stream: string;
    level: "debug" | "info" | "warn" | "error";
    msg: string;
    step?: string;
  }

  const apiBase = import.meta.env.VITE_REPOFORGE_API ?? "http://127.0.0.1:4077";

  let runs: RunRecord[] = [];
  let logs: LogEvent[] = [];
  let selectedRunId = "";
  let repoUrl = "https://github.com/mishankov/web-tail";
  let projectId = "repoforge";
  let model = "gpt-5.5";
  let prompt = "Inspect the repository, run the existing checks, and propose the smallest useful improvement. Keep changes focused.";
  let loading = false;
  let error = "";
  let source: EventSource | null = null;

  $: selectedRun = runs.find((run) => run.id === selectedRunId);

  async function loadRuns() {
    const response = await fetch(`${apiBase}/runs`);
    runs = await response.json();
    if (!selectedRunId && runs[0]) selectRun(runs[0].id);
  }

  async function createRun() {
    loading = true;
    error = "";
    try {
      const response = await fetch(`${apiBase}/runs`, {
        method: "POST",
        headers: { "content-type": "application/json" },
        body: JSON.stringify({ projectId, repoUrl, prompt, model })
      });
      if (!response.ok) throw new Error(await response.text());
      const run = await response.json();
      runs = [run, ...runs];
      selectRun(run.id);
    } catch (caught) {
      error = caught instanceof Error ? caught.message : String(caught);
    } finally {
      loading = false;
    }
  }

  async function selectRun(runId: string) {
    selectedRunId = runId;
    source?.close();
    const response = await fetch(`${apiBase}/runs/${runId}/logs`);
    logs = await response.json();
    source = new EventSource(`${apiBase}/runs/${runId}/events`);
    source.addEventListener("log", (event) => {
      logs = [...logs, JSON.parse((event as MessageEvent).data)];
    });
  }

  function statusClass(status: RunStatus) {
    return `status ${status}`;
  }

  loadRuns();
</script>

<main class="shell">
  <aside class="sidebar">
    <div class="brand">
      <strong>RepoForge</strong>
      <span>PM console</span>
    </div>

    <button class="refresh" type="button" on:click={loadRuns} title="Refresh runs">↻</button>

    <div class="run-list">
      {#each runs as run}
        <button class:selected={run.id === selectedRunId} type="button" on:click={() => selectRun(run.id)}>
          <span class={statusClass(run.status)}></span>
          <span>
            <strong>{run.repoUrl.split("/").slice(-1)[0]}</strong>
            <small>{run.status} · {new Date(run.updatedAt).toLocaleTimeString()}</small>
          </span>
        </button>
      {:else}
        <p class="empty">No runs yet</p>
      {/each}
    </div>
  </aside>

  <section class="workspace">
    <form class="composer" on:submit|preventDefault={createRun}>
      <label>
        <span>Project</span>
        <input bind:value={projectId} />
      </label>
      <label>
        <span>Repository</span>
        <input bind:value={repoUrl} />
      </label>
      <label>
        <span>Model</span>
        <input bind:value={model} />
      </label>
      <label class="prompt">
        <span>Task</span>
        <textarea bind:value={prompt}></textarea>
      </label>
      <button disabled={loading} type="submit">{loading ? "Queuing..." : "Start run"}</button>
      {#if error}<p class="error">{error}</p>{/if}
    </form>

    <div class="detail">
      <header>
        <div>
          <span class="eyebrow">Selected run</span>
          <h1>{selectedRun ? selectedRun.repoUrl : "Waiting for a task"}</h1>
        </div>
        {#if selectedRun}
          <span class={statusClass(selectedRun.status)}>{selectedRun.status}</span>
        {/if}
      </header>

      <div class="log">
        {#each logs as item}
          <div class="log-line {item.level}">
            <time>{new Date(item.ts).toLocaleTimeString()}</time>
            <span>{item.stream}</span>
            <code>{item.msg}</code>
          </div>
        {:else}
          <div class="empty log-empty">No logs for this run</div>
        {/each}
      </div>
    </div>
  </section>
</main>
