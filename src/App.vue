<template>
  <div style="height: 100vh">
    <VirtualList :dataSource="list" :itemHeight="22" :onReachBottom="()=>{
      console.log('到底')
    }">
      <template #default="{item}">
        <div>{{ item.value }}</div>
      </template>
    </VirtualList>
  </div>

</template>
<script>
import VirtualList from './components/VirtualList.vue'

export default {
  name: 'app',
  components: {VirtualList},
  data() {
    return {
      list: new Array(100).fill(0).map((_, index) => ({value: index + 1}))
    }
  }
}
</script>

