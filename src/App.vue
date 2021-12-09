<template>
  <div id="app">
    <div class="container">
      <VisualList 
        :size="40" 
        :items="items"
        :remain="8"
        :variable="true"
      >
        <template v-slot="item">
          <Item :info="item.id + ' ------  ' + item.value"></Item>
        </template>
      </VisualList>
    </div>
  </div>
</template>

<script>
import Mock from 'mockjs';
import VisualList from './components/visual-list'
import Item from './components/item.vue'

let items = []

for (let i = 0; i < 1000; i++) {
  items.push({ id: i, value: Mock.Random.sentence(5, 20) })
}

export default {
  name: 'App',
  components: {
    VisualList,
    Item
  },
  data() {
    return {
      items
    }
  }
}
</script>

<style lang="scss">
#app {
  margin: 0;
  padding: 0;
  * {
    box-sizing: border-box;
  }
  .container {
    width: 400px;
    margin: 200px auto;
    border: 1px solid #ccc;
    .item {
      padding: 5px;
    }
  }
}
</style>
