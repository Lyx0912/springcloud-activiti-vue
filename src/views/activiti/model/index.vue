<template>
  <div class="app-container">
    <el-table
      :row-style="{height:'45px'}"
      :cell-style="{padding:'0px'}"
      v-loading="listLoading"
      :data="list"
      element-loading-text="Loading"
      @selection-change="handleSelectionChange"
      border
      fit
      highlight-current-row
    >
      <el-table-column
        type="selection"
        width="55">
      </el-table-column>
      <el-table-column align="center" label="模型名称" prop="name" />
      <el-table-column label="模型标识" align="center" prop="key" />
      <el-table-column label="模型分类" align="center" prop="category" width="150"/>
      <el-table-column label="创建时间"  align="center" prop="createTime" />
      <el-table-column class-name="status-col" label="部署编号" width="110" prop="deploymentId" align="center"/>
      <el-table-column align="center" prop="lastUpdateTime" label="最近更新时间" width="150" />
      <el-table-column align="center" prop="created_at" label="操作" >
        <template v-slot="scope">
          <el-button type="primary" size="mini" @click="handleEdit(scope.row.id)">设计</el-button>
          <el-button type="warning" size="mini" v-if="!scope.row.deploymentId" @click="handleDeploy(scope.row.id)">发布</el-button>
          <el-button type="success" size="mini" @click="handleDownLoad(scope.row.name,scope.row.id)">导出</el-button>
          <el-button type="danger" size="mini" @click="handleDelete(scope.row.id)">删除</el-button>
        </template>
      </el-table-column>
    </el-table>
    <a ref="downloadLink" hidden />
    <pagination v-show="total>0" :total="total" :page.sync="queryParams.pageNo" :limit.sync="queryParams.pageSize" @pagination="getList" />
  </div>
</template>

<script>
import { getModelList, deleteModel, downloadResource, deployment } from '@/api/activiti/model'
import pagination from '@/components/Pagination'

export default {
  components: { pagination },
  data() {
    return {
      listLoading: true,
      total: 0,
      queryParams: {
        pageNo: 1,
        pageSize: 10,
        keyword: ''
      },
      list: null
    }
  },
  created() {
    this.getList()
  },
  methods: {
    handleDeploy(modelId) {
      deployment(modelId).then(res => {
        this.$message({
          type: 'success',
          message: '发布成功!'
        })
        this.getList()
      })
    },
    handleDownLoad(modelName, id) {
      downloadResource(id).then(res => {
        // 这里就获取到了之前设置的隐藏链接
        const downloadLink = this.$refs.downloadLink
        // 把输就转换为URI，下载要用到的
        const encodedData = encodeURIComponent(res.data)

        if (res.data) {
          // 将数据给到链接
          downloadLink.href = 'data:application/bpmn20-xml;charset=UTF-8,' + encodedData
          // 设置文件名
          downloadLink.download = modelName + '.bpmn20.xml'
          // 触发点击事件开始下载
          downloadLink.click()
        }
      })
    },
    handleEdit(modelId) {
      this.$router.push({ path: '/design', query: { modelId }})
    },
    getList() {
      this.listLoading = true
      console.log(this.queryParams)
      getModelList(this.queryParams).then(response => {
        this.list = response.data.list
        this.total = response.data.total
        this.listLoading = false
      })
    },
    handleDelete(id) {
      this.$confirm(`确定删除此模型吗？`, '是否继续?', '提示', {
        confirmButtonText: '删除',
        cancelButtonText: '算了吧',
        type: 'warning'
      }).then(() => {
        deleteModel(id).then(res => {
          this.$message({
            type: 'success',
            message: '操作成功!'
          })
          this.getList()
        })
      }).catch(() => {
        this.$message({
          type: 'info',
          message: '取消操作'
        })
      })
    }
  }
}
</script>

<style scoped>

</style>
