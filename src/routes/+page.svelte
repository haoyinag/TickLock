<script lang="ts">
  import '../app.css';
  import { onMount } from 'svelte';
  import Timer from '$lib/components/Timer.svelte';
  import { getSettings, getThemes, onSettingsChanged, onThemesChanged } from '$lib/ipc';
  import { settings } from '$lib/stores/settings';
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
  const unlocked = $derived(!$settings.overlay_locked_clickthrough);

  $effect(() => {
    function update() {
      const available = Math.max(72, Math.min(window.innerWidth, window.innerHeight) - 14);
      uiScale = Math.max(0.28, Math.min(available / 252, 1.25));
    }
    update();
    window.addEventListener('resize', update);
    return () => window.removeEventListener('resize', update);
  });

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
        try {
          const existing = await WebviewWindow.getByLabel('settings');
          if (existing) {
            await existing.show();
            await existing.setFocus();
            return;
          }
          const win = new WebviewWindow('settings', {
            url: '/settings',
            title: 'Pomotroid - Settings',
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
          await logError(`[main] failed to open settings from tray: ${err}`);
        }
      });
      cleanups.push(trayListener);
    })();

    return () => {
      for (const fn of cleanups) fn();
    };
  });

  async function startDrag() {
    if (!unlocked) return;
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
  role="application"
  aria-label="Pomodoro floating timer"
>
  <div class="shell-layer" data-tauri-drag-region></div>
  {#if unlocked}
    <button class="drag-zone" aria-label="Drag floating timer" onmousedown={startDrag}></button>
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
    overflow: hidden;
    border-radius: 50%;
    animation: ball-scale-in 0.28s var(--transition-slow) both;
  }

  .shell-layer {
    position: absolute;
    inset: 6px;
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
