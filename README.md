# element-datatable-plus
一个基于element-plus的表格组件

## Installation
```bash
yarn add element-datatable-plus
# or
npm install element-datatable-plus
```

## Adding to project
In you `main.js` or `app.js`
```js
import { createApp } from 'vue'
import ElementPlus from 'element-plus'
import ElementDatatablePlus from 'element-datatable-plus'

const app = createApp(/*...*/)
app.use(ElementPlus)
app.use(ElementDatatablePlus)

// or 

createApp(/*...*/)
  .use(ElementPlus)
  .use(ElementDatatablePlus)
```

## Basic usage
Besides from `tableData` and `tableHeaders` most [options from the default table](https://element-plus.org/en-US/component/table.html#table-attributes) work as descrived in their documentation
```vue
<template>
  <ElementDatatablePlus
    :tableData="items"
    :tableHeaders="headers"
  >
    <template #fullName="scope">
      {{`${scope.row.firstName} ${scope.row.lastName}`}}
    </template>
  <ElementDatatablePlus>
</template>

<script>
export default {
  setup() {
    const items = [
      { id: 1, username: 'liupei', firstName: 'Liu', lastName: 'Peiqiang' },
      // ...
    ]
    const headers = [
      { prop: 'id', label: 'Id', sortable: true },
      { prop: 'username', label: 'Username', sortable: true },
      { prop: 'firstName', label: 'First name', sortable: true },
      { prop: 'lastName', label: 'Last name', sortable: true },
      { prop: 'fullName', label: 'Full name' },
    ]
    
    return {
      items, headers
    }
  }
}
</script>
```
