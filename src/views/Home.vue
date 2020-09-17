<template>
  <div class="home">
    <sequence-diagram :sData="sequenceData"></sequence-diagram>
  </div>
</template>

<script>
// @ is an alias to /src
import SequenceDiagram from '@/components/sequenceDiagram.vue'

export default {
  name: 'Home',
  components: {
    SequenceDiagram
  },
  data () {
    return {
      sequenceData: {
        nodeDataArray: [
          { key: 'Fred', text: 'Fred: Patron', isGroup: true, loc: '0 0', duration: 9 },
          { key: 'Bob', text: 'Bob: Waiter', isGroup: true, loc: '100 0', duration: 9 },
          { key: 'Hank', text: 'Hank: Cook', isGroup: true, loc: '200 0', duration: 9 },
          { key: 'Renee', text: 'Renee: Cashier', isGroup: true, loc: '300 0', duration: 9 },
          { group: 'Bob', start: 1, duration: 2 },
          { group: 'Hank', start: 2, duration: 3 },
          { group: 'Fred', start: 3, duration: 1 },
          { group: 'Bob', start: 5, duration: 1 },
          { group: 'Fred', start: 6, duration: 2 },
          { group: 'Renee', start: 8, duration: 1 }
        ],
        linkDataArray: [
          { from: 'Fred', to: 'Bob', text: 'order', time: 1 },
          { from: 'Bob', to: 'Hank', text: 'order food', time: 2 },
          { from: 'Bob', to: 'Fred', text: 'serve drinks', time: 3 },
          { from: 'Hank', to: 'Bob', text: 'finish cooking', time: 5 },
          { from: 'Bob', to: 'Fred', text: 'serve food', time: 6 },
          { from: 'Fred', to: 'Renee', text: 'pay', time: 8 }
        ]
      }
    }
  }
}
</script>

<style scoped lang="scss">
.home {
  width: 1000px;
  margin: 0 auto;
}
</style>
