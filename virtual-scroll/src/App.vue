<template>
  <div id="app">
		<virtual-list :size="120" :keeps="8" :arrayData="list" :variable="true">
			<template #default="{ item }">
				<div style="border: 1px solid cadetblue; padding: 20px;">
					{{ item.id}}. {{ item.value }}
				</div>
			</template>
		</virtual-list>
  </div>
</template>

<script>
import VirtualList from './components/VirtualList.vue'
import Mock from 'mockjs'

const list = []
for (let i = 0; i < 1000; i++) {
	list.push({
		id: i,
		value: Mock.Random.cparagraph( 3, 10 )
	})
}
export default {
  name: 'App',
  components: {
    VirtualList
  },
	data() {
		return { list }
	}
}
</script>

<style lang="less">
* {
	margin: 0;
	padding: 0;
	box-sizing: border-box;
}
</style>
