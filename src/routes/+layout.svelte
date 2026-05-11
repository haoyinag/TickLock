<script lang="ts">
  import { onMount } from 'svelte';
  import { toggleWindowLock } from '$lib/ipc';
  import { settings } from '$lib/stores/settings';

  onMount(() => {
    let hoverReady = false;

    const onHoverReadyChanged = (event: Event) => {
      const customEvent = event as CustomEvent<boolean>;
      hoverReady = customEvent.detail === true;
    };

    const disableContextMenu = async (event: MouseEvent) => {
      const onMainRoute = window.location.pathname === '/';
      if (onMainRoute && $settings.overlay_mode_enabled && hoverReady) {
        event.preventDefault();
        try {
          settings.set(await toggleWindowLock());
        } catch {
          // Ignore transient toggle failures to avoid blocking the app.
        }
        return;
      }
      event.preventDefault();
    };

    window.addEventListener('overlay:hover-ready', onHoverReadyChanged as EventListener);
    document.addEventListener('contextmenu', disableContextMenu, { capture: true });

    return () => {
      window.removeEventListener('overlay:hover-ready', onHoverReadyChanged as EventListener);
      document.removeEventListener('contextmenu', disableContextMenu, { capture: true });
    };
  });
</script>

<slot />
