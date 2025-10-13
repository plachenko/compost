<script>
  let { type, size = 16 } = $props();
  
  const icons = {
    splitH: `<rect width="18" height="18" x="3" y="3" rx="2"/><path d="M12 3v18"/>`,
    splitV: `<rect width="18" height="18" x="3" y="3" rx="2"/><path d="M3 12h18"/>`,
    mergeUp: `<path d="m18 15-6-6-6 6"/>`,
    mergeDown: `<path d="m6 9 6 6 6-6"/>`,
    mergeLeft: `<path d="m15 18-6-6 6-6"/>`,
    mergeRight: `<path d="m9 18 6-6-6-6"/>`,
    popOut: `<path d="M21 13v6a2 2 0 0 1-2 2H5a2 2 0 0 1-2-2V5a2 2 0 0 1 2-2h6"/><path d="m21 3-9 9"/><path d="M15 3h6v6"/>`
  };
</script>

<svg 
  xmlns="http://www.w3.org/2000/svg" 
  width={size} 
  height={size} 
  viewBox="0 0 24 24" 
  fill="none" 
  stroke="currentColor" 
  stroke-width="2" 
  stroke-linecap="round" 
  stroke-linejoin="round"
>
  {@html icons[type]}
</svg>