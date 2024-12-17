<script setup lang="ts">

import { onMounted, reactive, ref } from "vue";
import useAxios from "@/api";
import { FormColumn, PageResult, TableColumn, TableOptButton, ViewColumnConfig, ViewConfig } from "#/conponent.ts";
import { ElMessage, Sort } from "element-plus";
import MainTable from "@/components/MainTable.vue";
import { booleanFormat, datetimeFormat } from "@/utils/util";
import { MD5 } from "crypto-js";

const { sendPut, sendGet, sendPost } = useAxios();
//页面配置
const displayControl = reactive({
  loading: false,
  addDialog: false,
  isEdit: false,

});
const viewConfig = ref<ViewConfig>();
const viewColumnConfig = ref<ViewColumnConfig[]>();
const viewData = ref<object[]>([]);
const viewPageData = ref<PageResult>();
const optButtons = ref<TableOptButton[]>([]);
const gridColumn: TableColumn[] = [];
const addFormRef = ref();
const addFormData = ref({});
const addFormItems = ref<FormColumn[]>([]);
const addFormRules = ref({});
const searchItem = ref();
const searchData = reactive({
  limit: 20,
  page: 1,
  data: {},
  dir: "ASC",
});
const sort: Sort = { prop: "createDatetime", order: "descending" };

/**
 * 取回页面配置
 * @param viewId 页面ID
 */
const getViewConfig = (viewId: number) => {
  sendGet<ViewConfig>(`/system/manage/view/${viewId}`)
    .then(req => {
      viewConfig.value = req;
    })
    .then(() => {
      initView(viewId);
    });
};

/**
 * 取回页面列配置
 * @param viewId 页面ID
 */
const getViewColumnConfig = (viewId: number) => {
  sendGet<ViewColumnConfig[]>(`/system/manage/view/column/list/${viewId}`)
    .then(req => {
      viewColumnConfig.value = req;
    })
    .then(() => {
      //视图表头初始化
      initViewColumns();
      initForm();
    });
};

/**
 * 初始化视图表头
 */
const initViewColumns = () => {
  viewColumnConfig.value?.map(viewColumn => {
    if (viewColumn.showColumn) {
      const tableColumn: TableColumn = {
        label: viewColumn.columnLabel,
        prop: viewColumn.columnName,
        width: viewColumn.showWidth,
        fixed: viewColumn.showFixed,
        sortable: viewColumn.sortable,
        format: getColumnFormat(viewColumn.showFormat),
      };
      gridColumn.push(tableColumn);
    }
  });
};

/**
 * 初始化表格
 */
const initForm = () => {
  if (viewConfig.value?.optAdd) {
    viewColumnConfig.value?.map(column => {
      //表格数据
      addFormItems.value?.push({
        label: column.columnLabel,
        prop: column.columnName,
        type: column.columnType,
        addHandle: column.addHandle,
        editHandle: column.editHandle,
      });

      //表格规则
      if (column.ruleRequired) {
        addFormRules.value[column.columnName] = {
          required: true,
          message: `${column.columnLabel}不能为空`,
        };
      }
    });
    console.log(addFormItems.value);
  }
};

const getColumnFormat = (formatName?: string) => {
  switch (formatName) {
    case "datetimeFormat":
      return datetimeFormat;
    case "booleanFormat":
      return booleanFormat;
    default:
      return undefined;
  }
};

/**
 * 初始化页面
 */
const initView = (viewId: number) => {
  if (viewConfig.value) {
    const config = viewConfig.value;
    if (config.initData) {
      //页面数据
      getViewData();
      getViewColumnConfig(viewId);
      initOptButton();
    }
  }
};

/**
 * 初始化操作按钮
 */
const initOptButton = () => {
  if (viewConfig.value) {
    const config = viewConfig.value;
    if (config.optAdd) {
      const optButton: TableOptButton = {
        label: config.optAddLabel,
        selectHandler: false,
        type: "primary",
        handle: showAddDialog,
      };
      optButtons.value.push(optButton);
    }
  }
};

/**
 * 显示添加窗口
 */
const showAddDialog = () => {
  displayControl.addDialog = true;
};

/**
 * 关闭添加窗口
 */
const hideAddDialog = () => {
  addFormRef.value.resetFields();
  displayControl.addDialog = false;
};

/**
 * 添加处理器
 */
const saveHandle = () => {
  addFormRef.value.validate(valid => {
    if (valid) {
      if (viewConfig.value) {
        if (displayControl.isEdit) {
          sendPut(viewConfig.value.optEditUrl, addFormData).then(() => {
            getViewData();
            hideAddDialog();
            ElMessage.success("操作成功");
          });
        } else {
          const submitData = JSON.parse(JSON.stringify(addFormData.value));
          submitData.password = MD5(submitData.password).toString().toLowerCase();
          sendPost(viewConfig.value.optAddUrl, submitData).then(() => {
            getViewData();
            hideAddDialog();
            ElMessage.success("操作成功");
          });
        }
      }
    }
  });
};

/**
 * 取回数据
 */
const getViewData = () => {
  if (viewConfig.value) {
    const config = viewConfig.value;
    sendPost(config.initDataUrl, searchData)
      .then((rep: any) => {
        viewPageData.value = rep;
        viewData.value = viewPageData.value ? viewPageData.value.records : [];
      });
  }
};


onMounted(() => {
  getViewConfig(2);
});
</script>

<template>
  <MainTable
    :grid-column="gridColumn"
    :grid-data="viewData"
    :page-data="viewPageData"
    :search-item="searchItem"
    :button-item="optButtons"
    :multi-select="true"
    :loading="displayControl.loading"
    :grid-data-fun="getViewData"
    :default-sort="sort"
    :page="true"
  />

  <el-drawer
    v-model="displayControl.addDialog"
    :title="displayControl.isEdit ? '编辑' : '添加'+'管理员'"
    direction="rtl"
    :before-close="hideAddDialog"
    size="400"
  >
    <el-form label-width="80" ref="addFormRef" :model="addFormData" :rules="addFormRules" label-position="top">
      <el-space fill>
        <el-alert v-if="!displayControl.isEdit" type="info" show-icon :closable="false">
          <p>"用户名"为6-20位字母开头，且只能包含字母数字下划线</p>
        </el-alert>
        <template v-for="item in addFormItems">
          <el-form-item v-if="item.addHandle" :label="item.label" :prop="item.prop">
            <el-input :type="item.type" v-model="addFormData[item.prop]" />
          </el-form-item>
        </template>
      </el-space>
    </el-form>
    <template #footer>
      <div class="dialog-footer">
        <el-button @click="hideAddDialog">取消</el-button>
        <el-button type="primary" @click="saveHandle">确认</el-button>
      </div>
    </template>
  </el-drawer>
</template>

<style scoped lang="scss">

</style>