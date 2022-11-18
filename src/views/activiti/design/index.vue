<template>
  <div class="app-container">
    <el-dialog title="保存模型" :visible.sync="modelFormShow" width="500px">
      <el-form :model="modelForm" label-width="80px">
        <el-form-item label="模型名称">
          <el-input v-model="modelForm.name" wu />
        </el-form-item>
        <el-form-item label="模型标识">
          <el-input v-model="modelForm.key" />
        </el-form-item>
        <el-form-item label="分类">
          <el-input v-model="modelForm.category" />
        </el-form-item>
        <el-form-item label="描述">
          <el-input v-model="modelForm.description" />
        </el-form-item>
      </el-form>
      <div slot="footer" class="dialog-footer">
        <el-button @click="roleFormShow = false">取 消</el-button>
        <el-button type="primary" @click="saveModel()">提 交</el-button>
      </div>
    </el-dialog>
    <!-- 把操作按钮写在这里面 -->
    <div class="action">
      <el-upload action class="upload-demo" :before-upload="openBpmn">
        <el-button icon="el-icon-folder-opened" />
      </el-upload>
      <el-button class="new" icon="el-icon-circle-plus" @click="newDiagram" />
      <el-button icon="el-icon-download" @click="downloadBpmn" />
      <el-button icon="el-icon-picture" @click="downloadSvg" />
      <el-button icon="el-icon-save" @click="handleSaveModel">保存模型</el-button>
      <a ref="downloadLink" hidden />
    </div>
    <!-- 创建一个canvas画布 npmn-js是通过canvas实现绘图的，并设置ref让vue获取到element -->
    <div ref="canvas" class="bpmn-canvas" />
    <!-- 工具栏显示的地方 -->
    <div id="js-properties-panel" class="bpmn-js-properties-panel" />

  </div>
</template>

<script>
import 'bpmn-js/dist/assets/diagram-js.css'
import 'bpmn-js/dist/assets/bpmn-font/css/bpmn.css'
import 'bpmn-js/dist/assets/bpmn-font/css/bpmn-codes.css'
import 'bpmn-js/dist/assets/bpmn-font/css/bpmn-embedded.css'
// 在这里引入一下Bpmn建模器对象
import BpmnModeler from 'bpmn-js/lib/Modeler'
// 左边工具栏以及编辑节点的样式
import 'bpmn-js-properties-panel/dist/assets/bpmn-js-properties-panel.css'
// 工具栏相关
import propertiesProviderModule from 'bpmn-js-properties-panel/lib/provider/camunda'
import propertiesPanelModule from 'bpmn-js-properties-panel'
import camundaModdleDescriptor from 'camunda-bpmn-moddle/resources/camunda'
import { saveModelInfo, loadModelXml, updateModel } from '@/api/activiti/model'
// 汉化文件夹
import customTranslate from '../customTranslate/customTranslate'

export default {
  data() {
    return {
      modelId: '',
      modelForm: {

      },
      modelFormShow: false,
      bpmnModeler: null,
      canvas: null,
      bpmnTemplate: `
          <?xml version="1.0" encoding="UTF-8"?>
          <definitions
              xmlns="http://www.omg.org/spec/BPMN/20100524/MODEL"
              xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
              xmlns:bpmndi="http://www.omg.org/spec/BPMN/20100524/DI"
              xmlns:omgdc="http://www.omg.org/spec/DD/20100524/DC"
              xmlns:camunda="http://camunda.org/schema/1.0/bpmn"
              xmlns:xsd="http://www.w3.org/2001/XMLSchema"
              id="m1577635100724"
              name=""
              targetNamespace="http://www.activiti.org/testm1577635100724"
            >
            <process id="process" processType="None" isClosed="false" isExecutable="true">
              <extensionElements>
                <camunda:properties>
                  <camunda:property name="a" value="1" />
                </camunda:properties>
              </extensionElements>
              <startEvent id="_2" name="开始" />
            </process>
            <bpmndi:BPMNDiagram id="BPMNDiagram_leave">
              <bpmndi:BPMNPlane id="BPMNPlane_leave" bpmnElement="leave">
                <bpmndi:BPMNShape id="BPMNShape__2" bpmnElement="_2">
                  <omgdc:Bounds x="144" y="368" width="32" height="32" />
                  <bpmndi:BPMNLabel>
                    <omgdc:Bounds x="149" y="400" width="23" height="14" />
                  </bpmndi:BPMNLabel>
                </bpmndi:BPMNShape>
              </bpmndi:BPMNPlane>
            </bpmndi:BPMNDiagram>
          </definitions>
      `
    }
  },
  created() {
    this.modelId = this.$route.query.modelId
    if (this.modelId) {
      loadModelXml(this.modelId).then(res => {
        this.bpmnTemplate = res.data.replace('activiti.org/bpmn', 'camunda.org/schema/1.0/bpmn').replace('<activiti:properties>', '<camunda:properties>').replace('</activiti:properties>', '</camunda:properties>')
        console.log(this.bpmnTemplate.indexOf('activiti'))
        this.createNewDiagram(this.bpmnTemplate)
      })
    }
  },
  watch: {
    '$route'(to, from) {
      if (!this.$route.query.modelId) {
        this.modelId = ''
        this.bpmnTemplate = `
          <?xml version="1.0" encoding="UTF-8"?>
          <definitions
              xmlns="http://www.omg.org/spec/BPMN/20100524/MODEL"
              xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
              xmlns:bpmndi="http://www.omg.org/spec/BPMN/20100524/DI"
              xmlns:omgdc="http://www.omg.org/spec/DD/20100524/DC"
              xmlns:camunda="http://camunda.org/schema/1.0/bpmn"
              xmlns:xsd="http://www.w3.org/2001/XMLSchema"
              id="m1577635100724"
              name=""
              targetNamespace="http://www.activiti.org/testm1577635100724"
            >
            <process id="process" processType="None" isClosed="false" isExecutable="true">
              <extensionElements>
                <camunda:properties>
                  <camunda:property name="a" value="1" />
                </camunda:properties>
              </extensionElements>
              <startEvent id="_2" name="开始" />
            </process>
            <bpmndi:BPMNDiagram id="BPMNDiagram_leave">
              <bpmndi:BPMNPlane id="BPMNPlane_leave" bpmnElement="leave">
                <bpmndi:BPMNShape id="BPMNShape__2" bpmnElement="_2">
                  <omgdc:Bounds x="144" y="368" width="32" height="32" />
                  <bpmndi:BPMNLabel>
                    <omgdc:Bounds x="149" y="400" width="23" height="14" />
                  </bpmndi:BPMNLabel>
                </bpmndi:BPMNShape>
              </bpmndi:BPMNPlane>
            </bpmndi:BPMNDiagram>
          </definitions>
      `
        this.createNewDiagram(this.bpmnTemplate)
      }
    }
  },

  mounted() {
    if(!this.$route.query.modelId){
      this.modelId = this.$route.query.modelId
    }
    this.init()
  },
  methods: {
    handleSaveModel() {
      if (!this.modelId) {
        this.$common.clear(this.modelForm)
        this.modelFormShow = true
        // eslint-disable-next-line no-empty
      } else {
        // 更新模型
        this.handleUpdateModel(this.modelId)
      }
    },
    handleUpdateModel(modelId) {
      this.bpmnModeler.saveXML({ format: true }, (err, data) => {
        if (!err) {
          var xml = data.replace(/camunda.org\/schema\/1.0\/bpmn/ig, 'activiti.org/bpmn').replace(/camunda/ig, 'activiti')
          updateModel(modelId, xml).then(res => {
            this.$message({
              type: 'success',
              message: '更新成功'
            })
            this.$router.push('/activiti')
          })
        }
      })
    },
    saveModel() {
      this.bpmnModeler.saveXML({ format: true }, (err, data) => {
        if (!err) {
          const xml = data.replace(/camunda.org\/schema\/1.0\/bpmn/ig, 'activiti.org/bpmn').replace(/camunda/ig, 'activiti')
          this.modelForm.xml = xml
          saveModelInfo(this.modelForm).then(res => {
            this.$message({
              type: 'success',
              message: '操作成功'
            })
            this.$common.clear(this.modelForm)
            this.modelFormShow = false
            this.createNewDiagram()
          })
        }
      })
    },

    newDiagram() {
      this.createNewDiagram(this.bpmnTemplate)
    },

    downloadBpmn() {
      this.bpmnModeler.saveXML({ format: true }, (err, xml) => {
        if (!err) {
          const newXml = xml.replace(/camunda.org\/schema\/1.0\/bpmn/ig, 'activiti.org/bpmn').replace(/camunda/ig, 'activiti')
          // 获取文件名
          const name = `${this.getFilename(newXml)}.bpmn20.xml`
          // 将文件名以及数据交给下载方法
          this.download({ name: name, data: newXml })
        }
      })
    },
    downloadSvg() {
      this.bpmnModeler.saveXML({ format: true }, (err, data) => {
        const xml = data.replace(/camunda.org\/schema\/1.0\/bpmn/ig, 'activiti.org/bpmn').replace(/camunda/ig, 'activiti')

        if (!err) {
          // 获取文件名
          const name = `${this.getFilename(xml)}.svg`

          // 从建模器画布中提取svg图形标签
          let context = ''
          const djsGroupAll = this.$refs.canvas.querySelectorAll('.djs-group')
          for (const item of djsGroupAll) {
            context += item.innerHTML
          }
          // 获取svg的基本数据，长宽高
          const viewport = this.$refs.canvas
            .querySelector('.viewport')
            .getBBox()

          // 将标签和数据拼接成一个完整正常的svg图形
          const svg = `
            <svg
              xmlns="http://www.w3.org/2000/svg"
              xmlns:xlink="http://www.w3.org/1999/xlink"
              width="${viewport.width}"
              height="${viewport.height}"
              viewBox="${viewport.x} ${viewport.y} ${viewport.width} ${viewport.height}"
              version="1.1"
              >
              ${context}
            </svg>
          `
          // 将文件名以及数据交给下载方法
          this.download({ name: name, data: svg })
        }
      })
    },

    openBpmn(file) {
      const reader = new FileReader()
      // 读取File对象中的文本信息，编码格式为UTF-8
      reader.readAsText(file, 'utf-8')
      reader.onload = () => {
        // 读取完毕后将文本信息导入到Bpmn建模器
        this.createNewDiagram(reader.result)
      }
      return false
    },

    getFilename(xml) {
      const start = xml.indexOf('process')
      let filename = xml.substr(start, xml.indexOf('>'))
      filename = filename.substr(filename.indexOf('id') + 4)
      filename = filename.substr(0, filename.indexOf('"'))
      return filename
    },

    download({ name = 'diagram.bpmn20.xml', data }) {
      // 这里就获取到了之前设置的隐藏链接
      const downloadLink = this.$refs.downloadLink
      // 把输就转换为URI，下载要用到的
      const encodedData = encodeURIComponent(data)

      if (data) {
        // 将数据给到链接
        downloadLink.href =
          'data:application/bpmn20-xml;charset=UTF-8,' + encodedData
        // 设置文件名
        downloadLink.download = name
        // 触发点击事件开始下载
        downloadLink.click()
      }
    },

    init() {
      // 获取画布 element
      this.canvas = this.$refs.canvas
      // 汉化配置
      let customTranslateModule = {
        translate: ['value', customTranslate]
      }
      // 创建Bpmn对象
      this.bpmnModeler = new BpmnModeler({
        // 设置bpmn的绘图容器为上门获取的画布 element
        container: this.canvas,

        // 加入工具栏支持
        propertiesPanel: {
          parent: '#js-properties-panel'
        },
        additionalModules: [propertiesProviderModule, propertiesPanelModule, customTranslateModule],
        moddleExtensions: {
          camunda: camundaModdleDescriptor
        }
      })
      this.createNewDiagram(this.bpmnTemplate)
    },
    createNewDiagram(bpmn) {
      console.log(bpmn)
      // 将字符串转换成图显示出来;
      this.bpmnModeler.importXML(bpmn, err => {
        if (err) {
          console.log(err)
          this.$Message.error('打开模型出错,请确认该模型符合Bpmn2.0规范')
        }
      })
    }
  }
}
</script>

<style scoped>
.bpmn-canvas {
  width: 100%;
  height: 90vh;
}

.action {
  position: relative;
  bottom: 10px;
  left: 10px;
  display: flex;
}
.upload-demo {
  margin-right: 10px;
}

.bpmn-js-properties-panel {
  position: absolute;
  top: 0;
  right: 0px;
  width: 300px;
}
</style>
