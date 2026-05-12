<script lang="ts">
  import { fade } from 'svelte/transition';
  import { timerToggle, timerRestartRound, timerSkip } from '$lib/ipc';
  import { timerState } from '$lib/stores/timer';

  let state = $derived($timerState);
</script>

<div class="mini-controls">
  <!-- Restart current round -->
  <button class="btn-side" onclick={timerRestartRound} aria-label="Restart round">
    <svg width="10" height="10" viewBox="0 0 16 16">
      <polygon points="15,1 6,8 15,15" fill="currentColor" />
      <rect x="1" y="1" width="3" height="14" rx="1" fill="currentColor" />
    </svg>
  </button>

  <!-- Play / Pause -->
  <button class="play-pause" onclick={timerToggle} aria-label={state.is_running ? 'Pause' : 'Play'}>
    {#key state.is_running}
      <span class="icon" in:fade={{ duration: 100 }}>
        {#if state.is_running}
          <svg width="12" height="12" viewBox="0 0 24 24">
            <rect x="4" y="3" width="5" height="18" rx="1.5" fill="currentColor" />
            <rect x="15" y="3" width="5" height="18" rx="1.5" fill="currentColor" />
          </svg>
        {:else}
          <svg width="12" height="12" viewBox="0 0 24 24">
            <polygon points="5,3 21,12 5,21" fill="currentColor" />
          </svg>
        {/if}
      </span>
    {/key}
  </button>

  <!-- Skip round -->
  <button class="btn-side" onclick={timerSkip} aria-label="Skip round">
    <svg width="10" height="10" viewBox="0 0 16 16">
      <polygon points="1,1 10,8 1,15" fill="currentColor" />
      <rect x="12" y="1" width="3" height="14" rx="1" fill="currentColor" />
    </svg>
  </button>
</div>

<style>
  .mini-controls {
    display: flex;
    align-items: center;
    gap: 12px;
    filter: drop-shadow(0 2px 8px color-mix(in oklch, black 35%, transparent));
  }

  .btn-side {
    display: flex;
    align-items: center;
    justify-content: center;
    width: 24px;
    height: 24px;
    background: color-mix(in oklch, var(--color-background-light) 48%, transparent);
    border: 1px solid color-mix(in oklch, var(--color-foreground) 24%, transparent);
    border-radius: 50%;
    cursor: pointer;
    color: var(--color-foreground);
    backdrop-filter: blur(10px);
    transition:
      color var(--transition-default),
      border-color var(--transition-default),
      background var(--transition-default);
  }

  .btn-side:hover {
    color: var(--color-accent);
    border-color: color-mix(in oklch, var(--color-accent) 70%, transparent);
    background: color-mix(in oklch, var(--color-accent) 16%, transparent);
  }

  .play-pause {
    display: flex;
    align-items: center;
    justify-content: center;
    width: 24px;
    height: 24px;
    background: color-mix(in oklch, var(--color-background-light) 54%, transparent);
    border: 1.5px solid color-mix(in oklch, var(--color-foreground) 70%, transparent);
    border-radius: 50%;
    cursor: pointer;
    color: var(--color-foreground);
    backdrop-filter: blur(10px);
    overflow: hidden;
    transition:
      color var(--transition-default),
      border-color var(--transition-default),
      background var(--transition-default);
  }

  .play-pause:hover {
    color: var(--color-accent);
    border-color: var(--color-accent);
    background: color-mix(in oklch, var(--color-accent) 16%, transparent);
  }

  .icon {
    display: flex;
    align-items: center;
    justify-content: center;
  }
</style>
