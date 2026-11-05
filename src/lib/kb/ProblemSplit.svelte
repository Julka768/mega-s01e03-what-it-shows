<script lang="ts">
  import reports from '#lib/data/reports-4471.json';
  import notes from '#lib/data/desk-notes.json';
  import { causeOf, uniqueById, type Report } from '#lib/kb/causes';
  import { tenantOf } from '#lib/kb/locales';

  // One row per problem in the ticket, each with the ticket it now lives in.
  type Problem = 'parser' | 'hint' | 'one-day move' | 'unplaced';
  let { ticket, issues }: { ticket: number; issues: Partial<Record<Problem, string>> } = $props();

  // The desk's own one-day-move list wins over the cause by tenant: these reports are not a swap.
  const problemOf = (r: Report): Problem =>
    notes.oneDayMove.includes(r.id) ? 'one-day move' : (causeOf(r, tenantOf(r)) ?? 'unplaced');

  const rows = $derived.by(() => {
    const all = uniqueById(reports as Report[]).filter((r) => r.mergedInto === ticket);
    const by = new Map<Problem, Report[]>();
    for (const r of all) by.set(problemOf(r), [...(by.get(problemOf(r)) ?? []), r]);
    return {
      total: all.length,
      groups: [...by.entries()]
        .map(([problem, rs]) => {
          const days = rs.map((r) => r.receivedAt.slice(0, 10)).sort();
          return {
            problem,
            count: rs.length,
            tenants: new Set(rs.map((r) => r.tenant)).size,
            firstSeen: days[0],
            lastSeen: days[days.length - 1],
          };
        })
        .sort((a, b) => b.count - a.count),
    };
  });
  const issueNo = (url: string) => '#' + url.split('/').pop();
</script>

<p class="sum">
  <strong>{rows.total}</strong> reports merged into #{ticket} ·
  <strong>{rows.groups.length}</strong>
  {rows.groups.length === 1 ? 'problem' : 'problems'}
</p>

<table>
  <thead>
    <tr><th>problem</th><th>reports</th><th>tenants</th><th>seen</th><th>ticket</th></tr>
  </thead>
  <tbody>
    {#each rows.groups as g (g.problem)}
      <tr>
        <td><code>{g.problem}</code></td>
        <td>{g.count}</td>
        <td>{g.tenants}</td>
        <td>{g.firstSeen} → {g.lastSeen}</td>
        <td>
          {#if issues[g.problem]}<a href={issues[g.problem]}>{issueNo(issues[g.problem]!)}</a>{:else}<span class="none">no ticket yet</span>{/if}
        </td>
      </tr>
    {/each}
  </tbody>
</table>

<style>
  .sum { font-size: 1.1rem; }
  .none { color: var(--warn); font-weight: 600; }
</style>
