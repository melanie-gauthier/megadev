<script lang="ts">
  import reports from '#lib/data/reports-4471.json';
  import notes from '#lib/data/desk-notes.json';
  import { groupByCause, uniqueById, type CauseGroup, type Report } from '#lib/kb/causes';
  import { tenantOf } from '#lib/kb/locales';

  // One row per problem in a ticket, each with the issue filed for it. causes.ts files the desk's
  // one-day moves under their tenant's cause; they are a problem of their own, so they get their own row.
  let { ticket, issues }: { ticket: number; issues: Record<string, string> } = $props();

  const merged = $derived(uniqueById(reports as Report[]).filter((r) => r.mergedInto === ticket));
  const moved = $derived(merged.filter((r) => notes.oneDayMove.includes(r.id)));
  const rows = $derived.by(() => {
    const groups = groupByCause(
      merged.filter((r) => !notes.oneDayMove.includes(r.id)),
      tenantOf,
    ).map((g) => ({ key: g.cause as string, ...g }));
    if (moved.length) {
      const [shift] = groupByCause(moved, () => undefined);
      groups.push({ ...shift, key: 'one-day move' });
    }
    return groups.sort((a, b) => b.reports.length - a.reports.length);
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

<style>
  .sum { font-size: 1.1rem; }
  .tags { color: var(--muted); font-size: 0.85em; }
</style>
