<script>
  import { onMount } from 'svelte';
  
  let logs = $state([
    { type: 'info', content: 'Console ready', timestamp: new Date() }
  ]);
  let input = $state('');
  let history = $state([]);
  let historyIndex = $state(-1);
  let inputRef = $state(null);
  let logsEndRef = $state(null);
  let showSettings = $state(false);
  
  // Color schemes
  const defaultThemes = {
    dark: {
      background: '#030712',
      containerBg: '#111827',
      borderColor: '#1f2937',
      textPrimary: '#f3f4f6',
      textSecondary: '#9ca3af',
      textTertiary: '#6b7280',
      stringColor: '#fb923c',
      numberColor: '#60a5fa',
      booleanColor: '#c084fc',
      nullColor: '#9ca3af',
      functionColor: '#9ca3af',
      errorBg: 'rgba(127, 29, 29, 0.3)',
      errorText: '#fca5a5',
      errorBorder: '#ef4444',
      warnBg: 'rgba(113, 63, 18, 0.3)',
      warnText: '#fcd34d',
      warnBorder: '#eab308',
      infoBg: 'rgba(30, 58, 138, 0.3)',
      infoText: '#93c5fd',
      infoBorder: '#3b82f6',
      resultText: '#86efac',
      commandBg: 'rgba(17, 24, 39, 0.5)',
      accentColor: '#60a5fa'
    },
    light: {
      background: '#ffffff',
      containerBg: '#f9fafb',
      borderColor: '#e5e7eb',
      textPrimary: '#111827',
      textSecondary: '#4b5563',
      textTertiary: '#9ca3af',
      stringColor: '#ea580c',
      numberColor: '#2563eb',
      booleanColor: '#9333ea',
      nullColor: '#6b7280',
      functionColor: '#6b7280',
      errorBg: 'rgba(254, 202, 202, 0.5)',
      errorText: '#dc2626',
      errorBorder: '#ef4444',
      warnBg: 'rgba(254, 240, 138, 0.5)',
      warnText: '#ca8a04',
      warnBorder: '#eab308',
      infoBg: 'rgba(191, 219, 254, 0.5)',
      infoText: '#2563eb',
      infoBorder: '#3b82f6',
      resultText: '#16a34a',
      commandBg: 'rgba(243, 244, 246, 0.8)',
      accentColor: '#2563eb'
    }
  };
  
  let theme = $state(defaultThemes.dark);
  let currentThemeName = $state('dark');
  
  // Store original console methods
  let originalConsole = $state({});
  
  onMount(() => {
    // Load theme from localStorage
    const savedTheme = localStorage.getItem('consoleTheme');
    const savedThemeName = localStorage.getItem('consoleThemeName');
    if (savedTheme) {
      theme = JSON.parse(savedTheme);
      currentThemeName = savedThemeName || 'custom';
    }
    
    // Intercept console methods
    originalConsole = {
      log: console.log,
      warn: console.warn,
      error: console.error,
      info: console.info,
      clear: console.clear
    };
    
    console.log = (...args) => {
      originalConsole.log(...args);
      addLog('log', args.length === 1 ? args[0] : args);
    };
    
    console.warn = (...args) => {
      originalConsole.warn(...args);
      addLog('warn', args.length === 1 ? args[0] : args);
    };
    
    console.error = (...args) => {
      originalConsole.error(...args);
      addLog('error', args.length === 1 ? args[0] : args);
    };
    
    console.info = (...args) => {
      originalConsole.info(...args);
      addLog('info', args.length === 1 ? args[0] : args);
    };
    
    console.clear = () => {
      originalConsole.clear();
      clearConsole();
    };
    
    // Smooth scroll to bottom on new logs
    $effect(() => {
      if (logsEndRef) {
        logsEndRef.scrollIntoView({ behavior: 'smooth', block: 'end' });
      }
    });
    
    return () => {
      // Restore original console methods
      console.log = originalConsole.log;
      console.warn = originalConsole.warn;
      console.error = originalConsole.error;
      console.info = originalConsole.info;
      console.clear = originalConsole.clear;
    };
  });
  
  function addLog(type, content) {
    logs = [...logs, { type, content, timestamp: new Date() }];
  }
  
  function formatValue(value, depth = 0) {
    if (depth > 3) return '...';
    
    if (value === null) return { type: 'null', value: 'null' };
    if (value === undefined) return { type: 'undefined', value: 'undefined' };
    if (typeof value === 'string') return { type: 'string', value };
    if (typeof value === 'number') return { type: 'number', value };
    if (typeof value === 'boolean') return { type: 'boolean', value: String(value) };
    if (typeof value === 'function') return { type: 'function', value: `ƒ ${value.name}()` };
    
    if (Array.isArray(value)) {
      return {
        type: 'array',
        length: value.length,
        items: value.map((item, idx) => ({ key: idx, value: formatValue(item, depth + 1) }))
      };
    }
    
    if (typeof value === 'object') {
      const entries = Object.entries(value);
      return {
        type: 'object',
        entries: entries.map(([key, val]) => ({ key, value: formatValue(val, depth + 1) }))
      };
    }
    
    return { type: 'string', value: String(value) };
  }
  
  function executeCommand(cmd) {
    if (!cmd.trim()) return;
    
    addLog('command', cmd);
    history = [...history, cmd];
    historyIndex = -1;
    
    try {
      // Handle clear
      if (cmd === 'clear()' || cmd === 'console.clear()') {
        clearConsole();
        return;
      }
      
      // Evaluate expression
      const result = eval(cmd);
      addLog('result', result);
    } catch (error) {
      addLog('error', error.message);
    }
  }
  
  function handleKeyDown(e) {
    if (e.key === 'Enter') {
      executeCommand(input);
      input = '';
    } else if (e.key === 'ArrowUp') {
      e.preventDefault();
      if (history.length > 0) {
        const newIndex = historyIndex === -1 ? history.length - 1 : Math.max(0, historyIndex - 1);
        historyIndex = newIndex;
        input = history[newIndex];
      }
    } else if (e.key === 'ArrowDown') {
      e.preventDefault();
      if (historyIndex > -1) {
        const newIndex = historyIndex + 1;
        if (newIndex >= history.length) {
          historyIndex = -1;
          input = '';
        } else {
          historyIndex = newIndex;
          input = history[newIndex];
        }
      }
    }
  }
  
  function clearConsole() {
    logs = [];
  }
  
  function applyTheme(themeName) {
    theme = { ...defaultThemes[themeName] };
    currentThemeName = themeName;
    saveTheme();
  }
  
  function saveTheme() {
    localStorage.setItem('consoleTheme', JSON.stringify(theme));
    localStorage.setItem('consoleThemeName', currentThemeName);
  }
  
  function updateThemeColor(key, value) {
    theme[key] = value;
    currentThemeName = 'custom';
    saveTheme();
  }
  
  function getLogStyle(type) {
    const styles = {
      error: {
        color: theme.errorText,
        background: theme.errorBg,
        borderLeft: `2px solid ${theme.errorBorder}`
      },
      warn: {
        color: theme.warnText,
        background: theme.warnBg,
        borderLeft: `2px solid ${theme.warnBorder}`
      },
      warning: {
        color: theme.warnText,
        background: theme.warnBg,
        borderLeft: `2px solid ${theme.warnBorder}`
      },
      info: {
        color: theme.infoText,
        background: theme.infoBg,
        borderLeft: `2px solid ${theme.infoBorder}`
      },
      command: {
        color: theme.textPrimary,
        background: theme.commandBg,
        borderLeft: 'none'
      },
      result: {
        color: theme.resultText,
        background: 'transparent',
        borderLeft: 'none'
      },
      log: {
        color: theme.resultText,
        background: 'transparent',
        borderLeft: 'none'
      }
    };
    
    return styles[type] || { color: theme.textPrimary, background: 'transparent', borderLeft: 'none' };
  }
</script>

<div class="h-full console-container w-full" style="background: {theme.background}; color: {theme.textPrimary};">
  <!-- Header -->
  <div class="header" style="background: {theme.containerBg}; border-bottom: 1px solid {theme.borderColor};">
    <div class="header-left">
      <svg class="icon" style="color: {theme.accentColor};" fill="none" stroke="currentColor" viewBox="0 0 24 24">
        <rect x="2" y="3" width="20" height="14" rx="2" stroke-width="2"/>
        <path d="M8 21h8M12 17v4" stroke-width="2"/>
      </svg>
      <span class="header-title">Console</span>
    </div>
    <div class="header-right">
      <button
        onclick={() => showSettings = !showSettings}
        class="btn"
        style="background: {theme.containerBg}; border: 1px solid {theme.borderColor}; color: {theme.textPrimary};"
        title="Settings"
      >
        <svg class="icon-sm" fill="none" stroke="currentColor" viewBox="0 0 24 24">
          <circle cx="12" cy="12" r="3" stroke-width="2"/>
          <path d="M12 1v6m0 6v6M23 12h-6m-6 0H1" stroke-width="2"/>
        </svg>
      </button>
      <button
        onclick={clearConsole}
        class="btn"
        style="background: {theme.containerBg}; border: 1px solid {theme.borderColor}; color: {theme.textPrimary};"
      >
        Clear
      </button>
    </div>
  </div>
  
  <!-- Settings Panel -->
  {#if showSettings}
    <div class="settings-panel" style="background: {theme.containerBg}; border-bottom: 1px solid {theme.borderColor}; color: {theme.textPrimary};">
      <div class="settings-header">
        <h3 style="color: {theme.textPrimary};">Theme Settings</h3>
        <button onclick={() => showSettings = false} class="close-btn" style="color: {theme.textSecondary};">×</button>
      </div>
      
      <div class="theme-presets">
        <button
          onclick={() => applyTheme('dark')}
          class="preset-btn"
          class:active={currentThemeName === 'dark'}
          style="background: {currentThemeName === 'dark' ? theme.accentColor : theme.background}; border: 1px solid {theme.borderColor}; color: {currentThemeName === 'dark' ? '#fff' : theme.textPrimary};"
        >
          Dark
        </button>
        <button
          onclick={() => applyTheme('light')}
          class="preset-btn"
          class:active={currentThemeName === 'light'}
          style="background: {currentThemeName === 'light' ? theme.accentColor : theme.background}; border: 1px solid {theme.borderColor}; color: {currentThemeName === 'light' ? '#fff' : theme.textPrimary};"
        >
          Light
        </button>
      </div>
      
      <div class="color-grid">
        <div class="color-item">
          <label style="color: {theme.textSecondary};">Background</label>
          <input type="color" value={theme.background} oninput={(e) => updateThemeColor('background', e.target.value)} />
        </div>
        <div class="color-item">
          <label style="color: {theme.textSecondary};">Container</label>
          <input type="color" value={theme.containerBg} oninput={(e) => updateThemeColor('containerBg', e.target.value)} />
        </div>
        <div class="color-item">
          <label style="color: {theme.textSecondary};">Text</label>
          <input type="color" value={theme.textPrimary} oninput={(e) => updateThemeColor('textPrimary', e.target.value)} />
        </div>
        <div class="color-item">
          <label style="color: {theme.textSecondary};">Accent</label>
          <input type="color" value={theme.accentColor} oninput={(e) => updateThemeColor('accentColor', e.target.value)} />
        </div>
        <div class="color-item">
          <label style="color: {theme.textSecondary};">String</label>
          <input type="color" value={theme.stringColor} oninput={(e) => updateThemeColor('stringColor', e.target.value)} />
        </div>
        <div class="color-item">
          <label style="color: {theme.textSecondary};">Number</label>
          <input type="color" value={theme.numberColor} oninput={(e) => updateThemeColor('numberColor', e.target.value)} />
        </div>
        <div class="color-item">
          <label style="color: {theme.textSecondary};">Error</label>
          <input type="color" value={theme.errorText} oninput={(e) => updateThemeColor('errorText', e.target.value)} />
        </div>
        <div class="color-item">
          <label style="color: {theme.textSecondary};">Warning</label>
          <input type="color" value={theme.warnText} oninput={(e) => updateThemeColor('warnText', e.target.value)} />
        </div>
      </div>
    </div>
  {/if}
  
  <!-- Logs Area -->
  <div class="logs-area">
    {#each logs as log, idx}
      <div class="log-entry first:border-b-0 border-b-2 border-slate-400/20 border-dashed" style="color: {getLogStyle(log.type).color}; background: {getLogStyle(log.type).background}; border-left: {getLogStyle(log.type).borderLeft};">
        <!-- Icon -->
        {#if log.type === 'error'}
          <svg class="log-icon" style="color: {theme.errorBorder};" fill="none" stroke="currentColor" viewBox="0 0 24 24">
            <path d="M18 6L6 18M6 6l12 12" stroke-width="2"/>
          </svg>
        {:else if log.type === 'warn' || log.type === 'warning'}
          <svg class="log-icon" style="color: {theme.warnBorder};" fill="none" stroke="currentColor" viewBox="0 0 24 24">
            <path d="M10.29 3.86L1.82 18a2 2 0 001.71 3h16.94a2 2 0 001.71-3L13.71 3.86a2 2 0 00-3.42 0z" stroke-width="2"/>
            <path d="M12 9v4M12 17h.01" stroke-width="2"/>
          </svg>
        {:else if log.type === 'info'}
          <svg class="log-icon" style="color: {theme.infoBorder};" fill="none" stroke="currentColor" viewBox="0 0 24 24">
            <circle cx="12" cy="12" r="10" stroke-width="2"/>
            <path d="M12 16v-4M12 8h.01" stroke-width="2"/>
          </svg>
        {:else}
          <svg class="log-icon" style="color: {theme.textTertiary};" fill="none" stroke="currentColor" viewBox="0 0 24 24">
            <path d="M9 5l7 7-7 7" stroke-width="2"/>
          </svg>
        {/if}
        
        <div class="log-content">
          {#if log.type === 'command'}
            <span style="color: {theme.textTertiary};">&gt; </span>{log.content}
          {:else if typeof log.content === 'string'}
            {log.content}
          {:else}
            {@render valueDisplay(formatValue(log.content))}
          {/if}
        </div>
        
        <span class="log-timestamp" style="color: {theme.textTertiary};">
          {log.timestamp.toLocaleTimeString()}
        </span>
      </div>
    {/each}
    <div bind:this={logsEndRef}></div>
  </div>
  
  <!-- Input Area -->
  <div class="input-area" style="border-top: 1px solid {theme.borderColor}; background: {theme.containerBg};">
    <div class="input-wrapper">
      <svg class="icon" style="color: {theme.accentColor};" fill="none" stroke="currentColor" viewBox="0 0 24 24">
        <path d="M9 5l7 7-7 7" stroke-width="2"/>
      </svg>
      <input
        bind:this={inputRef}
        bind:value={input}
        onkeydown={handleKeyDown}
        type="text"
        placeholder="Enter JavaScript expression..."
        class="input-field"
        style="color: {theme.textPrimary}; background: transparent;"
        autofocus
      />
    </div>
  </div>
  
  <!-- Helper Text -->
  <div class="helper-text" style="color: {theme.textTertiary}; background: {theme.containerBg}; border-top: 1px solid {theme.borderColor};">
    Try: <code style="color: {theme.accentColor};">console.log("hello")</code>,{' '}
    <code style="color: {theme.accentColor};">{'{ name: "Alice" }'}</code>,{' '}
    <code style="color: {theme.accentColor};">[1, 2, 3]</code>
  </div>
</div>

{#snippet valueDisplay(formatted)}
  {#if formatted.type === 'null'}
    <span style="color: {theme.nullColor};">{formatted.value}</span>
  {:else if formatted.type === 'undefined'}
    <span style="color: {theme.nullColor};">{formatted.value}</span>
  {:else if formatted.type === 'string'}
    <span style="color: {theme.stringColor};">"{formatted.value}"</span>
  {:else if formatted.type === 'number'}
    <span style="color: {theme.numberColor};">{formatted.value}</span>
  {:else if formatted.type === 'boolean'}
    <span style="color: {theme.booleanColor};">{formatted.value}</span>
  {:else if formatted.type === 'function'}
    <span style="color: {theme.functionColor};">{formatted.value}</span>
  {:else if formatted.type === 'array'}
    <details class="value-details">
      <summary style="color: {theme.textSecondary};">
        Array({formatted.length}) [{formatted.length > 0 ? '…' : ''}]
      </summary>
      <div class="nested-content">
        {#each formatted.items as item}
          <div>
            <span style="color: {theme.textTertiary};">{item.key}: </span>
            {@render valueDisplay(item.value)}
          </div>
        {/each}
      </div>
    </details>
  {:else if formatted.type === 'object'}
    <details class="value-details">
      <summary style="color: {theme.textSecondary};">
        Object {'{'}…{'}'}
      </summary>
      <div class="nested-content">
        {#each formatted.entries as entry}
          <div>
            <span style="color: {theme.booleanColor};">{entry.key}</span>
            <span style="color: {theme.textTertiary};">: </span>
            {@render valueDisplay(entry.value)}
          </div>
        {/each}
      </div>
    </details>
  {:else}
    {String(formatted.value)}
  {/if}
{/snippet}

<style>
  .console-container {
    display: flex;
    flex-direction: column;
    font-family: 'Consolas', 'Monaco', monospace;
    font-size: 0.875rem;
  }
  
  .header {
    display: flex;
    align-items: center;
    justify-content: space-between;
    padding: 0.5rem 1rem;
  }
  
  .header-left, .header-right {
    display: flex;
    align-items: center;
    gap: 0.5rem;
  }
  
  .header-title {
    font-weight: 600;
  }
  
  .icon {
    width: 1.25rem;
    height: 1.25rem;
  }
  
  .icon-sm {
    width: 1rem;
    height: 1rem;
  }
  
  .btn {
    padding: 0.25rem 0.75rem;
    font-size: 0.75rem;
    border-radius: 0.25rem;
    cursor: pointer;
    transition: opacity 0.2s;
  }
  
  .btn:hover {
    opacity: 0.8;
  }
  
  .settings-panel {
    padding: 1rem;
    max-height: 400px;
    overflow-y: auto;
  }
  
  .settings-header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    margin-bottom: 1rem;
  }
  
  .settings-header h3 {
    margin: 0;
    font-size: 1rem;
  }
  
  .close-btn {
    background: none;
    border: none;
    font-size: 2rem;
    cursor: pointer;
    line-height: 1;
  }
  
  .theme-presets {
    display: flex;
    gap: 0.5rem;
    margin-bottom: 1rem;
  }
  
  .preset-btn {
    padding: 0.5rem 1rem;
    border-radius: 0.25rem;
    cursor: pointer;
    transition: all 0.2s;
    font-size: 0.875rem;
  }
  
  .color-grid {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(120px, 1fr));
    gap: 1rem;
  }
  
  .color-item {
    display: flex;
    flex-direction: column;
    gap: 0.25rem;
  }
  
  .color-item label {
    font-size: 0.75rem;
  }
  
  .color-item input[type="color"] {
    width: 100%;
    height: 2rem;
    border: none;
    border-radius: 0.25rem;
    cursor: pointer;
  }
  
  .logs-area {
    flex: 1;
    overflow-y: auto;
    padding: 0.5rem;
    display: flex;
    flex-direction: column;
    gap: 0.25rem;
    scroll-behavior: smooth;
  }
  
  .log-entry {
    display: flex;
    align-items: center;
    gap: 0.5rem;
    padding: 0.5rem;
    border-radius: 0.25rem;
  }
  
  .log-icon {
    width: 1rem;
    height: 1rem;
    flex-shrink: 0;
    margin-top: 0.125rem;
  }
  
  .log-content {
    flex: 1;
    word-break: break-all;
  }
  
  .log-timestamp {
    font-size: 0.75rem;
    white-space: nowrap;
  }
  
  .input-area {
    padding: 0;
  }
  
  .input-wrapper {
    display: flex;
    align-items: center;
    padding: 0.5rem 1rem;
    gap: 0.5rem;
  }
  
  .input-field {
    flex: 1;
    border: none;
    outline: none;
    font-family: inherit;
    font-size: inherit;
  }
  
  .input-field::placeholder {
    opacity: 0.5;
  }
  
  .helper-text {
    padding: 0.5rem 1rem;
    font-size: 0.75rem;
  }
  
  .helper-text code {
    padding: 0.125rem 0.25rem;
    border-radius: 0.125rem;
  }
  
  .value-details {
    display: inline;
  }
  
  .value-details summary {
    cursor: pointer;
    list-style: none;
    display: inline;
  }
  
  .value-details summary::-webkit-details-marker {
    display: none;
  }
  
  .value-details summary::before {
    content: '▶';
    display: inline-block;
    margin-right: 0.25rem;
    transition: transform 0.2s;
    font-size: 0.75rem;
  }
  
  .value-details[open] summary::before {
    transform: rotate(90deg);
  }
  
  .value-details summary:hover {
    opacity: 0.8;
  }
  
  .nested-content {
    margin-left: 1rem;
    margin-top: 0.25rem;
    display: flex;
    flex-direction: column;
    gap: 0.25rem;
  }
</style>