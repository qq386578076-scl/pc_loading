<template>
  <section>
    <label>筛选条件 <input v-model="query" /></label>
    <button type="button" @click="search">查询（允许快速重查）</button>
    <button type="button" :disabled="batchBusy" @click="loadBoth">并发加载两类数据</button>
    <p v-if="error" role="alert">{{ error }}</p>
    <Loading ref="loader" :fullscreen="false" :delay="300"
      :min-duration="200" :timeout="20000" size="small"
      text="正在加载" @timeout="onTimeout">
      <div class="content">
        <pre>{{ result }}</pre>
        <pre>{{ details }}</pre>
      </div>
    </Loading>
  </section>
</template>

<script>
import Loading from '../Loading.vue';

// 父组件传入真实请求函数：(query, { signal }) => Promise<业务数据>。
// 请求函数应把 signal 转交给支持取消的请求库；不支持时仍由序号阻止旧结果回写。
// AbortController 需要运行环境支持；这里有检测，不要求新增 polyfill。
export default {
  name: 'LoadingUsage',
  components: { Loading },
  props: {
    fetchRows: { type: Function, required: true },
    fetchDetails: { type: Function, required: true }
  },
  data() {
    return { query: '', result: null, details: null, error: '', batchBusy: false };
  },
  created() {
    this.requestSequence = 0;
    this.disposed = false;
    this.controllers = new Set();
    this.searchController = null;
  },
  beforeDestroy() {
    this.disposed = true;
    this.requestSequence += 1;
    this.controllers.forEach(controller => controller.abort());
    this.controllers.clear();
  },
  methods: {
    makeController() {
      const controller = typeof AbortController !== 'undefined' ? new AbortController() : null;
      if (controller) this.controllers.add(controller);
      return controller;
    },
    async search() {
      // 避免查询与批量操作同时改写相同业务数据。
      if (this.batchBusy) return;
      const sequence = ++this.requestSequence;
      if (this.searchController) this.searchController.abort();
      const controller = this.makeController();
      this.searchController = controller;
      const loader = this.$refs.loader;
      const token = loader.open();
      const query = this.query;
      this.error = '';
      try {
        const data = await this.fetchRows(query, { signal: controller ? controller.signal : undefined });
        if (!this.disposed && sequence === this.requestSequence) this.result = data;
      } catch (error) {
        if (!this.disposed && sequence === this.requestSequence && error.name !== 'AbortError') {
          this.error = error.message || '查询失败';
        }
      } finally {
        if (controller) this.controllers.delete(controller);
        if (this.searchController === controller) this.searchController = null;
        // 每个 token 只释放一次，销毁后不再调用组件。
        if (!this.disposed) loader.done(token);
      }
    },
    async loadBoth() {
      if (this.batchBusy) return;
      this.batchBusy = true;
      const sequence = ++this.requestSequence;
      if (this.searchController) this.searchController.abort();
      const controller = this.makeController();
      const loader = this.$refs.loader;
      const query = this.query;
      const options = { signal: controller ? controller.signal : undefined };
      this.error = '';
      const run = async (fetcher, field) => {
        const token = loader.open();
        try {
          const data = await fetcher(query, options);
          if (!this.disposed && sequence === this.requestSequence) this[field] = data;
        } catch (error) {
          if (!this.disposed && sequence === this.requestSequence && error.name !== 'AbortError') {
            this.error = error.message || '加载失败';
          }
        } finally {
          if (!this.disposed) loader.done(token);
        }
      };
      try {
        await Promise.all([run(this.fetchRows, 'result'), run(this.fetchDetails, 'details')]);
      } finally {
        if (controller) this.controllers.delete(controller);
        if (!this.disposed) this.batchBusy = false;
      }
    },
    onTimeout() {
      // 原组件超时不重置计数，这里只展示错误；不能把它当作网络超时。
      this.error = '加载展示超时，请检查网络请求；当前组件存在超时后恢复限制';
    }
  }
};
</script>

<style scoped>
.content { min-height: 240px; }
pre { white-space: pre-wrap; overflow-wrap: anywhere; }
</style>
