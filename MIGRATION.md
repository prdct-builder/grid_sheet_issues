# Migration Guide: 2.2.3 → 3.0.0

## A note to everyone building with GridSheet

Thank you for building with GridSheet. Teams around the world rely on it in their products, and this release is about making that work simpler and more dependable.

This is a major release with a long list of API changes. That's deliberate. As GridSheet grew, it collected more than one way to do the same thing: index-based and key-based methods side by side, near-duplicate helpers, and names that didn't follow one pattern. Each of these was reasonable when it was added, but together they made the API harder to learn and easier to misuse. This release settles on one clear way to do each thing:

- **Keys everywhere.** Rows and columns are addressed by their stable keys, which stay correct through sorting, filtering, reordering and pagination. Code can no longer silently act on the wrong row because an index had shifted.
- **One name for one job.** Overlapping methods are merged, and names follow a single pattern, so the API reads the same wherever you look.
- **Fewer surprises.** Values keep their real types, callbacks receive the context they need, and several long-standing bugs are fixed along the way.

We know upgrading takes time, and we've tried to make it as quick as possible. Almost every change is a compile-time error, so the analyzer points you to each place that needs updating, and every 2.2.3 API below is listed next to its replacement. Most projects can upgrade by working through the analyzer errors with this guide open.

If something isn't covered here, or a change doesn't fit how you use GridSheet, please tell us on the [issue tracker](https://github.com/prdct-builder/grid_sheet_issues). Your feedback shapes the next release.

## What changed

- **The manager is grouped by area.** `GridSheetManager`'s members moved onto areas such as `gridManager.rows`, `gridManager.columns`, `gridManager.cells` and `gridManager.selection`. `gridManager.rows` and `gridManager.columns` are no longer lists: read the lists with `rows.all` and `columns.all`.
- **Keys only.** Every method that took a row/column index or a column name now takes a `ValueKey<String>`. With one addressing scheme left, the `...ByKeys`/`...ByKey` suffixes and the `get` prefixes are gone.
- **`GridSheetState` follows the manager.** It implements `GridSheetManager`, so code that called methods on `key.currentState!` changes the same way: `key.currentState!.deleteRowsByKeys(keys)` becomes `key.currentState!.rows.delete(keys)`.

Everything below is a compile-time break unless it's listed under [Behavior changes](#behavior-changes).

- [Resolving indexes and names to keys](#resolving-indexes-and-names-to-keys)
- [GridSheetManager](#gridsheetmanager)
- [List and row extensions](#list-and-row-extensions)
- [GridSheet widget](#gridsheet-widget)
- [Configuration, models and enums](#configuration-models-and-enums)
- [No longer exported](#no-longer-exported)
- [Removed deprecated symbols](#removed-deprecated-symbols)
- [Configuration flags gate the API](#configuration-flags-gate-the-api)
- [Behavior changes](#behavior-changes)
- [Mocking the manager](#mocking-the-manager)

---

## Resolving indexes and names to keys

If your code only has a row/column **position** (a loop counter, a builder's `index`), resolve it first:

```dart
final rowKey = gridManager.rows.at(index).key;       // throws RangeError if out of range
final columnKey = gridManager.columns.at(index).key;
final keys = gridManager.rows.atIndexes([0, 1, 2]).map((r) => r.key).toList();
```

If it only has column **names** (a saved layout, a form config):

```dart
final keys = gridManager.columns
    .findByNames(['status', 'amount'])  // unknown names are skipped
    .map((c) => c.key)
    .toList();
```

---

## GridSheetManager

The 2.2.3 `GridSheetManagerHelpers` extension is removed; its members are listed below with the rest.

### Rows — `gridManager.rows`

| 2.2.3 | Now |
|---|---|
| `rows` | `rows.all` |
| `filteredRows` / `currentPageRows` | `rows.filtered` / `rows.currentPage` |
| `totalFilteredRows` / `totalCurrentPageRows` | `rows.filteredCount` / `rows.currentPageCount` |
| `findRowByKey(key)` / `hasRowByKey(key)` | `rows.find(key)` / `rows.contains(key)` |
| `getRows()` | `rows.inScope()` |
| `getRows(onlyFilteredRows: true)` / `getRows(onlyCurrentPageRows: true)` | `rows.inScope(scope: GridSheetRowScope.filtered)` / `.currentPage` |
| `getRowRangeByKeys(startKey, endKey)` | `rows.range(startKey, endKey)` |
| `getFilteredRowRangeByKeys(...)` / `getCurrentPageRowRangeByKeys(...)` | `rows.range(..., scope: GridSheetRowScope.filtered)` / `.currentPage` |
| `getRowRange(start, end)` | `rows.range(rows.at(start).key, rows.at(end - 1).key)`. **`end` was exclusive; `rows.range` includes its end key.** |
| `getFilteredRowRange(start, end)` / `getCurrentPageRowRange(start, end)` | as above, with `scope: GridSheetRowScope.filtered` / `.currentPage`, resolving the keys from `rows.filtered` / `rows.currentPage` |
| `insertRowsAt(index, rows, {defaultValues})` | `rows.insertAt(key, rows, {defaultValues, position, band})`: inserts above `key` by default; `position: GridSheetRowInsertPosition.below` for below; `key: null` appends. The old `index` was relative to the current page. |
| `insertRows(rows, {position, defaultValues})` | `rows.insert(rows, {defaultValues})`. The `position` parameter is gone; placement comes from `GridSheetConfiguration.rowInsertPosition`. |
| `duplicateSelectedRows(index, rowKeys, newRowKeys)` | `rows.duplicate(keys, newKeys)`, or `rows.duplicateAt(key, keys, newKeys)` for an explicit target |
| `deleteRowsAt(indices)` / `deleteRowsByKeys(keys)` | `rows.delete(keys)` |
| `updateRowByKey(key, data)` | `rows.update(key, data)` |
| `reorderRowsByPositions(fromIndex:, toIndex:)` / `reorderRowsByKeys(fromRowKey:, toRowKey:)` | `rows.reorder(fromKey:, toKey:)` |
| `resetRowOrder()` | `rows.resetOrder()`, now returning `Future<void>` |
| `updateRowHeights(Map<int, double>)` / `updateRowHeightsByKeys(map)` | `rows.setHeights(Map<ValueKey<String>, double>)` |
| `setAllRowHeights(height)` | `rows.setHeights({for (final r in gridManager.rows.all) r.key: height})` |
| `autoFitRowHeights(keys)` | `rows.autoFitHeights(keys)` |
| `resetAllRowHeights()` / `autoFitAllRowHeights()` | `rows.resetHeights(keys)` / `rows.autoFitHeights(keys)` with `[for (final r in gridManager.rows.all) r.key]` |

### Columns — `gridManager.columns`

| 2.2.3 | Now |
|---|---|
| `columns` | `columns.all` |
| `findColumnByKey(key)` / `hasColumnByKey(key)` | `columns.find(key)` / `columns.contains(key)` |
| `findColumnByName(name)` | `columns.findByNames([name])`, which returns a list (use `.firstOrNull`) |
| `activeColumns` / `pinnedLeftColumns` / `pinnedRightColumns` | `columns.scrollable` / `columns.pinnedLeft` / `columns.pinnedRight` |
| `getAllColumnsInOrder()` / `getVisibleColumnsInOrder()` | `columns.inScope()` / `columns.inScope(scope: GridSheetColumnScope.visible)` |
| `insertColumnsAt(index, columns, {defaultValues})` | `columns.insertAt(key, columns, {defaultValues, position, band})`: inserts before `key` by default; `key: null` appends |
| `insertColumns(columns, {position, defaultValues})` | `columns.insert(columns, {defaultValues})`. The `position` parameter is gone; placement comes from `GridSheetConfiguration.columnInsertPosition`. |
| `duplicateSelectedColumns(index, List<String> newColumnKeys)` | `columns.duplicate(keys, newKeys)`: `keys` is now explicit (e.g. `selection.columnKeys`) and `newKeys` is `List<ValueKey<String>>`; `columns.duplicateAt` for an explicit target |
| `deleteColumnsAt(indices)` / `deleteColumnsByKeys(keys)` | `columns.delete(keys)` |
| `deleteColumnsOfSelectedCells(cells)` | `columns.delete(cells.map((c) => c.columnKey).toSet().toList())` |
| `renameColumns(Map<String, ...>)` → errors by name | `columns.rename(Map<ValueKey<String>, ...>)` → errors by key |
| `reorderColumnsByPositions(fromIndex:, toIndex:)` | `columns.reorder(fromKey:, toKey:)` |
| `setColumnOrderByNames(names)` / `resetColumnOrder()` | `columns.setOrder(keys)` / `columns.resetOrder()` |
| `hideColumns(names)` / `showColumns(names)` | `columns.hide(keys)` / `columns.show(keys)` |
| `pinColumnsToLeft(names)` / `pinColumnsToRight(names)` | `columns.pin(leftKeys: keys)` / `columns.pin(rightKeys: keys)`. Unlike the old methods, `pin` doesn't move the columns. |
| `pinColumnsLeftUptoIndex(i)` | `columns.pin(leftKeys: columns.inScope(scope: GridSheetColumnScope.visible).take(i + 1).map((c) => c.key).toList())` |
| `pinColumnsRightFromIndex(i)` | `columns.pin(rightKeys: columns.inScope(scope: GridSheetColumnScope.visible).skip(i).map((c) => c.key).toList())` |
| `clearPinColumns()` | `columns.unpinAll()` |
| `getColumnWidths()` → `Map<String, double>` | `columns.widths` → `Map<ValueKey<String>, double>` |
| `setColumnWidths(Map<String, double>)` | `columns.setWidths(Map<ValueKey<String>, double>)`, now returning `Future<void>` |
| `autoFitColumnWidths(names)` | `columns.autoFitWidths(keys)` |
| `autoFitAllColumnWidths()` | `columns.autoFitWidths(columns.inScope(scope: GridSheetColumnScope.visible).map((c) => c.key).toList())` |

### Cells — `gridManager.cells`

| 2.2.3 | Now |
|---|---|
| `getCellValueByKeys(rowKey, columnKey)` / `getCellAt(rowIndex, columnIndex)` | `cells.value(rowKey, columnKey)` |
| `getCellValuesByKeys(cells)` | `cells.values(cells)` |
| `updateCurrentCell(rowKey:, columnKey:, value:)` / `setCellAt(rowIndex, columnIndex, value)` | `cells.update(rowKey:, columnKey:, value:)` |
| `updateCurrentCells(updates)` | `cells.updateAll(updates)` |
| `fillCellsByKeys(cells, value)` | `cells.updateAll([for (final c in cells) GridSheetCellUpdate(rowKey: c.rowKey, columnKey: c.columnKey, value: value)])` |
| `clearCellsByKeys(cells)` | `cells.clear(cells)` |
| `fillCells(...)` | `cells.fill(...)` |
| `clearCellRange(startRow:, endRow:, startColumn:, endColumn:)` | `cells.clearRange(startRowKey:, endRowKey:, startColumnKey:, endColumnKey:)` |
| `getCellRange(startRow:, endRow:, startColumn:, endColumn:)` | `cells.range(startRowKey, endRowKey, startColumnKey, endColumnKey)` |
| `getFilteredCellRange(..., onlyCurrentPageRows:)` | `cells.range(..., scope: GridSheetRowScope.filtered)` / `.currentPage` |
| `findCells(predicate:, columnNames:)` | `cells.find(predicate:, columnKeys:)` |
| `findAndReplace(..., columnNames:)` | `cells.findAndReplace(..., columnKeys:)` |
| `autofillVerticalCells(...)` / `autofillHorizontalCells(...)` | `cells.autoFillVertical(...)` / `cells.autoFillHorizontal(...)` |
| `copyCellsToClipboard(...)` / `pasteCellsFromClipboard(...)` | `cells.copyToClipboard(...)` / `cells.pasteFromClipboard(...)` |
| `validateAllCells(...)` | `cells.validateAll(...)` |
| `scrollToCell(...)` | `cells.scrollTo(...)` |
| `getCellController(...)` / `getCellFocusNode(...)` | `cells.controller(...)` / `cells.focusNode(...)` |

### Selection — `gridManager.selection`

| 2.2.3 | Now |
|---|---|
| `selectedRowKeys` / `selectedColumnKeys` / `selectedCellKeys` | `selection.rowKeys` / `selection.columnKeys` / `selection.cellKeys` |
| `selectRowsByKeys` / `deselectRowsByKeys` | `selection.selectRows` / `selection.deselectRows` |
| `selectColumnsByKeys` / `deselectColumnsByKeys` | `selection.selectColumns` / `selection.deselectColumns` |
| `selectCellsByKeys` / `deselectCellsByKeys` | `selection.selectCells` / `selection.deselectCells` |
| `isRowSelected(key)` / `isColumnSelected(key)` / `isCellSelected(rowKey, columnKey)` | `selection.containsRow(key)` / `selection.containsColumn(key)` / `selection.containsCell(rowKey, columnKey)` |
| `getSelectedRows()` / `getSelectedColumns()` / `getSelectedCells()` | `selection.rows` / `selection.columns` / `selection.cells` |
| `getSelectedRowIndex()` / `getSelectedColumnIndex()` | `selection.rowIndex` / `selection.columnIndex` |
| `getSelectedRowsData(rows)` | `selection.rowsData` for the current selection, or `data.toMaps(rows)` for any rows |
| `getSelectedColumnsData(columns, {onlyCurrentPageRows})` | `selection.columnsData({onlyCurrentPageRows})` (reads the current selection) |
| `getSelectedColumnsNames(columns)` | `selection.columnNames` |
| `getSelectedCellsData(cells)` | `selection.cellsData` |

The selection events (`GridSheetRowsSelectedEvent`, `GridSheetColumnsSelectedEvent`, `GridSheetCellsSelectedEvent`) are unchanged.

### Filters and search — `gridManager.filters`, `gridManager.search`

| 2.2.3 | Now |
|---|---|
| `hasFilters` | `filters.hasAny` |
| `filters` → `Map<String, List>` | `filters.active` → `Map<ValueKey<String>, List>` |
| `addFilters({'status': [...]}, {comparison})` | `filters.add({statusKey: [...]}, {comparison})` |
| `removeFilter(name)` / `removeAllFilters()` | `filters.clear(key)` / `filters.clearAll()` |
| `search(values, {exactMatch, smartSyntax})` / `clearSearch()` | `search.apply(values, {exactMatch, smartSyntax})` / `search.clear()` |
| `activeSearchItems` / `searchExactMatchCase` / `searchSmartSyntax` | `search.terms` / `search.exactMatch` / `search.smartSyntax` |

### Sorting — `gridManager.sorting`

| 2.2.3 | Now |
|---|---|
| `sortKeys` | `sorting.keys` |
| `sortByColumn(name, {direction})` | `sorting.sortBy(key, {direction})` |
| `clearSort()` | `sorting.clear()` |
| `setColumnComparator(name, comparator)` | `sorting.setComparator(key, comparator)` |

### Pagination — `gridManager.pagination`

| 2.2.3 | Now |
|---|---|
| `currentPage` / `totalPages` / `rowsPerPage` / `serverTotalCount` | `pagination.current` / `.totalPages` / `.rowsPerPage` / `.serverTotalCount` |
| `isPaginationEnabled` / `hasMoreRows` | `pagination.isEnabled` / `pagination.hasMore` |
| `goToPage(page)` / `nextPage()` / `previousPage()` | `pagination.goTo(page)` / `pagination.next()` / `pagination.previous()` |
| `changeRowsPerPage(n)` | `pagination.setRowsPerPage(n)` |

### Conditional formatting — `gridManager.formatting`

Rules are now edited as one list: change a copy and hand it back. List order is priority order (the first rule wins).

```dart
final rules = [...gridManager.formatting.rules];
rules.insert(0, rules.removeAt(2));                             // reorder
rules.removeWhere((r) => r.scope == GridSheetFormatScope.row);  // remove by scope
await gridManager.formatting.setRules(rules);
```

| 2.2.3 | Now |
|---|---|
| `conditionalFormatRules` | `formatting.rules` |
| `addConditionalFormatRule(rule, {atIndex})` | `formatting.add(rule, {atIndex})` |
| `updateConditionalFormatRule(...)` / `removeConditionalFormatRule(...)` / `removeConditionalFormatRulesByScope(scope)` / `reorderConditionalFormatRulesByPositions(...)` | edit the list, then `formatting.setRules(rules)` |
| `findConditionalFormatRules(scope:, name:)` | `formatting.rules.where(...)` |
| `removeAllConditionalFormatRules()` | `formatting.clear()` |
| `refreshConditionalFormatting()` | `formatting.setRules(gridManager.formatting.rules)` (passing the current rules re-applies them) |
| `getConditionalFormattingStyles(onlyVisibleColumns:, onlyFilteredRows:, onlyCurrentPageRows:)` | `formatting.styles(columnScope:, rowScope:)` |

### Formulas — `gridManager.formulas`

| 2.2.3 | Now |
|---|---|
| `getCellFormula(rowKey:, columnKey:)` | `formulas.of(rowKey:, columnKey:)` |
| `evaluateFormula(formula)` | `formulas.evaluate(formula)` |
| `refreshAllFormulas()` | `formulas.refreshAll()` |
| `showFormulaOnCellFocus(...)` | removed (an internal editing hook) |

### Data and export — `gridManager.data`

`only...` flags became `GridSheetRowScope` (`all`/`filtered`/`currentPage`) and `GridSheetColumnScope` (`all`/`visible`). The defaults match the old ones.

| 2.2.3 | Now |
|---|---|
| `rowsDataToMapList(rows)` | `data.toMaps(rows)` |
| `getRowValuesByKey(key)` | `data.toMap(key)` |
| `getAllRowsInOrder()` | `data.toLists(rowScope: GridSheetRowScope.all)` |
| `getCurrentRowsInOrder()` / `getCurrentRowsInOrder(ignorePagination: false)` | `data.toLists()` / `data.toLists(rowScope: GridSheetRowScope.currentPage)` |
| `getVisibleColumnsAllRowsAsLists()` | `data.toLists(columnScope: GridSheetColumnScope.visible, rowScope: GridSheetRowScope.all)` |
| `getVisibleColumnsCurrentRowsAsLists()` | `data.toLists(columnScope: GridSheetColumnScope.visible)` |
| `exportToCSV(onlyVisibleColumns:, onlyFilteredRows:, onlyCurrentPageRows:)` | `data.toCsv(columnScope:, rowScope:)` |
| `exportToCSVWithFormulas(includeFormulas: true, ...)` | `data.toCsv(includeFormulas: true, ...)` |
| `getColumnStatistics(name, {hideAggregateStats})` | `data.statistics(key, {hideAggregateStats})` |
| `groupRowsByColumn(name)` | `data.groupBy(key)` |

### Change tracking — `gridManager.changes`

| 2.2.3 | Now |
|---|---|
| `insertedRows` / `modifiedRows` / `deletedRows` / `changedRows` | `changes.inserted` / `changes.modified` / `changes.deleted` / `changes.all` |
| `undoRowChanges(keys)` / `undoAllRowsChanges()` | `changes.undo(keys)` / `changes.undoAll()` |

### Configuration — `gridManager.config`

| 2.2.3 | Now |
|---|---|
| `configuration` / `updateConfiguration(config)` | `config.grid` / `config.updateGrid(config)` |
| `styleConfiguration` / `scrollConfiguration` / `contextMenuConfiguration` | `config.style` / `config.scroll` / `config.contextMenu` |
| `autofillConfiguration` / `rowExpansionConfiguration` | `config.autoFill` / `config.rowExpansion` |
| `formConfiguration` / `animationConfiguration` | `config.form` / `config.animation` |
| `tableGenerationConfiguration` | `config.dynamicGeneration` (nullable; see [Configuration, models and enums](#configuration-models-and-enums)) |

### Hover, loading, auto-reload, snapshots

| 2.2.3 | Now |
|---|---|
| `hoveredRowKey` / `isRowHovered(key)` | `cursor.hoveredRowKey` / `cursor.isRowHovered(key)` |
| `isLoading` / `setLoading(true)` / `setLoading(false)` | `loading.isActive` / `loading.start()` / `loading.stop()` |
| `startAutoReload()` / `stopAutoReload()` | `autoReload.start()` / `autoReload.stop()` |
| `isAutoReloadActive` / `autoReloadFailureCount` | `autoReload.isActive` / `autoReload.failureCount` |
| `createSnapshot()` / `currentSnapshot` | `snapshots.create()` |
| `restoreSnapshot(snapshot)` | `snapshots.restore(snapshot)` |
| `gridSheetTableKey` | `gridKey` |

`reload()`, `notifyListeners()` and `cleanup()` are unchanged.

---

## List and row extensions

The `List<GridSheetRow>` (`GridSheetRowHelper`) and `List<GridSheetColumn>` (`GridSheetColumnListHelper`) extensions are removed. The grid owns its rows and columns, so read them through the manager:

| 2.2.3 | Now |
|---|---|
| `rows.findByKey('id')` (a `String`) | `gridManager.rows.find(const ValueKey('id'))` |
| `rows.insertedRows` / `modifiedRows` / `deletedRows` / `changedRows` | `gridManager.changes.inserted` / `.modified` / `.deleted` / `.all` |
| `rows.existingRows` | `gridManager.rows.all.where((r) => r.isExisting).toList()` |
| `rows.selectedRows` / `rows.unselectedRows` | `gridManager.selection.rows` / `rows.where((r) => !gridManager.selection.containsRow(r.key))` |
| `rows.rowsDataToMapList(columns)` | `rows.map((r) => r.toParsedDataMap(columns)).toList()` |
| `rows.rowsAsListsInCurrentOrder()` | `rows.map((r) => List<dynamic>.from(r.data)).toList()` |
| `rows.rowsToVisibleLists(columns)` | `rows.map((r) => [for (final c in columns) r.data[c.index]]).toList()` |
| `columns.findByKey(key)` / `columns.findByName(name)` | `gridManager.columns.find(key)` / `gridManager.columns.findByNames([name])` |
| `columns.visibleColumns` / `columns.hiddenColumns` | `gridManager.columns.inScope(scope: GridSheetColumnScope.visible)` / `gridManager.columns.hidden` |
| `columns.activeColumns` / `pinnedLeftColumns` / `pinnedRightColumns` | `gridManager.columns.scrollable` / `.pinnedLeft` / `.pinnedRight` |

The `GridSheetRow` extension (`GridSheetRowExtension`) is removed too:

| 2.2.3 | Now |
|---|---|
| `row.rowKey` / `row.rowData` | `row.key` / `row.data` (`key` is non-nullable, so drop any `!`) |
| `row.rowDataToMap(columns)` | `row.toParsedDataMap(columns)`: a rename; it still parses values by column type |

---

## GridSheet widget

| 2.2.3 | Now |
|---|---|
| `contextMenuWidget` | `contextMenuBuilder` |
| `headerWidget` / `footerWidget` | `topBarBuilder` / `bottomBarBuilder` |
| `undoDeleteWidget` | `undoDeleteBuilder` |
| `noRowsWidget` / `noColumnsWidget` | `emptyRowsPlaceholder` / `emptyColumnsPlaceholder` |
| `loadingWidget` | `loadingIndicator` |
| `autofillConfiguration` | `autoFillConfiguration` |
| `tableGenerationConfiguration` | `dynamicGenerationConfiguration` |
| `headerWrapper: (context, child, isPinnedLeft, isPinnedRight)` | `headerWrapper: (context, child, details)`: `details.pinnedLeft`, `details.pinnedRight`, `details.gridManager` |
| `filterWrapper: (context, child, isPinnedLeft, isPinnedRight)` | `filterWrapper: (context, child, details)`: same fields as `headerWrapper` |
| `rowWrapper: (context, child, isPinnedLeft, isPinnedRight, isHovered, isSelected)` | `rowWrapper: (context, child, details)`: `details.pinnedLeft`, `.pinnedRight`, `.hovered`, `.selected`, `.row`, `.gridManager` |
| `rowExpandedWidgetBuilder: (context, row, columns)` | `rowExpandedBuilder: (context, row, columns, gridManager)` |
| `rowStateColorBuilder: (row) => color` + `rowStateIconBuilder: (row) => icon` | `rowStateIndicatorBuilder: (row) => GridSheetRowStateIndicator(color: color, icon: icon)` |
| `GridSheetRowStateColorBuilder` / `GridSheetRowStateIconBuilder` | `GridSheetRowStateIndicatorBuilder` |

### `rows` and `columns` now follow the widget

In 2.2.3 the grid read `rows` and `columns` once and ignored later changes. Now they work like any Flutter widget's input: when the parent passes different rows or columns, the grid shows them, keeping sort, filters, search, grouping, the page, and the selection and expansion of rows and columns that are still there. Passing equal data on a rebuild changes nothing, so edits made in the grid survive.

What to check:

- If your parent passes **different** `rows`/`columns` on later builds and relied on the grid ignoring them, the grid now updates. Pass the data you want shown, or keep the list unchanged.
- New rows from the parent become the change-tracking baseline, and replace unsaved edits to those rows.
- Workarounds for the old behavior can go: a new `Key` to force a rebuild, or `rowsLoader` + `reload()` used only to show new data.
- With `serverSidePagination` or `infiniteScroll`, rows still come from `onPageChange`; a changed `rows` is ignored.

---

## Configuration, models and enums

| 2.2.3 | Now |
|---|---|
| `GridSheetDynamicTableGeneration(enabled: true, ...)` | `GridSheetDynamicGenerationConfiguration(...)` (no `enabled`) |
| `GridSheetDynamicTableGeneration(enabled: false)` | leave `dynamicGenerationConfiguration` out (it's now nullable and defaults to `null`) |
| `GridSheetConfiguration.selectOnlyPageRows` | `selectOnlyCurrentPageRows` |
| `GridSheetConfiguration.filterBlanksText` / `filterNonBlanksText` | `blanksText` / `noBlanksText` |
| `GridSheetConstants.filterBlanksText` / `filterNoBlanksText` | `GridSheetConstants.blanksText` / `noBlanksText` |
| `GridSheetStyleConfiguration.ascSortIconColor` / `descSortIconColor` | `ascendingSortIconColor` / `descendingSortIconColor` |
| `GridSheetStyleConfiguration.footerRowBorderColor` | `bottomBarBorderColor` |
| `GridSheetFormConfiguration.mode` | `viewMode` |
| `GridSheetFormConfiguration.headerBuilder: (row) => ...` / `footerBuilder` | `topBarBuilder: (context, row, gridManager) => ...` / `bottomBarBuilder` |
| `GridSheetColumn.resize` | `resizable` |
| `GridSheetColumn(noTextControllerWidget: true)` | `GridSheetColumn(hasTextController: false)`: the meaning is inverted; the default is `true` |
| `GridSheetRow(originalData: ...)` / `row.copyWith(originalData: ...)` / `row.originalData = ...` | removed: pass `data`; the grid sets `originalData` (now read-only) on the first edit |
| `row.markAsInserted()` / `markAsModified()` / `markAsDeleted()` / `setDataAt(i, v)` / `replaceData(data)` | `gridManager.cells.update(...)` / `gridManager.rows.update(key, data)`, which also track changes and refresh formulas |
| `row.setHeight(h)` / `row.resetHeight()` | `gridManager.rows.setHeights({row.key: h})` / `gridManager.rows.resetHeights([row.key])` |
| `row.mutableDataForInternalUse`, `row.formatCacheForColumn(...)`, `row.clearFormatCache()`, `row.dispose()` | removed from the public API (grid internals). `row.rebuildRow()` is unchanged. |
| `GridSheetCellContext.gridManager` (nullable) | non-nullable: drop `?.` / `!` |
| `GridSheetCellContext.onFilterChanged({column.name: [...]})` | `onFilterChanged({column.key: [...]})` |
| `GridSheetSortState('age', direction)` / `sort.columnName` | `GridSheetSortState(key: ageKey, name: 'age', direction: direction)` / `sort.name` (plus `sort.key`) |
| `GridSheetPageRequest.searchExactMatchCase` / `GridSheetSnapshot.searchExactMatchCase` | `searchExactMatch` |
| `GridSheetConditionalFormatRule(backgroundColorExpression: ...)` | `GridSheetConditionalFormatRule(expression: ...)` |
| `GridSheetFormatScope.column` | `GridSheetFormatScope.cell` (identical behavior; for a whole-column color use a constant expression such as `'"#E3F2FD"'`) |
| Expression variables `row_state`, `cell_value`, `column_name`, `column_index` | `_row_state`, `_cell_value`, `_column_name`, `_column_index` (conditional formatting and `conditionalEditExpression`) |
| `GridSheetRowInsertPosition.aboveSelection` / `.belowSelection` | `.above` / `.below` |
| `GridSheetColumnInsertPosition.beforeSelection` / `.afterSelection` | `.before` / `.after` |
| `IGridSheetCustomCellWidgetBuilder` | `IGridSheetCustomCellBuilder` (same methods) |
| `CellValueChanged` typedef | removed (the grid never called it); declare your own callback type |
| `GridSheetCellWidgetType` enum | removed (unused); use `GridSheetColumnType` or your own enum |
| `GridSheetSortKey` class | removed (unused); the active sort is `sorting.keys` (`List<GridSheetSortState>`) |

Still identified by column **name**, unchanged: `GridSheetPageRequest.filters`/`filterComparisons`/`sortKeys` (what a server-side host receives), `GridSheetSnapshot.filters`, and data output such as `data.toMaps`, `data.toMap`, `selection.columnsData` and CSV headers.

---

## No longer exported

These were implementation details. Use the manager instead.

| 2.2.3 | Use instead |
|---|---|
| Formula engine: `GridSheetFormulaEngine`, `FormulaParser`, `ASTNode` and its node classes, `FormulaError`, `FormulaErrorType`, `GridSheetFormulaToken`, `GridSheetFormulaTokenType`, `Token`, `TokenType`, `GridSheetFormulaCell`, `CellAddress`, `DependencyGraph`, `CircularReferenceDetector` | `gridManager.formulas` (`of`, `evaluate`, `refreshAll`) |
| `GridSheetAutoFillEngine`, `GridSheetFillDirection` | `cells.autoFillVertical` / `cells.autoFillHorizontal` |
| `GridSheetColumnGroups` | `columns.pinnedLeft` / `columns.scrollable` / `columns.pinnedRight` |
| `GridSheetSorter` | `gridManager.sorting` (`sorting.setComparator` for a custom order) |
| `GridSheetCellSelectionState`, `GridSheetRowExpansionForm`, `RowExpansionLayout`, `GridSheetEvaluatedFormatCache` | none (internal) |
| `GridSheetNoScrollbarBehavior`, `NegativeKeyGenerator` | none (internal) |
| `GridSheetPaginationState` (with `buildPreviousPageButton`/`buildNextPageButton`) | none; the pagination state class is private |

---

## Removed deprecated symbols

These were `@Deprecated` in 2.2.3 and are now gone.

| Removed | Use instead |
|---|---|
| `GridSheetLayoutConfiguration`, `GridSheetManager.layoutConfiguration` | `GridSheetConfiguration`, `config.grid` |
| `reorderColumnsByOrder(fromIndex:, toIndex:)` | `columns.reorder(fromKey:, toKey:)` |
| `freezeColumnsUptoKey` / `freezeColumnsUptoIndex` | `columns.pin(leftKeys:)` |
| `GridSheetColumn.frozen` (getter and constructor parameter) | `pinnedLeft` |
| `GridSheetColumnGroups.frozen` | not exported (see above) |
| `GridSheetColumnContextInfo.isFrozen` | `isPinnedLeft` |
| `GridSheet.onCellValueChange` | `onCellValueChanged` |
| `GridSheetConfiguration.enableReorder` | `enableColumnReorder` |
| `GridSheetConfiguration.serialNumberColumn` / `serialNumberColumnWidth` | `GridSheet.indexColumn` |
| `GridSheetConfiguration.rowHeight` | `GridSheetRow.height` |
| `GridSheetDynamicTableGeneration.type` / `width` / `rowState` | `columnBuilder` / `rowBuilder` |
| `exportToCSV(onlyCurrentpageRows:)` (misspelled) | `data.toCsv(rowScope: GridSheetRowScope.currentPage)` |

---

## Configuration flags gate the API

A manager call now does nothing when the matching `GridSheetConfiguration` flag is off, the same as the UI. These flags default to off, so set them where you use the API:

| API | Needs |
|---|---|
| `selection.selectRows` / `selectColumns` / `selectCells` with several keys | `enableMultiSelection: true` (otherwise only the last key is selected, like a tap) |
| `rows.setHeights` / `resetHeights` / `autoFitHeights` | `enableRowResize: true` |
| `columns.setWidths` / `resetWidths` / `autoFitWidths` | `resizable: true` on each column you resize |
| `cells.autoFillVertical` / `autoFillHorizontal` | `GridSheetAutoFillConfiguration(enabled: true)`; its `enablePatternDetection` also limits the call's own `enablePatternDetection` |
| `rows.reorder` / `rows.resetOrder` | `enableRowReorder: true` |
| `columns.reorder` / `setOrder` / `resetOrder` | `enableColumnReorder: true` |

---

## Behavior changes

These compile unchanged but behave differently:

- **Cell text no longer wraps by default.** `GridSheetColumn.wrap` defaults to `false`: read-only cells truncate with an ellipsis and editable cells stay single-line. Set `wrap: true`, or call `columns.setWrap(keys, true)`, where you relied on wrapping.
- **`GridSheetFormConfiguration.formBuilder`** replaces only the form card's content, not the whole card.
- **`columns.insert`/`columns.duplicate`** always insert into the scrollable section; a pinned column selection is treated as no selection. Use `columns.insertAt`/`columns.duplicateAt` with `band:` to target a pinned pane.
- **The row-expansion panel** spans every column (pinned and scrollable) instead of only the scrollable ones.
- **`cells.pasteFromClipboard`** converts each value by column type, like typed input: pasting `99` into an integer column stores `99`, not `"99"`, and a blank value stores `null` instead of `''`.
- **`cells.findAndReplace`** converts a string `replaceValue` by column type the same way.
- **`GridSheetCellValueChangedEvent.newValue`** is the stored, typed value (like `oldValue`), not its string form: `99`, not `"99"`, in an integer column; `null`, not `''`, for a cleared cell.
- **`GridSheetDataSource.rowsFromJson` and `GridSheetRow.toParsedDataMap`** keep `List`, `Map` and custom-object values instead of turning them into `null`.
- **The (Blanks)/(Non-Blanks) filters** count an empty list, a list of empty items and an empty map as blank.
- **Cell-scoped conditional format rules follow `columns.rename`**: a rule whose `name` matched the renamed column takes the new name. Expressions that mention the old name still need updating.
- **A conditional format rule with a `textStyle` and no `expression`** now applies the text style to every cell it covers; it used to do nothing.
- **`GridSheetRow.index` is renumbered by position** on server-side pages and infinite-scroll batches, instead of keeping the value the host passed in.
- **Row drag-and-drop**: the dragged preview is lifted (`rowDragFeedbackElevation` now defaults to `8.0`, was `4.0`), and a drop lands where the drop gap shows. Dropping a row onto the row just below it no longer fires `onRowReorder`, since it doesn't move.

---

## Mocking the manager

If you mock `GridSheetManager` in tests, generate a mock for each area you use (for example `@GenerateMocks([GridSheetManager, GridSheetRowsAPI, GridSheetCellsAPI])`), stub the manager's getter to return it (`when(manager.rows).thenReturn(rowsMock)`), and stub the members on that area mock.
