<template>
  <div class="com-user-card">
    <div class="user-card-avatar">
      <!-- 如果是用户自己，则点击头像可以更换头像 -->
      <nuxt-link :to="'/user/' + user.id">
        <UserAvatar :user="user" />
      </nuxt-link>
    </div>
    <div class="user-card-username">
      <strong>{{ user.username }}</strong>
    </div>
    <div class="user-card-stat">
      <el-row class="help-block">
        <el-col :span="8">
          <div>文档</div>
          <div class="el-link el-link--primary">{{ user.doc_count || 0 }}</div>
        </el-col>
        <el-col :span="8">
          <div>文章</div>
          <div class="el-link el-link--primary">
            {{ user.article_count || 0 }}
          </div>
        </el-col>
        <el-col :span="8"
          ><div>{{ settings.system.credit_name || '魔豆' }}</div>
          <div class="el-link el-link--primary">
            {{ user.credit_count || 0 }}
          </div>
        </el-col>
      </el-row>
    </div>
    <div v-if="!hideSignature" class="user-card-signature">
      <div><small>个性签名</small></div>
      <div class="help-block">
        {{ user.signature || '暂无个性签名' }}
      </div>
    </div>
    <template v-if="!hideLatest">
      <template v-if="type === 'default'">
        <div v-if="latestDocuments.length > 0" class="mgt-20px latest">
          <div class="heading">
            <small
              ><strong>最近上传</strong> <i class="el-icon-time"></i
            ></small>
          </div>
          <div class="help-block">
            <document-simple-list :docs="latestDocuments" />
          </div>
        </div>
      </template>
      <template v-else-if="type === 'article'">
        <div v-if="latestArticles.length > 0" class="mgt-20px latest">
          <div class="heading">
            <small
              ><strong>最近发布</strong> <i class="el-icon-time"></i
            ></small>
          </div>
          <div class="help-block">
            <article-simple-list :articles="latestArticles" />
          </div>
        </div>
      </template>
    </template>
  </div>
</template>
<script>
import { mapGetters } from 'vuex'
import { listArticle } from '~/api/article'
import { listDocument } from '~/api/document'
// 用户信息卡片
export default {
  name: 'UserCard',
  props: {
    user: {
      type: Object,
      default: () => {
        return {
          id: 0,
          name: '',
          avatar: '',
          signature: '',
          doc_count: 0,
          favorite_count: 0,
          credit_count: 0,
        }
      },
    },
    hideActions: {
      type: Boolean,
      default: false,
    },
    hideSignature: {
      type: Boolean,
      default: false,
    },
    hideLatest: {
      type: Boolean,
      default: false,
    },
    type: {
      type: String,
      default: 'default',
    },
  },
  data() {
    return {
      loading: false,
      latestDocuments: [],
      latestArticles: [],
    }
  },
  computed: {
    ...mapGetters('setting', ['settings']),
  },
  watch: {
    user: {
      handler() {
        if (this.type === 'article') {
          this.getLatestArticles()
        } else {
          this.getLatestDocuments()
        }
      },
      immediate: true,
    },
  },
  methods: {
    // 获取最近上传的文档
    async getLatestDocuments() {
      if (
        this.user.id === 0 ||
        this.loading ||
        this.isMobile ||
        !this.user.doc_count ||
        this.hideLatest
      )
        return
      this.loading = true
      const res = await listDocument({
        page: 1,
        size: 5,
        user_id: this.user.id,
      })
      this.loading = false
      if (res.status === 200) {
        this.latestDocuments = res.data.document || []
      }
    },
    async getLatestArticles() {
      if (
        this.user.id === 0 ||
        this.loading ||
        this.isMobile ||
        !this.user.article_count ||
        this.hideLatest
      )
        return
      this.loading = true
      const res = await listArticle({
        page: 1,
        size: 5,
        user_id: this.user.id,
        type: 1,
        status: 1,
      })
      this.loading = false
      if (res.status === 200) {
        this.latestArticles = res.data.article || []
      }
    },
  },
}
</script>
<style lang="scss" scoped>
.com-user-card {
  text-align: center;
  .latest {
    text-align: left;
    margin-bottom: -10px;
    .heading {
      margin-bottom: 10px;
    }
  }
  .el-avatar {
    border: 2px solid #ddd;
    padding: 3px;
    background-color: #fff;
    width: 80px;
    height: 80px;
    &:hover {
      border: 2px solid #409eff;
    }
    img {
      border-radius: 50%;
    }
  }
  .user-card-username {
    margin: 10px 0;
    font-size: 16px;
    small {
      font-size: 13px;
    }
  }
  .user-card-stat {
    margin: 20px 0;
    .help-block {
      font-size: 12px;
      .el-link {
        font-size: 14px;
        margin-top: 5px;
      }
      .el-col {
        border-right: 1px solid #eee;
        &:first-child {
          border-left: 1px solid #eee;
        }
      }
    }
  }
  .user-card-signature {
    text-align: left;
    .help-block {
      margin-top: 10px;
      font-size: 13px;
      text-indent: 2em;
    }
  }
}
</style>
