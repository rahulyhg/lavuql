<template lang='pug'>
panel
  panel-body
    article.dataTableContainer
      data-table-search( v-if="showSearch", v-model="searchTerm", :placeholder="placeholder")

      table.dataTable
        data-table-header(
          :columns="columns", 
          :sort-key="internalSortKey", 
          :sort-direction="internalSortDirection", 
          @click="columnClick"
        ) 

        tbody.dataTableBody
          tr.dataTableRow( v-for="(row, i) in rows", :key="i", @click="rowClick(row)" )
            td.dataTableCell( v-for="(cell, cellIdx) in getVisibleCells(row)", :key="cellIdx" )
              slot( :name="cell.slot" :cell="cell", :row="getRowObject(row)" ) {{ cell.value }}
                    
          tr( v-if="count === 0" )
            td.dataTableNoResults( :colspan="columns.length - 1" )
              p No results

      grid-row
        grid-item
          pagination( :v-if="maxPages > 1", :pages="maxPages", :current-page="currentPage + 1", @change="paginationClick" )
        grid-item    
          lvql-select.dataTableSelect( placeholder="Items per page: ", :options="maxRowsOptions", name="dataTablePerPageSelect", id="dataTablePerPageSelect", v-model="maxRows" )
</template>

<script lang='ts'>
import { Component, Vue, Prop, Provide, Watch } from 'vue-property-decorator';
import { IDataTableHeaderItem, IComputedDataRowCell } from './interfaces';

@Component
export default class DataTable extends Vue {
  @Prop({ required: true }) header!: object;
  @Prop({ required: true }) data!: any[];
  @Prop({ default: 0 }) page!: number;
  @Prop({ default: 5 }) maxRows!: number;
  @Prop({ default: true }) showSearch!: boolean;
  @Prop({ default: '' }) sortKey!: string;
  @Prop({ default: '' }) placeholder!: string;
  @Prop({ default: 'asc' }) sortDirection!: string;

  @Provide() internalSortKey: string = '';
  @Provide() internalSortDirection: string = '';
  @Provide() currentPage: number = 0;
  @Provide() searchTerm: string = '';

  maxRowsOptions = [
    {
      label: '5',
      value: 5
    },
    {
      label: '10',
      value: 10
    },
    {
      label: '25',
      value: 25
    },
    {
      label: '50',
      value: 50
    }
  ];

  get count() {
    return this.filteredData.length;
  }

  get maxPages() {
    if (this.maxRows === 0) {
      return 0;
    }

    return Math.ceil(this.count / this.maxRows);
  }

  get filteredData() {
    const query = this.searchTerm.replace(/[.*+?^${}()|[\]\\]/g, '\\$&');

    if (query.length === 0) {
      return this.data;
    }

    const searchRegex: RegExp = new RegExp(`${query}`, 'gmi');
    const filter = (row: IDataTableHeaderItem[]) => {
      let match: boolean = false;

      Object.keys(row).forEach((key: string) => {
        const column: IDataTableHeaderItem = this.header[key];

        if (typeof column === 'undefined' || column.visible === false) {
          return;
        }

        if (column.visible && match === false) {
          match = searchRegex.exec(row[key].toString().toLowerCase()) !== null;
        }
      });

      return match;
    };

    return this.data.slice(0).filter(filter);
  }

  get sortedData() {
    if (!this.internalSortKey) {
      return this.filteredData;
    }

    const sort = (a: any, b: any) => {
      if (a[this.internalSortKey] < b[this.internalSortKey]) {
        return this.internalSortDirection === 'asc' ? -1 : 1;
      }

      if (a[this.internalSortKey] > b[this.internalSortKey]) {
        return this.internalSortDirection === 'asc' ? 1 : -1;
      }

      return 0;
    };

    return this.filteredData.slice(0).sort(sort);
  }

  get displayData() {
    if (this.maxRows === 0 || this.maxRows >= this.count) {
      return this.sortedData;
    }

    return this.sortedData.slice(
      this.currentPage * this.maxRows,
      (this.currentPage + 1) * this.maxRows
    );
  }

  get columns() {
    return Object.keys(this.header).map((key: string) => {
      const header: IDataTableHeaderItem = this.header[key];

      if (!header.title) {
        header.title = key;
      }

      if (typeof header.visible === 'undefined') {
        header.visible = true;
      }

      if (typeof header.sortable === 'undefined') {
        header.sortable = true;
      }

      header.sortKey = key;

      return header;
    });
  }

  get rows() {
    return this.displayData.map((row: any) => {
      const computedRow: IComputedDataRowCell[] = [];

      Object.keys(this.header).forEach((key: string) => {
        const column: IDataTableHeaderItem = this.header[key];

        const computedColumn: IComputedDataRowCell = {
          key,
          value: row[key],
          visible: column.visible,
          slot: column.slot
        };

        computedRow.push(computedColumn);
      });

      return computedRow;
    });
  }

  columnClick(column: IDataTableHeaderItem) {
    if (
      this.internalSortKey === column.sortKey &&
      this.internalSortDirection === 'desc'
    ) {
      this.internalSortKey = '';
      this.internalSortDirection = 'asc';
    } else if (
      this.internalSortKey === column.sortKey &&
      this.internalSortDirection === 'asc'
    ) {
      this.internalSortDirection = 'desc';
    } else {
      this.internalSortKey = column.sortKey as string;
      this.internalSortDirection = 'asc';
    }
  }

  getRowObject(cells: IComputedDataRowCell[]) {
    const row: any = {};

    cells.forEach((column: IComputedDataRowCell) => {
      row[column.key] = column.value;
    });

    return row;
  }

  rowClick(cells: IComputedDataRowCell[]) {
    this.$emit('click', this.getRowObject(cells));
  }

  paginationClick(page: number) {
    this.currentPage = page - 1;
  }

  getVisibleCells(cells: IComputedDataRowCell[]) {
    return cells.filter((cell: IDataTableHeaderItem) => cell.visible);
  }

  mounted() {
    this.currentPage = this.page;
  }

  @Watch('sortKey', { immediate: true })
  onSortKeyChanged(newSortKey: string) {
    this.internalSortKey = newSortKey;
  }

  @Watch('sortDirection', { immediate: true })
  onSortDirectionChanged(newSortDirection: string) {
    this.internalSortDirection = newSortDirection;
  }
}
</script>

<style lang="scss" scoped>
@import '~styles/app';
@import '~styles/components/datatable';

.dataTableContainer {
  overflow-x: scroll;
}

.dataTable {
  width: 100%;
}

.dataTableNoResults {
  border: $data-table-no-results-border;
  text-align: center;
  padding: $data-table-no-results-padding;
}

.dataTableRow {
  border: $data-table-row-border;
  cursor: pointer;
  min-width: $data-table-min-width;

  &:hover {
    background: $data-table-row-hover-bg;
    color: $data-table-row-hover-color;
  }
}

.dataTableCell {
  border-right: $data-table-row-border;
  padding: $data-table-row-column-padding;
  white-space: nowrap;

  &:last-child {
    border-right: none;
  }
}

.dataTableSelect {
  margin-top: space(4);
}
</style>
