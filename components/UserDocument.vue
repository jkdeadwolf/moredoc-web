<template>
  <div class="com-user-document">
    <!-- 文档搜索表单 -->
    <el-form
      :inline="true"
      :model="query"
      class="demo-form-inline"
      @submit.native.prevent
    >
      <el-form-item>
        <el-input
          v-model="query.wd"
          placeholder="请输入关键字"
          clearable
          size="medium"
          @keydown.enter.native="onSearch"
        ></el-input>
      </el-form-item>
      <el-form-item>
        <el-date-picker
          v-model="query.created_at"
          type="datetimerange"
          :picker-options="datetimePickerOptions"
          range-separator="至"
          start-placeholder="开始日期"
          end-placeholder="结束日期"
          align="right"
          size="medium"
          value-format="yyyy-MM-dd HH:mm:ss"
        >
        </el-date-picker>
      </el-form-item>
      <el-form-item>
        <el-button
          type="primary"
          size="medium"
          icon="el-icon-search"
          :loading="loading"
          @click="onSearch"
        >
          搜索
        </el-button>
      </el-form-item>
    </el-form>

    <el-table v-loading="loading" :data="docs" style="width: 100%">
      <el-table-column prop="title" label="名称" min-width="300">
        <template slot-scope="scope">
          <el-tooltip :content="scope.row.title" placement="right">
            <nuxt-link
              target="_blank"
              :to="{
                name: 'document-id',
                params: { id: scope.row.uuid || scope.row.id },
              }"
              class="el-link el-link--default doc-title"
            >
              <img :src="`/static/images/${scope.row.icon}_24.png`" alt="" />
              {{ scope.row.title }}
            </nuxt-link>
          </el-tooltip>
        </template>
      </el-table-column>
      <el-table-column
        v-if="(settings.language || []).length > 0"
        prop="language"
        label="语言"
        width="110"
      >
        <template slot-scope="scope">
          <el-tag
            v-if="filterLanguage(scope.row.language).language"
            size="mini"
            effect="plain"
          >
            {{ filterLanguage(scope.row.language).language }}
          </el-tag>
          <span v-else>{{ scope.row.language }}</span>
        </template>
      </el-table-column>
      <el-table-column prop="score" label="评分" width="110">
        <template slot-scope="scope">
          <el-rate
            :value="scope.row.score || 0.0"
            disabled
            score-template="{value}"
          ></el-rate>
        </template>
      </el-table-column>
      <el-table-column
        v-if="settings.display.show_document_view_count || showPrivateData"
        prop="view_count"
        label="浏览"
        width="70"
      >
        <template slot-scope="scope">{{ scope.row.view_count || 0 }}</template>
      </el-table-column>
      <el-table-column
        v-if="settings.display.show_document_download_count || showPrivateData"
        prop="download_count"
        label="下载"
        width="70"
      >
        <template slot-scope="scope">{{
          scope.row.download_count || 0
        }}</template>
      </el-table-column>
      <el-table-column
        v-if="settings.display.show_document_favorite_count || showPrivateData"
        prop="favorite_count"
        label="收藏"
        width="70"
      >
        <template slot-scope="scope">{{
          scope.row.favorite_count || 0
        }}</template>
      </el-table-column>
      <el-table-column
        v-if="showPrivateData"
        prop="status"
        label="状态"
        width="70"
      >
        <template slot-scope="scope">
          <el-tag
            :type="filterStatus(scope.row.status).type"
            size="mini"
            effect="plain"
          >
            {{ filterStatus(scope.row.status).label }}
          </el-tag>
        </template>
      </el-table-column>
      <el-table-column prop="page" label="页数" width="70">
        <template slot-scope="scope">{{ scope.row.pages || '-' }}</template>
      </el-table-column>
      <el-table-column prop="page" label="价格" width="70">
        <template slot-scope="scope">{{ scope.row.price || '0' }}</template>
      </el-table-column>
      <el-table-column prop="size" label="大小" width="100">
        <template slot-scope="scope">{{
          formatBytes(scope.row.size)
        }}</template>
      </el-table-column>
      <el-table-column prop="created_at" label="上传" width="100">
        <template slot-scope="scope">
          <el-tooltip
            :content="formatDatetime(scope.row.created_at)"
            placement="top"
          >
            <span>{{ formatRelativeTime(scope.row.created_at) }}</span>
          </el-tooltip>
        </template>
      </el-table-column>
      <el-table-column
        v-if="userId === user.id"
        label="操作"
        width="70"
        fixed="right"
      >
        <template slot-scope="scope">
          <el-tooltip content="编辑文档" placement="top">
            <el-button
              type="text"
              icon="el-icon-edit"
              :loading="updating"
              @click="updateDocument(scope.row)"
            ></el-button>
          </el-tooltip>
          <el-tooltip content="删除文档" placement="top">
            <el-button
              type="text"
              icon="el-icon-delete"
              @click="deleteDocument(scope.row)"
            ></el-button>
          </el-tooltip>
        </template>
      </el-table-column>
    </el-table>
    <el-pagination
      v-if="total > 0"
      :current-page="query.page"
      :page-size="query.size"
      :layout="
        isMobile
          ? 'total, prev, pager, next'
          : 'total, prev, pager, next, jumper'
      "
      :pager-count="isMobile ? 5 : 7"
      :small="isMobile"
      :total="total"
      class="mgt-20px"
      @current-change="pageChange"
    >
    </el-pagination>
    <el-dialog
      title="编辑文档"
      :visible.sync="updateDocumentVisible"
      :width="isMobile ? '95%' : '640px'"
    >
      <FormUpdateDocument
        :category-trees="categoryTrees"
        :init-document="document"
        :is-admin="false"
        @success="updateDocumentSuccess"
      />
    </el-dialog>
  </div>
</template>

<script>
import { mapGetters } from 'vuex'
import {
  deleteDocument,
  getDocument,
  listDocument,
  searchDocument,
} from '~/api/document'
import {
  formatBytes,
  formatDatetime,
  formatRelativeTime,
  getIcon,
} from '~/utils/utils'
import { datetimePickerOptions, documentStatusOptions } from '~/utils/enum'

export default {
  name: 'UserDocument',
  props: {
    userId: {
      type: Number,
      required: true,
    },
  },
  data() {
    return {
      datetimePickerOptions,
      documentStatusOptions,
      documentStatusOptionsMap: {},
      languageMap: {},
      docs: [],
      total: 0,
      loading: false,
      query: {
        page: parseInt(this.$route.query.page) || 1,
        size: 20,
        wd: '',
        created_at: [],
      },
      updateDocumentVisible: false,
      document: { id: 0 },
      updating: false,
    }
  },
  computed: {
    ...mapGetters('user', ['user', 'permissions']),
    ...mapGetters('category', ['categoryTrees']),
    ...mapGetters('setting', ['settings']),
    showPrivateData() {
      // 如果是用户自身或者是网站管理员则显示私有数据
      return this.userId === this.user.id || this.permissions.length > 0
    },
  },
  watch: {
    '$route.query': {
      handler() {
        this.query = {
          ...this.$route.query,
          page: parseInt(this.$route.query.page) || 1,
          size: parseInt(this.$route.query.size) || 20,
        }
        console.log(this.$route.query, this.query)
        this.getDocuments()
      },
      immediate: true,
    },
  },
  created() {
    console.log('created')
    const statusMap = {}
    this.documentStatusOptions.forEach((item) => {
      statusMap[item.value] = item
    })
    this.documentStatusOptionsMap = statusMap

    const languageMap = {}
    ;(this.settings.language || []).forEach((item) => {
      languageMap[item.code] = item
    })
    this.languageMap = languageMap
    this.getDocuments()
  },
  methods: {
    formatBytes,
    formatDatetime,
    formatRelativeTime,
    async updateDocument(row) {
      this.updating = true
      await this.getDocument(row.id)
      this.updating = false
      this.updateDocumentVisible = true
    },
    async getDocument(id) {
      const res = await getDocument({ id, with_all_content: true })
      if (res.status === 200) {
        this.document = res.data || { id: 0 }
      }
    },
    updateDocumentSuccess() {
      this.updateDocumentVisible = false
      this.getDocuments()
    },
    tabClick(tab) {
      this.activeTab = tab.name
    },
    onSearch() {
      this.query.page = 1
      this.$router.push({
        path: this.$route.path,
        query: this.query,
      })
    },
    async getDocuments() {
      if (this.userId === 0 || this.loading) return
      this.loading = true
      let res
      if (this.query.wd) {
        res = await searchDocument({
          ...this.query,
          user_id: this.userId,
        })
      } else {
        res = await listDocument({
          ...this.query,
          user_id: this.userId,
        })
      }

      console.log(res)

      if (res.status === 200) {
        const docs = res.data.document || []
        docs.map((item) => {
          item.score = item.score / 100 || 0.0
          try {
            item.icon = getIcon(item.ext)
          } catch (e) {
            console.log(e)
          }
          return item
        })
        this.docs = docs
        this.total = res.data.total || 0
      }
      this.loading = false
    },
    pageChange(page) {
      this.$router.push({
        query: { ...this.query, page },
      })
    },
    deleteDocument(row) {
      this.$confirm(`您确定要删除文档《${row.title}》吗？`, '提示', {
        confirmButtonText: '确定',
        cancelButtonText: '取消',
        type: 'warning',
      }).then(async () => {
        const res = await deleteDocument({ id: row.id })
        if (res.status === 200) {
          this.$message({
            type: 'success',
            message: '删除成功!',
          })
          this.getDocuments()
        }
      })
    },
    filterStatus(status) {
      return (
        this.documentStatusOptionsMap[status] || {
          value: status,
          label: '未知',
          type: 'info',
        }
      )
    },
    filterLanguage(lang) {
      return (
        this.languageMap[lang] || {
          value: lang || '-',
          label: lang || '-',
          type: 'info',
        }
      )
    },
  },
}
</script>

<style lang="scss">
.com-user-document {
  .doc-title {
    display: block;
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;
    line-height: 180%;
    img {
      height: 18px;
      position: relative;
      top: 3px;
    }
  }
}
</style>
