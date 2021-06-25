<template>
  <div class="element-datatable-plus">
    <el-table
      ref="dtRef"
      v-loading="isLoading"
      :data="viewList"
      :border="border"
      :stripe="stripe"
      style="width: 100%"
      :class="{ 'edp-clickable': !!onRowClick }"
      @row-click="handleRowClick"
      @selection-change="handleSelectionChange"
      @sort-change="handleSortChange"
    >
      <el-table-column
        v-if="onSelectionChange"
        :selectable="selectable"
        type="selection"
        :align="align"
      />
      <el-table-column
        v-for="(header, index) in tableHeaders"
        :key="`table-column-${index}`"
        :prop="header.prop"
        :label="header.label"
        :sortable="
          header.sortable || header.sortProp !== undefined ? 'custom' : false
        "
        :sort-orders="['ascending', 'descending', null]"
        :width="header.width ? header.width : ''"
        :align="align"
      >
        <template #default="scope">
          <slot :name="header.prop" :row="scope.row" :$index="scope.$index">
            <span>
              {{
                header.formatter
                  ? header.formatter(
                      extractProp(scope.row, header.prop, header.default),
                      scope.row
                    )
                  : extractProp(scope.row, header.prop, header.default)
              }}
            </span>
          </slot>
        </template>
      </el-table-column>
    </el-table>
    <div class="edp-bottom-bar">
      <slot name="__leftBottom" />
      <el-pagination
        v-if="pagination"
        ref="pagination"
        :disabled="isLoading"
        :current-page="currentPage"
        :page-size="size"
        :page-sizes="sizeOptions"
        :total="total"
        :layout="pagination"
        background
        @size-change="handleSizeChange"
        @current-change="handlePageChange"
      />
    </div>
  </div>
</template>
<script>
import axios from "axios";
import _ from "lodash";
import { ref, toRefs, onMounted, computed, watch } from "vue";

export default {
  name: "ElementDatatablePlus",
  props: {
    tableHeaders: {
      // 表头结构数据
      type: Array,
      default: () => {
        return [];
      },
    },
    tableData: {
      // 表格数据，如果有值则默认为前端分页模式
      type: Array,
      default: null,
    },
    page: {
      // 初始化时的页数
      type: Number,
      default: 1,
    },
    size: {
      // 初始化时的每页条数
      type: Number,
      default: 10,
    },
    pagination: {
      // 分页组件布局
      type: [String, Boolean],
      default: "total, sizes, prev, pager, next, jumper",
    },
    sizeOptions: {
      // 每页条数的可选项
      type: Array,
      default: () => {
        return [5, 10, 20, 50, 100];
      },
    },
    stripe: {
      // 是否显示间隔条纹
      type: Boolean,
      default: true,
    },
    border: {
      // 是否显示边框
      type: Boolean,
      default: true,
    },
    align: {
      // 对齐方式
      type: String,
      default: "left",
    },
    url: {
      // 请求数据的url
      type: String,
      default: "",
    },
    dataName: {
      // 在返回的数据中定位数据list的位置，可以用点连接
      type: String,
      default: "data",
    },
    totalName: {
      // 在返回的数据中定位数据总数的位置，可以用点连接
      type: String,
      default: "total",
    },
    pageName: {
      // 分页参数的名称
      type: String,
      default: "page",
    },
    sizeName: {
      // 每页条数参数的名称
      type: String,
      default: "size",
    },
    requestInstance: {
      // Axios实例，如没有则使用默认实例
      type: [Object, Function],
      default: null,
    },
    queryParams: {
      // 在这里指定查询参数
      type: Object,
      default: null,
    },
    onRowClick: {
      // 行点击事件的回调
      type: Function,
      default: null,
    },
    onSelectionChange: {
      // 选中行事件的回调
      type: Function,
      default: null,
    },
    selectable: {
      // 指定可选行的规则
      type: Function,
      default: null,
    },
    orderName: {
      // 自定义升降序名称
      type: Array,
      default: () => ["asc", "desc"],
    },
    preprocess: {
      // 预处理回调
      type: Function,
      default: null,
    },
  },
  setup(props) {
    const dtRef = ref(null);
    const {
      tableHeaders,
      tableData,
      requestInstance,
      queryParams,
      url,
      dataName,
      totalName,
      onRowClick,
      onSelectionChange,
      orderName,
    } = toRefs(props);

    const currentPage = ref(props.page); // 当前页数
    const currentSize = ref(props.size); // 当前每页条数
    const total = ref(0); // 表格总条数
    const viewList = ref([]); // 表格显示的部分
    const isLoading = ref(false); // 加载状态开关

    let sortedTableData = tableData.value ? [...tableData.value] : []; // 筛选后的数据
    const currentSortInfo = ref({}); // 排序信息

    const isFrontendMode = computed(() => {
      // 是否是后端模式
      return !!tableData.value;
    });

    const innerReqIns = axios.create();

    const reqIns = computed(() => {
      return requestInstance.value || innerReqIns;
    });

    const isSorted = computed(() => {
      return !!currentSortInfo.value.order;
    });

    const extractProp = (row, prop, defaultValue = "") => {
      return _.get(row, prop, defaultValue);
    };

    const reloadViewList = () => {
      const dataSource = isSorted.value ? sortedTableData : tableData.value;
      isLoading.value = true;
      if (isFrontendMode.value) {
        // 前端模式
        const from = (currentPage.value - 1) * currentSize.value;
        const to = from + currentSize.value;
        viewList.value = dataSource.slice(from, to);
        total.value = dataSource.length;
        isLoading.value = false;
      } else {
        // 后端模式
        const innerParams = {
          [props.pageName]: currentPage.value,
          [props.sizeName]: currentSize.value,
        };
        const sortParams = {};
        if (isSorted.value) {
          const sortCol = _.find(tableHeaders.value, {
            prop: currentSortInfo.value.prop,
          });
          const orderNameMap = {
            ascending: orderName.value[0],
            descending: orderName.value[1],
          };
          sortParams[sortCol.sortProp || sortCol.prop] = currentSortInfo.value
            .order
            ? orderNameMap[currentSortInfo.value.order]
            : undefined;
        }
        const params = { ...innerParams, ...sortParams, ...queryParams.value };
        reqIns.value
          .get(url.value, {
            params,
          })
          .then((res) => {
            let resData = res.data;
            if (props.preprocess) {
              resData = props.preprocess(resData, res);
            }
            viewList.value = _.get(resData, dataName.value);
            total.value = _.get(resData, totalName.value);
            isLoading.value = false;
          });
      }
    };

    watch(tableData, (newVal) => {
      clearSort();
      sortedTableData = [...newVal];
      currentPage.value = 1;
      reloadViewList();
    });

    watch(queryParams, () => {
      currentPage.value = 1;
      reloadViewList();
    });

    onMounted(() => {
      reloadViewList();
    });

    const handlePageChange = (page) => {
      currentPage.value = page;
      reloadViewList();
    };

    const handleSizeChange = (size) => {
      currentSize.value = size;
      // 解决改变size后，pagination组件自动调整页数后，触发handlePageChange重复请求接口问题
      if (Math.ceil(total.value / size) < currentPage.value) {
        return;
      }
      reloadViewList();
    };

    const handleRowClick = (row) => {
      if (onRowClick.value) onRowClick.value(row);
    };

    const handleSelectionChange = (rows) => {
      onSelectionChange.value(rows);
    };

    const toggleRowSelection = (row, selected) => {
      dtRef.value.toggleRowSelection(row, selected);
    };

    const clearSelection = () => {
      dtRef.value.clearSelection();
    };

    const handleSortChange = (info) => {
      isLoading.value = true;
      currentSortInfo.value = info;
      if (isFrontendMode.value) {
        // 前端模式
        if (info.order) {
          sortedTableData.sort((a, b) => {
            if (info.order === "ascending") {
              return _.get(a, info.prop) > _.get(b, info.prop) ? 1 : -1;
            } else if (info.order === "descending") {
              return _.get(a, info.prop) > _.get(b, info.prop) ? -1 : 1;
            }
          });
        }
        reloadViewList();
      } else {
        // 后端模式
        reloadViewList();
      }
    };

    const clearSort = () => {
      currentSortInfo.value = {};
      dtRef.value.clearSort();
    };

    const reload = (holdPage, isDelete) => {
      let offset = 0;
      if (isDelete && viewList.value.length <= 1) {
        offset = 1;
      }
      currentPage.value = holdPage ? currentPage.value - offset || 1 : 1;
      reloadViewList();
    };

    return {
      dtRef,
      isLoading,
      currentPage,
      currentSize,
      total,
      viewList,
      extractProp,
      handlePageChange,
      handleSizeChange,
      handleRowClick,
      handleSelectionChange,
      toggleRowSelection,
      clearSelection,
      handleSortChange,
      clearSort,
      reload,
    };
  },
};
</script>

<style scoped>
.element-datatable-plus:deep(.el-pagination__total),
.element-datatable-plus:deep(.el-pagination__sizes) {
  float: left;
}

.element-datatable-plus:deep(.edp-clickable .el-table__body tr) {
  cursor: pointer;
}

.edp-bottom-bar {
  display: flex;
  align-items: center;
  margin-top: 10px;
}

.el-pagination {
  flex-grow: 1;
  text-align: right;
}
</style>
