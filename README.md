# Budibase ClassicTable

The classic Budibase table (the `Table` component that lives inside a data provider
and was replaced by the grid based table), rebuilt as a plugin in Budibase's Svelte 5
plugin format, with a search field on top.

## Requirements
A Budibase version that runs Svelte 5 plugins (`schema.metadata.svelteMajor: 5`).

## Installation
1. Open Budibase and navigate to the "Plugins" section.
2. Click add plugin and select the GitHub source.
3. Enter the URL `https://github.com/Sagestum/budibase-classic-table`.

## Use
1. Add a **Data Provider** and choose its data source, sorting, limit and pagination.
2. Add the **ClassicTable** component inside the data provider.
3. Optionally choose the columns, the search fields and the row click actions.

## Features
- Click a column header to sort. Sorting is done by the data provider, so it covers all pages.
- One search field that searches several columns at once (a match in any column is enough).
  The search is added to the data provider's query, so it searches the whole data source and
  not just the loaded page. Text columns are searched with "contains", number columns need an
  exact match. Without selected search fields all text columns are searched.
- Columns setting with display name, width, alignment and value template.
- Row count (scroll limit), compact and quiet mode, medium and large size.
- Row selection with the `Selected Rows` binding and the `Clear Row Selection` action.
- `On Row Click` actions with the clicked row as context.
- Components placed inside the table are rendered as a first column for every row and can
  bind to the row.

## Build
```
yarn install
yarn build
```
The plugin bundle is written to `dist/budibase-classic-table-<version>.tar.gz`.
