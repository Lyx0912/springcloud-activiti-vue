<template>
  <div class="app-container">
    <el-row style="margin-bottom: 12px">
      <el-col :span="5">
        <el-button icon="el-icon-plus" size="mini" type="warning" @click="handleAdd()">录入</el-button>
      </el-col>
      <el-col :span="19">
        <el-form ref="queryForm" :model="queryParams" size="small" :inline="true" style="float: right">
          <el-form-item label="职工名称" prop="userName">
            <el-input
              v-model="queryParams.userName"
              placeholder="请输入职工名称"
              clearable
              @keyup.enter.native="handleQuery"
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
      v-loading="loading"
      :row-style="{height:'45px'}"
      :cell-style="{padding:'0px'}"
      :data="overWorkList"
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
      <el-table-column align="center" label="职工号" prop="userId" width="120" />
      <el-table-column align="center" label="职工名称" prop="userName" />
      <el-table-column label="上班时间" align="center" prop="startTime" />
      <el-table-column label="下班时间" align="center" prop="endTime" />
      <el-table-column label="出勤类型" align="center" prop="type" >
        <template v-slot="scope">
          <span>{{scope.row.type===0?'正常':scope.row.type===1?'迟到':scope.row.type===2?'早退':'矿工'}}</span>
        </template>
      </el-table-column>
      <el-table-column align="center" prop="created_at" label="操作">
        <template v-slot="scope">
          <el-button type="primary" size="mini" @click="handleUpdate(scope.row)">编辑</el-button>
          <el-button type="danger" size="mini" @click="handleDelete(scope.row)">删除</el-button>
        </template>
      </el-table-column>
    </el-table>
    <pagination v-show="total>0" :total="total" :page.sync="queryParams.pageNo" :limit.sync="queryParams.pageSize" @pagination="getList" />
    <!-- 添加或修改出勤信息对话框 -->
    <el-dialog :title="title" :visible.sync="open" width="500px" append-to-body>
      <el-form ref="form" :model="form" :rules="rules" label-width="80px">
        <el-form-item label="职工编号" prop="userName">
          <el-input v-model="form.userId" placeholder="请输入职工编号" />
        </el-form-item>
        <el-form-item label="职工名称" prop="userName">
          <el-input v-model="form.userName" placeholder="请输入职工名称" />
        </el-form-item>
        <el-form-item label="上班时间" prop="startTime">
          <el-date-picker
            v-model="form.startTime"
            clearable
            type="datetime"
            value-format="yyyy-MM-dd HH:mm:ss"
            placeholder="请选择开始时间"
          />
        </el-form-item>
        <el-form-item label="下班时间" prop="endTime">
          <el-date-picker
            v-model="form.endTime"
            clearable
            type="datetime"
            value-format="yyyy-MM-dd HH:mm:ss"
            placeholder="请选择结束时间"
          />
        </el-form-item>
        <el-form-item label="出勤类型" prop="status">
          <el-select v-model="form.type" placeholder="出勤类型" clearable>
            <el-option
              v-for="dict in options"
              :key="dict.value"
              :label="dict.label"
              :value="dict.value"
            />
          </el-select>
        </el-form-item>&nbsp;
      </el-form>
      <div slot="footer" class="dialog-footer">
        <el-button type="primary" @click="submitForm">确 定</el-button>
        <el-button @click="cancel">取 消</el-button>
      </div>
    </el-dialog>
  </div>
</template>

<script>
import pagination from '@/components/Pagination'
import { list, getRecord, delRecord, addRecord, updateRecord } from '@/api/attendance/record'

export default {
  name: 'record',
  components: { pagination },
  data() {
    return {
      options: [{
        value: 0,
        label: '正常'
      }, {
        value: 1,
        label: '迟到'
      }, {
        value: 2,
        label: '早退'
      }, {
        value: 4,
        label: '矿工'
      }],
      // 遮罩层
      loading: true,
      // 选中数组
      ids: [],
      // 非单个禁用
      single: true,
      // 非多个禁用
      multiple: true,
      // 显示搜索条件
      showSearch: true,
      // 总条数
      total: 0,
      // 出勤信息表格数据
      overWorkList: [],
      // 弹出层标题
      title: '',
      // 是否显示弹出层
      open: false,
      // 查询参数
      queryParams: {
        pageNo: 1,
        pageSize: 10,
        startTime: null,
        endTime: null,
        type: null,
        userName: null
      },
      // 表单参数
      form: {},
      // 表单校验
      rules: {
      }
    }
  },
  created() {
    this.getList()
  },
  methods: {
    /** 查询出勤信息列表 */
    getList() {
      this.loading = true
      list(this.queryParams).then(response => {
        this.overWorkList = response.data.list
        this.total = response.data.total
        this.loading = false
      })
    },
    // 取消按钮
    cancel() {
      this.open = false
      this.reset()
    },
    // 表单重置
    reset() {
      this.form = {
        id: null,
        userId: null,
        startTime: null,
        endTime: null,
        duration: null,
        remark: null,
        userName: null
      }
    },
    /** 搜索按钮操作 */
    handleQuery() {
      this.queryParams.pageNum = 1
      this.getList()
    },
    /** 重置按钮操作 */
    resetQuery() {
      this.handleQuery()
    },
    // 多选框选中数据
    handleSelectionChange(selection) {
      this.ids = selection.map(item => item.id)
      this.single = selection.length !== 1
      this.multiple = !selection.length
    },
    /** 新增按钮操作 */
    handleAdd() {
      this.reset()
      this.open = true
      this.title = '添加出勤信息'
    },
    /** 修改按钮操作 */
    handleUpdate(row) {
      this.reset()
      const id = row.id || this.ids
      getRecord(id).then(response => {
        this.form = response.data
        this.open = true
        this.title = '修改出勤信息'
      })
    },
    /** 提交按钮 */
    submitForm() {
      if (this.form.id) {
        updateRecord(this.form).then(response => {
          this.$message({
            type: 'success',
            message: '编辑成功!'
          })
          this.open = false
          this.getList()
        })
      } else {
        addRecord(this.form).then(response => {
          this.$message({
            type: 'success',
            message: '新增成功!'
          })
          this.open = false
          this.getList()
        })
      }
    },
    /** 删除按钮操作 */
    handleDelete(row) {
      this.$confirm(`确定删除此出勤信息吗？`, '是否继续?', '提示', {
        confirmButtonText: '删除',
        cancelButtonText: '算了吧',
        type: 'warning'
      }).then(() => {
        delRecord(row.id).then(res => {
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
