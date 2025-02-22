<script setup lang="ts">
import { useI18n } from '@/hooks/web/useI18n'
import { ElInput, ElCard, ElTree, ElTreeSelect, ElMessage, ElMessageBox } from 'element-plus'
import { handleTree } from '@/utils/tree'
import { onMounted, ref, unref, watch } from 'vue'
import * as DeptApi from '@/api/system/dept'
import { Form, FormExpose } from '@/components/Form'
import { modelSchema } from './dept.data'
import { DeptVO } from '@/api/system/dept/types'
interface Tree {
  id: number
  name: string
  children?: Tree[]
}

const defaultProps = {
  children: 'children',
  label: 'name',
  value: 'id'
}
const { t } = useI18n() // 国际化
const loading = ref(false) // 遮罩层
const dialogVisible = ref(false) // 是否显示弹出层
const showForm = ref(false) // 显示form表单
const formTitle = ref('部门信息') // 显示form标题
const deptParentId = ref(0) // 上级ID
// 创建form表单
const formRef = ref<FormExpose>()

// ========== 创建部门树结构 ==========
const filterText = ref('')
const deptOptions = ref([]) // 树形结构
const treeRef = ref<InstanceType<typeof ElTree>>()
const getTree = async () => {
  const res = await DeptApi.listSimpleDeptApi()
  deptOptions.value = handleTree(res)
}
const filterNode = (value: string, data: Tree) => {
  if (!value) return true
  return data.name.includes(value)
}
watch(filterText, (val) => {
  treeRef.value!.filter(val)
})
// 新增
const handleAdd = (data: { id: number }) => {
  // 重置表单
  deptParentId.value = data.id
  formTitle.value = '新增部门'
  unref(formRef)?.getElFormRef()?.resetFields()
  showForm.value = true
}
// 编辑
const handleUpdate = async (data: { id: number }) => {
  showForm.value = true
  const res = await DeptApi.getDeptApi(data.id)
  formTitle.value = '修改- ' + res?.name
  deptParentId.value = res.parentId
  unref(formRef)?.setValues(res)
}
// 删除
const handleDelete = async (data: { id: number }) => {
  ElMessageBox.confirm(t('common.delDataMessage'), t('common.confirmTitle'), {
    confirmButtonText: t('common.ok'),
    cancelButtonText: t('common.cancel'),
    type: 'warning'
  })
    .then(async () => {
      await DeptApi.deleteDeptApi(data.id)
      ElMessage.success(t('common.delSuccess'))
    })
    .catch(() => {})
  await getTree()
}
// 提交按钮
const submitForm = async () => {
  loading.value = true
  // 提交请求
  try {
    const data = unref(formRef)?.formModel as DeptVO
    data.parentId = deptParentId.value
    // TODO: 表单提交待完善
    if (formTitle.value.startsWith('新增')) {
      await DeptApi.createDeptApi(data)
    } else if (formTitle.value.startsWith('修改')) {
      await DeptApi.updateDeptApi(data)
    }
    // 操作成功，重新加载列表
    dialogVisible.value = false
  } finally {
    loading.value = false
  }
}
onMounted(async () => {
  await getTree()
})
</script>
<template>
  <div class="flex">
    <el-card class="w-1/3 dept" :gutter="12" shadow="always">
      <template #header>
        <div class="card-header">
          <span>部门列表</span>
          <el-button type="primary" v-hasPermi="['system:dept:create']" @click="handleAdd">
            新增根节点
          </el-button>
        </div>
      </template>
      <div class="custom-tree-container">
        <!-- <p>部门列表</p> -->
        <!-- 操作工具栏 -->
        <el-input v-model="filterText" placeholder="搜索部门" />
        <el-tree
          ref="treeRef"
          node-key="id"
          :data="deptOptions"
          :props="defaultProps"
          :highlight-current="true"
          default-expand-all
          :filter-node-method="filterNode"
        >
          <template #default="{ node, data }">
            <span class="custom-tree-node">
              <span>{{ node.label }}</span>
              <span>
                <el-button link v-hasPermi="['system:dept:create']" @click="handleAdd(data)">
                  <Icon icon="ep:plus" class="mr-1px" />
                </el-button>
                <el-button link v-hasPermi="['system:dept:update']" @click="handleUpdate(data)">
                  <Icon icon="ep:edit" class="mr-1px" />
                </el-button>
                <el-button link v-hasPermi="['system:dept:delete']" @click="handleDelete(data)">
                  <Icon icon="ep:delete" class="mr-1px" />
                </el-button>
              </span>
            </span>
          </template>
        </el-tree>
      </div>
    </el-card>
    <el-card class="w-2/3 dept" style="margin-left: 10px" :gutter="12" shadow="hover">
      <template #header>
        <div class="card-header">
          <span>{{ formTitle }}</span>
        </div>
      </template>
      <div v-if="!showForm">
        <span><p>请从左侧选择部门</p></span>
      </div>
      <div v-if="showForm">
        <!-- 操作工具栏 -->
        <Form :schema="modelSchema" ref="formRef">
          <template #parentId>
            <el-tree-select
              node-key="id"
              v-model="deptParentId"
              :props="defaultProps"
              :data="deptOptions"
              check-strictly
            />
          </template>
        </Form>
        <!-- 操作按钮 -->
        <el-button
          type="primary"
          v-hasPermi="['system:dept:update']"
          :loading="loading"
          @click="submitForm"
        >
          {{ t('action.save') }}
        </el-button>
        <el-button type="danger" @click="showForm = false">取消</el-button>
      </div>
    </el-card>
  </div>
</template>
<style scoped>
.dept {
  height: 600px;
  max-height: 1800px;
}
.card-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
}
.custom-tree-node {
  flex: 1;
  display: flex;
  align-items: center;
  justify-content: space-between;
  font-size: 14px;
  padding-right: 8px;
}
</style>
