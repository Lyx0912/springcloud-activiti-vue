<template>
  <div class="app-container">
    <el-row style="margin-bottom: 12px">
      <el-col :span="5">
        <el-button icon="el-icon-plus" size="mini" type="warning" @click="handleAdd()">新增</el-button>
      </el-col>
      <el-col :span="19">
        <el-form ref="queryForm" :model="queryParams" size="small" :inline="true" style="float: right">
          <el-form-item label="状态" prop="status">
            <el-select v-model="queryParams.result" placeholder="结果" clearable>
              <el-option
                v-for="dict in options"
                :key="dict.value"
                :label="dict.label"
                :value="dict.value"
              />
            </el-select>
          </el-form-item>&nbsp;
          <el-form-item label="申请时间">
            <el-date-picker
              v-model="queryParams.startTime"
              value-format="yyyy-MM-dd HH:mm:ss"
              type="date"
              placeholder="选择开始时间"
            />-
            <el-date-picker
              v-model="queryParams.endTime"
              value-format="yyyy-MM-dd HH:mm:ss"
              type="date"
              placeholder="选择时间"
            />
          </el-form-item>
          <el-form-item>
            <el-button type="primary" icon="el-icon-search" size="mini" @click="getList">搜索</el-button>
            <el-button icon="el-icon-refresh" size="mini" @click="resetQuery">重置</el-button>
          </el-form-item>
        </el-form>
      </el-col>
    </el-row>
    <el-table
      v-loading="listLoading"
      :row-style="{height:'45px'}"
      :cell-style="{padding:'0px'}"
      :data="list"
      element-loading-text="Loading"
      border
      fit
      highlight-current-row
      @selection-change="handleSelectionChange"
    >
      <el-table-column
        type="selection"
        width="55"
      />
      <el-table-column align="center" label="编号" prop="id" width="50" />
      <el-table-column align="center" label="审批结果" prop="result" width="80">
        <template v-slot="scope">
          <span>{{ scope.row.result===0?'审批中':scope.row.result===1?'同意':'拒绝' }}</span>
        </template>
      </el-table-column>
      <el-table-column label="当前节点" prop="currentNode" width="200" align="center" />
      <el-table-column align="center" prop="currentRole" label="当前处理人" width="150" />
      <el-table-column label="出差事由" show-overflow-tooltip="true" align="center" prop="travelReason" />
      <el-table-column label="出差天数" align="center" prop="travelDays" width="100" />
      <el-table-column label="开始时间" align="center" prop="startTime" width="160" />
      <el-table-column label="结束时间" align="center" prop="endTime" width="160" />
      <el-table-column label="操作" align="center" width="250">
        <template v-slot="scope">
          <el-button type="primary" size="mini" @click="handleProcess(scope.row.processInstance)">流程进度/结果</el-button>
          <el-button type="danger" size="mini" @click="handleDelete(scope.row)">删除</el-button>
        </template>
      </el-table-column>
    </el-table>
    <pagination v-show="total>0" :total="total" :page.sync="queryParams.pageNo" :limit.sync="queryParams.pageSize" @pagination="getList" />
    <el-dialog title="出差申请" :visible.sync="travelFormShow" width="500px">
      <el-form :model="travelForm" label-width="80px">
        <el-form-item label="开始时间">
          <el-date-picker
            v-model="travelForm.startTime"
            value-format="yyyy-MM-dd HH:mm:ss"
            type="date"
            placeholder="选择开始时间"
          />
        </el-form-item>
        <el-form-item label="结束时间">
          <el-date-picker
            v-model="travelForm.endTime"
            value-format="yyyy-MM-dd HH:mm:ss"
            type="date"
            placeholder="选择结束时间"
          />
        </el-form-item>
        <el-form-item label="出差事由">
          <el-input v-model="travelForm.travelReason" type="textarea" />
        </el-form-item>
      </el-form>
      <div slot="footer" class="dialog-footer">
        <el-button @click="leaveFormShow = false">取 消</el-button>
        <el-button type="primary" @click="saveTravelInfo()">提 交</el-button>
      </div>
    </el-dialog>
    <el-dialog title="流程进度" :visible.sync="travelTimelineShow" width="500px" class="spec-dialog">
      <el-scrollbar style="height:700px;">
        <el-timeline>
          <el-timeline-item v-for="item in processHistorys" :timestamp="item.startTime" placement="top" :color="!item.endTime?'#0bbd87':'#bbfae1'">
            <el-card>
              <h4>{{ item.name }}</h4>
              <p v-if="item.endTime">{{ item.assignee }} 办理于 {{ item.endTime }}</p>
              <p v-else>等待 {{ item.assignee }} 审批</p>
              <p v-if="item.comment">审批意见:{{ item.comment }}</p>
            </el-card>
          </el-timeline-item>
        </el-timeline>
      </el-scrollbar>
    </el-dialog>
  </div>
</template>

<script>
import pagination from '@/components/Pagination'
import {deleteTravel, getList, saveTravel} from '@/api/attendance/travel'
import { processHistory } from '@/api/activiti/task'
import { mapGetters } from 'vuex'
import {deleteLeave} from "@/api/attendance/leave";

export default {
  components: { pagination },
  computed: {
    ...mapGetters([
      'nickname'
    ])
  },
  data() {
    return {
      options: [{
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
      travelTimelineShow: false,
      travelForm: {

      },
      travelFormShow: false,
      list: [],
      total: 1,
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
    resetQuery() {
      this.queryParams = {
        manager: 0,
        pageNo: 1,
        pageSize: 10
      }
    },
    handleDelete(row) {
      this.$confirm(`确定撤销此申请吗？`, '是否继续?', '提示', {
        confirmButtonText: '删除',
        cancelButtonText: '算了吧',
        type: 'warning'
      }).then(() => {
        deleteTravel(row.id, row.processInstance).then(res => {
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
    saveTravelInfo() {
      this.travelFormShow = false
      this.listLoading = true
      this.travelForm.uname = this.nickname
      saveTravel(this.travelForm).then(res => {
        this.getList()
      })
    },
    handleAdd() {
      this.$common.clear(this.travelForm)
      this.travelFormShow = true
    },
    getList() {
      this.listLoading = true
      getList(this.queryParams).then(res => {
        this.list = res.data.list
        this.total = res.data.total
        this.listLoading = false
      })
    },
    handleProcess(id) {
      this.travelTimelineShow = true
      processHistory(id).then(res => {
        this.processHistorys = res.data
      })
    }
  }

}
</script>

<style scoped>

</style>
