<script lang="ts">
  import { onMount } from 'svelte';
  import { settings } from '$lib/stores/settings';
  import {
    setSetting,
    setWindowOpacity,
    resetSettings,
    clearSessionHistory,
    traySupported,
  } from '$lib/ipc';
  import SettingsToggle from '$lib/components/settings/SettingsToggle.svelte';
  import * as m from '$paraglide/messages.js';
  import { setLocale } from '$lib/locale.svelte.js';
  import { isMac, isLinux } from '$lib/utils/platform';

  // Language options: value stored in DB, label shown in native language.
  const LANGUAGES = [
    { value: 'auto', label: 'Auto' },
    { value: 'en', label: 'English' },
    { value: 'es', label: 'Español' },
    { value: 'fr', label: 'Français' },
    { value: 'de', label: 'Deutsch' },
    { value: 'ja', label: '日本語' },
    { value: 'zh', label: '中文' },
    { value: 'pt', label: 'Português' },
    { value: 'tr', label: 'Türkçe' },
  ];

  // On Linux, probe for libayatana-appindicator3 at runtime.  The tray section
  // is hidden entirely when the library is absent so users can't enable a
  // feature that would crash the app.  Non-Linux platforms always support tray.
  let trayAvailable = $state(!isLinux);
  onMount(async () => {
    if (isLinux) trayAvailable = await traySupported();
  });

  let localPort = $state(String($settings.websocket_port));
  let localOpacity = $state(Math.round($settings.window_opacity * 100));
  let localOverlaySize = $state($settings.overlay_size);
  let localMinSize = $state(String($settings.overlay_min_size));
  let localMaxSize = $state(String($settings.overlay_max_size));
  let localHoverActivationMs = $state($settings.overlay_hover_activation_ms);
  $effect(() => {
    localPort = String($settings.websocket_port);
    localOpacity = Math.round($settings.window_opacity * 100);
    localOverlaySize = $settings.overlay_size;
    localMinSize = String($settings.overlay_min_size);
    localMaxSize = String($settings.overlay_max_size);
    localHoverActivationMs = $settings.overlay_hover_activation_ms;
  });

  let langOpen = $state(false);
  let langEl: HTMLElement | undefined;

  const selectedLabel = $derived(
    LANGUAGES.find((l) => l.value === $settings.language)?.label ?? 'Auto'
  );

  async function selectLanguage(value: string) {
    langOpen = false;
    const updated = await setSetting('language', value);
    settings.set(updated);
    setLocale(value);
  }

  $effect(() => {
    if (!langOpen) return;
    function onOutside(e: MouseEvent) {
      if (langEl && !langEl.contains(e.target as Node)) langOpen = false;
    }
    window.addEventListener('mousedown', onOutside);
    return () => window.removeEventListener('mousedown', onOutside);
  });

  let confirmingReset = $state(false);
  let confirmingClear = $state(false);

  async function handleReset() {
    const updated = await resetSettings();
    settings.set(updated);
    confirmingReset = false;
  }

  async function handleClear() {
    await clearSessionHistory();
    confirmingClear = false;
  }

  async function toggle(dbKey: string, current: boolean) {
    const updated = await setSetting(dbKey, current ? 'false' : 'true');
    settings.set(updated);
  }

  async function setOverlayLock(enabled: boolean) {
    const updated = await setSetting('overlay_locked_clickthrough', enabled ? 'true' : 'false');
    settings.set(updated);
  }

  async function setOpacity(value: number) {
    const clamped = Math.max(20, Math.min(100, value));
    localOpacity = clamped;
    const opacityValue = clamped / 100;
    const updated = await setSetting('window_opacity', String(opacityValue));
    settings.set(updated);
    await setWindowOpacity(opacityValue).catch(() => undefined);
  }

  async function setHoverActivationMs(value: number) {
    const clamped = Math.max(3000, Math.min(10000, value));
    localHoverActivationMs = clamped;
    const updated = await setSetting('overlay_hover_activation_ms', String(clamped));
    settings.set(updated);
  }

  async function setOverlaySize(value: number) {
    const clamped = Math.max($settings.overlay_min_size, Math.min(value, $settings.overlay_max_size));
    localOverlaySize = clamped;
    const updated = await setSetting('overlay_size', String(clamped));
    settings.set(updated);
  }

  async function setNumberSetting(key: string, value: number) {
    const updated = await setSetting(key, String(value));
    settings.set(updated);
  }

  async function handleMinSizeBlur() {
    const raw = parseInt(localMinSize, 10);
    if (Number.isNaN(raw)) {
      localMinSize = String($settings.overlay_min_size);
      return;
    }
    const value = Math.max(120, Math.min(raw, $settings.overlay_max_size));
    localMinSize = String(value);
    await setNumberSetting('overlay_min_size', value);
  }

  async function handleMaxSizeBlur() {
    const raw = parseInt(localMaxSize, 10);
    if (Number.isNaN(raw)) {
      localMaxSize = String($settings.overlay_max_size);
      return;
    }
    const value = Math.max($settings.overlay_min_size, Math.min(raw, 1000));
    localMaxSize = String(value);
    await setNumberSetting('overlay_max_size', value);
  }

  async function setColorSetting(key: string, value: string) {
    const updated = await setSetting(key, value);
    settings.set(updated);
  }

  async function handlePortBlur() {
    const port = parseInt(localPort, 10);
    if (!isNaN(port) && port >= 1024 && port <= 65535) {
      const updated = await setSetting('websocket_port', String(port));
      settings.set(updated);
    } else {
      localPort = String($settings.websocket_port);
    }
  }
</script>

<div class="section">
  <div class="group-heading">{m.system_group_integrations()}</div>

  <SettingsToggle
    label={m.system_toggle_websocket()}
    description={m.system_toggle_websocket_desc({ port: $settings.websocket_port })}
    tooltip={m.tooltip_websocket()}
    checked={$settings.websocket_enabled}
    onclick={() => toggle('websocket_enabled', $settings.websocket_enabled)}
  />

  {#if $settings.websocket_enabled}
    <div class="row">
      <span class="label">{m.system_label_port()}</span>
      <input
        class="port-input"
        type="number"
        min="1024"
        max="65535"
        bind:value={localPort}
        onblur={handlePortBlur}
      />
    </div>
    <p class="note">
      Listens on <code>ws://127.0.0.1:{$settings.websocket_port}/ws</code>. Send
      <code>{'{ "type": "getState" }'}</code> to query the current timer state. Round changes are broadcast
      automatically.
    </p>
  {/if}

  <div class="group-heading">{m.system_group_language()}</div>

  <div class="lang-row">
    <div class="lang-dropdown" bind:this={langEl}>
      <button
        class="lang-trigger"
        class:open={langOpen}
        onclick={() => (langOpen = !langOpen)}
        onkeydown={(e) => {
          if (e.key === 'Escape') langOpen = false;
        }}
      >
        <span>{selectedLabel}</span>
        <svg
          class="chevron"
          class:open={langOpen}
          width="10"
          height="6"
          viewBox="0 0 10 6"
          aria-hidden="true"
        >
          <polyline
            points="0,0.5 5,5.5 10,0.5"
            fill="none"
            stroke="currentColor"
            stroke-width="1.5"
            stroke-linecap="round"
            stroke-linejoin="round"
          />
        </svg>
      </button>

      {#if langOpen}
        <ul class="lang-menu" role="listbox">
          {#each LANGUAGES as lang (lang.value)}
            <!-- svelte-ignore a11y_interactive_supports_focus -->
            <li
              class="lang-option"
              class:selected={$settings.language === lang.value}
              role="option"
              aria-selected={$settings.language === lang.value}
              onmousedown={() => selectLanguage(lang.value)}
            >
              {lang.label}
            </li>
          {/each}
        </ul>
      {/if}
    </div>
  </div>

  <div class="group-heading">{m.system_group_updates()}</div>

  <SettingsToggle
    label={m.system_toggle_check_updates()}
    description={m.system_toggle_check_updates_desc()}
    checked={$settings.check_for_updates}
    onclick={() => toggle('check_for_updates', $settings.check_for_updates)}
  />

  <div class="group-heading">{m.advanced_group_logging()}</div>

  <SettingsToggle
    label={m.advanced_toggle_verbose_logging()}
    description={m.advanced_toggle_verbose_logging_desc()}
    tooltip={m.tooltip_verbose_logging()}
    checked={$settings.verbose_logging}
    onclick={() => toggle('verbose_logging', $settings.verbose_logging)}
  />

  {#if trayAvailable}
    <div class="group-heading">{m.system_group_tray()}</div>

    <!-- Show in System Tray: available on all platforms. -->
    <SettingsToggle
      label={m.system_toggle_show_tray()}
      description={m.system_toggle_show_tray_desc()}
      tooltip={isLinux ? m.system_tray_gnome_hint() : undefined}
      checked={$settings.tray_icon_enabled}
      onclick={() => toggle('tray_icon_enabled', $settings.tray_icon_enabled)}
    />

    {#if $settings.tray_icon_enabled}
      <!-- Minimize to Tray is Windows/Linux only: the macOS yellow traffic-light
           button routes to the Dock and cannot be intercepted by the app. -->
      {#if !isMac}
        <SettingsToggle
          label={m.system_toggle_min_tray()}
          description={m.system_toggle_min_tray_desc()}
          checked={$settings.min_to_tray}
          onclick={() => toggle('min_to_tray', $settings.min_to_tray)}
        />
      {/if}
      <!-- Close to Tray is available on all platforms: the CloseRequested event
           fires on macOS (red button / Cmd+W) and can be intercepted. -->
      <SettingsToggle
        label={m.system_toggle_close_tray()}
        description={m.system_toggle_close_tray_desc()}
        checked={$settings.min_to_tray_on_close}
        onclick={() => toggle('min_to_tray_on_close', $settings.min_to_tray_on_close)}
      />
    {/if}
  {/if}

  <div class="group-heading">Floating Ball / &#24748;&#28014;&#29699;</div>

  <SettingsToggle
    label="Always on top / &#21560;&#39030;"
    description="Keep the floating timer above other windows."
    checked={$settings.always_on_top}
    onclick={() => toggle('always_on_top', $settings.always_on_top)}
  />

  <SettingsToggle
    label="Click-through lock / &#40736;&#26631;&#31359;&#36879;&#38145;&#23450;"
    description="When enabled, clicks pass through the floating ball to apps underneath."
    checked={$settings.overlay_locked_clickthrough}
    onclick={() => setOverlayLock(!$settings.overlay_locked_clickthrough)}
  />

  <div class="row hover-delay-row">
    <span class="label-block">
      <span class="label">Hover activation delay / &#24748;&#20572;&#35302;&#21457;&#24310;&#36831;</span>
      <span class="description">How long the mouse must stay over the locked ball before controls appear.</span>
    </span>
    <div class="delay-editor">
      <input
        type="range"
        min="3000"
        max="10000"
        step="500"
        value={localHoverActivationMs}
        oninput={(e) => {
          const value = Number((e.currentTarget as HTMLInputElement).value);
          setHoverActivationMs(value);
        }}
      />
      <span class="delay-value">{(localHoverActivationMs / 1000).toFixed(1)}s</span>
    </div>
  </div>

  <div class="row opacity-row">
    <span class="label-block">
      <span class="label">Shell opacity / &#22771;&#23618;&#36879;&#26126;&#24230;</span>
      <span class="description"
        >Only the floating shell fades; timer digits, ring, and controls stay clear.</span
      >
    </span>
    <div class="opacity-editor">
      <input
        type="range"
        min="20"
        max="100"
        value={localOpacity}
        oninput={(e) => {
          const value = Number((e.currentTarget as HTMLInputElement).value);
          setOpacity(value);
        }}
      />
      <span class="opacity-value">{localOpacity}%</span>
    </div>
  </div>

  <div class="row overlay-size-row">
    <span class="label-block">
      <span class="label">Floating ball size / &#24748;&#28014;&#29699;&#23610;&#23544;</span>
      <span class="description">Adjusts the whole always-on-top floating window and timer UI together.</span>
    </span>
    <div class="size-slider-editor">
      <input
        type="range"
        min={$settings.overlay_min_size}
        max={$settings.overlay_max_size}
        step="1"
        value={localOverlaySize}
        oninput={(e) => {
          const value = Number((e.currentTarget as HTMLInputElement).value);
          setOverlaySize(value);
        }}
      />
      <span class="size-value">{localOverlaySize}px</span>
    </div>
  </div>

  <div class="row size-row">
    <span class="label-block">
      <span class="label">Size limits / &#23610;&#23544;&#33539;&#22260;</span>
      <span class="description">Unlocked resize is constrained between these square sizes.</span>
    </span>
    <div class="size-editor">
      <label>
        <span>Min</span>
        <input
          class="size-input"
          type="number"
          min="120"
          max={$settings.overlay_max_size}
          bind:value={localMinSize}
          onblur={handleMinSizeBlur}
        />
      </label>
      <label>
        <span>Max</span>
        <input
          class="size-input"
          type="number"
          min={$settings.overlay_min_size}
          max="1000"
          bind:value={localMaxSize}
          onblur={handleMaxSizeBlur}
        />
      </label>
    </div>
  </div>

  <div class="row color-row">
    <span class="label-block">
      <span class="label">Progress gradient / &#36827;&#24230;&#28176;&#21464;</span>
      <span class="description">Customize the floating ball progress ring colors.</span>
    </span>
    <div class="color-editor">
      <input
        type="color"
        value={$settings.overlay_progress_color_start}
        aria-label="Progress gradient start color"
        oninput={(e) =>
          setColorSetting(
            'overlay_progress_color_start',
            (e.currentTarget as HTMLInputElement).value
          )}
      />
      <input
        type="color"
        value={$settings.overlay_progress_color_end}
        aria-label="Progress gradient end color"
        oninput={(e) =>
          setColorSetting(
            'overlay_progress_color_end',
            (e.currentTarget as HTMLInputElement).value
          )}
      />
    </div>
  </div>

  <div class="group-heading">{m.system_group_data()}</div>

  <div class="data-group">
    {#if !confirmingClear}
      <button class="data-row" onclick={() => (confirmingClear = true)}>
        <span>Clear Session History</span>
      </button>
    {:else}
      <div class="confirm-row">
        <span class="confirm-label"
          >This will permanently delete all session history. This cannot be undone.</span
        >
        <div class="confirm-actions">
          <button class="confirm-cancel" onclick={() => (confirmingClear = false)}>Cancel</button>
          <button class="confirm-destructive" onclick={handleClear}>Clear</button>
        </div>
      </div>
    {/if}
    {#if !confirmingReset}
      <button class="data-row" onclick={() => (confirmingReset = true)}>
        <span>{m.about_reset_all()}</span>
      </button>
    {:else}
      <div class="confirm-row">
        <span class="confirm-label">{m.about_reset_confirm()}</span>
        <div class="confirm-actions">
          <button class="confirm-cancel" onclick={() => (confirmingReset = false)}>Cancel</button>
          <button class="confirm-destructive" onclick={handleReset}>Reset</button>
        </div>
      </div>
    {/if}
  </div>
</div>

<style>
  .section {
    display: flex;
    flex-direction: column;
    padding-bottom: 20px;
  }

  .group-heading {
    font-size: 0.68rem;
    font-weight: 600;
    letter-spacing: 0.1em;
    text-transform: uppercase;
    color: var(--color-foreground-darker, var(--color-foreground));
    opacity: 0.6;
    margin: 0;
    padding: 16px 20px 6px;
  }

  .row {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 16px;
    padding: 10px 20px;
    border-bottom: 1px solid var(--color-separator);
  }

  .opacity-row,
  .overlay-size-row,
  .hover-delay-row {
    align-items: flex-start;
  }

  .size-row,
  .color-row {
    align-items: flex-start;
  }

  .label {
    font-size: 0.85rem;
    color: var(--color-foreground);
    letter-spacing: 0.02em;
  }

  .label-block {
    display: flex;
    flex-direction: column;
    gap: 3px;
    min-width: 0;
  }

  .description {
    font-size: 0.74rem;
    line-height: 1.35;
    color: var(--color-foreground-darker, var(--color-foreground));
    opacity: 0.72;
  }

  .port-input {
    background: var(--color-hover);
    border: 1px solid color-mix(in oklch, var(--color-foreground) 18%, transparent);
    border-radius: 4px;
    color: var(--color-foreground);
    font-size: 0.85rem;
    font-family: monospace;
    padding: 4px 10px;
    width: 90px;
    text-align: right;
    outline: none;
    transition: border-color 0.15s;
  }

  .port-input:focus {
    border-color: var(--color-accent);
  }

  .note {
    font-size: 0.75rem;
    color: var(--color-foreground-darker, var(--color-foreground));
    opacity: 0.65;
    padding: 16px 20px;
    line-height: 1.6;
  }

  .opacity-editor,
  .size-slider-editor,
  .delay-editor {
    display: flex;
    align-items: center;
    gap: 8px;
    flex-shrink: 0;
  }

  .size-editor,
  .color-editor {
    display: flex;
    align-items: center;
    gap: 10px;
    flex-shrink: 0;
  }

  .size-editor label {
    display: flex;
    align-items: center;
    gap: 5px;
    color: var(--color-foreground-darker, var(--color-foreground));
    font-size: 0.72rem;
  }

  .size-input {
    width: 64px;
    background: var(--color-hover);
    border: 1px solid color-mix(in oklch, var(--color-foreground) 18%, transparent);
    border-radius: 4px;
    color: var(--color-foreground);
    font-size: 0.8rem;
    padding: 4px 8px;
    text-align: right;
    outline: none;
  }

  .size-input:focus {
    border-color: var(--color-accent);
  }

  .color-editor input[type='color'] {
    width: 34px;
    height: 26px;
    padding: 0;
    border: 1px solid color-mix(in oklch, var(--color-foreground) 22%, transparent);
    border-radius: 4px;
    background: transparent;
    cursor: pointer;
  }

  .opacity-editor input[type='range'],
  .size-slider-editor input[type='range'],
  .delay-editor input[type='range'] {
    width: 130px;
  }

  .opacity-value,
  .size-value,
  .delay-value {
    font-size: 0.8rem;
    color: var(--color-foreground-darker, var(--color-foreground));
    min-width: 44px;
    text-align: right;
  }

  code {
    font-family: monospace;
    font-size: 0.75em;
    background: color-mix(in oklch, var(--color-foreground) 12%, transparent);
    padding: 1px 5px;
    border-radius: 3px;
  }

  .lang-row {
    padding: 10px 20px;
    border-bottom: 1px solid var(--color-separator);
  }

  .lang-dropdown {
    position: relative;
  }

  .lang-trigger {
    display: flex;
    align-items: center;
    justify-content: space-between;
    width: 100%;
    background: var(--color-hover);
    border: 1px solid color-mix(in oklch, var(--color-foreground) 18%, transparent);
    border-radius: 4px;
    color: var(--color-foreground);
    font-size: 0.85rem;
    padding: 6px 10px;
    cursor: pointer;
    outline: none;
    transition: border-color 0.15s;
    text-align: left;
  }

  .lang-trigger:hover,
  .lang-trigger:focus,
  .lang-trigger.open {
    border-color: var(--color-accent);
  }

  .chevron {
    flex-shrink: 0;
    margin-left: 6px;
    color: var(--color-foreground-darker, var(--color-foreground));
    opacity: 0.7;
    transition: transform 0.15s;
  }

  .chevron.open {
    transform: rotate(180deg);
  }

  .lang-menu {
    position: absolute;
    top: calc(100% + 3px);
    left: 0;
    right: 0;
    background: var(--color-background-light);
    border: 1px solid color-mix(in oklch, var(--color-foreground) 15%, transparent);
    border-radius: 4px;
    box-shadow: 0 4px 16px color-mix(in oklch, black 20%, transparent);
    z-index: 200;
    list-style: none;
    margin: 0;
    padding: 3px 0;
  }

  .lang-option {
    padding: 7px 12px;
    font-size: 0.85rem;
    color: var(--color-foreground);
    cursor: pointer;
    transition: background 0.1s;
  }

  .lang-option:hover {
    background: var(--color-hover);
  }

  .lang-option.selected {
    color: var(--color-accent);
    font-weight: 500;
  }

  .data-group {
    border: 1px solid var(--color-separator);
    border-radius: 6px;
    overflow: hidden;
    margin: 6px 20px 0;
  }

  .data-row {
    display: flex;
    align-items: center;
    width: 100%;
    padding: 12px 16px;
    background: none;
    border: none;
    border-bottom: 1px solid var(--color-separator);
    cursor: pointer;
    color: var(--color-foreground-darker, var(--color-foreground));
    font-size: 0.85rem;
    letter-spacing: 0.02em;
    text-align: left;
    transition:
      background 0.12s,
      color 0.12s;
  }

  .data-row:last-child {
    border-bottom: none;
  }

  .data-row:hover {
    background: var(--color-hover);
    color: color-mix(in oklch, var(--color-focus-round) 80%, var(--color-foreground));
  }

  .confirm-row {
    display: flex;
    flex-direction: column;
    gap: 10px;
    padding: 12px 16px;
    border-bottom: 1px solid var(--color-separator);
  }

  .confirm-row:last-child {
    border-bottom: none;
  }

  .confirm-label {
    font-size: 0.8rem;
    color: var(--color-foreground-darker, var(--color-foreground));
    opacity: 0.8;
  }

  .confirm-actions {
    display: flex;
    gap: 8px;
    justify-content: flex-end;
  }

  .confirm-cancel,
  .confirm-destructive {
    background: none;
    border: 1px solid color-mix(in oklch, var(--color-foreground) 18%, transparent);
    border-radius: 4px;
    font-size: 0.8rem;
    padding: 5px 14px;
    cursor: pointer;
    transition:
      border-color 0.15s,
      color 0.15s;
  }

  .confirm-cancel {
    color: var(--color-foreground-darker, var(--color-foreground));
  }

  .confirm-cancel:hover {
    border-color: color-mix(in oklch, var(--color-foreground) 40%, transparent);
    color: var(--color-foreground);
  }

  .confirm-destructive {
    color: var(--color-accent);
    border-color: color-mix(in oklch, var(--color-accent) 40%, transparent);
  }

  .confirm-destructive:hover {
    background: color-mix(in oklch, var(--color-accent) 10%, transparent);
    border-color: var(--color-accent);
  }
</style>
