<template>
  <div class="app-container">
    <el-row style="margin-bottom: 12px">
      <el-col :span="5">
        <el-button icon="el-icon-plus" size="mini" type="warning" @click="handleAdd()" >新增</el-button>
      </el-col>
      <el-col :span="19">
        <el-form :model="queryParams" ref="queryForm" size="small" :inline="true" style="float: right">
          <el-form-item label="状态" prop="status">
            <el-select v-model="queryParams.result" placeholder="状态" clearable>
              <el-option
                v-for="dict in resultOptions"
                :key="dict.value"
                :label="dict.label"
                :value="dict.value"
              />
            </el-select>
          </el-form-item>&nbsp;
          <el-form-item label="类型" prop="status">
            <el-select v-model="queryParams.type" placeholder="请假类型" clearable>
              <el-option
                v-for="dict in options"
                :key="dict.value"
                :label="dict.label"
                :value="dict.value"
              />
            </el-select>
          </el-form-item>&nbsp;
          <el-form-item label="申请时间" >
            <el-date-picker
              value-format="yyyy-MM-dd HH:mm:ss"
              v-model="queryParams.startTime"
              type="date"
              placeholder="选择开始时间">
            </el-date-picker>-
            <el-date-picker
              value-format="yyyy-MM-dd HH:mm:ss"
              v-model="queryParams.endTime"
              type="date"
              placeholder="选择时间">
            </el-date-picker>
          </el-form-item>
          <el-form-item>
            <el-button type="primary" icon="el-icon-search" size="mini" @click="getList">搜索</el-button>
            <el-button icon="el-icon-refresh" size="mini" @click="resetQuery">重置</el-button>
          </el-form-item>
        </el-form>
      </el-col>
    </el-row>
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
      <el-table-column align="center" label="编号" prop="id" width="50" />
      <el-table-column align="center" label="审批结果" prop="result" width="80">
        <template v-slot="scope">
          <span>{{ scope.row.result===0?'审批中':scope.row.result===1?'同意':'拒绝' }}</span>
        </template>
      </el-table-column>
      <el-table-column label="类型" prop="type" width="120" align="center">
        <template v-slot="scope">
          <span v-if="scope.row.type === 1">事假</span>
          <span v-if="scope.row.type === 2">病假</span>
          <span v-if="scope.row.type === 3">婚假</span>
          <span v-if="scope.row.type === 4">丧假</span>
          <span v-if="scope.row.type === 5">产假</span>
          <span v-if="scope.row.type === 6">探亲</span>
          <span v-if="scope.row.type === 7">其他</span>
        </template>
      </el-table-column>
      <el-table-column label="当前节点" prop="currentNode" width="200" align="center"/>
      <el-table-column align="center" prop="currentRole" label="当前处理人" width="150" />
      <el-table-column label="请假事由" show-overflow-tooltip=true align="center" prop="leaveReason" />
      <el-table-column label="请假天数"  align="center" prop="leaveDays" width="100"/>
      <el-table-column label="开始时间" align="center" prop="startTime" width="160"/>
      <el-table-column label="结束时间" align="center" prop="endTime" width="160"/>
      <el-table-column  label="操作"  align="center"  width="250">
        <template v-slot="scope">
          <el-button type="primary"  size="mini" @click="handleProcess(scope.row.processInstance)">流程进度/结果</el-button>
          <el-button type="danger" size="mini" @click="handleDelete(scope.row)">删除</el-button>
        </template>
      </el-table-column>
    </el-table>
    <pagination v-show="total>0" :total="total" :page.sync="queryParams.pageNo" :limit.sync="queryParams.pageSize" @pagination="getList" />
    <el-dialog title="请假申请" :visible.sync="leaveFormShow" width="500px">
      <el-form :model="leaveForm" label-width="80px">
        <el-form-item label="账号">
          <el-select v-model="leaveForm.type" placeholder="请假类型" clearable>
            <el-option
              v-for="dict in options"
              :key="dict.value"
              :label="dict.label"
              :value="dict.value"
            />
          </el-select>
        </el-form-item>
        <el-form-item label="开始时间">
          <el-date-picker
            value-format="yyyy-MM-dd HH:mm:ss"
            v-model="leaveForm.startTime"
            type="date"
            placeholder="选择开始时间">
          </el-date-picker>
        </el-form-item>
        <el-form-item label="结束时间">
          <el-date-picker
            value-format="yyyy-MM-dd HH:mm:ss"
            v-model="leaveForm.endTime"
            type="date"
            placeholder="选择结束时间">
          </el-date-picker>
        </el-form-item>
        <el-form-item label="请假事由">
          <el-input type="textarea" v-model="leaveForm.leaveReason"></el-input>
        </el-form-item>
      </el-form>
      <div slot="footer" class="dialog-footer">
        <el-button @click="leaveFormShow = false">取 消</el-button>
        <el-button type="primary" @click="saveLeaveInfo()">提 交</el-button>
      </div>
    </el-dialog>
    <el-dialog title="流程进度" :visible.sync="leaveTimelineShow" width="500px" class="spec-dialog">
      <el-scrollbar style="height:700px;">
      <el-timeline >
        <el-timeline-item v-for="item in processHistorys" :timestamp="item.startTime" placement="top" :color="!item.endTime?'#0bbd87':'#bbfae1'">
          <el-card>
            <h4>{{item.name}}</h4>
            <p v-if="item.endTime">{{item.assignee}} 办理于 {{item.endTime}}</p>
            <p v-else>等待 {{item.assignee}} 审批</p>
            <p v-if="item.comment">审批意见:{{item.comment}}</p>
          </el-card>
        </el-timeline-item>
      </el-timeline>
      </el-scrollbar>
    </el-dialog>
  </div>
</template>

<script>
import { deleteLeave, getList, saveLeave } from '@/api/attendance/leave'
import pagination from '@/components/Pagination'
import { mapGetters } from 'vuex'
import { processHistory } from '@/api/activiti/task'

export default {
  components: { pagination },
  computed: {
    ...mapGetters([
      'nickname'
    ])
  },
  data() {
    return {
      resultOptions: [{
        value: 0,
        label: '审批中'
      }, {
        value: 1,
        label: '同意'
      }, {
        value: 2,
        label: '拒绝'
      }],
      processHistorys: [],
      leaveTimelineShow: false,
      leaveForm: {

      },
      leaveFormShow: false,
      options: [{
        value: 1,
        label: '事假'
      }, {
        value: 2,
        label: '病假'
      }, {
        value: 3,
        label: '婚假'
      }, {
        value: 4,
        label: '丧假'
      }, {
        value: 5,
        label: '产假'
      }, {
        value: 6,
        label: '探亲'
      }, {
        value: 7,
        label: '其他'
      }],
      list: [],
      total: 0,
      queryParams: {
        manager: 0,
        pageNo: 1,
        pageSize: 10,
        type: '',
        startTime: '',
        endTime: ''
      },
      listLoading: true
    }
  },
  created() {
    this.getList()
  },
  methods: {
    handleDelete(row) {
      this.$confirm(`确定撤销此申请吗？`, '是否继续?', '提示', {
        confirmButtonText: '删除',
        cancelButtonText: '算了吧',
        type: 'warning'
      }).then(() => {
        deleteLeave(row.id, row.processInstance).then(res => {
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
    handleProcess(id) {
      this.leaveTimelineShow = true
      processHistory(id).then(res => {
        this.processHistorys = res.data
      })
    },
    saveLeaveInfo() {
      this.leaveFormShow = false
      this.listLoading = true
      this.leaveForm.uname = this.nickname
      saveLeave(this.leaveForm).then(res => {
        this.getList()
      })
    },
    handleAdd() {
      this.$common.clear(this.leaveForm)
      this.leaveFormShow = true
    },
    resetQuery() {
      this.queryParams = {
        manager: 0,
        pageNo: 1,
        pageSize: 10,
        type: '',
        startTime: '',
        endTime: ''
      }
    },
    getList() {
      this.listLoading = true
      getList(this.queryParams).then(res => {
        this.list = res.data.list
        this.total = res.data.total
        this.listLoading = false
      })
    }
  }
}
</script>

<style scoped>
.el-scrollbar__wrap {
  overflow-x: hidden!important;
}
</style>
