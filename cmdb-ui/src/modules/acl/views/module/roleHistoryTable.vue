<template>
  <div>
    <search-form
      ref="child"
      :attrList="roleTableAttrList"
      :hasSwitch="true"
      switchValue="角色关系"
      @onSwitchChange="onSwitchChange"
      @search="handleSearch"
      @searchFormReset="searchFormReset"
    ></search-form>
    <vxe-table
      ref="xTable"
      stripe
      class="ops-stripe-table"
      :data="tableData"
      size="small"
      :loading="loading"
      resizable
      :height="`${windowHeight - 310}px`"
    >
      <vxe-column field="created_at" width="144px" title="操作时间"></vxe-column>
      <vxe-column field="operate_uid" title="操作员" width="130px"></vxe-column>
      <vxe-column field="operate_type" title="操作" width="112px">
        <template #default="{ row }">
          <template>
            <a-tag :color="handleTagColor(row.operate_type)">{{ operateTypeMap.get(row.operate_type) }}</a-tag>
          </template>
        </template>
      </vxe-column>
      <vxe-column :title="checked ? '角色' : '角色'">
        <template #default="{ row }">
          <template v-if="!checked">
            <a-tag color="blue">{{ row.current.name || row.origin.name }}</a-tag>
          </template>
          <template v-else>
            <a-tag v-for="(id, index) in row.extra.child_ids" :key="'child_ids_' + id + index" color="blue">{{
              id
            }}</a-tag>
          </template>
        </template>
      </vxe-column>
      <vxe-column :title="checked ? '继承自' : '管理员'" :width="checked ? '350px' : '80px'">
        <template #default="{ row }">
          <template v-if="!checked">
            <a-icon type="check" v-if="row.current.is_app_admin" />
          </template>
          <template v-else>
            <a-tag v-for="(id, index) in row.extra.parent_ids" :key="'parent_ids_' + id + index" color="cyan">{{
              id
            }}</a-tag>
          </template>
        </template>
      </vxe-column>
      <vxe-column title="描述" v-if="!checked">
        <template #default="{ row }">
          <p>
            {{ row.description }}
          </p>
        </template>
      </vxe-column>
      <vxe-column field="source" title="来源" width="100px"></vxe-column>
    </vxe-table>
    <pager
      :current-page.sync="queryParams.page"
      :page-size.sync="queryParams.page_size"
      :page-sizes="[50, 100, 200]"
      :total="tableDataLength"
      :isLoading="loading"
      @change="onChange"
      @showSizeChange="onShowSizeChange"
      :style="{ marginTop: '10px' }"
    ></pager>
  </div>
</template>

<script>
import _ from 'lodash'
import Pager from './pager.vue'
import SearchForm from './searchForm.vue'
import { searchRoleHistory } from '@/modules/acl/api/history'
export default {
  components: { SearchForm, Pager },
  props: {
    allUsers: {
      type: Array,
      required: true,
    },
    allRoles: {
      type: Array,
      required: true,
    },
    allRolesMap: {
      type: Map,
      required: true,
    },
    allUsersMap: {
      type: Map,
      required: true,
    },
  },
  data() {
    return {
      loading: true,
      checked: false,
      tableData: [],
      app_id: this.$route.name.split('_')[0],
      operateTypeMap: new Map([
        ['create', '新建'],
        ['delete', '删除'],
        ['update', '修改'],
        ['role_relation_add', '添加角色关系'],
        ['role_relation_delete', '删除角色关系'],
      ]),
      colorMap: new Map([
        ['create', 'green'],
        ['delete', 'red'],
        ['update', 'orange'],
        ['role_relation_add', 'green'],
        ['role_relation_delete', 'red'],
      ]),
      queryParams: {
        page: 1,
        page_size: 50,
        app_id: this.$route.name.split('_')[0],
        scope: 'role',
        start: '',
        end: '',
      },
      roleTableAttrList: [
        {
          alias: '日期',
          is_choice: false,
          name: 'datetime',
          value_type: '3',
        },
        {
          alias: '操作员',
          is_choice: true,
          name: 'operate_uid',
          value_type: '2',
          choice_value: this.allUsers,
        },
        {
          alias: '操作',
          is_choice: true,
          name: 'operate_type',
          value_type: '2',
          choice_value: [
            { 新建: 'create' },
            { 修改: 'update' },
            { 删除: 'delete' },
            { 添加角色关系: 'role_relation_add' },
            { 删除角色关系: 'role_relation_delete' },
          ],
        },
      ],
    }
  },
  computed: {
    windowHeight() {
      return this.$store.state.windowHeight
    },
    tableDataLength() {
      return this.tableData.length
    },
  },
  async created() {
    await this.getTable(this.queryParams)
  },
  updated() {
    this.$refs.xTable.$el.querySelector('.vxe-table--body-wrapper').scrollTop = 0
  },
  watch: {
    '$route.name': async function(oldName, newName) {
      this.app_id = this.$route.name.split('_')[0]
      await this.getTable(this.queryParams)
    },
  },
  methods: {
    async getTable(queryParams) {
      try {
        this.loading = true
        const { data, id2roles, id2perms, id2resources } = await searchRoleHistory(this.handleQueryParams(queryParams))
        data.forEach((item) => {
          item.operate_uid = this.allUsersMap.get(item.operate_uid)
          if (item.operate_type === 'role_relation_add' || item.operate_type === 'role_relation_delete') {
            item.extra.child_ids.forEach((subItem, index) => {
              item.extra.child_ids[index] = id2roles[subItem].name
            })
            item.extra.parent_ids.forEach((subItem, index) => {
              item.extra.parent_ids[index] = id2roles[subItem].name
            })
          } else {
            this.handleChangeDescription(item, item.operate_type, id2roles, id2perms, id2resources)
          }
        })
        this.tableData = data
      } finally {
        this.loading = false
      }
    },

    // pager相关
    onShowSizeChange(size) {
      this.queryParams.page_size = size
      this.queryParams.page = 1
      this.getTable(this.queryParams)
    },
    onChange(pageNum) {
      this.queryParams.page = pageNum
      this.getTable(this.queryParams)
    },

    // searchForm相关
    onSwitchChange(checked) {
      this.checked = checked
      this.queryParams.scope = checked ? 'role_relation' : 'role'
      this.queryParams.page = 1
      this.getTable(this.queryParams)
    },
    handleSearch(queryParams) {
      this.queryParams = {
        ...this.queryParams,
        ...queryParams,
        app_id: this.app_id,
        scope: this.checked ? 'role_relation' : 'role',
      }
      this.getTable(this.queryParams)
    },
    searchFormReset() {
      this.$refs.child.checked = false
      this.checked = false
      this.queryParams = {
        page: 1,
        page_size: 50,
        app_id: this.$route.name.split('_')[0],
        scope: this.checked ? 'role_relation' : 'role',
      }
      this.getTable(this.queryParams)
    },

    // 处理查询参数
    handleQueryParams(queryParams) {
      const _queryParams = _.cloneDeep(queryParams)

      let flag = false
      let q = _queryParams.q ? _queryParams.q : ''
      for (const key in _queryParams) {
        if (
          key !== 'page' &&
          key !== 'page_size' &&
          key !== 'app_id' &&
          key !== 'q' &&
          key !== 'start' &&
          key !== 'end' &&
          _queryParams[key] !== undefined
        ) {
          flag = true
          if (q) q += `,${key}:${_queryParams[key]}`
          else q += `${key}:${_queryParams[key]}`
          delete _queryParams[key]
        }
      }
      const newQueryParams = { ..._queryParams, q }
      return flag ? newQueryParams : _queryParams
    },
    // 处理tag颜色
    handleTagColor(operateType) {
      return this.colorMap.get(operateType)
    },
    handleChangeDescription(item, operate_type, id2roles, id2perms, id2resources) {
      switch (operate_type) {
        // create
        case 'create': {
          item.description = `新建角色：${item.current.name}`
          break
        }
        case 'update': {
          item.description = ''
          for (const key in item.origin) {
            const newVal = item.current[key]
            const oldVal = item.origin[key]
            if (!_.isEqual(newVal, oldVal) && key !== 'updated_at' && key !== 'deleted_at' && key !== 'created_at') {
              if (oldVal === null) {
                const str = ` 【 ${key} : 改为 ${newVal} 】 `
                item.description += str
              } else {
                const str = ` 【 ${key} : 由 ${oldVal} 改为 ${newVal} 】 `
                item.description += str
              }
            }
          }
          if (!item.description) item.description = '没有修改'
          break
        }
        case 'delete': {
          const {
            extra: { child_ids, parent_ids, role_permissions },
          } = item
          child_ids.forEach((subItem, index) => {
            child_ids[index] = id2roles[subItem].name
          })
          parent_ids.forEach((subItem, index) => {
            parent_ids[index] = id2roles[subItem].name
          })

          const resourceMap = new Map()
          const permsArr = []
          role_permissions.forEach((subItem) => {
            const resource_id = subItem.resource_id
            const perm_id = subItem.perm_id
            if (resourceMap.has(resource_id)) {
              let resource_perms = resourceMap.get(resource_id)
              resource_perms += `,${id2perms[perm_id].name}`
              resourceMap.set(resource_id, resource_perms)
            } else {
              resourceMap.set(resource_id, String(id2perms[perm_id].name))
            }
          })
          resourceMap.forEach((value, key) => {
            permsArr.push(`${id2resources[key].name}：${value}`)
          })
          item.description = `继承者：${child_ids}\n继承自：${parent_ids}\n涉及资源及权限：\n${permsArr.join(`\n`)}`
          break
        }
      }
    },
  },
}
</script>

<style lang="less" scoped>
.row {
  margin-top: 5px;
}
.ant-tag {
  max-width: 100%;
  overflow: hidden;
  text-overflow: ellipsis;
}
p {
  margin-bottom: 0;
}
</style>
