<template>
  <div class="pod-container">
    <!-- 搜索框 -->
    <el-input
      v-model="searchQuery"
      placeholder="搜索Pod"
      clearable
      @clear="handleSearch"
      @input="handleSearch"
      class="search-input"
    />

    <!-- 新增Pod按钮 -->
    <el-button type="success" @click="handleAdd" class="add-pod-button">新增Pod</el-button>

    <el-table
      :data="filteredAndSortedData"
      border
      stripe
      style="width: 100%; margin-top: 20px;"
      @sort-change="handleSortChange"
    >
      <el-table-column prop="name" label="Pod名称" min-width="180" sortable />
      <el-table-column prop="namespace" label="命名空间" min-width="180" sortable />
      <el-table-column prop="status" label="状态" min-width="120" sortable>
        <template #default="{ row }">
          <el-tag :type="getStatusTagType(row.status)">
            {{ row.status }}
          </el-tag>
        </template>
      </el-table-column>
      <el-table-column prop="ip" label="IP地址" min-width="180" sortable />
      <el-table-column prop="node" label="节点" min-width="180" sortable />
      <el-table-column prop="createdAt" label="创建时间" min-width="200" sortable />
      <el-table-column label="操作" min-width="200">
        <template #default="{ row }">
          <el-button type="primary" size="small" @click="handleEdit(row)">编辑</el-button>
          <el-button type="danger" size="small" @click="handleDelete(row)">删除</el-button>
        </template>
      </el-table-column>
    </el-table>

    <!-- 分页组件 -->
    <el-pagination
      v-model:currentPage="currentPage"
      :page-size="pageSize"
      :total="totalPods"
      layout="total, prev, pager, next, jumper"
      @current-change="handlePageChange"
    />

    <!-- 新增/编辑Pod对话框 -->
    <el-dialog :title="dialogTitle" v-model="dialogVisible">
      <el-form :model="currentPod" label-width="120px">
        <el-form-item label="Pod名称">
          <el-input v-model="currentPod.name" />
        </el-form-item>
        <el-form-item label="命名空间">
          <el-input v-model="currentPod.namespace" />
        </el-form-item>
        <el-form-item label="镜像">
          <el-input v-model="currentPod.image" />
        </el-form-item>
      </el-form>
      <template #footer>
        <el-button @click="dialogVisible = false">取消</el-button>
        <el-button type="primary" @click="handleSave">保存</el-button>
      </template>
    </el-dialog>
  </div>
</template>

<script lang="ts">
import { defineComponent, onMounted, ref, computed } from "vue"
import { ElMessage, ElMessageBox } from "element-plus"
import { request } from "@/utils/service"

interface Pod {
  name: string
  namespace: string
  labels: { [key: string]: string }
  annotations: { [key: string]: string } | null
  status: string
  ip: string
  node: string
  createdAt: string
  image: string
}

export default defineComponent({
  name: "Pod",
  setup() {
    const podData = ref<Pod[]>([])
    const currentPage = ref(1)
    const pageSize = ref(10)
    const totalPods = ref(0)
    const dialogVisible = ref(false)
    const dialogTitle = ref("新增Pod")
    const currentPod = ref<Pod>({
      name: "",
      namespace: "default",
      labels: {},
      annotations: null,
      status: "Pending",
      ip: "",
      node: "",
      createdAt: "",
      image: ""
    })
    const searchQuery = ref("")
    const sortKey = ref("")
    const sortOrder = ref("")

    const currentPageData = computed(() => {
      const start = (currentPage.value - 1) * pageSize.value
      const end = start + pageSize.value
      return filteredAndSortedData.value.slice(start, end)
    })

    const filteredAndSortedData = computed(() => {
      let data = podData.value

      // 搜索过滤
      if (searchQuery.value) {
        const query = searchQuery.value.toLowerCase()
        data = data.filter(
          (pod) =>
            pod.name.toLowerCase().includes(query) ||
            pod.namespace.toLowerCase().includes(query) ||
            pod.status.toLowerCase().includes(query) ||
            pod.ip.toLowerCase().includes(query) ||
            pod.node.toLowerCase().includes(query)
        )
      }

      // 排序
      if (sortKey.value) {
        data = data.slice().sort((a, b) => {
          const aValue = a[sortKey.value]
          const bValue = b[sortKey.value]
          if (sortOrder.value === "ascending") {
            return aValue > bValue ? 1 : -1
          } else if (sortOrder.value === "descending") {
            return aValue < bValue ? 1 : -1
          }
          return 0
        })
      }

      return data
    })

    const fetchPodData = async () => {
      try {
        const response = await request({
          url: "/api/v1/namespaces/kube-system/pods",
          method: "get",
          baseURL: "http://192.168.1.100:8080"
        })

        console.log("API response:", response)
        
        if (response.code === 200 && response.data.items) {
          podData.value = response.data.items.map((item: any) => ({
            name: item.name,
            namespace: item.namespace,
            labels: item.labels || {},
            annotations: item.annotations || null,
            status: item.status,
            ip: item.ip,
            node: item.node,
            createdAt: item.createdAt
          }))
          totalPods.value = response.data.total
          console.log("Processed pod data:", podData.value)
        } else {
          console.error("Invalid response format:", response)
          ElMessage.error("获取Pod数据失败: " + response.message)
        }
      } catch (error) {
        console.error("获取Pod数据失败:", error)
        ElMessage.error("获取Pod数据失败")
      }
    }

    const handlePageChange = (page: number) => {
      currentPage.value = page
      fetchPodData()
    }

    const handleSearch = () => {
      currentPage.value = 1
    }

    const handleSortChange = ({ prop, order }: { prop: string; order: string }) => {
      sortKey.value = prop
      sortOrder.value = order
    }

    const handleAdd = () => {
      dialogTitle.value = "新增Pod"
      currentPod.value = {
        name: "",
        namespace: "default",
        labels: {},
        annotations: null,
        status: "Pending",
        ip: "",
        node: "",
        createdAt: "",
        image: ""
      }
      dialogVisible.value = true
    }

    const handleEdit = (pod: Pod) => {
      dialogTitle.value = "编辑Pod"
      currentPod.value = JSON.parse(JSON.stringify(pod))
      dialogVisible.value = true
    }

    const handleSave = async () => {
      try {
        if (dialogTitle.value === "新增Pod") {
          await request({
            url: "/api/v1/namespaces/default/pods",
            method: "post",
            data: currentPod.value,
            baseURL: "http://192.168.1.100:8080" // 可根据需要调整 baseURL
          })
          ElMessage.success("Pod新增成功")
        } else {
          await request({
            url: `/api/v1/namespaces/default/pods/${currentPod.value.name}`,
            method: "put",
            data: currentPod.value,
            baseURL: "http://192.168.1.100:8080" // 可根据需要调整 baseURL
          })
          ElMessage.success("Pod编辑成功")
        }
        dialogVisible.value = false
        fetchPodData()
      } catch (error) {
        console.error("保存Pod失败:", error)
        ElMessage.error("保存Pod失败")
      }
    }

    const handleDelete = (pod: Pod) => {
      ElMessageBox.confirm(`确定删除Pod ${pod.name} 吗？`, "提示", {
        confirmButtonText: "确定",
        cancelButtonText: "取消",
        type: "warning"
      }).then(async () => {
        try {
          await request({
            url: `/api/v1/namespaces/default/pods/${pod.name}`,
            method: "delete",
            baseURL: "http://192.168.1.100:8080" // 可根据需要调整 baseURL
          })
          ElMessage.success("Pod删除成功")
          fetchPodData()
        } catch (error) {
          console.error("删除Pod失败:", error)
          ElMessage.error("删除Pod失败")
        }
      })
    }

    const getStatusTagType = (status: string) => {
      switch (status) {
        case "Running":
          return "success"
        case "Pending":
          return "warning"
        case "Failed":
          return "danger"
        default:
          return "info"
      }
    }

    onMounted(() => {
      fetchPodData()
    })

    return {
      podData,
      currentPage,
      pageSize,
      totalPods,
      currentPageData,
      filteredAndSortedData,
      searchQuery,
      sortKey,
      sortOrder,
      handlePageChange,
      handleSearch,
      handleSortChange,
      dialogVisible,
      dialogTitle,
      currentPod,
      handleAdd,
      handleEdit,
      handleSave,
      handleDelete,
      getStatusTagType
    }
  }
})
</script>

<style scoped>
.pod-container {
  padding: 20px;
}

.el-tag {
  font-size: 12px;
}

.el-table-column .cell {
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.add-pod-button {
  margin-bottom: 20px;
}

.search-input {
  margin-bottom: 20px;
  width: 300px;
}
</style>