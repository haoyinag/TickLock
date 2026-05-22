<script lang="ts">
  import { cubicOut } from 'svelte/easing';
  import { tweened } from 'svelte/motion';
  import type { TimerState } from '$lib/types';

  interface Props {
    snap: TimerState;
    countdown?: boolean;
    colorStart?: string;
    colorEnd?: string;
  }

  let { snap, countdown = false, colorStart = '#ff7a45', colorEnd = '#ff2d75' }: Props = $props();

  const CENTER = 115;
  const RADIUS = 85;
  const CIRCUMFERENCE = 2 * Math.PI * RADIUS;
  const SEGMENT_COUNT = 96;

  const dashOffset = tweened(CIRCUMFERENCE, { duration: 800, easing: cubicOut });

  let prevRound = $state<string>('');
  const visibleProgress = $derived(Math.max(0, Math.min(1, 1 - $dashOffset / CIRCUMFERENCE)));
  const segments = $derived(buildSegments(visibleProgress, colorStart, colorEnd));

  function parseHexColor(hex: string): [number, number, number] | null {
    const raw = hex.trim().replace(/^#/, '');
    const normalized =
      raw.length === 3
        ? raw
            .split('')
            .map((c) => c + c)
            .join('')
        : raw;
    if (!/^[0-9a-fA-F]{6}$/.test(normalized)) return null;
    return [
      Number.parseInt(normalized.slice(0, 2), 16),
      Number.parseInt(normalized.slice(2, 4), 16),
      Number.parseInt(normalized.slice(4, 6), 16),
    ];
  }

  function mixColor(start: string, end: string, t: number) {
    const a = parseHexColor(start) ?? [255, 122, 69];
    const b = parseHexColor(end) ?? [255, 45, 117];
    const mixed = a.map((channel, i) => Math.round(channel + (b[i] - channel) * t));
    return `rgb(${mixed[0]} ${mixed[1]} ${mixed[2]})`;
  }

  function pointOnCircle(ratio: number) {
    const angle = -Math.PI / 2 + ratio * Math.PI * 2;
    return {
      x: CENTER + RADIUS * Math.cos(angle),
      y: CENTER + RADIUS * Math.sin(angle),
    };
  }

  function arcPath(startRatio: number, endRatio: number) {
    const start = pointOnCircle(startRatio);
    const end = pointOnCircle(endRatio);
    const largeArc = endRatio - startRatio > 0.5 ? 1 : 0;
    return `M ${start.x.toFixed(3)} ${start.y.toFixed(3)} A ${RADIUS} ${RADIUS} 0 ${largeArc} 1 ${end.x.toFixed(3)} ${end.y.toFixed(3)}`;
  }

  function buildSegments(progress: number, start: string, end: string) {
    if (progress <= 0) return [];

    const total = Math.ceil(progress * SEGMENT_COUNT);
    return Array.from({ length: total }, (_, i) => {
      const startRatio = i / SEGMENT_COUNT;
      const endRatio = Math.min((i + 1) / SEGMENT_COUNT, progress);
      const localT = progress <= 0 ? 0 : (startRatio + endRatio) / 2 / progress;
      return {
        key: `${i}-${endRatio.toFixed(3)}`,
        d: arcPath(startRatio, endRatio),
        color: mixColor(start, end, Math.max(0, Math.min(1, localT))),
      };
    }).filter((segment) => segment.d.length > 0);
  }

  $effect(() => {
    const rt = snap.round_type;
    const progress = snap.total_secs > 0 ? snap.elapsed_secs / snap.total_secs : 0;
    const target = countdown ? CIRCUMFERENCE * progress : CIRCUMFERENCE * (1 - progress);
    const startOffset = countdown ? 0 : CIRCUMFERENCE;

    if (rt !== prevRound) {
      dashOffset.set(startOffset, { duration: 0 });
      prevRound = rt;
    } else {
      dashOffset.set(target);
    }
  });
</script>

<svg class="dial" viewBox="0 0 230 230" aria-hidden="true" style:--progress-glow={colorStart}>
  <circle
    class="track"
    cx={CENTER}
    cy={CENTER}
    r={RADIUS}
    fill="none"
    stroke="currentColor"
    stroke-width="2"
  />
  {#each segments as segment (segment.key)}
    <path
      class="progress-segment"
      d={segment.d}
      fill="none"
      stroke={segment.color}
      stroke-width="10"
      stroke-linecap="round"
    />
  {/each}
</svg>

<style>
  .dial {
    width: 220px;
    height: 220px;
    display: block;
    color: color-mix(in oklch, var(--color-foreground) 42%, transparent);
    filter: drop-shadow(0 0 8px color-mix(in oklch, var(--color-foreground) 18%, transparent))
      drop-shadow(0 2px 10px color-mix(in oklch, black 42%, transparent));
  }

  .progress-segment {
    filter: drop-shadow(0 0 7px var(--progress-glow));
  }
</style>
