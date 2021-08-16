<template>
	<!-- 展示区域 -->
	<div class="wrap" ref="wrap" @scroll="scrollFn">
		<!-- 为了显示滚动条 -->
		<div ref="scrollHeight"></div>
		<!-- 展示的内容 -->
		<div class="visible-wrap" :style="{transform: `translateY(${offset}px)`}">
			<div v-for="item in visibleData" :key="item.id" :id="item.id" ref="items">
				<slot :item="item"></slot>
			</div>
		</div>
	</div>
</template>

<script>
import throttle from 'lodash/throttle'
export default {
  name: 'VirtualList',
  props: {
    size: Number, // 每一项的高度或预估高度
		keeps: Number, // 渲染的项目数
		arrayData: Array, // 总列表的数据
		variable: Boolean // 每一项是否高度固定
  },
	data() {
		return {
			start: 0, // 展示的开始项
			end: this.keeps, // 展示的结尾项（不包括）
			offset: 0 // 列表内容的偏移量
		}
	},
	computed: {
		visibleData() {
			// 下面有用到 Math.min 是防止前面或后面没有 this.keeps 项，也就是刚开始或快到底的情况
			const prevCount = Math.min(this.start, this.keeps) // 展示的项数之前多渲染几项
			this.prevCount = prevCount // handleScroll 方法里要用
			const renderStart = this.start -  prevCount // 真正的渲染开始项
			const nextCount = Math.min(this.arrayData.length - this.end, this.keeps) // 展示的项数之后多渲染几项
			const renderEnd = this.end +  nextCount // 真正的渲染结束项的后一项，因为 slice 是不包括 end 参数的
			return this.arrayData.slice(renderStart, renderEnd) 
		}
	},
	created() {
		this.scrollFn = throttle(this.handleScroll, 200, { 'leading': false })
	},
	mounted() {
		// 计算列表如果全部渲染应该有的高度
		this.$refs.scrollHeight.style.height = this.arrayData.length * this.size + 'px'
		// 计算可视区域的高度
		this.$refs.wrap.style.height = this.keeps * this.size + 'px'
		// 缓存每一项的高度
		this.cacheListPosition()
	},
	updated() {
		this.$nextTick(() => {
			// 确保 dom 已经更新
			const domArr = this.$refs.items
			if (!(domArr && domArr.length > 0)) return // 如果没有 dom 就返回
			domArr.forEach(item => {
				const { height } = item.getBoundingClientRect() // 获取每个节点的高度
				// 更新缓存在 positionListArr 里的数据，先拿到这个节点的 id，id 跟 index 是一一对应的
				const id = item.getAttribute('id') // 先拿到这个节点的 id
				const oldHeight = this.positionListArr[id].height // 获取缓存数组里对应的这一项的原来的高度
				const difference = oldHeight - height
				if (difference) { // 如果高度变了，更新这一项的高度和 bottom（height 的变化不影响 top）
					this.positionListArr[id].height = height
					this.positionListArr[id].bottom = this.positionListArr[id].bottom - difference
					// 后面的每一项相应的也要调整
					for (let i = id + 1; i < this.positionListArr.length; i++) {
						if (this.positionListArr[i]) {
							this.positionListArr[i].top = this.positionListArr[i - 1].bottom // 后一项的 top 等于前一项的 bottom
							this.positionListArr[i].bottom = this.positionListArr[i].bottom - difference
						}
					}
				}
			})
			// 重新计算列表如果全部渲染应该有的高度
			this.$refs.scrollHeight.style.height = this.positionListArr[this.positionListArr.length - 1].bottom + 'px'
		})
	},
	methods: {
		handleScroll() {
			const scrollTop = this.$refs.wrap.scrollTop // 滚动条滚动距离
			if (this.variable) { // 每一项高度不固定
				this.start = this.getStartIndex(scrollTop) // 获取开始展示的那一项的下标
				this.end = this.start + this.keeps
				// 下面这句用三元运算符是因为 this.positionListArr[this.start - this.prevCount] 的值可能为 undefined，
				// 比如当从下往回滚动的时候 this.prevCount 的值有可能会有等于前一次滚动时的 this.start 的值的时候，大于现在 this.start 的值
				this.offset = this.positionListArr[this.start - this.prevCount] ? this.positionListArr[this.start - this.prevCount].top : 0 
			} else { // 每一项高度固定
				// 计获取开始展示的那一项的下标，减 1 是因为展示的数据是从第 0 项开始的
				this.start = Math.ceil(scrollTop / this.size) - 1 >= 0 ? Math.ceil(scrollTop / this.size) - 1 : 0
				this.end = this.start + this.keeps
				// 当列表向上（下）滚动时，为了让展示的列表一直处于可视范围内，就要把列表向下（上）挪
				this.offset = (this.start - this.prevCount) * this.size
			}
		},
	
		getStartIndex(scrollTop) {
			// 用二分法查找滚动距离是 positionListArr 哪一项的 bottom 的值
			let start = 0, // positionListArr 的第一项的下标
			end = this.positionListArr.length - 1, // positionListArr 的最后一项的下标
			// 用 temp 存储得不到精确值时最终返回的值，因为可能滚动的距离的值不会等于 positionListArr 数组里
			// 任何一项的 bottom 的值，所以只好存储一个最接近的值 
			temp = null
			while (start <= end) {
				let midIndex = parseInt(start + (end - start) / 2), // 中间那一项的 index，也可以用 Math.floor 代替 parseInt
				midVal = this.positionListArr[midIndex].bottom // 中间那一项的 bottom 值
				if (scrollTop === midVal) {
					// 滚动的值刚好等于中间那一项的 bottom 值，则此时的展示起点 start 应该为中间那一项的下一项 
					return midIndex + 1
				} else if (scrollTop < midVal) {
					// 滚动的值小于中间那一项的 bottom 值:
					end = midIndex - 1 // 让 end 移动到中间的那一项的前面一项
					if (temp === null || temp > midIndex ) // temp > midIndex 保证了目标值所在范围越变越小
						temp = midIndex
				} else if (scrollTop > midVal) {
					// 滚动的值大于中间那一项的 bottom 值:
					start = midIndex + 1 // 让 start 移动到中间的那一项的后面一项
					if (temp === null || temp < midIndex )
						temp = midIndex
				}
			}
			return temp
		},
		
		cacheListPosition() {
			// 缓存数据数组每一项的高度，top 值和 bottom 值
			// 注意: () => ({}) 为返回对象的写法
			this.positionListArr = this.arrayData.map((item, index) => ({
				height: this.size,
				top: index * this.size,
				bottom: (index + 1) * this.size
			}))
		}
	}
}
</script>


<style scoped lang="less">
.wrap {
	position: relative;
	overflow-y: scroll;
}

.visible-wrap {
	position: absolute;
	left: 0;
	top: 0;
	width: 100%;
}
</style>
