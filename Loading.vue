<template>
    <!--
        局部模式：组件本身作为相对定位容器，包裹默认插槽内容，遮罩在其内部绝对定位
        全屏模式：根节点为占满视口的 fixed 遮罩
        SSR 安全：shown / isMounted 双守卫，服务端渲染阶段不输出遮罩，避免水合不一致
    -->
    <div
        :class="rootClass"
        :style="rootStyle"
        :aria-busy="shown ? 'true' : 'false'"
    >
        <!-- 局部模式下的真实内容，始终占位，避免内容插入时页面跳动 -->
        <slot v-if="!fullscreen"></slot>

        <transition name="loading-fade" @after-enter="onAfterEnter" @after-leave="onAfterLeave">
            <div
                v-if="shown && isMounted"
                class="loading-overlay"
                :class="{ 'loading-overlay--local': !fullscreen }"
                role="status"
                aria-live="polite"
                :style="overlayStyle"
            >
                <!-- 骨架屏优先：提供 skeleton 插槽时用占位骨架替代纯 spinner，避免视觉/布局跳变 -->
                <div v-if="hasSkeleton" class="loading-skeleton">
                    <slot name="skeleton"></slot>
                </div>
                <template v-else>
                    <div class="loading-box" :style="scaleStyle">
                        <div class="loading-spinner">
                            <div class="spinner-ring spinner-ring--outer"></div>
                            <div class="spinner-ring spinner-ring--inner"></div>
                            <div class="spinner-core">
                                <span
                                    class="spinner-dot"
                                    v-for="n in 8"
                                    :key="n"
                                    :style="{ '--i': n }"
                                ></span>
                            </div>
                        </div>
                        <p class="loading-text" aria-live="polite">
                            {{ text }}<span class="loading-dots"><i>.</i><i>.</i><i>.</i></span>
                        </p>
                    </div>
                </template>
                <!-- 供屏幕阅读器朗读的隐藏文本 -->
                <span class="loading-sr-only">{{ text }}，请稍候</span>
            </div>
        </transition>
    </div>
</template>

<script>
// 尺寸档位 -> 基准像素（spinner 原始设计为 120px，通过 scale 统一缩放避免重写关键帧）
// default 对齐 index.vue 的 120px，保证默认观感一致
const SIZE_MAP = { small: 64, default: 120, large: 144 };
const BASE_SIZE = 120;
// 模块级引用计数：多个实例/多个请求共用同一份 body 滚动锁，避免互相解锁
let bodyLockCount = 0;

export default {
    name: 'Loading',
    props: {
        // 受控模式：父组件用布尔值驱动（兼容既有 <Loading :loading="flag" /> 用法）
        loading: {
            type: Boolean,
            default: false
        },
        // 问题1：请求很快返回时不闪一下 —— 延迟多少毫秒才真正显示（0 表示立即显示）
        delay: {
            type: Number,
            default: 300
        },
        // 问题7：真正显示后至少停留多久再消失，避免闪现即灭造成的跳动（毫秒）
        minDuration: {
            type: Number,
            default: 200
        },
        // 问题2：请求失败/卡住时的兜底超时，超时后自动关闭并 emit('timeout')（毫秒，0 表示不超时）
        timeout: {
            type: Number,
            default: 20000
        },
        // 问题11：尺寸可配置
        size: {
            type: [String, Number],
            default: 'default'
        },
        // 问题11：全屏或局部
        fullscreen: {
            type: Boolean,
            default: true
        },
        // 问题8：遮罩打开时锁定背景滚动与点击
        lock: {
            type: Boolean,
            default: true
        },
        text: {
            type: String,
            default: '数据加载中'
        },
        // 问题11：遮罩层级可配置（仅全屏模式生效，局部模式跟随容器层叠上下文）
        zIndex: {
            type: Number,
            default: 9999
        }
    },
    data() {
        return {
            shown: false,          // 遮罩是否处于显示状态
            isMounted: false,      // SSR/水合守卫
            pending: 0,            // 问题5：进行中的请求数（引用计数，多个请求共用一个圈）
            generation: 0,         // 问题4：代数令牌，快速筛选时旧结果不会覆盖新结果
            propLoading: false,    // 受控 loading prop 对应的内部状态
            scale: 1               // 根据 size 计算的缩放
        };
    },
    computed: {
        rootClass() {
            return this.fullscreen ? 'loading-root loading-root--fullscreen' : 'loading-root loading-root--local';
        },
        rootStyle() {
            // 局部模式撑满父容器需要父级有定位/尺寸；此处只负责遮罩外层
            return {};
        },
        overlayStyle() {
            return {
                zIndex: String(this.zIndexInternal)
            };
        },
        scaleStyle() {
            // 默认尺寸(1:1)时不施加任何 transform，保证与 index.vue 的动画合成路径完全一致，
            // 避免多余的变换层导致子元素动画被提升为不重绘的合成层
            if (Math.abs(this.scale - 1) < 0.001) return {};
            return { transform: `scale(${this.scale})` };
        },
        hasSkeleton() {
            // 是否提供了 #skeleton 插槽
            return !!(this.$scopedSlots && this.$scopedSlots.skeleton);
        }
    },
    watch: {
        // 受控模式：外部布尔值进入/退出引用计数体系
        loading(val) {
            if (val) this.acquireProp();
            else this.releaseProp();
        },
        // pending 变化统一驱动显示/隐藏时序
        pending(val, old) {
            if (val > 0 && old === 0) this.startShow();
            else if (val === 0 && old > 0) this.stopShow();
        }
    },
    created() {
        this.delayTimer = null;
        this.timeoutTimer = null;
        this.minDurationTimer = null;
        // 计算缩放
        this.scale = this.resolveScale();
        // 遮罩层级（响应式依赖 prop，局部模式由 CSS 控制，这里仅作全屏内联样式）
        this.zIndexInternal = this.fullscreen ? this.zIndex : 10;
        // 初始化受控值（created 阶段 prop 已就绪）
        this.propLoading = false;
    },
    mounted() {
        // 问题10：水合完成后才允许渲染遮罩
        this.isMounted = true;
        if (this.loading) this.acquireProp();
        // size 变化时重算缩放
        this.$watch('size', () => {
            this.scale = this.resolveScale();
        });
    },
    beforeDestroy() {
        // 问题3：卸载时清理所有计时器并释放 body 锁，防止计时器泄漏继续跑
        this.clearAllTimers();
        this.releaseBodyLock();
    },
    methods: {
        resolveScale() {
            let px;
            if (typeof this.size === 'number') px = this.size;
            else px = SIZE_MAP[this.size] || SIZE_MAP.default;
            return Math.max(0.4, px / BASE_SIZE);
        },
        // ====== 引用计数：解决多个请求共用一个 loading（问题5） ======
        // prop 源用布尔守卫（一个布尔最多占 1 个计数）；manual 源用真实计数（每次 open 各占 1）
        acquireProp() {
            if (this.propLoading) return;
            this.propLoading = true;
            this.pending += 1;
        },
        releaseProp() {
            if (!this.propLoading) return;
            this.propLoading = false;
            this.pending = Math.max(0, this.pending - 1);
        },
        // ====== 对外命令式 API：支持 async 请求包裹，天然处理竞态与多请求 ======
        // 每次 open() 返回一个 token，请求结束时调用 done(token)
        // 多个并发请求各自 +1，全部 done 后 pending 归零才关圈（问题5）
        open() {
            const token = ++this.generation;
            this.pending += 1;
            return token;
        },
        // done(token)：始终释放一个计数；返回值告知调用方这是否是最新一次请求，
        // 便于其丢弃陈旧响应、避免旧结果覆盖新结果（问题4）
        done(token) {
            this.pending = Math.max(0, this.pending - 1);
            return token == null || token === this.generation;
        },
        // ====== 显示时序 ======
        startShow() {
            // 新的加载开始：取消任何待执行的隐藏 / 超时 / 延迟，避免闪灭与叠加
            this.clearAllTimers();
            if (this.shown) {
                // 已在显示中，仅需重置超时兜底
                this.armTimeout();
                return;
            }
            if (this.delay > 0) {
                // 问题1：延迟窗口内如果请求已完成（pending 归零会走 stopShow 取消它），则不显示，避免闪烁
                this.delayTimer = setTimeout(() => {
                    this.delayTimer = null;
                    if (this.pending > 0) this.doShow();
                }, this.delay);
            } else {
                this.doShow();
            }
        },
        doShow() {
            if (!this.isMounted) return;
            this.shown = true;
            this.showStartAt = Date.now();
            // 问题8：锁定背景滚动
            this.applyBodyLock();
            // 问题2：卡住/失败兜底
            this.armTimeout();
        },
        armTimeout() {
            this.clearTimeoutTimer();
            if (this.timeout > 0) {
                this.timeoutTimer = setTimeout(() => {
                    this.timeoutTimer = null;
                    // 视为异常结束：强制隐藏并通知外部
                    this.forceHide();
                    this.$emit('timeout');
                }, this.timeout);
            }
        },
        stopShow() {
            // 延迟窗口内就被取消：从未显示，直接撤销即可（问题1）
            if (this.delayTimer) {
                clearTimeout(this.delayTimer);
                this.delayTimer = null;
                return;
            }
            if (!this.shown) return;
            // 问题7：保证最短显示时长，避免闪现即灭造成内容跳动
            const elapsed = Date.now() - (this.showStartAt || 0);
            const remaining = Math.max(0, this.minDuration - elapsed);
            this.clearTimeoutTimer();
            if (remaining > 0) {
                this.minDurationTimer = setTimeout(() => {
                    this.minDurationTimer = null;
                    this.doHide();
                }, remaining);
            } else {
                this.doHide();
            }
        },
        forceHide() {
            this.clearAllTimers();
            this.doHide();
        },
        doHide() {
            if (!this.shown) return;
            this.shown = false;
            this.clearTimeoutTimer();
            // body 锁在 after-leave 释放，保证淡出动画期间仍锁定
        },
        onAfterEnter() {
            this.$emit('shown');
        },
        onAfterLeave() {
            this.releaseBodyLock();
            this.$emit('hidden');
        },
        // ====== body 滚动/点击锁（模块级引用计数，多实例安全） ======
        applyBodyLock() {
            if (this._bodyLocked) return;
            if (typeof document === 'undefined') return; // 问题10 SSR 守卫
            if (!(this.lock && this.fullscreen)) return;
            bodyLockCount += 1;
            document.body.classList.add('loading-scroll-lock');
            this._bodyLocked = true;
        },
        releaseBodyLock() {
            if (!this._bodyLocked) return;
            this._bodyLocked = false;
            if (typeof document === 'undefined') return;
            bodyLockCount = Math.max(0, bodyLockCount - 1);
            if (bodyLockCount === 0) {
                document.body.classList.remove('loading-scroll-lock');
            }
        },
        // ====== 计时器清理（问题3） ======
        clearTimeoutTimer() {
            if (this.timeoutTimer) {
                clearTimeout(this.timeoutTimer);
                this.timeoutTimer = null;
            }
        },
        clearAllTimers() {
            if (this.delayTimer) { clearTimeout(this.delayTimer); this.delayTimer = null; }
            if (this.timeoutTimer) { clearTimeout(this.timeoutTimer); this.timeoutTimer = null; }
            if (this.minDurationTimer) { clearTimeout(this.minDurationTimer); this.minDurationTimer = null; }
        }
    }
};
</script>

<style scoped>
/* ========== 容器 ========== */
.loading-root--local {
    position: relative;
    display: block;
}

.loading-root--fullscreen {
    display: contents; /* 全屏遮罩用 fixed，容器本身不参与布局 */
}

.loading-overlay {
    position: fixed;
    width: 100vw;
    height: 100vh;
    top: 0;
    left: 0;
    z-index: 9999;
    background: radial-gradient(ellipse at center, rgba(10, 25, 60, 0.85) 0%, rgba(0, 0, 0, 0.42) 100%);
    display: flex;
    flex-direction: column;
    justify-content: center;
    align-items: center;
    /* 问题8：遮罩层拦截所有指针事件，背景不可点击 */
    pointer-events: auto;
}

/* 局部模式：改为相对容器绝对定位，只覆盖内容区 */
.loading-overlay--local {
    position: absolute;
    width: 100%;
    height: 100%;
    z-index: 10;
}

/* ========== Spinner（用 scale 统一缩放，避免重写关键帧） ========== */
.loading-box {
    display: flex;
    flex-direction: column;
    justify-content: center;
    align-items: center;
    transform-origin: center center;
}

.loading-spinner {
    position: relative;
    width: 120px;
    height: 120px;
    display: flex;
    justify-content: center;
    align-items: center;
}

.spinner-ring {
    position: absolute;
    border-radius: 50%;
    border: 3px solid transparent;
}

.spinner-ring--outer {
    width: 120px;
    height: 120px;
    border-top-color: #53aaf9;
    border-right-color: #53aaf9;
    animation: spin 1.5s linear infinite;
    box-shadow: 0 0 20px rgba(83, 170, 249, 0.4), inset 0 0 20px rgba(83, 170, 249, 0.1);
}

.spinner-ring--inner {
    width: 90px;
    height: 90px;
    border-bottom-color: #00e5ff;
    border-left-color: #00e5ff;
    animation: spin 1s linear infinite reverse;
    box-shadow: 0 0 15px rgba(0, 229, 255, 0.4), inset 0 0 15px rgba(0, 229, 255, 0.1);
}

.spinner-core {
    position: relative;
    width: 50px;
    height: 50px;
}

.spinner-dot {
    position: absolute;
    width: 12px;
    height: 6px;
    background: #53aaf9;
    border-radius: 50%;
    top: 50%;
    left: 50%;
    margin: -3px 0 0 -6px;
    animation: dotPulse 1.8s ease-in-out infinite;
    animation-delay: calc((var(--i) - 1) * -0.225s);
    transform: rotate(calc((var(--i) - 1) * 45deg)) translateX(18px);
    box-shadow: 0 0 10px rgba(83, 170, 249, 0.8);
}

.spinner-dot:nth-child(odd) {
    background: #00e5ff;
    box-shadow: 0 0 10px rgba(0, 229, 255, 0.8);
}

.loading-text {
    margin-top: 24px;
    color: #e0f0ff;
    font-size: 16px;
    font-weight: 500;
    letter-spacing: 2px;
    text-shadow: 0 0 10px rgba(83, 170, 249, 0.6);
}

.loading-dots i {
    animation: dotBlink 1.4s infinite;
    font-style: normal;
}

.loading-dots i:nth-child(2) {
    animation-delay: 0.2s;
}

.loading-dots i:nth-child(3) {
    animation-delay: 0.4s;
}

/* ========== 骨架屏占位 ========== */
.loading-skeleton {
    width: 100%;
    height: 100%;
    display: flex;
    justify-content: center;
    align-items: center;
    overflow: hidden;
}

/* ========== 无障碍：屏幕阅读器专用隐藏文本 ========== */
.loading-sr-only {
    position: absolute;
    width: 1px;
    height: 1px;
    padding: 0;
    margin: -1px;
    overflow: hidden;
    clip: rect(0, 0, 0, 0);
    white-space: nowrap;
    border: 0;
}

/* ========== 淡入淡出（局部内容交换用不透明度避免跳变，问题7） ========== */
.loading-fade-enter-active,
.loading-fade-leave-active {
    transition: opacity 0.4s ease;
}

.loading-fade-enter,
.loading-fade-leave-to {
    opacity: 0;
}

@keyframes spin {
    0% { transform: rotate(0deg); }
    100% { transform: rotate(360deg); }
}

/* 减小明暗与尺寸反差，负延迟使首次展示即处于错峰呼吸状态。 */
@keyframes dotPulse {
    0%, 100% {
        opacity: 0.55;
        transform: rotate(calc((var(--i) - 1) * 45deg)) translateX(18px) scale(0.85);
    }
    50% {
        opacity: 0.95;
        transform: rotate(calc((var(--i) - 1) * 45deg)) translateX(18px) scale(1.05);
    }
}
@keyframes dotBlink {
    0%, 20% { opacity: 0; }
    50% { opacity: 1; }
    100% { opacity: 0; }
}

/* ========== 问题9：尊重系统「减少动态效果」偏好 ========== */
@media (prefers-reduced-motion: reduce) {
    .spinner-ring--outer,
    .spinner-ring--inner {
        animation-duration: 4s; /* 保留极缓的旋转作为加载指示，避免完全静止让人误以为卡死 */
    }
    /* 中心圆点保留与 index.vue 一致的透明度脉冲。 */
    .loading-dots i {
        animation: none;
        opacity: 1;
    }
    .loading-fade-enter-active,
    .loading-fade-leave-active {
        transition: none;
    }
}
</style>

<style>
/* 全局：body 滚动锁（不能用 scoped，否则命中不到 body）—— 问题8 */
body.loading-scroll-lock {
    overflow: hidden !important;
    touch-action: none;
    overscroll-behavior: contain;
}
</style>
