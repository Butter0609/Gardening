<script>
  const tileTypes = ['stone', 'gravel', 'grass', 'dirt'];
  const conditions = ['dry', 'moist'];

  let setupComplete = false;
  let formError = '';
  let selectedTile = null;
  let tileForm = { type: 'grass', condition: 'dry', contents: '' };
  let yard = {
    length: 100,
    width: 75,
    conditions: { sunny: false, shaded: false, sloped: false, drainage: false },
    tiles: {}
  };

  $: columns = setupComplete ? Math.ceil(yard.length / 5) : 0;
  $: rows = setupComplete ? Math.ceil(yard.width / 5) : 0;
  $: tileCount = columns * rows;
  $: markedTiles = Object.keys(yard.tiles).length;

  function isValidDimension(value) {
    return Number.isInteger(Number(value)) && Number(value) > 0 && Number(value) <= 1000;
  }

  function startPlanning() {
    if (!isValidDimension(yard.length) || !isValidDimension(yard.width)) {
      formError = 'Use a positive whole number of feet between 1 and 1,000.';
      return;
    }

    formError = '';
    yard = { ...yard, length: Number(yard.length), width: Number(yard.width), tiles: {} };
    setupComplete = true;
  }

  function openTile(row, column) {
    selectedTile = `${row}-${column}`;
    tileForm = yard.tiles[selectedTile]
      ? { ...yard.tiles[selectedTile] }
      : { type: 'grass', condition: 'dry', contents: '' };
  }

  function saveTile() {
    yard = { ...yard, tiles: { ...yard.tiles, [selectedTile]: { ...tileForm } } };
    selectedTile = null;
  }

  function clearTile() {
    const tiles = { ...yard.tiles };
    delete tiles[selectedTile];
    yard = { ...yard, tiles };
    selectedTile = null;
  }

  function tileLabel(row, column) {
    return yard.tiles[`${row}-${column}`]?.type || 'Unmarked';
  }
</script>

<svelte:head>
  <title>{setupComplete ? 'Map your backyard' : 'Backyard planner'}</title>
</svelte:head>

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
          {#each Object.entries(yard.conditions) as [condition, active]}
            {#if active}<span>{condition}</span>{/if}
          {/each}
          {#if !Object.values(yard.conditions).some(Boolean)}<span>None recorded</span>{/if}
        </div>
        <button class="text-button" on:click={() => (setupComplete = false)}>← Edit dimensions</button>
      </aside>

      <div class="map-area">
        <div class="map-heading"><div><p class="eyebrow">Click any tile to add details</p><h2>Map your space</h2></div><span class="scale-key"><i></i> 5 ft</span></div>
        <div class="map-scroll">
          <div class="yard-grid" style={`--columns: ${columns}; --rows: ${rows}`} aria-label="Backyard map">
            {#each Array(tileCount) as _, index}
              {@const row = Math.floor(index / columns)}
              {@const column = index % columns}
              {@const tile = yard.tiles[`${row}-${column}`]}
              <button class:marked={tile} class="grid-tile {tile?.type || ''}" on:click={() => openTile(row, column)} aria-label={`Tile row ${row + 1}, column ${column + 1}: ${tileLabel(row, column)}`}>
                {#if tile}<span>{tile.type}</span>{/if}
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
    <section class="tile-modal" role="dialog" aria-modal="true" aria-labelledby="tile-title">
      <div class="modal-heading"><div><p class="eyebrow">Tile details</p><h2 id="tile-title">5 × 5 foot area</h2></div><button class="close-button" aria-label="Close tile details" on:click={() => (selectedTile = null)}>×</button></div>
      <label>Tile type<select bind:value={tileForm.type}>{#each tileTypes as type}<option value={type}>{type[0].toUpperCase() + type.slice(1)}</option>{/each}</select></label>
      <label>Condition<select bind:value={tileForm.condition}>{#each conditions as condition}<option value={condition}>{condition[0].toUpperCase() + condition.slice(1)}</option>{/each}</select></label>
      <label>Contents <span class="label-help">plants, structures, or notable objects</span><textarea rows="3" bind:value={tileForm.contents} placeholder="e.g. raised bed, apple tree"></textarea></label>
      <div class="modal-actions"><button class="text-button danger" on:click={clearTile}>Clear tile</button><button class="primary-button" on:click={saveTile}>Save details</button></div>
    </section>
  </div>
{/if}
