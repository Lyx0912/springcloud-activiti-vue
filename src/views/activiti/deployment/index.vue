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
      <el-table-column align="center" label="流程编号" prop="id" />
      <el-table-column label="部署编号" align="center" prop="deploymentId" />
      <el-table-column label="流程标识" align="center" prop="key" width="150"/>
      <el-table-column label="流程名称"  align="center" prop="name" width="200"/>
      <el-table-column label="资源名称" prop="resourceName" width="200" align="center"/>
      <el-table-column align="center" prop="created_at" label="操作" width="200" >
        <template v-slot="scope">
          <el-button v-if="scope.row.status===2" type="primary" icon="el-icon-video-play" size="mini" @click="changeStatus(scope.row)">激活</el-button>
          <el-button v-else type="warning" icon="el-icon-video-pause" size="mini" @click="changeStatus(scope.row)">挂起</el-button>
          <el-button type="danger" size="mini" @click="handleDelete(scope.row.deploymentId)">删除</el-button>
        </template>
      </el-table-column>
    </el-table>
    <pagination v-show="total>0" :total="total" :page.sync="queryParams.pageNo" :limit.sync="queryParams.pageSize" @pagination="getList" />
  </div>
</template>

<script>
import {changeStatus, deleteDef, list} from "@/api/activiti/def"
import pagination from '@/components/Pagination'
import {deleteModel} from "@/api/activiti/model";

export default {
  components: { pagination },
  data() {
    return {
      list: [],
      total: 0,
      queryParams: {
        pageNo: 1,
        pageSize: 10,
        keyword: ''
      }
    }
  },
  created() {
    this.getList()
  },
  methods: {
    getList() {
      list(this.queryParams).then(res=>{
        this.list = res.data.list
        this.total = res.data.total
      })
    },
    handleDelete(id) {
      this.$confirm(`确定删除此流程吗？`, '是否继续?', '提示', {
        confirmButtonText: '删除',
        cancelButtonText: '算了吧',
        type: 'warning'
      }).then(() => {
        deleteDef(id).then(res => {
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
    },
    changeStatus(row) {
      this.$confirm(`确定${row.status === 1 ? '挂起' : '激活'}此流程吗？`, '是否继续?', '提示', {
        confirmButtonText: '删除',
        cancelButtonText: '算了吧',
        type: 'warning'
      }).then(() => {
        console.log(row.id)
        changeStatus(row.id).then(res => {
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
