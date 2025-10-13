<script lang="ts">
  import { onMount, onDestroy } from 'svelte';
  import { Settings, X, Save, Edit2, Trash2, Plus, Move } from 'lucide-svelte'; 
  // (or whatever your Svelte icon library is)

  // Define your states via runes
  const debug = $state(true);
  const showSettings = $state(false);
  const pointer = $state<{ x: number; y: number }>({ x: 0, y: 0 });
  const startPoint = $state<{ x: number; y: number } | null>(null);
  const currentPoint = $state<{ x: number; y: number } | null>(null);
  const isPointerDown = $state(false);
  const savedRectangles = $state<Array<any>>([]);
  const editingRect = $state<any>(null);
  const pan = $state<{ x: number; y: number }>({ x: 0, y: 0 });
  const isPanning = $state(false);
  const panStart = $state<{ x: number; y: number }>({ x: 0, y: 0 });
  const panMode = $state(false);

  const settings = $state({
    rulerIncrement: 50,
    snapToCursor: false,
    snapX: true,
    snapY: true,
    gridOpacity: 0.2,
    showGrid: true,
  });

  let containerEl: HTMLElement;
  let nextId = 0;

  function handlePointerMove(e: PointerEvent) {
    if (!containerEl) return;
    const rect = containerEl.getBoundingClientRect();
    let x = Math.trunc(e.clientX - rect.left - pan.x);
    let y = Math.trunc(e.clientY - rect.top - pan.y);

    if (settings.snapToCursor && !isPanning()) {
      if (settings.snapX) x = Math.trunc(Math.trunc(x / settings.rulerIncrement) * settings.rulerIncrement);
      if (settings.snapY) y = Math.trunc(Math.trunc(y / settings.rulerIncrement) * settings.rulerIncrement);
    }

    pointer.set({ x, y });

    if (isPanning()) {
      const dx = e.clientX - panStart.x;
      const dy = e.clientY - panStart.y;
      pan.set({ x: Math.trunc(pan.x + dx), y: Math.trunc(pan.y + dy) });
      panStart.set({ x: e.clientX, y: e.clientY });
    } else if (isPointerDown() && !panMode()) {
      currentPoint.set({ x, y });
    }
  }

  function handlePointerDown(e: PointerEvent) {
    if (!containerEl) return;
    if (panMode() || e.shiftKey) {
      isPanning.set(true);
      panStart.set({ x: e.clientX, y: e.clientY });
      return;
    }
    const rect = containerEl.getBoundingClientRect();
    let x = Math.trunc(e.clientX - rect.left - pan.x);
    let y = Math.trunc(e.clientY - rect.top - pan.y);

    if (settings.snapToCursor) {
      if (settings.snapX) x = Math.trunc(Math.trunc(x / settings.rulerIncrement) * settings.rulerIncrement);
      if (settings.snapY) y = Math.trunc(Math.trunc(y / settings.rulerIncrement) * settings.rulerIncrement);
    }

    startPoint.set({ x, y });
    currentPoint.set({ x, y });
    isPointerDown.set(true);
  }

  function handlePointerUp() {
    isPointerDown.set(false);
    isPanning.set(false);
  }

  function saveRectangle() {
    const sp = startPoint();
    const cp = currentPoint();
    if (sp && cp) {
      const newRect = {
        id: nextId++,
        x: Math.trunc(Math.min(sp.x, cp.x)),
        y: Math.trunc(Math.min(sp.y, cp.y)),
        width: Math.trunc(Math.abs(cp.x - sp.x)),
        height: Math.trunc(Math.abs(cp.y - sp.y)),
        children: [],
      };
      savedRectangles.set([...savedRectangles(), newRect]);
      startPoint.set(null);
      currentPoint.set(null);
    }
  }

  function deleteRectangle(id: number) {
    savedRectangles.set(savedRectangles().filter(r => r.id !== id));
    if (editingRect()?.id === id) {
      editingRect.set(null);
    }
  }

  function updateRectangle(id: number, updates: any) {
    savedRectangles.set(
      savedRectangles().map(r => (r.id === id ? { ...r, ...updates } : r))
    );
    if (editingRect()?.id === id) {
      editingRect.set({ ...editingRect(), ...updates });
    }
  }

  function addChildRectangle(parentId: number) {
    const parent = savedRectangles().find(r => r.id === parentId);
    if (parent) {
      const child = {
        id: nextId++,
        x: Math.trunc(parent.x + 10),
        y: Math.trunc(parent.y + 10),
        width: Math.trunc(Math.max(50, parent.width - 20)),
        height: Math.trunc(Math.max(50, parent.height - 20)),
        children: [],
      };
      updateRectangle(parentId, { children: [...parent.children, child] });
    }
  }

  function renderGrid() {
    if (!settings.showGrid || !containerEl) return [];
    const w = containerEl.offsetWidth;
    const h = containerEl.offsetHeight;
    const lines: Array<any> = [];

    for (let x = pan.x % settings.rulerIncrement; x < w; x += settings.rulerIncrement) {
      lines.push({ x1: x, y1: 0, x2: x, y2: h });
    }
    for (let y = pan.y % settings.rulerIncrement; y < h; y += settings.rulerIncrement) {
      lines.push({ x1: 0, y1: y, x2: w, y2: y });
    }
    return lines;
  }

  function renderRuler() {
    if (!containerEl) return [];
    const w = containerEl.offsetWidth;
    const markers: Array<any> = [];
    for (let i = 0; i <= w; i += settings.rulerIncrement) {
      const pos = Math.trunc(i - (pan.x % settings.rulerIncrement));
      const value = Math.trunc(i - pan.x);
      if (pos >= 0 && pos <= w) {
        markers.push({ pos, value });
      }
    }
    return markers;
  }

  function renderVerticalRuler() {
    if (!containerEl) return [];
    const h = containerEl.offsetHeight;
    const markers: Array<any> = [];
    for (let i = 0; i <= h; i += settings.rulerIncrement) {
      const pos = Math.trunc(i - (pan.y % settings.rulerIncrement));
      const value = Math.trunc(i - pan.y);
      if (pos >= 0 && pos <= h) {
        markers.push({ pos, value });
      }
    }
    return markers;
  }

  // If you want side‑effects (e.g. event listeners), use $effect
  $effect(() => {
    // Example: attach window-level key handlers
    const onKey = (e: KeyboardEvent) => {
      if (e.key === 's' || e.key === 'S') {
        saveRectangle();
      }
    };
    window.addEventListener('keydown', onKey);
    return () => {
      window.removeEventListener('keydown', onKey);
    };
  });
 /** 
   * Helper to render a rectangle and its children. Svelte doesn't support recursive blocks directly,
   * so one strategy is to render HTML string (dangerous) or break into a separate component.
   * Here is a simplistic “HTML” helper (you could instead make a <Rect> component).
   */
  function renderRectangleHtml(rect: any, isChild = false): string {
    const x = Math.trunc(rect.x + pan().x);
    const y = Math.trunc(rect.y + pan().y);
    const border = isChild ? 'border-yellow-500' : 'border-purple-500';
    const bg = isChild ? 'rgba(234, 179, 8, 0.05)' : 'rgba(168, 85, 247, 0.05)';
    const labelBg = isChild ? 'bg-yellow-600' : 'bg-purple-600';

    let html = `
      <div class="absolute border-2 ${border} pointer-events-none"
        style="left: ${x}px; top: ${y}px; width: ${rect.width}px; height: ${rect.height}px; background: ${bg};">
        <div class="absolute -top-8 left-0 ${labelBg} text-white px-2 py-1 rounded text-xs font-mono flex gap-1 pointer-events-auto">
          <span>${rect.width} × ${rect.height}</span>
          ${!isChild
            ? `<button onclick="window.__svelte_bridge_updateRect(${rect.id})" class="hover:bg-white hover:text-purple-600 rounded px-1">
                <svg><!-- icon markup or component bridging --></svg>
              </button>
              <button onclick="window.__svelte_bridge_deleteRect(${rect.id})" class="hover:bg-white hover:text-red-600 rounded px-1">
                <svg><!-- trash icon --></svg>
              </button>`
            : ''}
        </div>
      </div>
    `;

    if (rect.children) {
      for (const child of rect.children) {
        html += renderRectangleHtml(child, true);
      }
    }
    return html;
  }

  // Bridge functions if using inner HTML buttons to call Svelte state
  // (Better is to use a dedicated <Rect> component rather than inner HTML + onclick strings)
  window.__svelte_bridge_updateRect = (id: number) => {
    editingRect.set(savedRectangles().find(r => r.id === id));
  };
  window.__svelte_bridge_deleteRect = (id: number) => {
    deleteRectangle(id);
  };
</script>

<style>
  /* port over your CSS or Tailwind classes as needed */
</style>

<div class="w-full h-screen bg-gray-900 relative overflow-hidden">
  <!-- Toolbar -->
  <div class="absolute top-4 left-4 z-50 flex gap-2">
    <button
      on:click={() => panMode.set(!panMode())}
      class={`${panMode() ? 'bg-green-600' : 'bg-gray-700'} text-white p-2 rounded-lg hover:bg-green-700 transition-colors`}
      title="Pan Mode (or hold Shift)"
    >
      <Move size={24} />
    </button>
    {#if startPoint() && currentPoint()}
      <button
        on:click={saveRectangle}
        class="bg-green-600 text-white p-2 rounded-lg hover:bg-green-700 transition-colors"
      >
        <Save size={24} />
      </button>
    {/if}
  </div>

  <!-- Settings Button -->
  <button
    on:click={() => showSettings.set(!showSettings())}
    class="absolute top-4 right-4 z-50 bg-blue-600 text-white p-2 rounded-lg hover:bg-blue-700 transition-colors"
  >
    <Settings size={24} />
  </button>

  <!-- Settings Panel -->
  {#if showSettings()}
    <div class="absolute top-4 right-20 z-50 bg-gray-800 text-white p-6 rounded-lg shadow-2xl w-80 border border-gray-700 max-h-[80vh] overflow-y-auto">
      <div class="flex justify-between items-center mb-4">
        <h3 class="text-lg font-bold">Settings</h3>
        <button on:click={() => showSettings.set(false)} class="text-gray-400 hover:text-white">
          <X size={20} />
        </button>
      </div>
      <div class="space-y-4">
        <label class="flex items-center space-x-2">
          <input
            type="checkbox"
            bind:checked={debug()}
            on:change={(e) => debug.set(e.currentTarget.checked)}
            class="w-4 h-4"
          />
          <span>Debug Mode</span>
        </label>

        <label class="flex items-center space-x-2">
          <input
            type="checkbox"
            bind:checked={settings().showGrid}
            on:change={ (e) => settings.set({ ...settings(), showGrid: e.currentTarget.checked }) }
            class="w-4 h-4"
          />
          <span>Show Grid</span>
        </label>

        <div>
          <label class="block mb-2">Grid Opacity</label>
          <input
            type="range"
            min="0"
            max="1"
            step="0.05"
            bind:value={settings().gridOpacity}
            on:change={(e) => settings.set({ ...settings(), gridOpacity: Number(e.currentTarget.value) })}
            class="w-full"
          />
          <span class="text-sm text-gray-400">{(settings().gridOpacity * 100).toFixed(0)}%</span>
        </div>

        <div>
          <label class="block mb-2">Ruler Increment (px)</label>
          <input
            type="number"
            bind:value={settings().rulerIncrement}
            on:change={(e) => settings.set({ ...settings(), rulerIncrement: Number(e.currentTarget.value) })}
            class="w-full bg-gray-700 text-white px-3 py-2 rounded border border-gray-600"
            min="10"
            step="10"
          />
        </div>

        <label class="flex items-center space-x-2">
          <input
            type="checkbox"
            bind:checked={settings().snapToCursor}
            on:change={(e) => settings.set({ ...settings(), snapToCursor: e.currentTarget.checked })}
            class="w-4 h-4"
          />
          <span>Snap to Grid</span>
        </label>

        {#if settings().snapToCursor}
          <div class="ml-6 space-y-2">
            <label class="flex items-center space-x-2">
              <input
                type="checkbox"
                bind:checked={settings().snapX}
                on:change={(e) => settings.set({ ...settings(), snapX: e.currentTarget.checked })}
                class="w-4 h-4"
              />
              <span>Snap X Axis</span>
            </label>
            <label class="flex items-center space-x-2">
              <input
                type="checkbox"
                bind:checked={settings().snapY}
                on:change={(e) => settings.set({ ...settings(), snapY: e.currentTarget.checked })}
                class="w-4 h-4"
              />
              <span>Snap Y Axis</span>
            </label>
          </div>
        {/if}
      </div>
    </div>
  {/if}

  <!-- Edit Panel -->
  {#if editingRect()}
    <div class="absolute top-20 left-1/2 transform -translate-x-1/2 z-50 bg-gray-800 text-white p-6 rounded-lg shadow-2xl w-96 border border-gray-700">
      <div class="flex justify-between items-center mb-4">
        <h3 class="text-lg font-bold">Edit Rectangle #{editingRect().id}</h3>
        <button on:click={() => editingRect.set(null)} class="text-gray-400 hover:text-white">
          <X size={20} />
        </button>
      </div>
      <div class="space-y-3">
        <div class="grid grid-cols-2 gap-2">
          <div>
            <label class="block text-sm mb-1">X Position</label>
            <input
              type="number"
              value={editingRect().x}
              on:change={(e) => updateRectangle(editingRect().id, { x: Math.trunc(Number(e.currentTarget.value)) })}
              class="w-full bg-gray-700 text-white px-2 py-1 rounded border border-gray-600"
            />
          </div>
          <div>
            <label class="block text-sm mb-1">Y Position</label>
            <input
              type="number"
              value={editingRect().y}
              on:change={(e) => updateRectangle(editingRect().id, { y: Math.trunc(Number(e.currentTarget.value)) })}
              class="w-full bg-gray-700 text-white px-2 py-1 rounded border border-gray-600"
            />
          </div>
        </div>

        <div class="grid grid-cols-2 gap-2">
          <div>
            <label class="block text-sm mb-1">Width</label>
            <input
              type="number"
              value={editingRect().width}
              on:change={(e) => updateRectangle(editingRect().id, { width: Math.trunc(Number(e.currentTarget.value)) })}
              class="w-full bg-gray-700 text-white px-2 py-1 rounded border border-gray-600"
            />
          </div>
          <div>
            <label class="block text-sm mb-1">Height</label>
            <input
              type="number"
              value={editingRect().height}
              on:change={(e) => updateRectangle(editingRect().id, { height: Math.trunc(Number(e.currentTarget.value)) })}
              class="w-full bg-gray-700 text-white px-2 py-1 rounded border border-gray-600"
            />
          </div>
        </div>

        <button
          on:click={() => addChildRectangle(editingRect().id)}
          class="w-full bg-yellow-600 text-white px-3 py-2 rounded hover:bg-yellow-700 transition-colors flex items-center justify-center gap-2"
        >
          <Plus size={16} />
          Add Child Rectangle
        </button>

        {#if editingRect().children && editingRect().children.length > 0}
          <div class="mt-4">
            <div class="text-sm font-bold mb-2">Children: {editingRect().children.length}</div>
            <div class="text-xs text-gray-400">Child rectangles are shown in yellow</div>
          </div>
        {/if}
      </div>
    </div>
  {/if}

  <!-- Main Canvas -->
  <div
    bind:this={containerEl}
    class={`w-full h-full relative ${isPanning() || panMode() ? 'cursor-move' : 'cursor-crosshair'}`}
    on:pointermove={handlePointerMove}
    on:pointerdown={handlePointerDown}
    on:pointerup={handlePointerUp}
    on:pointerleave={handlePointerUp}
  >
    {#if settings().showGrid}
      <svg class="absolute top-0 left-0 w-full h-full pointer-events-none" style="z-index:0">
        {#each renderGrid() as line}
          <line
            x1={line.x1} y1={line.y1} x2={line.x2} y2={line.y2}
            stroke="#4b5563" stroke-width="1" opacity={settings().gridOpacity}
          />
        {/each}
      </svg>
    {/if}

    {#if debug()}
      <div
        class="absolute left-0 right-0 h-px bg-blue-400 opacity-50 pointer-events-none"
        style="top: {Math.trunc(pointer().y + pan().y)}px"
      ></div>
      <div
        class="absolute top-0 bottom-0 w-px bg-blue-400 opacity-50 pointer-events-none"
        style="left: {Math.trunc(pointer().x + pan().x)}px"
      ></div>
      <div
        class="absolute bg-blue-600 text-white px-3 py-1 rounded text-sm font-mono pointer-events-none"
        style="left: {Math.trunc(pointer().x + pan().x + 10)}px; top: {Math.trunc(pointer().y + pan().y + 10)}px"
      >
        X: {Math.trunc(pointer().x)}px, Y: {Math.trunc(pointer().y)}px
      </div>
      <div class="absolute top-0 left-0 right-0 h-8 pointer-events-none bg-gray-900 bg-opacity-90" style="z-index:40">
        {#each renderRuler() as mark}
          <div class="absolute top-0" style="left: {mark.pos}px">
            <div class="w-px h-3 bg-blue-500"></div>
            <span class="text-xs text-blue-500 ml-1">{mark.value}</span>
          </div>
        {/each}
      </div>
      <div class="absolute top-0 left-0 bottom-0 w-16 pointer-events-none bg-gray-900 bg-opacity-90" style="z-index:40">
        {#each renderVerticalRuler() as mark}
          <div class="absolute left-0" style="top: {mark.pos}px">
            <div class="h-px w-3 bg-blue-500"></div>
            <span class="text-xs text-blue-500 ml-4" style="margin-top: -8px; display: inline-block">{mark.value}</span>
          </div>
        {/each}
      </div>
    {/if}

    {#each savedRectangles() as rect}
      {@html renderRectangleHtml(rect, false)}
    {/each}

    {#if startPoint() && currentPoint() && !isPanning()}
      <!-- Start marker -->
      <div
        class="absolute w-3 h-3 bg-green-500 rounded-full pointer-events-none transform -translate-x-1/2 -translate-y-1/2"
        style="left: {Math.trunc(startPoint().x + pan().x)}px; top: {Math.trunc(startPoint().y + pan().y)}px"
      ></div>
      {#if isPointerDown()}
        <div
          class="absolute w-3 h-3 bg-red-500 rounded-full pointer-events-none transform -translate-x-1/2 -translate-y-1/2"
          style="left: {Math.trunc(currentPoint().x + pan().x)}px; top: {Math.trunc(currentPoint().y + pan().y)}px"
        ></div>
      {/if}
      <svg
        class="absolute top-0 left-0 w-full h-full pointer-events-none"
        style="overflow: visible"
      >
        <line
          x1={Math.trunc(startPoint().x + pan().x)}
          y1={Math.trunc(startPoint().y + pan().y)}
          x2={Math.trunc(currentPoint().x + pan().x)}
          y2={Math.trunc(currentPoint().y + pan().y)}
          stroke="#f59e0b" stroke-width="2"
          stroke-dasharray={isPointerDown() ? '0' : '5,5'}
        />
      </svg>
      <div
        class="absolute border-2 border-purple-500 pointer-events-none"
        style="
          left: {Math.trunc(Math.min(startPoint().x, currentPoint().x) + pan().x)}px;
          top: {Math.trunc(Math.min(startPoint().y, currentPoint().y) + pan().y)}px;
          width: {Math.trunc(Math.abs(currentPoint().x - startPoint().x))}px;
          height: {Math.trunc(Math.abs(currentPoint().y - startPoint().y))}px;
          background: {isPointerDown() ? 'rgba(168,85,247,0.1)' : 'rgba(168,85,247,0.05)'}
        "
      ></div>
      {#if debug()}
        <div
          class="absolute bg-purple-600 text-white px-2 py-1 rounded text-xs font-mono pointer-events-none"
          style="
            left: {Math.trunc(Math.min(startPoint().x, currentPoint().x) + pan().x)}px;
            top: {Math.trunc(Math.min(startPoint().y, currentPoint().y) + pan().y - 25)}px
          "
        >
          {Math.trunc(Math.abs(currentPoint().x - startPoint().x))} × {Math.trunc(Math.abs(currentPoint().y - startPoint().y))}
        </div>
      {/if}
    {/if}

    <div class="absolute bottom-4 left-4 bg-gray-800 bg-opacity-90 text-white px-4 py-3 rounded-lg text-sm">
      <div class="font-bold mb-2">Instructions:</div>
      <div>• Click and drag to create selection</div>
      <div>• Click Save button or press S to save rectangle</div>
      <div>• Click Edit to modify saved rectangles</div>
      <div>• Hold Shift or enable Pan Mode to pan canvas</div>
      <div>• Add child rectangles from edit panel</div>
    </div>

    {#if debug()}
      <div class="absolute bottom-4 right-4 bg-gray-800 bg-opacity-90 text-white px-3 py-2 rounded text-xs font-mono">
        Pan: {Math.trunc(pan().x)}, {Math.trunc(pan().y)}
      </div>
    {/if}
  </div>
</div>
