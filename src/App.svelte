<script>
  const tileTypes = ['stone', 'gravel', 'grass', 'dirt', 'house'];
  const conditions = ['dry', 'moist'];
  const maxImageBytes = 750 * 1024;
  const premadeFlags = [
    { id: 'sun', name: 'Sun', symbol: '☼', color: '#f0b84b', image: '/flags/sun.svg' },
    { id: 'water', name: 'Water', symbol: '≈', color: '#5c9fc5', image: '/flags/water.svg' },
    { id: 'tree', name: 'Tree', symbol: '✦', color: '#4b8757', image: '/flags/tree.svg' },
    { id: 'bed', name: 'Bed', symbol: '▦', color: '#bd7651', image: '/flags/bed.svg' },
    { id: 'path', name: 'Path', symbol: '↝', color: '#8c8572', image: '/flags/path.svg' },
    { id: 'shed', name: 'Shed', symbol: '⌂', color: '#88634e', image: '/flags/shed.svg' },
    { id: 'compost', name: 'Compost', symbol: '♻', color: '#718c58', image: '/flags/compost.svg' },
    { id: 'vegetable', name: 'Vegetable', symbol: '✿', color: '#6b9b55', image: '/flags/vegetable.svg' },
    { id: 'flower', name: 'Flower', symbol: '❀', color: '#c56b78', image: '/flags/flower.svg' },
    { id: 'fountain', name: 'Fountain', symbol: '♒', color: '#4f91a8', image: '/flags/fountain.svg' },
    { id: 'tools', name: 'Tools', symbol: '⚒', color: '#756b5d', image: '/flags/tools.svg' },
    { id: 'bench', name: 'Bench', symbol: '▰', color: '#9b6b45', image: '/flags/bench.svg' }
  ];

  let setupComplete = false;
  let formError = '';
  let selectedTile = null;
  let selectedTiles = new Set();
  let selecting = false;
  let selectionAnchor = null;
  let dragMoved = false;
  let dragSelecting = true;
  let selectionBase = new Set();
  let selectionMode = false;
  let bulkModal = false;
  let libraryOpen = false;
  let plantInfo = null;
  let editingPlantId = null;
  let gridDragging = false;
  let gridDragMoved = false;
  let suppressTileClick = false;
  let gridPointerTile = null;
  let gridDragStart = { x: 0, y: 0, left: 0, top: 0 };
  let libraryTab = 'flags';
  let importError = '';
  let imageError = '';
  let plantForm = { name: '', matureSize: '', light: 'Full sun', seasonal: 'Perennial', notes: '' };
  let bulkForm = { type: '', condition: '', contents: '', flagId: '' };
  let tileForm = { name: '', type: 'grass', condition: 'dry', contents: '', plantId: '', flagId: '', image: '' };
  let yard = {
    length: 100,
    width: 75,
    conditions: { sunny: false, shaded: false, sloped: false, drainage: false },
    tiles: {},
    customFlags: [],
    plantLibrary: []
  };

  $: columns = setupComplete ? Math.ceil(yard.width / 5) : 0;
  $: rows = setupComplete ? Math.ceil(yard.length / 5) : 0;
  $: tileCount = columns * rows;
  $: markedTiles = Object.keys(yard.tiles).length;
  $: selectedCount = selectedTiles.size;
  $: currentItems = Object.entries(yard.tiles)
    .filter(([, tile]) => tile.contents?.trim())
    .map(([id, tile]) => {
      const [row, column] = id.split('-').map(Number);
      return { id, row, column, contents: tile.contents, name: tile.name };
    });

  function isValidDimension(value) {
    return Number.isInteger(Number(value)) && Number(value) > 0 && Number(value) <= 1000;
  }

  function startPlanning() {
    if (!isValidDimension(yard.length) || !isValidDimension(yard.width)) {
      formError = 'Use a positive whole number of feet between 1 and 1,000.';
      return;
    }

    formError = '';
    yard = { ...yard, length: Number(yard.length), width: Number(yard.width), tiles: {}, customFlags: [], plantLibrary: [] };
    selectedTiles = new Set();
    setupComplete = true;
  }

  function openTile(row, column) {
    const id = `${row}-${column}`;
    if (selectionMode) {
      toggleSelection(id);
      return;
    }
    selectedTiles = new Set();
    selectedTile = id;
    tileForm = yard.tiles[selectedTile]
      ? { ...yard.tiles[selectedTile] }
      : { name: '', type: 'grass', condition: 'dry', contents: '', plantId: '', flagId: '', image: '' };
  }

  function saveTile() {
    const savedTile = tileForm.type === 'house'
      ? { name: tileForm.name, type: 'house', condition: 'dry', contents: '', plantId: '', flagId: '', image: '' }
      : { ...tileForm };
    yard = { ...yard, tiles: { ...yard.tiles, [selectedTile]: savedTile } };
    selectedTile = null;
  }

  function clearTile() {
    const tiles = { ...yard.tiles };
    delete tiles[selectedTile];
    yard = { ...yard, tiles };
    selectedTile = null;
  }

  function toggleSelection(id) {
    const next = new Set(selectedTiles);
    next.has(id) ? next.delete(id) : next.add(id);
    selectedTiles = next;
  }

  function startDragSelection(row, column) {
    if (!selectionMode) return;
    selecting = true;
    dragMoved = false;
    selectionAnchor = { row, column };
    dragSelecting = !selectedTiles.has(`${row}-${column}`);
    selectionBase = new Set(selectedTiles);
  }

  function extendDragSelection(row, column) {
    if (!selecting || !selectionAnchor) return;
    if (selectionAnchor.row === row && selectionAnchor.column === column) return;
    dragMoved = true;
    selectRectangle(row, column);
  }

  function stopDragSelection() {
    selecting = false;
  }

  function startGridDrag(event) {
    if (selectionMode || event.button !== 0) return;
    const viewport = event.currentTarget;
    const tile = event.target.closest?.('.grid-tile');
    gridPointerTile = tile ? { element: tile, row: Number(tile.dataset.row), column: Number(tile.dataset.column) } : null;
    gridDragging = true;
    gridDragMoved = false;
    gridDragStart = { x: event.clientX, y: event.clientY, left: viewport.scrollLeft, top: viewport.scrollTop };
    viewport.setPointerCapture(event.pointerId);
  }

  function startTilePointer(event) {
    if (selectionMode || event.button !== 0) return;
    event.currentTarget.dataset.pointerStartX = event.clientX;
    event.currentTarget.dataset.pointerStartY = event.clientY;
  }

  function finishTilePointer(event) {
    if (!selectionMode && gridDragMoved) event.currentTarget.dataset.dragged = 'true';
  }

  function handleTilePointerUp(event, row, column) {
    if (selectionMode || gridDragMoved) return;
    openTile(row, column);
    event.currentTarget.dataset.handled = 'true';
  }

  function moveGridDrag(event) {
    if (!gridDragging) return;
    const viewport = event.currentTarget;
    const distanceX = event.clientX - gridDragStart.x;
    const distanceY = event.clientY - gridDragStart.y;
    if (Math.abs(distanceX) > 5 || Math.abs(distanceY) > 5) gridDragMoved = true;
    viewport.scrollLeft = gridDragStart.left - (event.clientX - gridDragStart.x);
    viewport.scrollTop = gridDragStart.top - (event.clientY - gridDragStart.y);
  }

  function stopGridDrag(event) {
    suppressTileClick = gridDragMoved;
    if (!selectionMode && !gridDragMoved && gridPointerTile) {
      openTile(gridPointerTile.row, gridPointerTile.column);
      gridPointerTile.element.dataset.handled = 'true';
    }
    gridDragging = false;
    gridPointerTile = null;
    if (event.currentTarget.hasPointerCapture(event.pointerId)) event.currentTarget.releasePointerCapture(event.pointerId);
  }

  function leaveSelectionMode() {
    selectionMode = false;
    selectedTiles = new Set();
  }

  function handleTileSelection(row, column) {
    if (!selectionMode) {
      if (suppressTileClick) {
        suppressTileClick = false;
        return;
      }
      openTile(row, column);
      return;
    }
    if (!dragMoved) toggleSelection(`${row}-${column}`);
    dragMoved = false;
  }

  function handleTileClick(event, row, column) {
    if (event.currentTarget.dataset.handled === 'true') {
      event.currentTarget.dataset.handled = 'false';
      return;
    }
    if (!selectionMode && event.currentTarget.dataset.dragged === 'true') {
      event.currentTarget.dataset.dragged = 'false';
      return;
    }
    handleTileSelection(row, column);
  }

  function selectRectangle(row, column) {
    const next = new Set(selectionBase);
    const minRow = Math.min(selectionAnchor.row, row);
    const maxRow = Math.max(selectionAnchor.row, row);
    const minColumn = Math.min(selectionAnchor.column, column);
    const maxColumn = Math.max(selectionAnchor.column, column);
    for (let currentRow = minRow; currentRow <= maxRow; currentRow += 1) {
      for (let currentColumn = minColumn; currentColumn <= maxColumn; currentColumn += 1) {
        const id = `${currentRow}-${currentColumn}`;
        if (dragSelecting) next.add(id);
        else next.delete(id);
      }
    }
    selectedTiles = next;
  }

  function changeTileType(event) {
    const type = event.currentTarget.value;
    tileForm = type === 'house'
      ? { ...tileForm, type, condition: 'dry', contents: '', plantId: '', flagId: '', image: '' }
      : { ...tileForm, type };
  }

  function handleEscape() {
    if (plantInfo) { plantInfo = null; return; }
    if (bulkModal) { bulkModal = false; selectedTiles = new Set(); return; }
    if (libraryOpen) { libraryOpen = false; selectedTiles = new Set(); return; }
    if (selectedTile !== null) { selectedTile = null; return; }
    if (selectedCount) { selectedTiles = new Set(); return; }
    if (selectionMode) leaveSelectionMode();
  }

  function hasDifferentNeighbor(row, column, direction) {
    const currentType = yard.tiles[`${row}-${column}`]?.type;
    if (!currentType) return false;
    const neighbors = {
      top: [row - 1, column],
      right: [row, column + 1],
      bottom: [row + 1, column],
      left: [row, column - 1]
    };
    const [neighborRow, neighborColumn] = neighbors[direction];
    const neighborType = yard.tiles[`${neighborRow}-${neighborColumn}`]?.type;
    return Boolean(neighborType && neighborType !== currentType);
  }

  function applyToSelection() {
    if (!selectedCount) return;
    bulkForm = { type: '', condition: '', contents: '', flagId: '' };
    bulkModal = true;
  }

  function saveBulkChanges() {
    const tiles = { ...yard.tiles };
    selectedTiles.forEach((id) => {
      const current = tiles[id] || { type: 'grass', condition: 'dry', contents: '', flagId: '', image: '' };
      tiles[id] = {
        ...current,
        ...(bulkForm.type ? { type: bulkForm.type } : {}),
        ...(bulkForm.condition ? { condition: bulkForm.condition } : {}),
        ...(bulkForm.contents ? { contents: bulkForm.contents } : {}),
        ...(bulkForm.flagId ? { flagId: bulkForm.flagId } : {})
      };
      if (bulkForm.type === 'house') tiles[id] = { name: tiles[id].name || '', type: 'house', condition: 'dry', contents: '', plantId: '', flagId: '', image: '' };
    });
    yard = { ...yard, tiles };
    selectedTiles = new Set();
    bulkModal = false;
  }

  function clearSelection() {
    const tiles = { ...yard.tiles };
    selectedTiles.forEach((id) => delete tiles[id]);
    yard = { ...yard, tiles };
    selectedTiles = new Set();
  }

  function openPlantInfo(plantId) {
    plantInfo = (yard.plantLibrary || []).find((plant) => plant.id === plantId) || null;
  }

  function choosePlant(plantId) {
    const plant = (yard.plantLibrary || []).find((item) => item.id === plantId);
    tileForm = { ...tileForm, plantId, contents: plant?.name || '' };
  }

  function exportGarden() {
    const blob = new Blob([JSON.stringify(yard, null, 2)], { type: 'application/json' });
    const link = document.createElement('a');
    link.href = URL.createObjectURL(blob);
    link.download = 'backyard-plan.json';
    link.click();
    URL.revokeObjectURL(link.href);
  }

  function importGarden(event) {
    const file = event.target.files[0];
    if (!file) return;
    const reader = new FileReader();
    reader.onload = () => {
      try {
        const imported = JSON.parse(reader.result);
        if (!imported.length || !imported.width || !imported.tiles) throw new Error('Invalid garden plan');
        yard = imported;
        setupComplete = true;
        importError = '';
      } catch (error) {
        importError = 'That file is not a valid backyard plan.';
      }
    };
    reader.readAsText(file);
    event.target.value = '';
  }

  function addCustomFlag(event) {
    const file = event.target.files[0];
    if (!file) return;
    if (!file.type.startsWith('image/') || file.size > maxImageBytes) {
      imageError = 'Choose an image under 750 KB.';
      event.target.value = '';
      return;
    }
    const reader = new FileReader();
    reader.onload = () => {
      const flag = { id: `custom-${Date.now()}`, name: file.name.replace(/\.[^/.]+$/, ''), symbol: '', color: '', image: reader.result };
      yard = { ...yard, customFlags: [...(yard.customFlags || []), flag] };
      imageError = '';
    };
    reader.readAsDataURL(file);
    event.target.value = '';
  }

  function flagById(id) {
    return [...premadeFlags, ...(yard.customFlags || [])].find((flag) => flag.id === id);
  }

  function savePlant() {
    if (!plantForm.name.trim()) return;
    const plant = { ...plantForm, id: editingPlantId || `plant-${Date.now()}` };
    yard = { ...yard, plantLibrary: editingPlantId ? (yard.plantLibrary || []).map((item) => item.id === editingPlantId ? plant : item) : [...(yard.plantLibrary || []), plant] };
    plantForm = { name: '', matureSize: '', light: 'Full sun', seasonal: 'Perennial', notes: '' };
    editingPlantId = null;
  }

  function editPlant(plant) {
    plantForm = { name: plant.name, matureSize: plant.matureSize || '', light: plant.light, seasonal: plant.seasonal, notes: plant.notes || '' };
    editingPlantId = plant.id;
  }

  function cancelPlantEdit() {
    plantForm = { name: '', matureSize: '', light: 'Full sun', seasonal: 'Perennial', notes: '' };
    editingPlantId = null;
  }

  function removePlant(id) {
    yard = { ...yard, plantLibrary: (yard.plantLibrary || []).filter((plant) => plant.id !== id) };
  }

  function openLibraryItem(item) {
    libraryOpen = false;
    selectedTiles = new Set();
    openTile(item.row, item.column);
  }

  function tileLabel(row, column) {
    return yard.tiles[`${row}-${column}`]?.type || 'Unmarked';
  }
</script>

<svelte:head>
  <title>{setupComplete ? 'Map your backyard' : 'Backyard planner'}</title>
</svelte:head>

<svelte:window on:keydown={(event) => event.key === 'Escape' && handleEscape()} />

<main class="page-shell">
  <header class="topbar">
    <div class="brand-mark">BP</div>
    <div>
      <p class="eyebrow">Outdoor planning tool</p>
      <h1>Backyard planner</h1>
    </div>
    {#if setupComplete}
      <div class="progress"><strong>{markedTiles}</strong> / {tileCount} tiles marked</div>
    {/if}
  </header>

  {#if !setupComplete}
    <section class="intro-panel">
      <div class="intro-copy">
        <p class="eyebrow">Start with the shape of the space</p>
        <h2>Build a clear picture of your backyard.</h2>
        <p>Set the dimensions and note the conditions that will guide your garden decisions.</p>
      </div>

      <form on:submit|preventDefault={startPlanning} class="setup-form">
        <div class="dimension-grid">
          <label>Length <span>feet</span><input type="number" min="5" max="1000" step="5" bind:value={yard.length} required /></label>
          <label>Width <span>feet</span><input type="number" min="5" max="1000" step="5" bind:value={yard.width} required /></label>
        </div>
        <fieldset>
          <legend>General conditions</legend>
          <div class="flag-grid">
            <label class="check-option"><input type="checkbox" bind:checked={yard.conditions.sunny} /> Mostly sunny</label>
            <label class="check-option"><input type="checkbox" bind:checked={yard.conditions.shaded} /> Mostly shaded</label>
            <label class="check-option"><input type="checkbox" bind:checked={yard.conditions.sloped} /> Sloped ground</label>
            <label class="check-option"><input type="checkbox" bind:checked={yard.conditions.drainage} /> Drainage concerns</label>
          </div>
        </fieldset>
        {#if formError}<p class="form-error">{formError}</p>{/if}
        <button class="primary-button" type="submit">Open backyard map <span aria-hidden="true">→</span></button>
        <p class="form-note">Dimensions must be positive whole feet between 1 and 1,000. Tiles represent 5 × 5 feet.</p>
      </form>
    </section>
  {:else}
    <section class="planner-layout">
      <aside class="yard-summary">
        <p class="eyebrow">Your backyard</p>
        <h2>{yard.length} × {yard.width}<span> ft</span></h2>
        <dl>
          <div><dt>Grid size</dt><dd>5 × 5 ft</dd></div>
          <div><dt>Rows</dt><dd>{rows}</dd></div>
          <div><dt>Columns</dt><dd>{columns}</dd></div>
        </dl>
        <div class="condition-list">
          <p class="eyebrow">Conditions</p>
          <div class="area-conditions">
            <label class="check-option"><input type="checkbox" bind:checked={yard.conditions.sunny} /> Sunny</label>
            <label class="check-option"><input type="checkbox" bind:checked={yard.conditions.shaded} /> Shaded</label>
            <label class="check-option"><input type="checkbox" bind:checked={yard.conditions.sloped} /> Sloped</label>
            <label class="check-option"><input type="checkbox" bind:checked={yard.conditions.drainage} /> Drainage concerns</label>
          </div>
        </div>
        <button class="text-button" on:click={() => (setupComplete = false)}>← Edit dimensions</button>
      </aside>

      <div class="map-area">
        <div class="map-heading"><div><p class="eyebrow">{selectionMode ? 'Click or drag across tiles' : 'Click any tile to add details'}</p><h2>Map your space</h2></div><span class="scale-key"><i></i> 5 ft</span></div>
        <div class="map-tools">
          <button class:active={selectionMode} class="tool-button" on:click={() => selectionMode ? leaveSelectionMode() : (selectionMode = true)}>{selectionMode ? 'Selecting tiles' : 'Select multiple'}</button>
          {#if selectionMode && selectedCount}<button class="tool-button" on:click={applyToSelection}>Apply details ({selectedCount})</button><button class="tool-button quiet" on:click={clearSelection}>Clear selected</button>{/if}
          <button class="tool-button quiet" on:click={() => { selectedTiles = new Set(); libraryTab = 'plants'; libraryOpen = true; }}>Plant library</button>
          <button class="tool-button quiet" on:click={exportGarden}>Export plan</button>
          <label class="tool-button quiet file-button">Import plan<input type="file" accept="application/json" on:change={importGarden} /></label>
        </div>
        {#if importError}<p class="form-error">{importError}</p>{/if}
        <div class:grid-dragging={gridDragging} class="map-scroll" role="region" aria-label="Scrollable backyard grid" on:pointerdown={startGridDrag} on:pointermove={moveGridDrag} on:pointerup={stopGridDrag} on:pointercancel={stopGridDrag}>
          <div class="yard-grid" style={`--columns: ${columns}; --rows: ${rows}`} aria-label="Backyard map">
            {#each Array(tileCount) as _, index}
              {@const row = Math.floor(index / columns)}
              {@const column = index % columns}
              {@const tile = yard.tiles[`${row}-${column}`]}
              <button data-row={row} data-column={column} class:boundary-top={hasDifferentNeighbor(row, column, 'top')} class:boundary-right={hasDifferentNeighbor(row, column, 'right')} class:boundary-bottom={hasDifferentNeighbor(row, column, 'bottom')} class:boundary-left={hasDifferentNeighbor(row, column, 'left')} class:marked={tile} class:selected={selectedTiles.has(`${row}-${column}`)} class="grid-tile {tile?.type || ''}" on:pointerdown={startTilePointer} on:pointerup={(event) => { finishTilePointer(event); handleTilePointerUp(event, row, column); }} on:click={(event) => handleTileClick(event, row, column)} on:mousedown={() => startDragSelection(row, column)} on:mouseenter={() => extendDragSelection(row, column)} on:mouseup={stopDragSelection} aria-label={`Tile row ${row + 1}, column ${column + 1}: ${tileLabel(row, column)}`}>
                {#if tile?.image || flagById(tile?.flagId)?.image}<img class="tile-image" src={tile.image || flagById(tile.flagId).image} alt="" />{:else if tile?.flagId}<span class="tile-flag" style={`--flag-color: ${flagById(tile.flagId)?.color}`}>{flagById(tile.flagId)?.symbol}</span>{/if}
              </button>
            {/each}
          </div>
        </div>
      </div>
    </section>
  {/if}
</main>

{#if selectedTile !== null}
  <div class="modal-backdrop" role="presentation" on:click={(event) => event.target === event.currentTarget && (selectedTile = null)}>
    <div class="tile-modal" role="dialog" aria-modal="true" aria-labelledby="tile-title" tabindex="-1">
      <div class="modal-heading"><div><p class="eyebrow">Tile details</p><h2 id="tile-title">{tileForm.name || 'Unnamed tile'}</h2></div><button class="close-button" aria-label="Close tile details" on:click={() => (selectedTile = null)}>×</button></div>
      <label>Tile name <span class="label-help">Used as this tile's heading</span><input maxlength="60" bind:value={tileForm.name} placeholder="e.g. North herb bed" /></label>
      <label>Tile type<select value={tileForm.type} on:change={changeTileType}>{#each tileTypes as type}<option value={type}>{type[0].toUpperCase() + type.slice(1)}</option>{/each}</select></label>
      {#if tileForm.type === 'house'}<p class="house-note">House tiles mark the boundary and cannot store additional tile information.</p>{/if}
      <label>Condition<select bind:value={tileForm.condition}>{#each conditions as condition}<option value={condition}>{condition[0].toUpperCase() + condition.slice(1)}</option>{/each}</select></label>
      <label>Contents <span class="label-help">Choose a catalogue plant or type freely</span><select value={tileForm.plantId} on:change={(event) => choosePlant(event.currentTarget.value)}><option value="">Manual entry</option>{#each yard.plantLibrary || [] as plant}<option value={plant.id}>{plant.name}</option>{/each}</select><textarea maxlength="160" rows="3" bind:value={tileForm.contents} on:input={() => (tileForm = { ...tileForm, plantId: '' })} placeholder="e.g. raised bed, apple tree"></textarea>{#if tileForm.plantId}<button class="plant-link" type="button" on:click={() => openPlantInfo(tileForm.plantId)}>{tileForm.contents} · view plant info</button>{/if}</label>
      <div class="flag-library"><div class="library-heading"><span class="field-label">Tile flag</span></div>
        <div class="flag-picker">{#each [...premadeFlags, ...(yard.customFlags || [])] as flag}<button class:selected={tileForm.flagId === flag.id} class="flag-option" on:click={() => (tileForm = { ...tileForm, flagId: flag.id })}>{#if flag.image}<img src={flag.image} alt="" />{:else}<span style={`color: ${flag.color}`}>{flag.symbol}</span>{/if}<small>{flag.name}</small></button>{/each}</div>
        <label class="upload-label">Add custom flag <input type="file" accept="image/png,image/jpeg,image/webp" on:change={addCustomFlag} /></label>
        {#if imageError}<p class="form-error">{imageError}</p>{/if}
      </div>
      <div class="modal-actions"><button class="text-button danger" on:click={clearTile}>Clear tile</button><button class="primary-button" on:click={saveTile}>Save details</button></div>
    </div>
  </div>
{/if}

{#if bulkModal}
  <div class="modal-backdrop" role="presentation" on:click={(event) => event.target === event.currentTarget && (bulkModal = false)}>
    <div class="tile-modal" role="dialog" aria-modal="true" aria-labelledby="bulk-title" tabindex="-1">
      <div class="modal-heading"><div><p class="eyebrow">{selectedCount} selected tiles</p><h2 id="bulk-title">Change group</h2></div><button class="close-button" aria-label="Close group editor" on:click={() => (bulkModal = false)}>×</button></div>
      <p class="library-note">Only fields you change will be applied. Existing differences remain where a field is kept.</p>
      <label>Tile type<select bind:value={bulkForm.type}><option value="">Keep each tile's type</option>{#each tileTypes as type}<option value={type}>{type[0].toUpperCase() + type.slice(1)}</option>{/each}</select></label>
      <label>Condition<select bind:value={bulkForm.condition}><option value="">Keep each tile's condition</option>{#each conditions as condition}<option value={condition}>{condition[0].toUpperCase() + condition.slice(1)}</option>{/each}</select></label>
      <label>Contents <span class="label-help">Leave empty to keep each tile's contents</span><input maxlength="160" bind:value={bulkForm.contents} placeholder="Apply the same plant or item" /></label>
      <label>Flag<select bind:value={bulkForm.flagId}><option value="">Keep each tile's flag</option>{#each [...premadeFlags, ...(yard.customFlags || [])] as flag}<option value={flag.id}>{flag.name}</option>{/each}</select></label>
      <div class="modal-actions"><button class="text-button" on:click={() => (bulkModal = false)}>Cancel</button><button class="primary-button" on:click={saveBulkChanges}>Apply changes</button></div>
    </div>
  </div>
{/if}

{#if libraryOpen}
  <div class="modal-backdrop" role="presentation" on:click={(event) => event.target === event.currentTarget && (libraryOpen = false)}>
    <div class="tile-modal library-modal" role="dialog" aria-modal="true" aria-labelledby="library-title" tabindex="-1">
      <div class="modal-heading"><div><p class="eyebrow">Garden records</p><h2 id="library-title">Plant library</h2></div><button class="close-button" aria-label="Close plant library" on:click={() => (libraryOpen = false)}>×</button></div>
      <nav class="library-tabs" aria-label="Garden library sections">
        <button class:active={libraryTab === 'flags'} type="button" on:click={() => (libraryTab = 'flags')}>Flags ({(yard.customFlags || []).length})</button>
        <button class:active={libraryTab === 'plants'} type="button" on:click={() => (libraryTab = 'plants')}>Plants ({(yard.plantLibrary || []).length})</button>
        <button class:active={libraryTab === 'items'} type="button" on:click={() => (libraryTab = 'items')}>Items ({currentItems.length})</button>
      </nav>
      {#if libraryTab === 'flags'}
        <div class="library-records flag-records">{#if (yard.customFlags || []).length}{#each yard.customFlags as flag}<article class="library-record"><div class="library-record-icon">{#if flag.image}<img src={flag.image} alt="" />{:else}<span style={`color: ${flag.color}`}>{flag.symbol}</span>{/if}</div><div><strong>{flag.name}</strong><p>Custom flag available in tile details</p></div></article>{/each}{:else}<p class="library-note">Your custom flag library is empty.</p>{/if}</div>
      {:else if libraryTab === 'plants'}
        <form class="plant-form" on:submit|preventDefault={savePlant}>
          <label>Plant name<input maxlength="80" bind:value={plantForm.name} placeholder="e.g. Lavender" required /></label>
          <div class="dimension-grid"><label>Mature size<input maxlength="60" bind:value={plantForm.matureSize} placeholder="e.g. 2 × 2 ft" /></label><label>Light<select bind:value={plantForm.light}><option>Full sun</option><option>Partial shade</option><option>Full shade</option></select></label></div>
          <label>Seasonal behavior<select bind:value={plantForm.seasonal}><option>Annual</option><option>Perennial</option><option>Evergreen</option><option>Deciduous</option><option>Cool-season</option><option>Warm-season</option></select></label>
          <label>Notes<textarea maxlength="500" rows="2" bind:value={plantForm.notes} placeholder="Care, spacing, bloom time, or source"></textarea></label>
          <div class="form-actions"><button class="primary-button" type="submit">{editingPlantId ? 'Save plant' : 'Add plant'}</button>{#if editingPlantId}<button class="text-button" type="button" on:click={cancelPlantEdit}>Cancel edit</button>{/if}</div>
        </form>
        <div class="catalogue"><p class="eyebrow">Catalogue ({(yard.plantLibrary || []).length})</p>{#if (yard.plantLibrary || []).length}{#each yard.plantLibrary as plant}<article class="plant-record"><div><strong>{plant.name}</strong><p>{plant.matureSize || 'Size not recorded'} · {plant.light} · {plant.seasonal}</p>{#if plant.notes}<small>{plant.notes}</small>{/if}</div><div class="record-actions"><button class="text-button" on:click={() => editPlant(plant)}>Edit</button><button class="text-button danger" on:click={() => removePlant(plant.id)}>Remove</button></div></article>{/each}{:else}<p class="library-note">Your plant catalogue is empty.</p>{/if}</div>
      {:else}
        <div class="library-records">{#if currentItems.length}{#each currentItems as item}<button class="library-record item-record" type="button" on:click={() => openLibraryItem(item)}><span><strong>{item.contents}</strong>{#if item.name}<small>{item.name}</small>{/if}</span><small>Row {item.row + 1}, column {item.column + 1}</small></button>{/each}{:else}<p class="library-note">Your garden has no current plant or item entries.</p>{/if}</div>
      {/if}
    </div>
  </div>
{/if}

{#if plantInfo}
  <div class="modal-backdrop" role="presentation" on:click={(event) => event.target === event.currentTarget && (plantInfo = null)}>
    <div class="tile-modal plant-info-modal" role="dialog" aria-modal="true" aria-labelledby="plant-info-title" tabindex="-1">
      <div class="modal-heading"><div><p class="eyebrow">Plant record</p><h2 id="plant-info-title">{plantInfo.name}</h2></div><button class="close-button" aria-label="Close plant information" on:click={() => (plantInfo = null)}>×</button></div>
      <dl class="plant-details"><div><dt>Mature size</dt><dd>{plantInfo.matureSize || 'Not recorded'}</dd></div><div><dt>Light</dt><dd>{plantInfo.light}</dd></div><div><dt>Seasonal behavior</dt><dd>{plantInfo.seasonal}</dd></div></dl>
      {#if plantInfo.notes}<p class="plant-notes">{plantInfo.notes}</p>{/if}
    </div>
  </div>
{/if}
