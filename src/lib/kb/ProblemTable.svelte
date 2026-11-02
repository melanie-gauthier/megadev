<script lang="ts">
  import reports from '#lib/data/reports-4471.json';
  import notes from '#lib/data/desk-notes.json';
  import { groupByCause, readAs, uniqueById, type CauseGroup, type Report } from '#lib/kb/causes';
  import { tenantOf } from '#lib/kb/locales';

  // One row per problem in a ticket, each with the issue filed for it. Counted the desk's way: the
  // one-day moves (desk-notes.json) stay counted under their tenant's cause, as CauseTable files them,
  // and also get a row of their own, footnoted with where they are filed.
  let { ticket, issues }: { ticket: number; issues: Record<string, string> } = $props();

  const merged = $derived(uniqueById(reports as Report[]).filter((r) => r.mergedInto === ticket));
  const groups = $derived(groupByCause(merged, tenantOf));
  const moved = $derived(merged.filter((r) => notes.oneDayMove.includes(r.id)));
  const filedUnder = $derived(groups.filter((g) => readAs(g, notes.oneDayMove)).map((g) => g.cause));
  const rows = $derived.by(() => {
    const out = groups.map((g) => ({ key: g.cause as string, ...g }));
    if (moved.length) {
      const [shift] = groupByCause(moved, () => undefined);
      out.push({ ...shift, key: 'one-day move' });
    }
    return out;
  });
  const tags = (g: Pick<CauseGroup, 'taggedAs'>) =>
    Object.entries(g.taggedAs)
      .sort((a, b) => b[1] - a[1])
      .map(([tag, n]) => `${tag} ×${n}`)
      .join(' · ');
</script>

<p class="sum">
  <strong>{merged.length}</strong> reports merged into #{ticket} ·
  <strong>{rows.length}</strong>
  {rows.length === 1 ? 'problem' : 'problems'}
</p>

<table>
  <thead>
    <tr><th>problem</th><th>reports</th><th>tagged as</th><th>tenants</th><th>seen</th><th>issue</th></tr>
  </thead>
  <tbody>
    {#each rows as g (g.key)}
      <tr>
        <td><code>{g.key}</code></td>
        <td>{g.reports.length}</td>
        <td class="tags">{tags(g)}</td>
        <td>{g.tenants}</td>
        <td>{g.firstSeen} → {g.lastSeen}</td>
        <td>{#if issues[g.key]}<a href={issues[g.key]}>{issues[g.key].replace(/^.*\/issues\//, '#')}</a>{:else}none yet{/if}</td>
      </tr>
    {/each}
  </tbody>
</table>

{#if moved.length}
  <p class="note">
    The {moved.length} one-day moves are counted under {#each filedUnder as c, i (c)}{i ? ', ' : ''}<code>{c}</code>{/each}, where
    the desk files them by tenant, and have an issue of their own.
  </p>
{/if}

<style>
  .note { color: var(--muted); font-size: 0.85em; }
  .sum { font-size: 1.1rem; }
  .tags { color: var(--muted); font-size: 0.85em; }
</style>
