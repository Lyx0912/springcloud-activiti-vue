<template>
  <div class="app-container">
    <el-row style="margin-bottom: 12px">
      <el-col :span="19">
        <el-form :model="queryParams" ref="queryForm" size="small" :inline="true" >
          <el-form-item label="类型" prop="status">
            <el-input></el-input>
          </el-form-item>&nbsp;
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
      <el-table-column align="center" label="流程编号" prop="processDefnitionId" width="180" />
      <el-table-column label="流程名称" prop="processDefnitionName" width="150" align="center" />
      <el-table-column label="执行实例编号" prop="processInstanceId" width="150" align="center"/>
      <el-table-column align="center" prop="id" label="任务编号" width="150" />
      <el-table-column label="任务名称" align="center" prop="name" />
      <el-table-column label="办理角色码" align="center" prop="assignee" width="180"/>
      <el-table-column label="业务编号" align="center" prop="bussinessKey" width="100"/>
      <el-table-column label="开始时间" align="center" prop="createTime" width="220"/>
      <el-table-column  label="操作"  align="center"  width="180">
        <template v-slot="scope">
          <el-button type="primary" icon="el-icon-message" size="mini" @click="handle(scope.row)">办理</el-button>
        </template>
      </el-table-column>
    </el-table>
    <pagination v-show="total>0" :total="total" :page.sync="queryParams.pageNo" :limit.sync="queryParams.pageSize" @pagination="getList" />
    <el-dialog title="请假申请" :visible.sync="leaveCheckShow" width="500px">
      <el-form :model="leaveInfo" label-width="80px">
        <el-form-item label="请假人" >
          <el-input  v-model="leaveInfo.uname" disabled="true"></el-input>
        </el-form-item>
        <el-form-item label="类型" >
          <el-select v-model="leaveInfo.type" placeholder="请假类型" clearable disabled="true">
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
            :readonly="true"
            value-format="yyyy-MM-dd HH:mm:ss"
            v-model="leaveInfo.startTime"
            type="date"
            disabled="true"
            placeholder="选择开始时间">
          </el-date-picker>
        </el-form-item>
        <el-form-item label="结束时间">
          <el-date-picker
            value-format="yyyy-MM-dd HH:mm:ss"
            v-model="leaveInfo.endTime"
            type="date"
            disabled="true"
            placeholder="选择结束时间">
          </el-date-picker>
        </el-form-item>
        <el-form-item label="请假事由">
          <el-input type="textarea" v-model="leaveInfo.leaveReason" disabled="true"></el-input>
        </el-form-item>
        <el-form-item label="审批结果">
          <el-radio-group v-model="leaveTaskRes.result">
            <el-radio label="同意"></el-radio>
            <el-radio label="拒绝"></el-radio>
          </el-radio-group>
        </el-form-item>
        <el-form-item label="意见">
          <el-input type="textarea" v-model="leaveTaskRes.opinion"></el-input>
        </el-form-item>
      </el-form>
      <div slot="footer" class="dialog-footer">
        <el-button @click="leaveCheckShow = false">取 消</el-button>
        <el-button type="primary" @click="doHandle()">办 理 </el-button>
      </div>
    </el-dialog>
    <el-dialog title="出差申请" :visible.sync="travelCheckShow" width="500px">
      <el-form :model="leaveInfo" label-width="80px">
        <el-form-item label="申请人" >
          <el-input  v-model="travelInfo.uname" disabled="true"></el-input>
        </el-form-item>
        <el-form-item label="开始时间">
          <el-date-picker
            :readonly="true"
            value-format="yyyy-MM-dd HH:mm:ss"
            v-model="travelInfo.startTime"
            type="date"
            disabled="true"
            placeholder="选择开始时间">
          </el-date-picker>
        </el-form-item>
        <el-form-item label="结束时间">
          <el-date-picker
            value-format="yyyy-MM-dd HH:mm:ss"
            v-model="travelInfo.endTime"
            type="date"
            disabled="true"
            placeholder="选择结束时间">
          </el-date-picker>
        </el-form-item>
        <el-form-item label="请假事由">
          <el-input type="textarea" v-model="travelInfo.travelReason" disabled="true"></el-input>
        </el-form-item>
        <el-form-item label="审批结果">
          <el-radio-group v-model="leaveTaskRes.result">
            <el-radio :label="1" >同意</el-radio>
            <el-radio :label="0" >拒绝</el-radio>
          </el-radio-group>
        </el-form-item>
        <el-form-item label="意见">
          <el-input type="textarea" v-model="leaveTaskRes.opinion"></el-input>
        </el-form-item>
      </el-form>
      <div slot="footer" class="dialog-footer">
        <el-button @click="leaveCheckShow = false">取 消</el-button>
        <el-button type="primary" @click="doHandle()">办 理 </el-button>
      </div>
    </el-dialog>
  </div>
</template>

<script>
import { handle, list } from '@/api/activiti/task'
import pagination from '@/components/Pagination'
import { leaveInfo } from '@/api/attendance/leave'
import { travelInfo } from '@/api/attendance/travel'

export default {
  components: { pagination },
  data() {
    return {
      travelInfo: {

      },
      travelCheckShow: false,
      leaveInfo: {

      },
      leaveTaskRes: {

      },
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
      leaveCheckShow: false,
      list: [],
      total: 0,
      queryParams: {
        pageNo: 1,
        pageSize: 10,
        type: '',
        startTime: '',
        endTime: ''
      },
      listLoading: false
    }
  },
  created() {
    this.getList()
  },
  methods: {
    doHandle() {
      this.travelCheckShow = false
      this.leaveCheckShow = false
      this.listLoading = true
      handle(this.leaveTaskRes).then(res=>{
        this.$message({
          type: 'success',
          message: '办理成功!'
        })
        this.getList()
      })
    },
    getList() {
      this.listLoading = true
      list(this.queryParams).then(res => {
        this.list = res.data.list
        this.total = res.data.total
        this.listLoading = false
      })
    },
    handle(row) {
      // 设置任务编号和实例编号
      this.leaveTaskRes.taskId = row.id
      this.leaveTaskRes.processInstance = row.processInstanceId
      if (row.processDefnitionName === '请假流程') {
        leaveInfo(row.bussinessKey).then(res => {
          this.leaveInfo = res.data
        })
        this.leaveCheckShow = true
      }
      if (row.processDefnitionName === '出差流程') {
        travelInfo(row.bussinessKey).then(res => {
          this.travelInfo = res.data
        })
        this.travelCheckShow = true
      }
    }
  }
}
</script>

<style scoped>

</style>
