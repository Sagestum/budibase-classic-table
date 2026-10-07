<script>
  import { getContext, onDestroy, tick } from "svelte"
  import Cell from "./Cell.svelte"
  import RowContext from "./RowContext.svelte"

  export let dataProvider
  export let columns
  export let rowCount
  export let quiet
  export let size
  export let allowSelectRows
  export let selectedText
  export let compact
  export let onClick
  export let noRowsMessage
  export let showSearch = true
  export let searchColumns
  export let searchPlaceholder
  export let searchAlign
  export let allowResize = true
  export let showTooltip = true

  const component = getContext("component")
  const context = getContext("context")
  const { styleable, Provider, ActionTypes, rowSelectionStore } =
    getContext("sdk")

  const HEADER_HEIGHT = 36
  const SEARCH_DEBOUNCE = 300
  const TOOLTIP_DELAY = 600
  const TOOLTIP_GAP = 6
  const MIN_COLUMN_WIDTH = 40
  const WIDTHS_KEY = `classic-table-widths-${$component.id}`
  const RULE_COUNT = 5
  const CUSTOM_COLUMN = `custom-${Math.random()}`
  const SELECT_COLUMN = `select-${Math.random()}`
  const TEXT_TYPES = ["string", "longform", "options", "formula", "barcodeqr"]
  const NUMBER_TYPES = ["number", "bigint"]
  const UNSORTABLE_TYPES = [
    "link",
    "attachment",
    "attachment_single",
    "signature_single",
    "json",
    "bb_reference",
    "bb_reference_single",
    "ai",
  ]

  let selectedRows = []
  let sortColumn
  let sortOrder
  let searchTerm = ""
  let searchTimeout
  let extensionProviderId
  let columnWidths = {}
  let resizing
  let tooltip
  let tooltipElement
  let tooltipTarget
  let tooltipTimeout

  // Column widths set by dragging are remembered per browser
  try {
    columnWidths = JSON.parse(localStorage.getItem(WIDTHS_KEY)) || {}
  } catch (error) {
    columnWidths = {}
  }

  $: snippets = $context.snippets
  $: hasChildren = $component.children
  $: data = dataProvider?.rows || []
  $: fullSchema = dataProvider?.schema ?? {}
  $: primaryDisplay = dataProvider?.primaryDisplay
  $: fieldList = getFields(fullSchema, columns, primaryDisplay)
  $: schema = getFilteredSchema(fullSchema, fieldList, hasChildren)
  $: fields = Object.keys(schema)
  $: rows = fields.length ? data : []
  $: canSelectRows =
    allowSelectRows &&
    ["table", "viewV2"].includes(dataProvider?.datasource?.type)
  $: rowHeight = compact ? 46 : 55
  $: resizedWidths = allowResize !== false ? columnWidths : {}
  $: hasResized = fields.some(field => resizedWidths[field])
  $: gridStyle = getGridStyle(fields, schema, canSelectRows, resizedWidths)
  $: heightStyle = getHeightStyle(rows.length, rowCount, rowHeight)
  $: cellStyles = computeCellStyles(schema)
  $: rules = getRules($$props)
  $: ruleStyles = computeRuleStyles(rules, rows, fields)
  $: allSelected =
    rows.length > 0 && rows.every(row => isSelected(selectedRows, row))
  $: searchFields = getSearchFields(fullSchema, searchColumns)
  $: applySearch($context, dataProvider?.id, showSearch, searchFields, false)
  $: {
    rowSelectionStore.actions.updateSelection(
      $component.id,
      selectedRows.length ? selectedRows[0].tableId : "",
      selectedRows.map(row => row._id)
    )
  }

  // If the data changes, double check that the selected rows still exist
  $: if (data) {
    const rowIds = data.map(row => row._id)
    if (rowIds.length) {
      selectedRows = selectedRows.filter(row => rowIds.includes(row._id))
    }
  }

  $: dataContext = { selectedRows }

  const actions = [
    {
      type: ActionTypes.ClearRowSelection,
      callback: () => (selectedRows = []),
    },
  ]

  const getProviderAction = (providerId, type) =>
    providerId ? $context[`${providerId}_${type}`] : null

  const getFields = (schema, customColumns, primaryDisplay) => {
    if (customColumns?.length) {
      return customColumns
    }

    // Otherwise generate columns: normal columns first, then auto columns
    let normalColumns = []
    let autoColumns = []
    Object.entries(schema).forEach(([field, fieldSchema]) => {
      if (fieldSchema?.visible === false) {
        return
      }
      if (fieldSchema?.autocolumn) {
        autoColumns.push(field)
      } else {
        normalColumns.push(field)
      }
    })
    const byOrder = (a, b) => {
      if (a === primaryDisplay) {
        return -1
      }
      if (b === primaryDisplay) {
        return 1
      }
      const aOrder = schema[a].order
      const bOrder = schema[b].order
      if (aOrder === bOrder) {
        return 0
      }
      if (aOrder == null) {
        return 1
      }
      if (bOrder == null) {
        return -1
      }
      return aOrder < bOrder ? -1 : 1
    }
    return normalColumns.sort(byOrder).concat(autoColumns.sort(byOrder))
  }

  const canBeSortColumn = fieldSchema => {
    if (UNSORTABLE_TYPES.includes(fieldSchema?.type)) {
      return false
    }
    return fieldSchema?.type !== "formula" || fieldSchema.formulaType === "static"
  }

  const getFilteredSchema = (schema, fields, hasChildren) => {
    let newSchema = {}
    if (hasChildren) {
      newSchema[CUSTOM_COLUMN] = {
        name: CUSTOM_COLUMN,
        displayName: null,
        sortable: false,
        divider: true,
        width: "auto",
        custom: true,
      }
    }
    fields.forEach(field => {
      const columnName = typeof field === "string" ? field : field.name
      if (!schema[columnName]) {
        return
      }
      newSchema[columnName] = {
        ...schema[columnName],
        // Additional column settings like width, alignment or template
        ...(typeof field === "object" ? field : {}),
        name: columnName,
      }
      if (!canBeSortColumn(schema[columnName])) {
        newSchema[columnName].sortable = false
      }

      // Numeric only widths are grid widths and should be ignored
      const width = newSchema[columnName].width
      if (width != null && `${width}`.trim().match(/^[0-9]+$/)) {
        delete newSchema[columnName].width
      }
    })
    return newSchema
  }

  const getDisplayName = fieldSchema =>
    (fieldSchema.displayName === undefined
      ? fieldSchema.name
      : fieldSchema.displayName) || ""

  const getGridStyle = (fields, schema, canSelectRows, resizedWidths) => {
    let style = "grid-template-columns:"
    if (canSelectRows) {
      style += " auto"
    }
    fields.forEach(field => {
      const width = schema[field].width
      if (resizedWidths[field]) {
        style += ` ${resizedWidths[field]}px`
      } else {
        style += width && typeof width === "string" ? ` ${width}` : " minmax(auto, 1fr)"
      }
    })
    return `${style};`
  }

  const getHeightStyle = (totalRowCount, rowCount, rowHeight) => {
    if (!rowCount || totalRowCount <= rowCount) {
      return ""
    }
    return `height: ${HEADER_HEIGHT + rowCount * rowHeight}px;`
  }

  const computeCellStyles = schema => {
    let styles = {}
    Object.keys(schema).forEach(field => {
      const fieldSchema = schema[field]
      let style = ""
      if (fieldSchema.color) {
        style += `color: ${fieldSchema.color};`
      }
      if (fieldSchema.background) {
        style += `background-color: ${fieldSchema.background};`
      }
      if (fieldSchema.align === "Center") {
        style += "justify-content: center; text-align: center;"
      }
      if (fieldSchema.align === "Right") {
        style += "justify-content: flex-end; text-align: right;"
      }
      if (fieldSchema.borderLeft) {
        style += "border-left: 1px solid var(--spectrum-global-color-gray-200);"
      }
      if (fieldSchema.borderRight) {
        style += "border-right: 1px solid var(--spectrum-global-color-gray-200);"
      }
      if (fieldSchema.minWidth) {
        style += `min-width: ${fieldSchema.minWidth};`
      }
      styles[field] = style
    })
    return styles
  }

  const deepGet = (row, path) => {
    if (row == null || path in row) {
      return row?.[path]
    }
    return path.split(".").reduce((value, key) => value?.[key], row)
  }

  // The color rules are numbered settings (rule1Field, rule1Operator, ...)
  const getRules = props => {
    let rules = []
    for (let i = 1; i <= RULE_COUNT; i++) {
      const rule = {
        field: props[`rule${i}Field`],
        operator: props[`rule${i}Operator`],
        value: props[`rule${i}Value`],
        background: props[`rule${i}Background`],
        color: props[`rule${i}Color`],
        scope: props[`rule${i}Scope`],
      }
      if (rule.field && (rule.background || rule.color)) {
        rules.push(rule)
      }
    }
    return rules
  }

  const toText = value => {
    if (value == null) {
      return ""
    }
    if (Array.isArray(value)) {
      return value.map(toText).join(", ")
    }
    if (typeof value === "object") {
      return `${value.primaryDisplay ?? value.name ?? JSON.stringify(value)}`
    }
    return `${value}`
  }

  const toNumber = text => (text === "" ? NaN : Number(text.replace(",", ".")))

  // Numbers are compared as numbers, everything else as text ignoring case
  const compare = (a, b) => {
    const numberA = toNumber(a)
    const numberB = toNumber(b)
    if (!isNaN(numberA) && !isNaN(numberB)) {
      return numberA - numberB
    }
    return a === b ? 0 : a < b ? -1 : 1
  }

  const matchesRule = (rule, value) => {
    const text = toText(value).trim().toLowerCase()
    const expected = toText(rule.value).trim().toLowerCase()
    switch (rule.operator) {
      case "notEqual":
        return compare(text, expected) !== 0
      case "contains":
        return text.includes(expected)
      case "notContains":
        return !text.includes(expected)
      case "greater":
        return text !== "" && compare(text, expected) > 0
      case "less":
        return text !== "" && compare(text, expected) < 0
      case "empty":
        return text === ""
      case "notEmpty":
        return text !== ""
      default:
        return compare(text, expected) === 0
    }
  }

  // Returns the rule colors of every row as styles by column. Earlier rules
  // win over later ones, separately for background and text color.
  const computeRuleStyles = (rules, rows, fields) => {
    if (!rules.length) {
      return []
    }
    const colors = rows.map(() => ({}))
    rules.forEach(rule => {
      const matches = rows.map(row => matchesRule(rule, deepGet(row, rule.field)))
      const anyMatch = matches.includes(true)
      const columns =
        rule.scope === "row" ? [SELECT_COLUMN, ...fields] : [rule.field]
      rows.forEach((row, idx) => {
        // A column is colored as a whole as soon as one of its cells matches
        if (rule.scope === "column" ? !anyMatch : !matches[idx]) {
          return
        }
        columns.forEach(column => {
          const cell = (colors[idx][column] ??= {})
          cell.background ||= rule.background
          cell.color ||= rule.color
        })
      })
    })
    return colors.map(row => {
      let styles = {}
      Object.entries(row).forEach(([column, cell]) => {
        styles[column] =
          (cell.background ? `background-color: ${cell.background};` : "") +
          (cell.color ? `color: ${cell.color};` : "")
      })
      return styles
    })
  }

  // Sorting is delegated to the data provider so it covers all pages
  const sortBy = fieldSchema => {
    if (fieldSchema.sortable === false) {
      return
    }
    if (fieldSchema.name === sortColumn) {
      sortOrder = sortOrder === "Descending" ? "Ascending" : "Descending"
    } else {
      sortColumn = fieldSchema.name
      sortOrder = "Descending"
    }
    const setSorting = getProviderAction(
      dataProvider?.id,
      ActionTypes.SetDataProviderSorting
    )
    setSorting?.({ column: sortColumn, order: sortOrder })
  }

  const isSelected = (selectedRows, row) =>
    selectedRows.some(selectedRow => selectedRow._id === row._id)

  const toggleSelectRow = row => {
    if (!canSelectRows) {
      return
    }
    if (isSelected(selectedRows, row)) {
      selectedRows = selectedRows.filter(
        selectedRow => selectedRow._id !== row._id
      )
    } else {
      selectedRows = [...selectedRows, row]
    }
  }

  const toggleSelectAll = () => {
    if (allSelected) {
      selectedRows = selectedRows.filter(
        selectedRow => !rows.some(row => row._id === selectedRow._id)
      )
    } else {
      selectedRows = [
        ...selectedRows,
        ...rows.filter(row => !isSelected(selectedRows, row)),
      ]
    }
  }

  const clickRow = row => {
    if (onClick) {
      onClick({ row })
    }
    toggleSelectRow(row)
  }

  const getSearchFields = (schema, searchColumns) => {
    const searchable = field =>
      TEXT_TYPES.includes(schema[field]?.type) ||
      NUMBER_TYPES.includes(schema[field]?.type)
    if (searchColumns?.length) {
      return searchColumns.filter(searchable)
    }
    return Object.keys(schema).filter(field =>
      TEXT_TYPES.includes(schema[field]?.type)
    )
  }

  // Builds a query matching rows where any of the search fields matches
  const buildSearchQuery = (term, fields) => {
    let conditions = []
    fields.forEach(field => {
      if (NUMBER_TYPES.includes(fullSchema[field]?.type)) {
        const number = Number(term.replace(",", "."))
        if (!isNaN(number)) {
          conditions.push({ equal: { [field]: number } })
        }
      } else {
        conditions.push({ fuzzy: { [field]: term } })
      }
    })
    if (!conditions.length) {
      // Nothing can match this term, so make sure no rows are returned
      conditions.push({ equal: { _id: "no-search-match" } })
    }
    return { $or: { conditions } }
  }

  const removeSearch = () => {
    getProviderAction(
      extensionProviderId,
      ActionTypes.RemoveDataProviderQueryExtension
    )?.($component.id)
    extensionProviderId = null
  }

  // The search is added to the data provider's query, so it searches all rows
  // of the data source and not just the rows that are currently loaded
  let appliedSearch
  const applySearch = (_context, providerId, showSearch, fields, force) => {
    const term = showSearch ? searchTerm.trim() : ""
    const searchKey = JSON.stringify([providerId, term, term ? fields : []])
    if (searchKey === appliedSearch && !force) {
      return
    }
    const addExtension = getProviderAction(
      providerId,
      ActionTypes.AddDataProviderQueryExtension
    )
    if (term && !addExtension) {
      // The data provider has not registered its actions yet
      return
    }
    if (extensionProviderId && (!term || extensionProviderId !== providerId)) {
      removeSearch()
    }
    if (term) {
      addExtension($component.id, buildSearchQuery(term, fields))
      extensionProviderId = providerId
    }
    appliedSearch = searchKey
  }

  const onSearchInput = () => {
    clearTimeout(searchTimeout)
    searchTimeout = setTimeout(
      () => applySearch($context, dataProvider?.id, showSearch, searchFields),
      SEARCH_DEBOUNCE
    )
  }

  const onSearchKeydown = e => {
    if (e.key === "Enter") {
      clearTimeout(searchTimeout)
      applySearch($context, dataProvider?.id, showSearch, searchFields)
    } else if (e.key === "Escape") {
      clearSearch()
    }
  }

  const clearSearch = () => {
    clearTimeout(searchTimeout)
    searchTerm = ""
    applySearch($context, dataProvider?.id, showSearch, searchFields)
  }

  const saveColumnWidths = () => {
    try {
      localStorage.setItem(WIDTHS_KEY, JSON.stringify(columnWidths))
    } catch (error) {
      // Without storage the widths just last until the page is left
    }
  }

  const startResize = (e, field) => {
    if (e.button !== 0) {
      return
    }
    hideTooltip()
    e.currentTarget.setPointerCapture(e.pointerId)
    resizing = {
      field,
      startX: e.clientX,
      startWidth: e.currentTarget.parentElement.getBoundingClientRect().width,
    }
  }

  const resize = e => {
    if (!resizing) {
      return
    }
    const width = Math.round(resizing.startWidth + e.clientX - resizing.startX)
    columnWidths = {
      ...columnWidths,
      [resizing.field]: Math.max(MIN_COLUMN_WIDTH, width),
    }
  }

  const stopResize = () => {
    if (!resizing) {
      return
    }
    resizing = null
    saveColumnWidths()
  }

  // A double click on the handle gives the column its original width back
  const resetWidth = field => {
    columnWidths = { ...columnWidths }
    delete columnWidths[field]
    saveColumnWidths()
  }

  const hideTooltip = () => {
    clearTimeout(tooltipTimeout)
    tooltipTarget = null
    tooltip = null
  }

  // Shows the full content of a cell after hovering a truncated text for a while
  const onMouseOver = e => {
    const target =
      showTooltip !== false && !resizing
        ? e.target.closest?.("[data-overflow-tip]")
        : null
    if (target === tooltipTarget) {
      return
    }
    hideTooltip()
    tooltipTarget = target
    if (target) {
      tooltipTimeout = setTimeout(() => openTooltip(target), TOOLTIP_DELAY)
    }
  }

  const openTooltip = async target => {
    if (!target.isConnected || target.scrollWidth <= target.clientWidth) {
      return
    }
    tooltip = { text: target.textContent.trim(), style: "visibility: hidden;" }
    await tick()
    if (!tooltipElement || tooltipTarget !== target) {
      return
    }
    const cell = target.getBoundingClientRect()
    const tip = tooltipElement.getBoundingClientRect()
    let left = Math.min(cell.left, window.innerWidth - tip.width - TOOLTIP_GAP)
    let top = cell.bottom + TOOLTIP_GAP
    const above = cell.top - TOOLTIP_GAP - tip.height
    if (top + tip.height > window.innerHeight - TOOLTIP_GAP && above >= 0) {
      top = above
    }
    // A transformed ancestor moves the origin of fixed elements, the measured
    // position of the still unplaced tooltip tells by how much
    left = Math.max(TOOLTIP_GAP, left) - tip.left
    top -= tip.top
    tooltip = { ...tooltip, style: `left: ${left}px; top: ${top}px;` }
  }

  onDestroy(() => {
    clearTimeout(searchTimeout)
    clearTimeout(tooltipTimeout)
    removeSearch()
    rowSelectionStore.actions.updateSelection($component.id, "", [])
  })
</script>

<svelte:window on:scroll|capture={hideTooltip} />

<div use:styleable={$component.styles} class="classic-table {size || ''}">
  <Provider {actions} data={dataContext}>
    {#if showSearch}
      <div class="search-bar" class:search-bar--right={searchAlign === "right"}>
        <div class="search">
          <svg class="search-icon" viewBox="0 0 24 24" aria-hidden="true">
            <circle cx="10.5" cy="10.5" r="6.5" />
            <line x1="15.5" y1="15.5" x2="21" y2="21" />
          </svg>
          <input
            type="text"
            bind:value={searchTerm}
            placeholder={searchPlaceholder || ""}
            aria-label={searchPlaceholder || "Search"}
            on:input={onSearchInput}
            on:keydown={onSearchKeydown}
          />
          {#if searchTerm}
            <button
              type="button"
              class="search-clear"
              aria-label="Clear search"
              on:click={clearSearch}
            >
              <svg viewBox="0 0 24 24" aria-hidden="true">
                <line x1="6" y1="6" x2="18" y2="18" />
                <line x1="18" y1="6" x2="6" y2="18" />
              </svg>
            </button>
          {/if}
        </div>
      </div>
    {/if}
    <!-- svelte-ignore a11y-no-static-element-interactions -->
    <!-- svelte-ignore a11y-click-events-have-key-events -->
    <!-- svelte-ignore a11y-mouse-events-have-key-events -->
    <div
      class="wrapper"
      class:wrapper--quiet={quiet}
      class:wrapper--compact={compact}
      style="--row-height: {rowHeight}px; --header-height: {HEADER_HEIGHT}px;"
    >
      <div
        class="spectrum-Table"
        class:no-scroll={!rowCount}
        class:has-resized={hasResized}
        style="{heightStyle}{gridStyle}"
        on:mouseover={onMouseOver}
        on:mouseleave={hideTooltip}
        on:mousedown={hideTooltip}
      >
        {#if fields.length}
          <div class="spectrum-Table-head">
            {#if canSelectRows}
              <div
                class="spectrum-Table-headCell spectrum-Table-headCell--divider spectrum-Table-headCell--edit"
              >
                <input
                  type="checkbox"
                  class="select"
                  checked={allSelected}
                  on:change={toggleSelectAll}
                />
              </div>
            {/if}
            {#each fields as field}
              <div
                class="spectrum-Table-headCell"
                class:spectrum-Table-headCell--alignCenter={schema[field]
                  .align === "Center"}
                class:spectrum-Table-headCell--alignRight={schema[field]
                  .align === "Right"}
                class:is-sortable={schema[field].sortable !== false}
                on:click={() => sortBy(schema[field])}
              >
                <div class="title" title={schema[field].custom ? "" : field}>
                  <span class="title-text">{getDisplayName(schema[field])}</span>
                  {#if sortColumn === field}
                    <svg
                      class="sort-icon"
                      class:sort-icon--asc={sortOrder === "Ascending"}
                      viewBox="0 0 24 24"
                      aria-hidden="true"
                    >
                      <polyline points="6 9 12 15 18 9" />
                    </svg>
                  {/if}
                </div>
                {#if allowResize !== false && !schema[field].custom}
                  <div
                    class="resize-handle"
                    class:is-active={resizing?.field === field}
                    on:pointerdown|stopPropagation={e => startResize(e, field)}
                    on:pointermove={resize}
                    on:pointerup={stopResize}
                    on:pointercancel={stopResize}
                    on:click|stopPropagation
                    on:dblclick|stopPropagation={() => resetWidth(field)}
                  ></div>
                {/if}
              </div>
            {/each}
          </div>
        {/if}
        {#if rows.length}
          {#each rows as row, idx}
            <div class="spectrum-Table-row clickable">
              {#if canSelectRows}
                <div
                  class="spectrum-Table-cell spectrum-Table-cell--divider spectrum-Table-cell--edit"
                  style={ruleStyles[idx]?.[SELECT_COLUMN]}
                  on:click|stopPropagation={() => toggleSelectRow(row)}
                >
                  <input
                    type="checkbox"
                    class="select"
                    checked={isSelected(selectedRows, row)}
                  />
                </div>
              {/if}
              {#each fields as field}
                {#if schema[field].custom}
                  <div
                    class="spectrum-Table-cell spectrum-Table-cell--divider"
                    style="{cellStyles[field]}{ruleStyles[idx]?.[field] || ''}"
                  >
                    <RowContext {row}>
                      <slot />
                    </RowContext>
                  </div>
                {:else}
                  <div
                    class="spectrum-Table-cell"
                    class:is-resized={resizedWidths[field]}
                    style="{cellStyles[field]}{ruleStyles[idx]?.[field] || ''}"
                    on:click={() => clickRow(row)}
                  >
                    <Cell
                      schema={schema[field]}
                      fullWidth={!!resizedWidths[field]}
                      value={deepGet(row, field)}
                      {snippets}
                    />
                  </div>
                {/if}
              {/each}
            </div>
          {/each}
        {:else}
          <div class="placeholder" class:placeholder--no-fields={!fields.length}>
            <div class="placeholder-content">
              <svg class="placeholder-icon" viewBox="0 0 24 24" aria-hidden="true">
                <rect x="3" y="4" width="18" height="16" rx="1" />
                <line x1="3" y1="10" x2="21" y2="10" />
                <line x1="3" y1="15" x2="21" y2="15" />
                <line x1="10" y1="4" x2="10" y2="20" />
              </svg>
              <div>{noRowsMessage || "No rows found"}</div>
            </div>
          </div>
        {/if}
      </div>
    </div>
  </Provider>
  {#if tooltip}
    <div class="tooltip" style={tooltip.style} bind:this={tooltipElement}>
      {tooltip.text}
    </div>
  {/if}
  {#if canSelectRows && selectedRows.length}
    <div class="row-count">
      {selectedRows.length}
      {selectedText || ""}
    </div>
  {/if}
</div>

<style>
  .classic-table {
    background-color: var(--spectrum-alias-background-color-secondary);
  }
  .row-count {
    margin-top: var(--spacing-l);
  }

  /* Search */
  .search-bar {
    display: flex;
    flex-direction: row;
    justify-content: flex-start;
    margin-bottom: var(--spacing-l);
  }
  .search-bar--right {
    justify-content: flex-end;
  }
  .search {
    position: relative;
    display: flex;
    align-items: center;
    width: 300px;
    max-width: 100%;
  }
  .search input {
    width: 100%;
    height: var(--spectrum-alias-item-height-m, 32px);
    box-sizing: border-box;
    padding: 0 32px;
    font-family: inherit;
    font-size: var(--spectrum-alias-font-size-default, 14px);
    color: var(--spectrum-alias-text-color);
    background-color: var(--spectrum-global-color-gray-50);
    border: 1px solid var(--spectrum-alias-border-color-mid);
    border-radius: var(--spectrum-alias-border-radius-regular, 4px);
    outline: none;
    transition: border-color 130ms ease-out;
  }
  .search input:hover {
    border-color: var(--spectrum-alias-border-color-hover);
  }
  .search input:focus {
    border-color: var(--spectrum-alias-border-color-mouse-focus);
  }
  .search input::placeholder {
    color: var(--spectrum-global-color-gray-600);
  }
  .search svg {
    width: 16px;
    height: 16px;
    fill: none;
    stroke: var(--spectrum-global-color-gray-600);
    stroke-width: 2;
    stroke-linecap: round;
  }
  .search-icon {
    position: absolute;
    left: 10px;
    pointer-events: none;
  }
  .search-clear {
    position: absolute;
    right: 6px;
    display: flex;
    padding: 2px;
    background: none;
    border: none;
    cursor: pointer;
  }
  .search-clear:hover svg {
    stroke: var(--spectrum-global-color-gray-900);
  }

  /* Wrapper */
  .wrapper {
    position: relative;
    --table-bg: var(--spectrum-global-color-gray-50);
    --table-border: 1px solid var(--spectrum-alias-border-color-mid);
    --cell-padding: var(--spectrum-global-dimension-size-250);
    overflow: auto;
    display: contents;
  }
  .wrapper--quiet {
    --table-bg: var(--spectrum-alias-background-color-transparent);
  }
  .wrapper--compact {
    --cell-padding: var(--spectrum-global-dimension-size-150);
  }

  /* Table */
  .spectrum-Table {
    width: 100%;
    border-radius: 0;
    display: grid;
    overflow: auto;
    border: none;
  }
  .spectrum-Table.no-scroll {
    overflow: visible;
  }
  /* Resized columns can be wider than the table, so it has to scroll */
  .spectrum-Table.no-scroll.has-resized {
    overflow: auto;
  }

  /* Header */
  .spectrum-Table-head {
    display: contents;
  }
  .spectrum-Table-head > :first-child {
    border-left: 1px solid transparent;
    padding-left: var(--cell-padding);
  }
  .spectrum-Table-head > .spectrum-Table-headCell--edit:first-child {
    padding-left: calc(var(--cell-padding) / 1.33);
    /* adding 1px to compensate for lack of right border in header */
    padding-right: calc(var(--cell-padding) / 1.33 + 1px);
  }
  .spectrum-Table-head > :last-child {
    border-right: 1px solid transparent;
    padding-right: var(--cell-padding);
  }
  .spectrum-Table-headCell {
    height: var(--header-height);
    box-sizing: border-box;
    position: sticky;
    top: 0;
    text-overflow: ellipsis;
    white-space: nowrap;
    background-color: var(--spectrum-alias-background-color-secondary);
    z-index: 2;
    border-bottom: var(--table-border);
    padding: 0 calc(var(--cell-padding) / 1.33);
    display: flex;
    flex-direction: row;
    justify-content: flex-start;
    align-items: center;
    user-select: none;
    border-top: var(--table-border);
    border-radius: 0;
    cursor: default;
    font-size: var(--spectrum-table-header-text-size, 11px);
    font-weight: var(--spectrum-table-header-text-font-weight, 700);
    letter-spacing: var(--spectrum-table-header-text-letter-spacing, 0.06em);
    text-transform: uppercase;
    color: var(
      --spectrum-table-header-text-color,
      var(--spectrum-global-color-gray-700)
    );
  }
  .spectrum-Table-headCell.is-sortable {
    cursor: pointer;
  }
  .spectrum-Table-headCell.is-sortable:hover {
    color: var(--spectrum-global-color-gray-900);
  }
  .spectrum-Table-headCell:first-of-type {
    border-left: var(--table-border);
  }
  .spectrum-Table-headCell:last-of-type {
    border-right: var(--table-border);
  }
  .spectrum-Table-headCell--alignCenter {
    justify-content: center;
  }
  .spectrum-Table-headCell--alignRight {
    justify-content: flex-end;
  }
  .spectrum-Table-headCell--edit {
    z-index: 3;
    left: 0;
    justify-content: center;
  }
  .spectrum-Table-headCell .title {
    min-width: 0;
    display: flex;
    align-items: center;
    gap: 4px;
  }
  .title-text {
    overflow: hidden;
    text-overflow: ellipsis;
  }
  .resize-handle {
    position: absolute;
    top: 0;
    right: 0;
    bottom: 0;
    width: 9px;
    cursor: col-resize;
    touch-action: none;
  }
  .resize-handle::after {
    content: "";
    position: absolute;
    top: 25%;
    bottom: 25%;
    right: 0;
    width: 2px;
    border-radius: 1px;
    background-color: transparent;
    transition: background-color 130ms ease-out;
  }
  .spectrum-Table-head:hover .resize-handle::after {
    background-color: var(--spectrum-global-color-gray-300);
  }
  .resize-handle:hover::after,
  .resize-handle.is-active::after {
    background-color: var(--spectrum-global-color-blue-500);
  }
  .sort-icon {
    width: 14px;
    height: 14px;
    fill: none;
    stroke: var(--spectrum-global-color-gray-700);
    stroke-width: 2.5;
    stroke-linecap: round;
    stroke-linejoin: round;
  }
  .sort-icon--asc {
    transform: rotate(180deg);
  }

  /* Table rows */
  .spectrum-Table-row {
    display: contents;
    cursor: auto;
    border: none;
  }
  .spectrum-Table-row.clickable {
    cursor: pointer;
  }
  .spectrum-Table-row.clickable:hover .spectrum-Table-cell {
    background-color: var(--spectrum-global-color-gray-100);
  }
  .spectrum-Table-row > :first-child {
    border-left: var(--table-border);
    padding-left: var(--cell-padding);
  }
  .spectrum-Table-row > .spectrum-Table-cell--edit:first-child {
    padding-left: calc(var(--cell-padding) / 1.33);
  }
  .spectrum-Table-row > :last-child {
    border-right: var(--table-border);
    padding-right: var(--cell-padding);
  }

  /* Table cells */
  .spectrum-Table-cell {
    flex: 1 1 auto;
    box-sizing: border-box;
    padding: 0 calc(var(--cell-padding) / 1.33);
    border-top: none;
    border-radius: 0;
    text-overflow: ellipsis;
    white-space: nowrap;
    height: var(--row-height);
    min-height: 0;
    display: flex;
    flex-direction: row;
    justify-content: flex-start;
    align-items: center;
    gap: 4px;
    border-bottom: 1px solid var(--spectrum-alias-border-color-mid);
    background-color: var(--table-bg);
    z-index: auto;
    transition: background-color 130ms ease-out;
    font-size: var(
      --spectrum-table-cell-text-size,
      var(--spectrum-alias-font-size-default)
    );
    color: var(
      --spectrum-table-cell-text-color,
      var(--spectrum-alias-text-color)
    );
  }
  .spectrum-Table-cell.is-resized {
    overflow: hidden;
  }
  .spectrum-Table-cell--divider {
    border-right: 1px solid var(--spectrum-alias-border-color-mid);
  }
  .spectrum-Table-cell--edit {
    position: sticky;
    left: 0;
    z-index: 1;
    justify-content: center;
  }
  .select {
    margin: 0;
    width: 14px;
    height: 14px;
    cursor: pointer;
    accent-color: var(--spectrum-global-color-blue-500);
  }

  /* Tooltip */
  .tooltip {
    position: fixed;
    left: 0;
    top: 0;
    z-index: 999;
    box-sizing: border-box;
    max-width: min(420px, calc(100vw - 12px));
    max-height: 60vh;
    overflow: hidden;
    padding: 6px 10px;
    white-space: pre-wrap;
    overflow-wrap: anywhere;
    pointer-events: none;
    font-size: var(--spectrum-alias-font-size-default, 14px);
    line-height: 1.4;
    color: var(--spectrum-alias-text-color);
    background-color: var(--spectrum-global-color-gray-50);
    border: 1px solid var(--spectrum-alias-border-color-mid);
    border-radius: var(--spectrum-alias-border-radius-regular, 4px);
    box-shadow: 0 2px 8px rgba(0, 0, 0, 0.2);
  }

  /* Placeholder */
  .placeholder {
    display: flex;
    flex-direction: row;
    justify-content: center;
    align-items: center;
    border: var(--table-border);
    border-top: none;
    grid-column: 1 / -1;
    background-color: var(--table-bg);
    padding: 40px;
  }
  .placeholder--no-fields {
    border-top: var(--table-border);
  }
  .wrapper--quiet .placeholder {
    border-left: none;
    border-right: none;
  }
  .placeholder-content {
    display: flex;
    flex-direction: column;
    justify-content: center;
    align-items: center;
    color: var(
      --spectrum-table-cell-text-color,
      var(--spectrum-alias-text-color)
    );
  }
  .placeholder-content div {
    margin-top: 10px;
    font-size: var(
      --spectrum-table-cell-text-size,
      var(--spectrum-alias-font-size-default)
    );
    text-align: center;
  }
  .placeholder-icon {
    width: 48px;
    height: 48px;
    fill: none;
    stroke: var(--spectrum-global-color-gray-600);
    stroke-width: 1.2;
  }
</style>
