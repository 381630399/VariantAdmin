<template>
  <el-container class="field-props-container">
    <el-header class="field-props-header" v-if="!!!showingInDialog">[多对多引用]字段属性设置</el-header>
    <el-main class="field-props-pane">
      <el-form ref="editorForm" :model="fieldProps" :rules="rules" label-position="left"
               label-width="220px" size="mini" @submit.native.prevent>
        <el-form-item label="字段名称" prop="name">
          <el-input v-model="fieldProps.name" :disabled="fieldState !== 1"></el-input>
        </el-form-item>
        <el-form-item label="显示名称" prop="label">
          <el-input v-model="fieldProps.label"></el-input>
        </el-form-item>
        <el-form-item label="引用实体设置">
          <el-button style="float: right" icon="el-icon-plus" @click="addRefEntity">添加</el-button>
        </el-form-item>
        <hr style="border: 0;border-top: 1px dotted #cccccc" />
        <el-form-item :label="'引用实体' + (refIdx + 1)" v-for="(refEntity, refIdx) in referenceList" :key="refIdx">
          <el-input v-model="refEntity.refEntityAndFields" readonly>
            <el-button slot="append" icon="el-icon-close" title="清除" v-if="!!refEntity.currentRefEntity"
                       @click="clearReferTo(refIdx)"></el-button>
            <el-button slot="append" icon="el-icon-search" title="选择"
                       @click="showRefEntitySettingDialog(refIdx)"></el-button>
            <el-button slot="append" icon="el-icon-delete" title="删除" @click="deleteRefEntity(refIdx)"></el-button>
          </el-input>
        </el-form-item>
        <hr style="border: 0;border-top: 1px dotted #cccccc;margin: -8px 0 12px" />
        <el-form-item label="搜索弹窗宽度（单位：像素）：">
          <el-input-number type="number" v-model.number="fieldProps.fieldViewModel.searchDialogWidth" style="width: 100%">
          </el-input-number>
        </el-form-item>
        <el-form-item label="字段校验函数(可多选)" prop="fieldViewModel.validators">
          <el-select multiple allow-create filterable default-first-option :popper-append-to-body="false"
                     v-model="fieldProps.fieldViewModel.validators" style="width: 100%">
            <el-option
                    v-for="(vt, vtIdx) in validators"
                    :key="vtIdx"
                    :label="vt.label"
                    :value="vt.value">
            </el-option>
          </el-select>
        </el-form-item>
        <el-form-item label="是否在列表中默认显示">
          <el-radio-group v-model="fieldProps.defaultMemberOfListFlag" style="float: right">
            <el-radio :label="true">是</el-radio>
            <el-radio :label="false">否</el-radio>
          </el-radio-group>
        </el-form-item>
        <el-form-item label="是否允许空值">
          <el-radio-group v-model="fieldProps.nullable" style="float: right">
            <el-radio :label="true">是</el-radio>
            <el-radio :label="false">否</el-radio>
          </el-radio-group>
        </el-form-item>
        <el-form-item label="新建记录时允许修改字段">
          <el-radio-group v-model="fieldProps.creatable" style="float: right">
            <el-radio :label="true">是</el-radio>
            <el-radio :label="false">否</el-radio>
          </el-radio-group>
        </el-form-item>
        <el-form-item label="更新记录时允许修改字段">
          <el-radio-group v-model="fieldProps.updatable" style="float: right">
            <el-radio :label="true">是</el-radio>
            <el-radio :label="false">否</el-radio>
          </el-radio-group>
        </el-form-item>
        <hr style="border: 0;border-top: 1px dotted #cccccc" />
        <el-form-item>
          <el-button type="primary" size="medium" style="width: 120px" @click="saveField">保存字段</el-button>
          <el-button size="medium" v-if="!!showingInDialog" @click="cancelSave">取消</el-button>
        </el-form-item>
      </el-form>
    </el-main>

    <el-dialog title="引用实体设置" class="refer-entity-dialog" :visible.sync="showRefEntityDialogFlag"
               :append-to-body="true" width="560px">
      <el-container>
        <el-header>
          <el-input size="small" placeholder="请选择引用实体" v-model="refEntityFullName">
            <el-button slot="append" icon="el-icon-search" title="选择" @click="showEntityListDialog">
            </el-button></el-input>
        </el-header>
        <el-main>
          <div>
            <div style="margin-bottom: 6px">选择实体搜索列表字段：</div>
            <!--            <hr style="border: 0;border-top: 1px solid rgb(220, 223, 230)" />-->
            <el-card shadow="never">
              <div style="font-style: italic" v-if="fieldItems.length <= 0">暂无字段可选</div>
              <el-checkbox v-for="(fieldItem, idx) in fieldItems" :key="idx"
                           @change="(value) => setRefEntityListField(fieldItem, value)">
                {{fieldItem.label}}({{fieldItem.name}})</el-checkbox>
            </el-card>
          </div>
          <div>
            <div style="margin: 20px 0 6px">已选择的显示字段：</div>
            <el-card shadow="never">
              <div style="font-style: italic" v-if="selectedFieldItems.length <= 0">未选择显示字段</div>
              <div v-for="(selectedField, selectedIdx) in selectedFieldItems" :key="selectedIdx">
                {{selectedField.label}}({{selectedField.name}})
              </div>
            </el-card>
          </div>
        </el-main>
      </el-container>
      <div slot="footer" class="dialog-footer">
        <el-button @click="showRefEntityDialogFlag = false">取 消</el-button>
        <el-button type="primary" @click="setRefEntity">确 定</el-button>
      </div>
    </el-dialog>

    <el-dialog ref="entityListDlg" title="选择引用实体" :visible.sync="showEntityListDialogFlag" :append-to-body="true"
               class="entity-list-dialog" width="560px">
      <SimpleTable :show-pagination="false" :show-check-box="false" :table-size="'mini'" :columns="columns" :data="tableData"
                   :max-height="420">
        <el-table-column slot="table_operation" align="center" label="" width="150" :resizable="false">
          <template slot-scope="scope">
            <el-button class="" icon="el-icon-check" @click="selectEntity(scope.row)">选择</el-button>
          </template>
        </el-table-column>
      </SimpleTable>
    </el-dialog>

  </el-container>
</template>

<script>
  import {
    addAnyRefField,
    updateAnyRefField,
    getEntitySet,
    getFieldSet,
    getField,
    getRefFieldExtras
  } from '@/api/system-manager'
  import FieldState from "@/views/system/field-state-variables";
  import {copyObj} from "@/utils/util";

  export default {
    name: "ReferenceListWidgetEditor",
    props: {
      entity: String,
      fieldName: String,
      showingInDialog: Boolean,
      fieldState: {
        type: Number,
        default: FieldState.NEW,
      }
    },
    data() {
      return {
        fieldProps: {
          'name': '',
          'label': '',
          'type': 'ReferenceList',
          'defaultMemberOfListFlag': false,
          'nullable': false,
          'creatable': true,
          'updatable': true,
          'fieldViewModel': {
            'searchDialogWidth': 520,
            'validators': [],
          },
          'referTo': [],
          'referenceSetting': [],
        },

        rules: {
          name: [
            {required: true, message: '请输入字段名称', trigger: 'blur'},
            {pattern: /^[A-Za-z\d]+$/, message: '请以英文字母、数字开头，不能包含中文字符', trigger: 'blur'},
            {min: 2, max: 30, message: '请输入至少两个字符', trigger: 'blur'},
          ],
          label: [
            {required: true, message: '请输入显示名称', trigger: 'blur'},
            {pattern: /^[A-Za-z\d\u4e00-\u9fa5]+[_-]*/, message: '请以中文、英文字母、数字开头，中间可输入下划线或横杠',
              trigger: 'blur'},
            {min: 2, max: 30, message: '请输入至少两个字符', trigger: 'blur'},
          ],
        },
        validators: [],

        refEntityIndex: -1,
        referenceList: [
          {
            currentRefEntity: '',
            selectedFieldItems: [],
            refEntityAndFields: '',
          },
        ],

        refEntityName: '',
        refEntityLabel: '',
        refEntityFullName: '',

        showRefEntityDialogFlag: false,
        showEntityListDialogFlag: false,

        fieldItems: [],
        selectedFieldItems: [],

        columns: [
          {prop: 'name', label: '实体名称', width: '150', align: 'center'},
          {prop: 'label', label: '显示名称', width: '200', align: 'center', formatter: this.formatter},
        ],
        tableData: [],
      }
    },
    mounted() {
      if (this.fieldState === FieldState.EDIT) {
        this.getFieldProps()
      }
    },
    methods: {
      getFieldProps() {
        getField(this.fieldName, this.entity).then(res => {
          if (res.error != null) {
            this.$message({ message: res.error, type: 'error' })
            return
          }

          this.readFieldProps(res.data)
        }).catch(res => {
          this.$message({ message: res.message, type: 'error' })
        })
      },

      readFieldProps(savedProps) {
        copyObj(this.fieldProps, savedProps)
        //console.log(this.fieldProps)
        if (!!!this.fieldProps.fieldViewModel) {  //设置搜索弹窗默认宽度
          this.fieldProps['fieldViewModel'] = {
            searchDialogWidth: 520
          }
        }
        if (!!savedProps.entityCode) {
          this.fieldProps.entityCode = savedProps.entityCode
        }
        //console.log(JSON.stringify(this.fieldProps))

        /*
        // TODO: 需参考reference-widget-editor.vue实现字段已保存属性的加载显示！！
         */
      },

      saveField() {
        let validResult = true
        this.$refs['editorForm'].validate((success) => {
          if (!success) {
            validResult = false
            return false
          }
        })
        if (!validResult) return

        if ((this.fieldProps.referTo.length <= 0) || !!!this.fieldProps.referenceSetting) {
          this.$message.error('请设置引用实体！')
          return
        }

        this.fieldProps.type = 'ReferenceList'
        let referToEntities = ''
        this.fieldProps.referTo.forEach((item, idx) => {
          if (idx !== this.fieldProps.referTo.length - 1) {
            referToEntities += item + ','
          } else {
            referToEntities += item
          }
        })

        let saveMethod = addAnyRefField
        if (this.fieldState === FieldState.EDIT) {
          saveMethod = updateAnyRefField
        }
        saveMethod(this.fieldProps, this.entity, referToEntities).then(res => {
          if (res.error != null) {
            this.$message({ message: res.error, type: 'error' })
            return
          }

          this.$message.success('保存成功')
          this.$emit('fieldSaved')
        }).catch(res => {
          this.$message({ message: res.message, type: 'error' })
        })
      },

      cancelSave() {
        this.$emit('cancelSave')
      },

      clearReferTo(refIdx) {
        this.referenceList[refIdx].currentRefEntity = ''
        this.referenceList[refIdx].refEntityAndFields = ''
        this.buildReferToAndReferenceSetting()
      },

      showRefEntitySettingDialog(refIdx) {
        this.showRefEntityDialogFlag = true
        this.refEntityIndex = refIdx

        this.refEntityName = ''
        this.refEntityLabel = ''
        this.refEntityFullName = ''
        this.fieldItems.length = 0
        this.selectedFieldItems.length = 0
      },

      setRefEntity() {
        if(this.selectedFieldItems.length <= 0) {
          this.$message.info('请至少选择一个显示字段！')
          return
        }

        let tempStr = this.refEntityLabel + '['
        for (let i = 0; i < this.selectedFieldItems.length; i++) {
          tempStr += this.selectedFieldItems[i].label + ','
        }
        tempStr += ']'
        this.referenceList[this.refEntityIndex].refEntityAndFields = tempStr

        this.referenceList[this.refEntityIndex].currentRefEntity = this.refEntityName
        this.referenceList[this.refEntityIndex].selectedFieldItems = JSON.parse(JSON.stringify(this.selectedFieldItems))
        this.buildReferToAndReferenceSetting()
        // console.log( JSON.stringify(this.fieldProps.referenceSetting) )

        this.showRefEntityDialogFlag = false
      },

      showEntityListDialog() {
        this.tableData.length = 0
        getEntitySet().then(res => {
          if (res.error != null) {
            this.$message({ message: res.error, type: 'error' })
            return
          }
          let entityItems = res.data
          if (!!entityItems) {
            entityItems.filter( entity => {
              if (entity.detailEntityFlag === false) {
                this.tableData.push({name: entity.name, label: entity.label})
              }
            })
          }
          this.showEntityListDialogFlag = true
        }).catch(res => {
          this.$message({ message: res.message, type: 'error' })
        })
      },

      selectEntity(row) {
        this.refEntityName = row.name
        this.refEntityLabel = row.label
        this.refEntityFullName = this.refEntityLabel + '(' + this.refEntityName + ')'
        this.showEntityListDialogFlag = false

        this.fieldItems.length = 0
        getFieldSet(this.refEntityName).then(res => {
          if (res.error != null) {
            this.$message({ message: res.error, type: 'error' })
            return
          }
          let resultList = res.data
          if (!!resultList) {
            resultList.filter( item => {
              if (item.type !== 'PrimaryKey') {
                this.fieldItems.push(item)
              }
            })
          }
        }).catch(res => {
          this.$message({ message: res.message, type: 'error' })
        })
      },

      setRefEntityListField(fieldItem, value) {
        if (!!value) {
          this.selectedFieldItems.push(fieldItem)
        } else {
          for (let i = this.selectedFieldItems.length - 1; i >= 0; i--) {
            if (this.selectedFieldItems[i].name === fieldItem.name) {
              this.selectedFieldItems.splice(i, 1)
            }
          }
        }
      },

      addRefEntity() {
        this.referenceList.push({
          currentRefEntity: '',
          selectedFieldItems: [],
          refEntityAndFields: ''
        })
      },

      deleteRefEntity(refIdx) {
        this.referenceList.splice(refIdx, 1)
        this.buildReferToAndReferenceSetting()
      },

      buildReferToAndReferenceSetting() {
        this.fieldProps.referTo.length = 0
        this.fieldProps.referenceSetting.length = 0

        this.referenceList.forEach( ref => {
          if (!!ref.currentRefEntity && !!ref.selectedFieldItems) {
            let fieldList = []
            ref.selectedFieldItems.forEach( item => {
              fieldList.push(item.name)
            })
            this.fieldProps.referTo.push(ref.currentRefEntity)
            this.fieldProps.referenceSetting.push({'entityName': ref.currentRefEntity, 'fieldList': fieldList})
          }
        })

        //console.log(JSON.stringify(this.fieldProps.referTo))
        //console.log(JSON.stringify(this.fieldProps.referenceSetting))
      },

    }
  }
</script>

<style lang="scss" scoped>
  @import "@/styles/form-layout/field-editor-common.scss";

  .refer-entity-dialog, .entity-list-dialog {
    ::v-deep .el-dialog__header {
      padding: 15px;
      padding-bottom: 3px;
    }

    ::v-deep .el-dialog__body {
      padding: 6px;
    }
  }

</style>
