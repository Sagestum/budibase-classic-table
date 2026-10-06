<script>
  import { getContext } from "svelte"

  export let value
  export let schema
  export let snippets

  const { processStringSync } = getContext("sdk")

  const DISPLAY_LIMIT = 5
  const IMAGE_EXTENSIONS = ["png", "tiff", "gif", "raw", "jpg", "jpeg"]
  const TYPE_MAP = {
    boolean: "boolean",
    datetime: "datetime",
    link: "relationship",
    bb_reference: "relationship",
    bb_reference_single: "relationship",
    attachment: "attachment",
    attachment_single: "attachment",
    signature_single: "attachment",
    array: "array",
  }

  $: cellValue = getCellValue(value, schema?.template, snippets)
  $: kind = getKind(schema)
  $: list = toList(cellValue)
  $: visible = list.slice(0, DISPLAY_LIMIT)
  $: leftover = list.length - visible.length

  const getCellValue = (value, template, snippets) => {
    if (!template) {
      return value
    }
    return processStringSync(template, { value, snippets })
  }

  const getKind = schema => {
    // A custom template always produces text
    if (schema?.template) {
      return "string"
    }
    return TYPE_MAP[schema?.type] ?? "string"
  }

  const toList = value => {
    if (Array.isArray(value)) {
      return value
    }
    return value == null || value === "" ? [] : [value]
  }

  const isImage = extension =>
    IMAGE_EXTENSIONS.includes(extension?.toLowerCase() ?? "")

  const formatDate = (value, schema) => {
    // Time only values arrive as HH:mm:ss and are no valid dates on their own
    const time = new Date(`0-${value}`)
    if (!isNaN(time) || schema?.timeOnly) {
      const date = isNaN(time) ? new Date(value) : time
      return isNaN(date) ? value : date.toLocaleTimeString()
    }
    const date = new Date(value)
    if (isNaN(date)) {
      return value
    }
    if (schema?.dateOnly) {
      // Date only values carry no timezone, keep the stored calendar day
      return date.toLocaleDateString(undefined, {
        dateStyle: "long",
        timeZone: "UTC",
      })
    }
    return date.toLocaleString(undefined, {
      dateStyle: "long",
      timeStyle: "short",
    })
  }
</script>

{#if cellValue != null && cellValue !== ""}
  {#if kind === "boolean"}
    <input type="checkbox" class="boolean" disabled checked={!!cellValue} />
  {:else if kind === "datetime"}
    <div class="date">{formatDate(cellValue, schema)}</div>
  {:else if kind === "relationship"}
    {#each visible as relationship}
      {#if relationship?.primaryDisplay}
        <span class="badge">{relationship.primaryDisplay}</span>
      {/if}
    {/each}
  {:else if kind === "array"}
    {#each visible as item}
      <span class="badge">{item}</span>
    {/each}
  {:else if kind === "attachment"}
    {#each visible as attachment}
      <a
        class="attachment"
        class:file={!isImage(attachment.extension)}
        target="_blank"
        download={attachment.name}
        href={attachment.url}
        title={attachment.name}
        on:click|stopPropagation
      >
        {#if isImage(attachment.extension)}
          <img src={attachment.url} alt={attachment.extension} />
        {:else}
          {attachment.extension}
        {/if}
      </a>
    {/each}
  {:else}
    <div
      class="text"
      class:capitalise={schema?.capitalise}
      style="--max-cell-width: {schema?.width ? 'none' : '200px'};"
    >
      {typeof cellValue === "object" ? JSON.stringify(cellValue) : cellValue}
    </div>
  {/if}
  {#if leftover > 0 && kind !== "string" && kind !== "datetime" && kind !== "boolean"}
    <div>+{leftover} more</div>
  {/if}
{/if}

<style>
  .text {
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
    max-width: var(--max-cell-width);
    width: 0;
    flex: 1 1 auto;
  }
  .date {
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
  }
  .text.capitalise {
    text-transform: capitalize;
  }
  .boolean {
    margin: 0;
    width: 14px;
    height: 14px;
    accent-color: var(--spectrum-global-color-blue-500);
  }
  .badge {
    display: inline-flex;
    align-items: center;
    height: 20px;
    padding: 0 8px;
    border-radius: 4px;
    font-size: 12px;
    font-weight: 600;
    white-space: nowrap;
    color: var(--spectrum-global-color-gray-900);
    background-color: var(--spectrum-global-color-gray-300);
  }
  .attachment {
    display: flex;
    flex-direction: row;
    justify-content: flex-start;
    align-items: center;
  }
  .attachment img {
    height: 32px;
    max-width: 64px;
  }
  .attachment.file {
    height: 32px;
    padding: 0 8px;
    color: var(--spectrum-global-color-gray-800);
    border: 1px solid var(--spectrum-global-color-gray-300);
    border-radius: 4px;
    text-transform: uppercase;
    text-decoration: none;
    font-weight: 600;
    font-size: 11px;
  }
</style>
