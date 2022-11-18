<template>
  <div class="app-container">
    <el-row style="margin-bottom: 12px">
      <el-col :span="5">
        <el-button icon="el-icon-plus" size="mini" type="warning" @click="handleAdd()">录入</el-button>
        <el-upload
          ref="upload"
          :show-file-list="false"
          style="display: inline;margin-left: 15px"
          class="upload-demo"
          :action="baseUrl+'/cloud-attendance/overwork/saveBatch'"
          :headers="headers"
          :on-success="onsuccess"
          :before-remove="beforeRemove"
          multiple
          :auto-upload="true"
          :limit="1"
          :on-exceed="handleExceed">
          <el-button icon="el-icon-plus" size="mini" type="primary">批量录入</el-button>
        </el-upload>
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
      <el-table-column align="center" label="职工号" prop="id" width="80" />
      <el-table-column align="center" label="职工名称" prop="userName" />
      <el-table-column label="备注" align="center" prop="remark" />
      <el-table-column label="开始时间" align="center" prop="startTime" />
      <el-table-column label="结束时间" align="center" prop="endTime" />
      <el-table-column label="加班时长" align="center" prop="duration" />
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
        <el-form-item label="开始时间" prop="startTime">
          <el-date-picker
            v-model="form.startTime"
            clearable
            type="datetime"
            value-format="yyyy-MM-dd HH:mm:ss"
            placeholder="请选择开始时间"
          />
        </el-form-item>
        <el-form-item label="结束时间" prop="endTime">
          <el-date-picker
            v-model="form.endTime"
            clearable
            type="datetime"
            value-format="yyyy-MM-dd HH:mm:ss"
            placeholder="请选择结束时间"
          />
        </el-form-item>
        <el-form-item label="备注" prop="remark">
          <el-input v-model="form.remark" placeholder="请输入备注" />
        </el-form-item>
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
import { listOverwork, getOverwork, delOverwork, addOverwork, updateOverwork } from '@/api/attendance/overwork'
import {getToken} from "@/utils/auth";

export default {
  name: 'Overwork',
  components: { pagination },
  data() {
    return {
      headers: {},
      baseUrl: process.env.VUE_APP_BASE_API,
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
    this.headers = { 'Authorization': 'Bearer ' + getToken() }
    this.getList()
  },
  methods: {
    onsuccess(response, file, fileList) {
      this.$refs.upload.clearFiles()
      this.getList()
    },
    handleExceed(files, fileList) {
      this.$message.warning(`当前限制选择 1 个文件，本次选择了 ${files.length} 个文件，共选择了 ${files.length + fileList.length} 个文件`);
    },
    beforeRemove(file, fileList) {
      return this.$confirm(`确定移除 ${file.name}？`)
    },
    /** 查询出勤信息列表 */
    getList() {
      this.loading = true
      listOverwork(this.queryParams).then(response => {
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
      this.resetForm('queryForm')
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
      getOverwork(id).then(response => {
        this.form = response.data
        this.open = true
        this.title = '修改出勤信息'
      })
    },
    /** 提交按钮 */
    submitForm() {
      if (this.form.id) {
        updateOverwork(this.form).then(response => {
          this.$message({
            type: 'success',
            message: '编辑成功!'
          })
          this.open = false
          this.getList()
        })
      } else {
        addOverwork(this.form).then(response => {
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
      this.$confirm(`确定删除此加班信息吗？`, '是否继续?', '提示', {
        confirmButtonText: '删除',
        cancelButtonText: '算了吧',
        type: 'warning'
      }).then(() => {
        delOverwork(row.id).then(res => {
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
