<template>
  <div>
    <h1>奇</h1>
    <virtual-ruler
      ref="ruler"
      style=""
      auto-resize
      :height="50"
      :min="1"
      :max="10000"
      :default-value="v2"
      :visible="11"
      :point-width="4"
      :linear-gradient="linearGradient"
      :value-gap="0.1" @change="v2 = $event">
    </virtual-ruler>
    <div class="text">{{v2}}</div>
    <virtual-ruler
      ref="ruler2"
      style="margin-top: 100px;"
      auto-resize
      :height="50"
      :min="1"
      :max="10000"
      :default-value="v2"
      :visible="30"
      :point-width="4"
      :linear-gradient="linearGradient"
      :value-gap="1000" @change="v2 = $event">
    </virtual-ruler>
    <h1>偶</h1>
    <input v-model="inputValue" type="number"/>
    <button @click="handleSend">发送</button>
  </div>
</template>

<script>
import VirtualRuler from '@/components/VirtualRuler.vue';
export default {
  name: 'VirtualRulerView',
  components: {
    VirtualRuler
  },
  data () {
    return {
      v2: 50000,
      linearGradient: false,
      inputValue: 50000,
    };
  },
  methods: {
    handleSend() {
      const value = Number(this.inputValue)
      this.$refs.ruler2.setValue(value)
    }
  }
};
</script>

<style>
.text {
  color: black;
  font-size: 32px;
}
</style>