<template>
  <a-card>
    <template slot="title">
      查看分数
    </template>
    <template slot="extra" >
      <download-excel
        class="export-excel-wrapper"
        :data="DetailsForm"
        :fields="json_fields"
        :header="title"
        :name="title"
      >
        <a-button>
          一键下载excel
        </a-button>
      </download-excel>
    </template>
    <a-table
      :columns="columns"
      bordered
      :components="components"
      :row-key="problem => problem.id"
      :data-source="data"
      :pagination="pagination"
      :loading="loading"
      ref="table"
      @change="handleTableChange"
    >
      <a-icon
        slot="filterIcon"
        slot-scope="filtered"
        type="search"
        :style="{ color: filtered ? '#108ee9' : undefined }"
      />
    </a-table>
  </a-card>
</template>

<script>
import Vue from 'vue'
import { getProblemSetGrades } from '@/api/class'
import ResizableTableHeader from '@/components/Table/ResizableTableHeader.js'
import moment from 'moment'

export default {
  data () {
    const columns = [
      {
        title: '学号',
        dataIndex: 'user.username',
        scopedSlots: {
          customRender: 'user_username',
          filterDropdown: 'filterDropdown',
          filterIcon: 'filterIcon',
        width: 180
        },
        searching: '',
        sorter: (a, b) => a.user.username - b.user.username,
        sortDirections: ['descend', 'ascend']
      },
      {
        title: '姓名',
        dataIndex: 'user.nickname',
        scopedSlots: {
          customRender: 'user_nickname',
          filterDropdown: 'filterDropdown',
          filterIcon: 'filterIcon',
        width: 180
        },
        searching: ''
      },
      {
        title: '总分',
        dataIndex: 'total',
        scopedSlots: { customRender: 'total' },
        width: 180
      }
    ]

    const ResizableTableHeaderWithColumns = (h, props, children) => {
      const draggingMap = {}
      columns.forEach(col => {
        draggingMap[col.dataIndex] = col.width
      })
      const draggingState = Vue.observable(draggingMap)
      return ResizableTableHeader(h, props, children, columns, draggingState)
    }
    return {
      problemSetID: this.$route.params.problemSetID,
      classID: this.$route.params.classID,
      now_user_id: this.$store.state.user.info.id,
      problemSet: {},
      columns: columns,
      components: {
        header: {
          cell: ResizableTableHeaderWithColumns
        }
      },
      problem_set: {
        name: '',
        description: '',
        time: [moment(), moment()],
        problems: []
      },
      title: '',
      json_fields: {
        学号: 'user.username',
        姓名: 'user.nickname'
      },
      DetailsForm: [],
      deleting: {},
      deleteModelText: '',
      visible: false,
      confirmLoading: false,
      data: [],
      pagination: {
        showTotal: (total, range) => `共 ${total} 条数据`
      },
      loading: false,
      sorter: {
        sortField: 'user.username',
        sortOrder: 'ascend'
      }
    }
  },
  mounted () {
    console.log(this.columns)
    // console.log(this.$route.params)
    getProblemSetGrades(this.classID, this.problemSetID).then(resp => {
      console.log(resp)
      const dynamicColumn = resp.problem_set.problems.map(item => {
        return {
          title: `${item.id} ${item.name}`,
          dataIndex: item.id,
          width: 200
        }
      })
      for (let i = 0; i < dynamicColumn.length; i++) {
        const _index = i + 2
        this.columns.splice(_index, 0, dynamicColumn[i])
      }
      this.problemSet = resp.problem_set
      this.title = `${resp.problem_set.name}成绩表格`
      this.data = resp.problem_set.grades.map(item => {
        const _detail = JSON.parse(item.detail)
        for (const problem of resp.problem_set.problems) {
          const problemID = problem.id.toString()
          if (!(_detail.hasOwnProperty(problemID))) {
            _detail[problemID] = 0
          }
        }
        return {
          ...item,
          ..._detail
        }
      })
      this.DetailsForm = this.data
      this.DetailsForm.sort((a, b) => a.user.username - b.user.username)
      this.problemSet.problems.forEach((problem) => {
        // use problemname as key,problemID as value
        this.$set(this.json_fields, problem.name, problem.id.toString())
      })
      this.$set(this.json_fields, '总分', 'total')
      this.$nextTick(() => {
          for (const col of this.columns) {
            if (col.thDom) {
              col.width = col.thDom.getBoundingClientRect().width
            }
          }
        })
    }).catch(err => {
      this.$error({
        title: '发生错误',
        content: err.response.message + err.response.error
      })
    })
  },
  watch: {
    '$route' (r) {
      this.search_user_id = this.$route.query && +this.$route.query.user_id || null
    }
  },
  methods: {
    format (time) {
      return moment(time).format('lll')
    },
    handleTableChange (pagination, filters, sorter) {
      console.log(pagination, filters, sorter)
      this.pagination = pagination
      this.sorter = sorter
    }
  }
}
</script>
