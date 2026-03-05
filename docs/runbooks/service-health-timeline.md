# Service Health Timeline Runbook

This runbook validates T4.3.6 timeline behavior on service detail pages.

## 1. Scope

Service detail page (`/services/:serviceId`) includes a compact health timeline that:

- displays status segments (`healthy`, `degraded`, `down`)
- supports selectable windows (`6h`, `24h` default, `7d`)
- shows timestamp + reason metadata on hover/click selection
- remains readable on mobile via compact layout and horizontal overflow support

## 2. Data source behavior

Load order for timeline data:

1. `GET /api/services/{serviceId}/health-timeline?range={window}`
2. local fallback file `apps/portal/frontend/service-health-timeline.sample.json`
3. safe empty timeline state

## 3. Validation steps

1. Open `/services/homelab-api`.
2. Confirm `Service Health Timeline` section exists and default window is `24h`.
3. Switch between `6h`, `24h`, and `7d` and verify segment updates.
4. Hover and click segments; verify metadata panel updates with:
   - status
   - start/end timestamp
   - duration
   - reason (if present)
5. Validate mobile width behavior (no overlap, timeline remains legible).

## 4. Troubleshooting

- Empty timeline:
  - check API response shape (`startAt`, `endAt`, `status`)
  - verify fallback sample includes chosen `window`
- All segments unknown:
  - verify status normalization accepts expected backend strings
- Timeline unreadable on small screens:
  - confirm overflow container is present and minimum width constraints are intact

## 5. Evidence files

- `apps/portal/frontend/src/components/service-health-timeline.tsx`
- `apps/portal/frontend/src/lib/adapters/service-health-timeline.ts`
- `apps/portal/frontend/src/pages/service-details-page.tsx`
- `apps/portal/frontend/service-health-timeline.sample.json`
