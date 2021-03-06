<template>
  <div class="app-container">
    <!--工具栏-->
    <div class="head-container">
      <div v-if="crud.props.searchToggle">
        <!-- 搜索 -->
        <el-input v-model="query.blurry" size="small" clearable :placeholder="$t('role.searchPlaceholder')" :title="$t('role.searchPlaceholder')" style="width: 200px;" class="filter-item" @keyup.enter.native="crud.toQuery" />
        <el-date-picker
          v-model="query.createTime"
          :default-time="['00:00:00','23:59:59']"
          type="daterange"
          range-separator=":"
          size="small"
          class="date-item"
          value-format="yyyy-MM-dd HH:mm:ss"
          :start-placeholder="$t('common.startTime')"
          :end-placeholder="$t('common.endTime')"
        />
        <rrOperation :crud="crud" />
      </div>
      <crudOperation :permission="permission" />
    </div>
    <!-- 表单渲染 -->
    <el-dialog append-to-body :close-on-click-modal="false" :before-close="crud.cancelCU" :visible.sync="crud.status.cu > 0" :title="crud.status.title" width="630px">
      <el-form ref="form" :inline="true" :model="form" :rules="rules" size="small" label-width="130px">
        <el-form-item :label="$t('role.name')" prop="name">
          <el-input v-model="form.name" style="width: 145px;" />
        </el-form-item>
        <el-form-item :label="$t('role.permission')" prop="permission">
          <el-input v-model="form.permission" style="width: 145px;" />
        </el-form-item>
        <el-form-item :label="$t('role.dataScope')" prop="dataScope">
          <el-select v-model="form.dataScope" style="width: 145px" :placeholder="$t('role.dataScopePlaceholder')" @change="changeScope">
            <el-option
              v-for="item in dateScopes"
              :key="item"
              :label="item"
              :value="item"
            />
          </el-select>
        </el-form-item>
        <el-form-item :label="$t('role.level')" prop="level">
          <el-input-number v-model.number="form.level" :min="1" controls-position="right" style="width: 145px;" />
        </el-form-item>
        <el-form-item v-if="form.dataScope === '自定义'" :label="$t('role.dataPermission')" prop="depts">
          <treeselect v-model="form.depts" :options="depts" multiple style="width: 430px" :placeholder="$t('role.selectPlaceholder')" />
        </el-form-item>
        <el-form-item :label="$t('role.remark')" prop="remark">
          <el-input v-model="form.remark" style="width: 430px;" rows="5" type="textarea" />
        </el-form-item>
      </el-form>
      <div slot="footer" class="dialog-footer">
        <el-button type="text" @click="crud.cancelCU">{{ $t('crud.cancel') }}</el-button>
        <el-button :loading="crud.cu === 2" type="primary" @click="crud.submitCU">{{ $t('crud.confirm') }}</el-button>
      </div>
    </el-dialog>
    <el-row :gutter="15">
      <!--角色管理-->
      <el-col :xs="24" :sm="24" :md="16" :lg="16" :xl="17" style="margin-bottom: 10px">
        <el-card class="box-card" shadow="never">
          <div slot="header" class="clearfix">
            <span class="role-span">{{ $t('role.roleList') }}</span>
          </div>
          <el-table ref="table" v-loading="crud.loading" highlight-current-row style="width: 100%;" :data="crud.data" @selection-change="crud.selectionChangeHandler" @current-change="handleCurrentChange" @sort-change="crud.doTitleOrder">
            <el-table-column :selectable="checkboxT" type="selection" width="55" />
            <el-table-column v-if="columns.visible('name')" prop="name" :label="$t('role.name')" sortable="custom" />
            <el-table-column v-if="columns.visible('dataScope')" prop="dataScope" :label="$t('role.dataScope')" sortable="custom" />
            <el-table-column v-if="columns.visible('permission')" prop="permission" :label="$t('role.permission')" sortable="custom" />
            <el-table-column v-if="columns.visible('level')" prop="level" :label="$t('role.level')" sortable="custom" />
            <el-table-column v-if="columns.visible('remark')" :show-overflow-tooltip="true" prop="remark" :label="$t('role.remark')" sortable="custom" />
            <el-table-column v-if="columns.visible('createTime')" :show-overflow-tooltip="true" width="135px" prop="createTime" :label="$t('be.createTime')" sortable="custom">
              <template slot-scope="scope">
                <span>{{ parseTime(scope.row.createTime) }}</span>
              </template>
            </el-table-column>
            <el-table-column v-permission="['admin','roles:edit','roles:del']" :label="$t('be.operate')" width="130px" align="center" fixed="right">
              <template slot-scope="scope">
                <udOperation
                  v-if="scope.row.level >= level"
                  :data="scope.row"
                  :permission="permission"
                />
              </template>
            </el-table-column>
          </el-table>
          <!--分页组件-->
          <pagination />
        </el-card>
      </el-col>
      <!-- 菜单授权 -->
      <el-col :xs="24" :sm="24" :md="8" :lg="8" :xl="7">
        <el-card class="box-card" shadow="never">
          <div slot="header" class="clearfix">
            <el-tooltip class="item" effect="dark" :content="$t('role.menuTips')" placement="top">
              <span class="role-span">{{ $t('role.menuAssignment') }}</span>
            </el-tooltip>
            <el-button
              v-permission="['admin','roles:edit']"
              :disabled="!showButton"
              :loading="menuLoading"
              icon="el-icon-check"
              size="mini"
              style="float: right; padding: 6px 9px"
              type="primary"
              @click="saveMenu"
            >{{ $t('crud.save') }}</el-button>
          </div>
          <el-tree
            ref="menu"
            :data="menus"
            :default-checked-keys="menuIds"
            :props="defaultProps"
            accordion
            show-checkbox
            node-key="id"
          />
        </el-card>
      </el-col>
    </el-row>
  </div>
</template>

<script>
import crudRoles from '@/api/system/role'
import { getDepts } from '@/api/system/dept'
import { getMenusTree } from '@/api/system/menu'
import CRUD, { presenter, header, form, crud } from '@crud/crud'
import rrOperation from '@crud/RR.operation'
import crudOperation from '@crud/CRUD.operation'
import udOperation from '@crud/UD.operation'
import pagination from '@crud/Pagination'
import Treeselect from '@riophae/vue-treeselect'
import '@riophae/vue-treeselect/dist/vue-treeselect.css'
import i18n from '../../../lang'

// crud交由presenter持有
const adSearchFields = new Map([['name', i18n.t('role.name')], ['dataScope', i18n.t('role.dataScope')], ['permission', i18n.t('role.permission')], ['level', i18n.t('role.level')], ['remark', i18n.t('role.remark')], ['createTime', i18n.t('be.createTime')]]) // 需要高级搜索的字段
const defaultCrud = CRUD({ title: i18n.t('role.TITLE'), url: 'api/roles/page', exportUrl: 'api/roles/download', sort: 'level,asc', crudMethod: { ...crudRoles }, adSearchFields: adSearchFields })
const defaultForm = { id: null, name: null, depts: [], remark: null, dataScope: '全部', level: 3, permission: null }
export default {
  name: 'Role',
  components: { Treeselect, pagination, crudOperation, rrOperation, udOperation },
  mixins: [presenter(defaultCrud), header(), form(defaultForm), crud()],
  data() {
    return {
      defaultProps: { children: 'children', label: 'label' },
      dateScopes: ['全部', '本级', '自定义'], level: 3,
      currentId: 0, menuLoading: false, showButton: false,
      menus: [], menuIds: [], depts: [],
      permission: {
        add: ['admin', 'roles:add'],
        edit: ['admin', 'roles:edit'],
        del: ['admin', 'roles:del']
      },
      rules: {
        name: [
          { required: true, message: i18n.t('role.nameRequired'), trigger: 'blur' }
        ],
        permission: [
          { required: true, message: i18n.t('role.permissionRequired'), trigger: 'blur' }
        ]
      }
    }
  },
  created() {
    this.getMenus()
    crudRoles.getLevel().then(res => {
      if (res.code === 0) {
        this.level = res.data.level
        console.log(this.level)
      } else {
        crud.notify(res.msg, CRUD.NOTIFICATION_TYPE.ERROR)
      }
    })
    this.$nextTick(() => {
      this.crud.toQuery()
    })
  },
  methods: {
    [CRUD.HOOK.afterRefresh]() {
      this.$refs.menu.setCheckedKeys([])
    },
    // 编辑前
    [CRUD.HOOK.beforeToEdit](crud, form) {
      if (form.dataScope === '自定义') {
        this.getDepts()
      }
      const depts = []
      form.depts.forEach(function(dept, index) {
        depts.push(dept.id)
      })
      form.depts = depts
    },
    // 提交前做的操作
    [CRUD.HOOK.afterValidateCU](crud) {
      if (crud.form.dataScope === '自定义' && crud.form.depts.length === 0) {
        this.$message({
          message: i18n.t('role.customChk'),
          type: 'warning'
        })
        return false
      } else if (crud.form.dataScope === '自定义') {
        const depts = []
        crud.form.depts.forEach(function(data, index) {
          const dept = { id: data }
          depts.push(dept)
        })
        crud.form.depts = depts
      } else {
        crud.form.depts = []
      }
      return true
    },
    [CRUD.HOOK.afterAddError](crud) {
      this.afterErrorMethod(crud)
    },
    [CRUD.HOOK.afterEditError](crud) {
      this.afterErrorMethod(crud)
    },
    afterErrorMethod(crud) {
      const depts = []
      crud.form.depts.forEach(function(dept, index) {
        depts.push(dept.id)
      })
      crud.form.depts = depts
    },
    // 获取所有菜单
    getMenus() {
      getMenusTree().then(res => {
        if (res.code === 0) {
          this.menus = res.data
        } else {
          crud.notify(res.msg, CRUD.NOTIFICATION_TYPE.ERROR)
        }
      })
    },
    // 触发单选
    handleCurrentChange(val) {
      if (val) {
        const _this = this
        // 清空菜单的选中
        this.$refs.menu.setCheckedKeys([])
        // 保存当前的角色id
        this.currentId = val.id
        this.showButton = this.level <= val.level
        // 初始化
        this.menuIds = []
        // 菜单数据需要特殊处理
        val.menus.forEach(function(data, index) {
          _this.menuIds.push(data.id)
        })
      }
    },
    // 保存菜单
    saveMenu() {
      this.menuLoading = true
      const role = { id: this.currentId, menus: [] }
      // 得到半选的父节点数据，保存起来
      this.$refs.menu.getHalfCheckedNodes().forEach(function(data, index) {
        const menu = { id: data.id }
        role.menus.push(menu)
      })
      // 得到已选中的 key 值
      this.$refs.menu.getCheckedKeys().forEach(function(data, index) {
        const menu = { id: data }
        role.menus.push(menu)
      })
      crudRoles.editMenu(role).then(res => {
        if (res.code === 0) {
          this.crud.notify(i18n.t('common.success'), CRUD.NOTIFICATION_TYPE.SUCCESS)
          this.menuLoading = false
          this.update()
        } else {
          this.menuLoading = false
          this.crud.notify(res.msg, CRUD.NOTIFICATION_TYPE.ERROR)
        }
      }).catch(err => {
        this.menuLoading = false
        console.log(err.response.data.message)
      })
    },
    // 改变数据
    update() {
      // 无刷新更新 表格数据
      crudRoles.get(this.currentId).then(res => {
        if (res.code === 0) {
          for (let i = 0; i < this.crud.data.length; i++) {
            if (res.data.id === this.crud.data[i].id) {
              this.crud.data[i] = res.data
              break
            }
          }
        } else {
          crud.notify(res.msg, CRUD.NOTIFICATION_TYPE.ERROR)
        }
      })
    },
    // 获取部门数据
    getDepts() {
      getDepts({ enabled: true }).then(res => {
        if (res.code === 0) {
          this.depts = res.data.content
        } else {
          crud.notify(res.msg, CRUD.NOTIFICATION_TYPE.ERROR)
        }
      })
    },
    // 如果数据权限为自定义则获取部门数据
    changeScope() {
      if (this.form.dataScope === '自定义') {
        this.getDepts()
      }
    },
    checkboxT(row, rowIndex) {
      return row.level >= this.level
    }
  }
}
</script>

<style rel="stylesheet/scss" lang="scss">
  .role-span {
    font-weight: bold;color: #303133;
    font-size: 15px;
  }
</style>

<style rel="stylesheet/scss" lang="scss" scoped>
  /deep/ .el-input-number .el-input__inner {
    text-align: left;
  }
</style>
