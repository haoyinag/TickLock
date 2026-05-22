<script lang="ts">
  import '../app.css';
  import { onMount } from 'svelte';
  import Timer from '$lib/components/Timer.svelte';
  import {
    getSettings,
    getThemes,
    onSettingsChanged,
    onThemesChanged,
    setWindowVisibility,
    timerRestartRound,
    timerSkip,
    timerToggle,
  } from '$lib/ipc';
  import { settings } from '$lib/stores/settings';
  import { timerState } from '$lib/stores/timer';
  import { applyTheme } from '$lib/stores/theme';
  import { resolveThemeName } from '$lib/utils/theme';
  import { isMac } from '$lib/utils/platform';
  import { setLocale } from '$lib/locale.svelte.js';
  import { WebviewWindow, getCurrentWebviewWindow } from '@tauri-apps/api/webviewWindow';
  import { listen, type UnlistenFn } from '@tauri-apps/api/event';
  import { info, error as logError } from '@tauri-apps/plugin-log';
  import { createLocalShortcutHandler } from '$lib/utils/localShortcuts';

  let localVolume = $state(1.0);
  let preMuteVolume = $state(0.5);
  let isFullscreen = $state(false);
  let uiScale = $state(0.78);
  let shellInset = $state(6);
  let hoverReady = $state(false);
  const unlocked = $derived(!$settings.overlay_locked_clickthrough);
  const showHoverControls = $derived($settings.overlay_locked_clickthrough && hoverReady);
  const canDrag = $derived(unlocked || showHoverControls);
  const timerSnap = $derived($timerState);

  $effect(() => {
    function update() {
      const shortest = Math.min(window.innerWidth, window.innerHeight);
      shellInset = Math.max(2, Math.min(shortest * 0.04, 6));
      const available = Math.max(96, shortest - shellInset * 2 - 10);
      uiScale = Math.max(0.42, Math.min(available / 230, 1.25));
    }
    update();
    window.addEventListener('resize', update);
    return () => window.removeEventListener('resize', update);
  });

  async function openSettingsWindow() {
    try {
      const existing = await WebviewWindow.getByLabel('settings');
      if (existing) {
        await existing.show();
        await existing.setFocus();
        return;
      }
      const win = new WebviewWindow('settings', {
        url: '/settings',
        title: 'dicda - Settings',
        width: 720,
        height: 520,
        decorations: isMac,
        titleBarStyle: isMac ? ('Overlay' as any) : undefined,
        hiddenTitle: isMac ? true : undefined,
        resizable: false,
        visible: false,
      });
      await win.show();
      await win.setFocus();
    } catch (err) {
      await logError(`[main] failed to open settings: ${err}`);
    }
  }

  async function hideFloatingBall() {
    await setWindowVisibility(false);
  }

  onMount(() => {
    const cleanups: UnlistenFn[] = [];

    const shortcutHandler = createLocalShortcutHandler({
      getSettings: () => $settings,
      getVolume: () => localVolume,
      setVolume: (v) => {
        localVolume = v;
      },
      getPreMuteVolume: () => preMuteVolume,
      setPreMuteVolume: (v) => {
        preMuteVolume = v;
      },
      getFullscreen: () => isFullscreen,
      setFullscreen: (v) => {
        isFullscreen = v;
      },
    });
    document.addEventListener('keydown', shortcutHandler);
    cleanups.push(() => document.removeEventListener('keydown', shortcutHandler));

    (async () => {
      try {
        const s = await getSettings();
        settings.set(s);
        localVolume = s.volume;

        setLocale(s.language);
        await info(`[main] settings loaded, locale=${s.language}`);

        const themes = await getThemes();
        const osDark = window.matchMedia('(prefers-color-scheme: dark)').matches;
        const active = themes.find((t) => t.name === resolveThemeName(s, osDark)) ?? themes[0];
        if (active) applyTheme(active);
        await getCurrentWebviewWindow().show();
        await info(`[main] initialized floating ball, theme=${active?.name ?? 'none'}`);
      } catch (e) {
        await logError(`[main] initialization failed: ${e}`);
        throw e;
      }

      const mq = window.matchMedia('(prefers-color-scheme: dark)');
      const mqListener = async (e: MediaQueryListEvent) => {
        if ($settings.theme_mode !== 'auto') return;
        const allThemes = await getThemes();
        const t = allThemes.find((th) => th.name === resolveThemeName($settings, e.matches));
        if (t) applyTheme(t);
      };
      mq.addEventListener('change', mqListener);
      cleanups.push(() => mq.removeEventListener('change', mqListener));

      cleanups.push(
        await onSettingsChanged(async (updated) => {
          const prevMode = $settings.theme_mode;
          const prevLight = $settings.theme_light;
          const prevDark = $settings.theme_dark;
          const prevLanguage = $settings.language;
          settings.set(updated);
          localVolume = updated.volume;
          if (updated.language !== prevLanguage) {
            setLocale(updated.language);
          }
          if (
            updated.theme_mode !== prevMode ||
            updated.theme_light !== prevLight ||
            updated.theme_dark !== prevDark
          ) {
            const allThemes = await getThemes();
            const dark = window.matchMedia('(prefers-color-scheme: dark)').matches;
            const t = allThemes.find((th) => th.name === resolveThemeName(updated, dark));
            if (t) applyTheme(t);
          }
        }),
        await onThemesChanged((updated) => {
          const dark = window.matchMedia('(prefers-color-scheme: dark)').matches;
          const current =
            updated.find((t) => t.name === resolveThemeName($settings, dark)) ?? updated[0];
          if (current) applyTheme(current);
        })
      );

      const trayListener = await listen('tray:open-settings', async () => {
        await openSettingsWindow();
      });
      cleanups.push(trayListener);

      const hoverListener = await listen<boolean>('overlay:hover-ready', (event) => {
        hoverReady = event.payload;
      });
      cleanups.push(hoverListener);
    })();

    return () => {
      for (const fn of cleanups) fn();
    };
  });

  async function startDrag() {
    if (!canDrag) return;
    await getCurrentWebviewWindow().startDragging();
  }

  async function startResize(direction: string) {
    if (!unlocked || isMac) return;
    // eslint-disable-next-line @typescript-eslint/no-explicit-any
    await getCurrentWebviewWindow().startResizeDragging(direction as any);
  }
</script>

<div
  class="floating-ball"
  class:locked={$settings.overlay_locked_clickthrough}
  class:unlocked
  style:--shell-opacity={$settings.window_opacity}
  style:--shell-inset={`${shellInset}px`}
  role="application"
  aria-label="Pomodoro floating timer"
>
  <div class="shell-layer" data-tauri-drag-region></div>
  {#if canDrag}
    <button class="drag-zone" aria-label="Drag floating timer" onmousedown={startDrag}></button>
  {/if}
  {#if unlocked}
    {#if !isMac}
      <button
        class="resize-handle resize-n"
        aria-label="Resize floating timer north"
        onmousedown={() => startResize('North')}
      ></button>
      <button
        class="resize-handle resize-e"
        aria-label="Resize floating timer east"
        onmousedown={() => startResize('East')}
      ></button>
      <button
        class="resize-handle resize-s"
        aria-label="Resize floating timer south"
        onmousedown={() => startResize('South')}
      ></button>
      <button
        class="resize-handle resize-w"
        aria-label="Resize floating timer west"
        onmousedown={() => startResize('West')}
      ></button>
      <button
        class="resize-handle resize-ne"
        aria-label="Resize floating timer north east"
        onmousedown={() => startResize('NorthEast')}
      ></button>
      <button
        class="resize-handle resize-nw"
        aria-label="Resize floating timer north west"
        onmousedown={() => startResize('NorthWest')}
      ></button>
      <button
        class="resize-handle resize-se"
        aria-label="Resize floating timer south east"
        onmousedown={() => startResize('SouthEast')}
      ></button>
      <button
        class="resize-handle resize-sw"
        aria-label="Resize floating timer south west"
        onmousedown={() => startResize('SouthWest')}
      ></button>
    {/if}
  {/if}
  <main>
    <Timer isCompact={true} {uiScale} />
  </main>

  {#if showHoverControls}
    <div class="hover-controls" aria-label="Floating timer controls">
      <button
        class="hover-btn hover-close"
        type="button"
        aria-label="Hide floating timer"
        onclick={hideFloatingBall}
      >
        X
      </button>
      <button
        class="hover-btn hover-reset"
        type="button"
        aria-label="Restart round"
        onclick={timerRestartRound}
      >
        <svg width="14" height="14" viewBox="0 0 16 16" aria-hidden="true">
          <polygon points="15,1 6,8 15,15" fill="currentColor" />
          <rect x="1" y="1" width="3" height="14" rx="1" fill="currentColor" />
        </svg>
      </button>
      <button
        class="hover-btn hover-play"
        type="button"
        aria-label={timerSnap.is_running ? 'Pause' : 'Play'}
        onclick={timerToggle}
      >
        {#if timerSnap.is_running}
          <svg width="18" height="18" viewBox="0 0 24 24" aria-hidden="true">
            <rect x="6" y="4" width="4" height="16" rx="1.2" fill="currentColor" />
            <rect x="14" y="4" width="4" height="16" rx="1.2" fill="currentColor" />
          </svg>
        {:else}
          <svg width="16" height="16" viewBox="0 0 24 24" aria-hidden="true">
            <polygon points="6,3 21,12 6,21" fill="currentColor" />
          </svg>
        {/if}
      </button>
      <button
        class="hover-btn hover-skip"
        type="button"
        aria-label="Skip round"
        onclick={timerSkip}
      >
        <svg width="14" height="14" viewBox="0 0 16 16" aria-hidden="true">
          <polygon points="1,1 10,8 1,15" fill="currentColor" />
          <rect x="12" y="1" width="3" height="14" rx="1" fill="currentColor" />
        </svg>
      </button>
      <button
        class="hover-btn hover-settings"
        type="button"
        aria-label="Open settings"
        onclick={openSettingsWindow}
      >
        <svg width="14" height="14" viewBox="0 0 24 24" aria-hidden="true">
          <path
            d="M19.4 13.4c.1-.5.1-.9.1-1.4s0-.9-.1-1.4l2-1.5-2-3.5-2.4 1a8 8 0 0 0-2.4-1.4L14.3 2h-4.1l-.4 3.2c-.9.3-1.7.8-2.4 1.4l-2.4-1-2 3.5 2 1.5a8.4 8.4 0 0 0 0 2.8l-2 1.5 2 3.5 2.4-1c.7.6 1.5 1.1 2.4 1.4l.4 3.2h4.1l.4-3.2c.9-.3 1.7-.8 2.4-1.4l2.4 1 2-3.5-2.1-1.5ZM12.3 15.5A3.5 3.5 0 1 1 12.3 8a3.5 3.5 0 0 1 0 7.5Z"
            fill="currentColor"
          />
        </svg>
      </button>
    </div>
  {/if}
</div>

<style>
  :global(html),
  :global(body) {
    background: transparent;
  }

  .floating-ball {
    position: relative;
    width: 100%;
    height: 100%;
    display: grid;
    place-items: center;
    /* overflow: hidden; */
    border-radius: 50%;
    animation: ball-scale-in 0.28s var(--transition-slow) both;
    padding: 12px;
  }

  .shell-layer {
    position: absolute;
    inset: var(--shell-inset, 6px);
    border-radius: 50%;
    background:
      radial-gradient(
        circle at 38% 28%,
        color-mix(in oklch, white 22%, transparent),
        transparent 36%
      ),
      color-mix(in oklch, var(--color-background) calc(var(--shell-opacity) * 100%), transparent);
    border: 1px solid
      color-mix(
        in oklch,
        var(--color-foreground) calc((0.18 + var(--shell-opacity) * 0.32) * 100%),
        transparent
      );
    box-shadow:
      inset 0 0 18px color-mix(in oklch, white 10%, transparent),
      0 10px 30px color-mix(in oklch, black calc(var(--shell-opacity) * 28%), transparent);
    opacity: 1;
    transition:
      opacity var(--transition-default),
      border-color var(--transition-default),
      box-shadow var(--transition-default);
    pointer-events: none;
  }

  .floating-ball.locked .shell-layer {
    opacity: 0;
    border-color: transparent;
    box-shadow: none;
  }

  main {
    position: relative;
    z-index: 1;
    width: 100%;
    height: 100%;
    display: grid;
    place-items: center;
    overflow: hidden;
  }

  .hover-controls {
    position: absolute;
    inset: 0;
    z-index: 4;
    pointer-events: none;
    animation: controls-fade-in 0.14s ease-out both;
  }

  .hover-btn {
    position: absolute;
    z-index: 1;
    display: grid;
    place-items: center;
    width: 26px;
    height: 26px;
    padding: 0;
    border: 1px solid color-mix(in oklch, var(--color-foreground) 28%, transparent);
    border-radius: 50%;
    color: var(--color-foreground);
    background: color-mix(in oklch, var(--color-background) 56%, transparent);
    box-shadow: 0 0 10px color-mix(in oklch, black 22%, transparent);
    cursor: pointer;
    pointer-events: auto;
    transition:
      background var(--transition-default),
      color var(--transition-default),
      transform var(--transition-default);
  }

  .hover-btn:hover {
    color: var(--color-accent);
    background: color-mix(in oklch, var(--color-background) 78%, transparent);
    transform: scale(1.08);
  }

  .hover-play {
    top: 50%;
    left: 50%;
    width: 34px;
    height: 34px;
    transform: translate(-50%, -50%);
  }

  .hover-play:hover {
    transform: translate(-50%, -50%) scale(1.08);
  }

  .hover-reset {
    top: 50%;
    left: 19%;
    transform: translateY(-50%);
  }

  .hover-reset:hover {
    transform: translateY(-50%) scale(1.08);
  }

  .hover-skip {
    top: 50%;
    right: 19%;
    transform: translateY(-50%);
  }

  .hover-skip:hover {
    transform: translateY(-50%) scale(1.08);
  }

  .hover-settings {
    left: 50%;
    bottom: 18%;
    transform: translateX(-50%);
  }

  .hover-settings:hover {
    transform: translateX(-50%) scale(1.08);
  }

  .hover-close {
    top: 16%;
    right: 18%;
    font-size: 18px;
    line-height: 1;
  }

  @keyframes controls-fade-in {
    from {
      opacity: 0;
    }
    to {
      opacity: 1;
    }
  }

  .drag-zone,
  .resize-handle {
    position: absolute;
    z-index: 3;
    appearance: none;
    border: 0;
    padding: 0;
    background: transparent;
  }

  .drag-zone {
    top: 10px;
    left: 50%;
    width: 54%;
    height: 28px;
    transform: translateX(-50%);
    border-radius: 999px;
    cursor: move;
  }

  .drag-zone::after {
    content: '';
    position: absolute;
    left: 50%;
    top: 8px;
    width: 30px;
    height: 3px;
    transform: translateX(-50%);
    border-radius: 999px;
    background: color-mix(in oklch, var(--color-foreground) 32%, transparent);
    box-shadow: 0 0 8px color-mix(in oklch, var(--color-foreground) 22%, transparent);
  }

  .resize-handle {
    opacity: 0;
  }

  .resize-n,
  .resize-s {
    left: 24%;
    width: 52%;
    height: 10px;
  }

  .resize-e,
  .resize-w {
    top: 24%;
    width: 10px;
    height: 52%;
  }

  .resize-n {
    top: 0;
    cursor: n-resize;
  }

  .resize-e {
    right: 0;
    cursor: e-resize;
  }

  .resize-s {
    bottom: 0;
    cursor: s-resize;
  }

  .resize-w {
    left: 0;
    cursor: w-resize;
  }

  .resize-ne,
  .resize-nw,
  .resize-se,
  .resize-sw {
    width: 34px;
    height: 34px;
  }

  .resize-ne {
    top: 2px;
    right: 2px;
    cursor: ne-resize;
  }

  .resize-nw {
    top: 2px;
    left: 2px;
    cursor: nw-resize;
  }

  .resize-se {
    right: 2px;
    bottom: 2px;
    cursor: se-resize;
  }

  .resize-sw {
    left: 2px;
    bottom: 2px;
    cursor: sw-resize;
  }

  @keyframes ball-scale-in {
    from {
      transform: scale(0.94);
    }
    to {
      transform: scale(1);
    }
  }
</style>
