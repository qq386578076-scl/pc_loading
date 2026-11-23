# pc_loading

loading加载

# Vue 2 Loading 组件

适用于 Vue 2.7 的单文件组件。直接复制源码到现有 Vue 项目使用，无额外 UI 库依赖，不是已发布的 npm 包。

> 当前版本已更新动画样式，尚未修复已知状态管理缺陷，不应直接视为生产就绪。不要在不了解下方限制时直接替换业务加载逻辑。

## 本次更新（2026-11-23）

中心圆点改为 12 × 6px 椭圆，采用 1.8 秒柔和呼吸，透明度 0.55～0.95、缩放 0.85～1.05，八个椭圆通过负延迟错峰。恢复圆环与文字光晕，遮罩淡入淡出调整为 0.4 秒，默认尺寸调整为 120px，large 为 144px。

完整前后对比、兼容影响与验证范围见 [CHANGELOG.md](./CHANGELOG.md)。这些动画参数目前通过组件 CSS 调整，未新增对应 Props。

## 文件

- `Loading.vue`：从原项目 `indexs.vue` 原样复制的组件。
- `examples/Usage.vue`：可复制的使用示例，通过 props 接收实际请求函数，不包含虚构接口或模拟数据。

## 接入

将 `Loading.vue` 放到项目 `src/components/loading/Loading.vue`，局部注册：

```vue
<template>
  <div>
    <button :disabled="busy" @click="load">加载</button>
    <Loading :loading="busy" @timeout="onTimeout" />
    <p v-if="error" role="alert">{{ error }}</p>
  </div>
</template>
<script>
import Loading from '@/components/loading/Loading.vue';
export default {
  components: { Loading },
  props: { fetchData: { type: Function, required: true } },
  data: () => ({ busy: false, error: '', result: null }),
  methods: {
    async load() {
      if (this.busy) return;
      this.busy = true;
      this.error = '';
      try { this.result = await this.fetchData(); }
      catch (error) { this.error = error.message || '加载失败'; }
      finally { this.busy = false; }
    },
    onTimeout() { this.error = '加载展示超时，请检查请求状态'; }
  }
};
</script>
```

以上仅用于单请求；组件超时不会取消 `fetchData`。生产调用方还需根据请求库增加取消和卸载保护，完整示例见 `examples/Usage.vue`。

也可在入口注册 `Vue.component('Loading', Loading)`。原项目 `@/components/loading` 默认解析 `index.vue`，不会自动加载 `indexs.vue`；需显式导入本文件。

## 参数 Props

| 参数 | 类型 | 默认值 | 作用及注意事项 |
| --- | --- | --- | --- |
| loading | Boolean | false | 受控开关。true 开始加载，false 结束；一个布尔值只能代表一个聚合状态，不会自动统计并发请求。 |
| delay | Number | 300 | 开始后延迟多少毫秒显示；延迟内结束则不显示。0 表示立即显示。 |
| minDuration | Number | 200 | 实际显示后至少保留多少毫秒；另外有约 400ms 淡出动画（减少动态效果模式下禁用过渡）。 |
| timeout | Number | 20000 | 遮罩显示后多少毫秒强制隐藏并触发 timeout；0 关闭超时。不是网络超时，不取消请求、不清零计数。 |
| size | String / Number | 'default' | small=64、default=120、large=144；数字为 spinner 缩放基准像素，最低缩放为 0.4。整个 loading-box 包含文字一起缩放；不是容器尺寸。未知字符串回退 default。 |
| fullscreen | Boolean | true | true 为视口 fixed 遮罩；false 为包裹默认插槽的局部绝对定位遮罩。局部容器须有稳定高度。 |
| lock | Boolean | true | 全屏展示时添加 body 滚动锁。false 只关闭滚动锁，遮罩仍拦截指针点击；局部模式不锁滚动。 |
| text | String | '数据加载中' | spinner 文本及辅助朗读文本。 |
| zIndex | Number | 9999 | 全屏层级；局部内部使用 10。当前仅创建时读取，不支持动态更新。 |

数值建议传有限非负值；size 应为正数。当前源码没有完整的参数合法性校验。除 loading、size 外，不应依赖展示过程中动态修改其他配置来立即同步内部状态或锁。

## 事件

| 事件 | 参数 | 时机 |
| --- | --- | --- |
| shown | 无 | 淡入动画完成。 |
| hidden | 无 | 淡出动画完成并释放滚动锁。 |
| timeout | 无 | 展示超时后请求隐藏；此时淡出可能尚未完成，实际网络请求仍可能进行中。 |

## 插槽

- 默认插槽：仅在 `fullscreen=false` 时渲染真实内容，始终保留在布局中。
- `skeleton`：提供后替换 spinner。骨架位于遮罩内，不占文档流，不自动撑高容器。

```vue
<Loading :loading="busy" :fullscreen="false" size="small">
  <section style="min-height: 240px">实际内容</section>
  <template #skeleton>
    <div aria-hidden="true">请替换为项目已有骨架组件</div>
  </template>
</Loading>
```

## 命令式 API 与并发

通过 `<Loading ref="loader" />` 获取实例；挂载后才能使用。

| 方法 | 返回值 | 用途 |
| --- | --- | --- |
| open() | 数字 token | pending 加 1，返回递增 token。 |
| done(token) | Boolean | pending 减 1；返回 token 是否为最后一次 open 的 token。不传 token 始终返回 true，但仍减计数。 |

每次 open 必须且只能对应一次 done。不同并发请求应保存各自 token，全部结束才隐藏。不要把多个独立请求直接绑定同一个布尔值。

快速筛选时，应在写结果之前判断 `done(token)` 返回值，而且必须在失败路径释放 token。示例文件同时使用调用方序号：序号负责业务结果新旧判断，token 只负责加载计数。共享 loader 的所有 open 都共享同一个“最新 token”，不适合把多个独立业务的结果新旧判断混在一起。

`forceHide()` 只隐藏展示，不重置请求计数，不能当作请求取消或重置方法。其他方法属于内部实现，不建议业务直接调用。

## 已知缺陷与边界

1. 超时只隐藏遮罩，pending/propLoading 不清零；卡住请求未释放时，后续 open 可能无法重新显示。需要先修复状态机再用于长期运行场景。
2. done 不校验 token 身份和重复完成，错误调用可能提前关圈。
3. 不自动取消请求或阻止旧数据回写；调用方必须处理 finally、请求序号及卸载取消。beforeDestroy 只清理组件现有计时器和 body 锁。
4. 骨架不参与占位；内容区须预留稳定尺寸，不能保证任意内容没有布局跳动。
5. 遮罩拦截指针，不限制背景键盘焦点；局部模式不锁容器滚动。动态切换 lock/fullscreen 不会完整同步滚动锁。
6. zIndex 初始化后不更新；delay/minDuration/timeout 等没有完整动态重配置和输入校验。
7. 有 role=status、aria-live 和减少动态效果样式，但可能重复朗读；减少动态效果下仍有缓慢旋转和中心椭圆呼吸；文字省略号动画与遮罩过渡被禁用。
8. SSR 初始隐藏并在 mounted 后显示，但未进行真实 SSR/SSG 水合验证。fixed 遮罩也可能受祖先 transform 或层叠上下文影响。
9. transform/opacity 动画不能保证页面无掉帧，仍需结合地图、图表等业务进行性能测量。

## 建议验收

- 请求在 50ms 内结束不展示；超过 delay 后展示；结束后满足最短时间再淡出。
- 成功、失败、超时、卸载分别验证；超时后再发新请求检查上述已知问题。
- 两个并发请求分别结束，检查计数；重复 done 属于当前不支持的错误调用。
- 快速筛选先后请求反序返回，确保仅最新序号写入结果。
- 局部空内容、骨架、移动端滚动、键盘焦点、屏幕阅读器和减少动态效果实测。

本次交付未运行浏览器、辅助技术或真实后端验收；没有添加第三方依赖，也没有指定开源许可证，发布或再分发前由代码所有者确定许可。
