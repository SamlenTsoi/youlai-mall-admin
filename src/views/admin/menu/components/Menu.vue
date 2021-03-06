<template>
  <div>
    <el-row :gutter="10">
      <el-col :sm="12" :xs="24">
        <el-card class="box-card" shadow="always">
          <div class="clearfix" slot="header">
            <b>
              <svg-icon icon-class="menu"/>
              菜单列表</b>
          </div>
          <!-- 搜索表单 -->
          <el-form
            ref="queryForm"
            size="mini"
            :model="queryParams"
            :inline="true"
          >
            <el-form-item>
              <el-button type="success" icon="el-icon-plus" @click="handleAdd">新增</el-button>
            </el-form-item>
            <el-form-item>
              <el-input
                v-model="queryParams.name"
                placeholder="菜单名称"
                clearable
                @keyup.enter.native="handleQuery"
              />
            </el-form-item>
            <el-form-item>
              <el-button type="primary" icon="el-icon-search" @click="handleQuery">搜索</el-button>
              <el-button icon="el-icon-refresh" @click="handleReset">重置</el-button>
            </el-form-item>
          </el-form>

          <!-- 数据表格 -->
          <el-table
            v-loading="loading"
            :data="pageList"
            row-key="id"
            highlight-current-row
            :tree-props="{children: 'children', hasChildren: 'hasChildren'}"
            @row-click="handleRowClick"
            border
            size="mini"
          >
            <el-table-column label="菜单名称">
              <template slot-scope="scope">
                <svg-icon :icon-class="scope.row.icon"/>
                {{ scope.row.name }}
              </template>
            </el-table-column>
            <el-table-column label="状态" width="60">
              <template slot-scope="scope">
                <el-tag size="mini" v-if="scope.row.visible==1" type="success">显示</el-tag>
                <el-tag size="mini" v-else type="info">隐藏</el-tag>
              </template>
            </el-table-column>
            <el-table-column label="操作" align="center" width="130">
              <template slot-scope="scope">
                <el-button
                  type="primary"
                  icon="el-icon-edit"
                  size="mini"
                  circle
                  plain
                  @click.stop="handleUpdate(scope.row)"
                />
                <el-button
                  type="success"
                  icon="el-icon-plus"
                  size="mini"
                  circle
                  plain
                  @click.stop="handleAdd(scope.row)"
                />
                <el-button
                  type="danger"
                  icon="el-icon-delete"
                  size="mini"
                  circle
                  plain
                  @click.stop="handleDelete(scope.row)"
                />
              </template>
            </el-table-column>
          </el-table>


        </el-card>
      </el-col>
      <el-col :sm="12" :xs="24">
        <el-card class="box-card" shadow="always">
          <div class="clearfix" slot="header">
            <b>
              <svg-icon icon-class="form"></svg-icon>
              {{ title }}</b>
          </div>

          <el-form
            ref="form"
            :model="form"
            :rules="rules"
            label-width="80px"
          >

            <el-form-item label="上级菜单" prop="parentId">
              <tree-select
                v-model="form.parentId"
                :options="menuOptions"
                placeholder="选择上级菜单"
              />
            </el-form-item>

            <el-form-item label="菜单名称" prop="name">
              <el-input v-model="form.name" placeholder="请输入菜单名称"/>
            </el-form-item>

            <el-form-item label="菜单图标">
              <el-popover
                placement="bottom-start"
                width="460"
                trigger="click"
                @show="$refs['iconSelect'].reset()">
                <IconSelect ref="iconSelect" @selected="handleIconSelected"/>
                <el-input slot="reference" v-model="form.icon" placeholder="点击选择图标" readonly>
                  <svg-icon
                    v-if="form.icon"
                    slot="prefix"
                    :icon-class="form.icon"
                    class="el-input__icon"
                    style="height: 40px;width: 16px; "/>
                  <i v-else slot="prefix" class="el-icon-search el-input__icon"/>
                </el-input>
              </el-popover>
            </el-form-item>

            <el-form-item label="路由路径" prop="path">
              <el-input v-model="form.path" placeholder="父路由例如：/admin；子路由省略斜杠/，例如：user"/>
            </el-form-item>

            <el-form-item label="组件">
              <el-input v-model="form.component" placeholder="admin/user/index">
                <template v-if="form.parentId!=0" slot="prepend">src/views/</template>
                <template v-if="form.parentId!=0" slot="append">.vue</template>
              </el-input>
            </el-form-item>

            <el-form-item label="状态">
              <el-radio-group v-model="form.visible">
                <el-radio :label="1">显示</el-radio>
                <el-radio :label="0">隐藏</el-radio>
              </el-radio-group>
            </el-form-item>

            <el-form-item label="排序" prop="sort">
              <el-input-number v-model="form.sort" style="width: 100px" controls-position="right" :min="0"/>
            </el-form-item>

            <el-form-item label="跳转路径">
              <el-input v-model="form.redirect" placeholder="请输入跳转路径" maxlength="50"/>
            </el-form-item>

            <el-form-item>
              <el-button type="primary" @click="handleSubmit">保存</el-button>
              <el-button icon="el-icon-refresh" @click="resetForm">重置</el-button>
            </el-form-item>

          </el-form>
        </el-card>
      </el-col>
    </el-row>
  </div>
</template>

<script>
import {add, del, list, patch, update} from "@/api/admin/menu";
import '@riophae/vue-treeselect/dist/vue-treeselect.css'
import TreeSelect from '@riophae/vue-treeselect'
import IconSelect from '@/components/IconSelect'

export default {
  name: "Menu",
  components: {TreeSelect, IconSelect},
  data() {
    return {
      loading: true,
      ids: [],
      single: true,
      multiple: true,
      queryParams: {
        name: undefined,
        queryMode: undefined
      },
      pagination: {
        page: 1,
        limit: 10,
        total: 0
      },
      pageList: [],
      dialog: {
        title: undefined,
        visible: false
      },
      form: {
        parentId: 0,
        visible: 1,
        icon: undefined,
        sort: 1,
        component: 'Layout',
      },
      rules: {
        parentId: [
          {required: true, message: '请选择顶级菜单', trigger: 'blur'}
        ],
        name: [
          {required: true, message: '请输入菜单名称', trigger: 'blur'}
        ],
        path: [
          {required: true, message: '请输入路由路径', trigger: 'blur'}
        ],
        component: [
          {required: true, message: '请输入组件', trigger: 'blur'}
        ]
      },
      title: '新增菜单',
      menuOptions: [],
      currentRow: undefined
    }
  },
  created() {
    this.loadMenuOptions()
    this.handleQuery()
  },
  methods: {
    handleQuery() {
      this.resetForm()
      this.queryParams.page = this.pagination.page
      this.queryParams.limit = this.pagination.limit
      this.queryParams.queryMode = 'list'
      list(this.queryParams).then(response => {
        this.pageList = response.data
        this.pagination.total = response.total
        this.loading = false
      })
    },
    handleReset() {
      this.pagination = {
        page: 1,
        limit: 10,
        total: 0
      }
      this.queryParams.name = undefined
      this.handleQuery()
    },
    handleRowClick(row) {
      const currentRow = JSON.parse(JSON.stringify(row));
      this.currentRow = currentRow
      this.form = currentRow
      this.title = '【' + currentRow.name + '】' + '菜单修改'
      this.$emit('menuClick', row)
    },
    handleSelectionChange(selection) {
      this.ids = selection.map(item => item.id)
      this.single = selection.length != 1
      this.multiple = !selection.length
    },
    handleStatusChange(row) {
      const text = row.status === 1 ? '启用' : '停用'
      this.$confirm('确认要"' + text + row.name + '"数据项吗?', '警告', {
        confirmButtonText: '确定',
        cancelButtonText: '取消',
        type: 'warning'
      }).then(function () {
        return patch(row.id, 1, {status: row.status})
      }).then(() => {
        this.$message.success(text + '成功')
      }).catch(function () {
        row.status = row.status === 1 ? 0 : 1
      })
    },

    handleAdd(row) {
      this.resetForm()
      if (row.id) { // 行点击新增
        this.form.parentId = row.id
      } else { // 工具栏点击新增
        if (this.currentRow) {
          this.form.parentId = this.currentRow.id
        } else {
          this.form.parentId = 0
          this.form.component = 'Layout'
        }
      }
      this.title = '新增菜单'
    },
    handleUpdate(row) {
      this.resetForm()
      this.title = '修改菜单'
      this.form = JSON.parse(JSON.stringify(row))
    },

    handleSubmit: function () {
      this.$refs['form'].validate(valid => {
        if (valid) {
          const id = this.form.id
          if (id != undefined) {
            update(id, this.form).then(() => {
              this.$message.success('修改成功')
              this.handleQuery()
            })
          } else {
            add(this.form).then(() => {
              this.$message.success('新增成功')
              this.handleQuery()
            })
          }
        }
      })
    },
    handleDelete(row) {
      const ids = [row.id || this.ids].join(',')
      this.$confirm('确认删除已选中的数据项？', '警告', {
        confirmButtonText: '确定',
        cancelButtonText: '取消',
        type: 'warning'
      }).then(() => {
        del(ids).then(() => {
          this.$message.success('删除成功')
          this.handleQuery()
        })
      }).catch(() =>
        this.$message.info('已取消删除')
      )
    },
    resetForm() {
      this.form = {
        visible: 1,
        parentId: 0,
        icon: undefined,
        sort: 1,
        component: 'Layout'
      }
      if (this.$refs['form']) {
        this.$refs['form'].resetFields()
      }
    },
    loadMenuOptions() {
      this.menuOptions = []
      list({queryMode: 'tree'}).then(response => {
        const menuOption = {id: 0, label: '顶级菜单', children: response.data}
        this.menuOptions.push(menuOption)
      })
    },
    handleIconSelected(name) {
      this.form.icon = name
    }
  }
}
</script>

<style scoped>

</style>
