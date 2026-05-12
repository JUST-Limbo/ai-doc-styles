# 关于 Vue 2 选择器组件开发思路的若干描述

## 前言

技术文章，尤其是前端技术文章具有时效性。

若文中出现 breaking change、事实错误或表述不当，欢迎在评论区或仓库 issue 中指出。

相关仓库：<https://github.com/JUST-Limbo/informal-essay>

## 摘要

本文从一段列表单选示例出发，说明两件事：其一，用 `v-model` 把「当前选中 id」从列表项数据中剥离出来，避免给接口数据随意挂载 `selected` 字段；其二，在更复杂的场景下，可借鉴 `el-select` 思路，用 `provide` / `inject` 把「选择容器」与「选项内容」拆开，降低重复代码。

## 背景

业务里常见一类「点选卡片 / 行」的需求，形态像选择器，却未必使用现成的 UI 库组件。部分实现会把激活态写回列表元素本身（例如在数据对象上临时加 `selected`），接口返回的数据往往又不带该字段，可读性与数据边界都容易变差。

下面用精简示例说明重构路径。

## 案例与问题定位

```vue
<template>
  <div>
    <div class="list">
      <div
        v-for="(item, index) in tableData"
        :key="index"
        class="item"
        @click="selectAddress(item)"
        :class="{ active: item.selected }"
      >
        <div>{{ item.date }}</div>
        <div>{{ item.name }}</div>
        <div>{{ item.address }}</div>
      </div>
    </div>
    <div>{{ selectId }}</div>
  </div>
</template>

<script>
export default {
  data() {
    return {
      selectId: "",
      tableData: [
        {
          id: 1,
          date: "2016-05-02",
          name: "王小虎",
          address: "上海市普陀区金沙江路 1518 弄",
        },
        {
          id: 2,
          date: "2016-05-04",
          name: "王小虎",
          address: "上海市普陀区金沙江路 1517 弄",
        },
        {
          id: 3,
          date: "2016-05-01",
          name: "王小虎",
          address: "上海市普陀区金沙江路 1519 弄",
        },
      ],
    };
  },
  methods: {
    selectAddress(item) {
      this.tableData.forEach((tableItem) => {
        this.$set(tableItem, "selected", false);
      });
      this.selectId = item.id;
      this.$set(item, "selected", true);
    },
  },
};
</script>

<style lang="scss">
.list {
  display: flex;
  .item {
    border: 1px dashed #d9d9d9;
    border-radius: 2px;
    width: 178px;
    height: 178px;
    border-radius: 6px;
    margin-left: 6px;
    &.active {
      background-color: #409eff;
    }
  }
}
</style>
```

核心问题在于：`.active` 依赖 `tableData` 元素上的 `selected` 字段，而接口数据通常不含该字段。仅为视图态污染源数据，后续维护成本高。

## 用计算激活态替代写入 selected

激活条件改为「当前 `selectId` 与项 id 是否相等」，点击时只更新 `selectId`，不再批量 `$set` `selected`：

```vue
<template>
  <div>
    <div class="list">
      <div
        v-for="(item, index) in tableData"
        :key="index"
        class="item"
        @click="selectAddress(item)"
        :class="{ active: selectId == item.id }"
      >
        ...略,
      </div>
    </div>
    ...略,
  </div>
</template>

<script>
export default {
  ...略,
  methods: {
    selectAddress(item) {
      this.selectId = item.id;
    },
  },
};
</script>

<style lang="scss">
...略
</style>
```

## 支持 v-model 的选择器子组件

单选、多选本质上是「当前值」与父组件同步，可沿用自定义组件 `v-model` 约定（`model` 选项或 Vue 3 的 `modelValue` 等，视版本而定）。

父组件：

```vue
<template>
  <div>
    <div>{{ selectId }}</div>
    <AddressSelect v-model="selectId" :data="tableData" />
  </div>
</template>

<script>
import AddressSelect from "./AddressSelect.vue";
export default {
  components: {
    AddressSelect,
  },
  data() {
    return {
      selectId: "",
      tableData: [
        {
          id: 1,
          date: "2016-05-02",
          name: "王小虎",
          address: "上海市普陀区金沙江路 1518 弄",
        },
        {
          id: 2,
          date: "2016-05-04",
          name: "王小虎",
          address: "上海市普陀区金沙江路 1517 弄",
        },
        {
          id: 3,
          date: "2016-05-01",
          name: "王小虎",
          address: "上海市普陀区金沙江路 1519 弄",
        },
      ],
    };
  },
};
</script>
```

子组件 `AddressSelect.vue`：

```vue
<template>
  <div class="list">
    <div
      v-for="(item, index) in data"
      :key="index"
      class="item"
      @click="selectAddress(item)"
      :class="{ active: value == item.id }"
    >
      <div>{{ item.date }}</div>
      <div>{{ item.name }}</div>
      <div>{{ item.address }}</div>
    </div>
  </div>
</template>

<script>
export default {
  name: "AddressSelect",
  model: {
    prop: "value",
    event: "select",
  },
  props: {
    value: [String, Number],
    data: Array,
  },
  methods: {
    selectAddress(item) {
      this.$emit("select", item.id);
    },
  },
};
</script>
<style lang="scss">
.list {
  display: flex;
  .item {
    border: 1px dashed #d9d9d9;
    border-radius: 2px;
    width: 178px;
    height: 178px;
    border-radius: 6px;
    margin-left: 6px;
    &.active {
      background-color: #409eff;
    }
  }
}
</style>
```

## 用 provide 与 inject 拆分容器与选项

当选择器变多、布局变复杂时，重复写 `model` 与单多选分支成本高。可参考 Element UI `el-select`：由外层 `Selector` 负责当前值、单多选策略与布局类名，内层 `SelectOption` 只关心展示与点击上报。

父页面示意：

```vue
<template>
  <div>
    <div>{{ selectedValue }}</div>
    <Selector v-model="selectedValue" custom-class="horizontal-list">
      <SelectOption1
        v-for="item in tableData"
        :key="item.id"
        :value="item.id"
        v-bind="item"
      ></SelectOption1>
    </Selector>
    <div>{{ selectedValue2 }}</div>
    <Selector v-model="selectedValue2" multiple custom-class="vertical-list">
      <SelectOption2
        v-for="item in tableData"
        :key="item.id"
        :value="item.id"
        v-bind="item"
      ></SelectOption2>
    </Selector>
  </div>
</template>

<script>
import Selector from "./Selector.vue";
import SelectOption1 from "./SelectOption1.vue";
import SelectOption2 from "./SelectOption2.vue";

export default {
  components: {
    Selector,
    SelectOption1,
    SelectOption2,
  },
  data() {
    return {
      selectedValue: "",
      selectedValue2: "",
      tableData: [
        {
          id: 1,
          date: "2016-05-02",
          name: "王小虎",
          address: "上海市普陀区金沙江路 1518 弄",
        },
        {
          id: 2,
          date: "2016-05-04",
          name: "王小虎",
          address: "上海市普陀区金沙江路 1517 弄",
        },
        {
          id: 3,
          date: "2016-05-01",
          name: "王小虎",
          address: "上海市普陀区金沙江路 1519 弄",
        },
      ],
    };
  },
};
</script>

<style lang="scss">
.horizontal-list {
  display: flex;
}
.vertical-list {
}
</style>
```

`Selector.vue`（节选，说明 `provide` 与 `onOptionSelect`）：

```vue
<template>
  <div :class="[customClass]">
    <slot></slot>
  </div>
</template>

<script>
export default {
  name: "Selector",
  inheritAttrs: false,
  props: {
    value: {
      required: true,
    },
    multiple: Boolean,
    customClass: String,
  },
  provide() {
    return {
      $Selector: this,
    };
  },
  created() {
    if (this.multiple && !Array.isArray(this.value)) {
      this.$emit("input", []);
    }
    if (!this.multiple && Array.isArray(this.value)) {
      this.$emit("input", "");
    }
  },
  methods: {
    onOptionSelect(option) {
      if (this.multiple) {
        const targetIndex = this.value.indexOf(option.value);
        const valueClone = this.value.slice();
        if (targetIndex > -1) {
          valueClone.splice(targetIndex, 1);
        } else {
          valueClone.push(option.value);
        }
        this.$emit("input", valueClone);
      } else {
        this.$emit("input", option.value);
      }
    },
    calcItemActive(itemValue) {
      if (this.multiple) {
        return this.value.includes(itemValue);
      } else {
        return this.value == itemValue;
      }
    },
  },
};
</script>
```

`SelectOption1.vue` 与 `SelectOption2.vue` 通过 `inject: ["$Selector"]` 调用 `calcItemActive` 与 `onOptionSelect`，样式上区分横排与竖排即可（完整代码见原仓库，此处不重复粘贴）。

## Vue 2.6 之前 v-model 与 $attrs 的交互

在 `vue@<2.6` 中，`v-model` 与 `v-bind="$attrs"`、`v-on="$listeners"` 同时使用时，可能出现 `value` 未进入 `$attrs` 等问题。升级 Vue 或调整透传策略前，可先阅读以下讨论：

1. [vuejs/vue#6216](https://github.com/vuejs/vue/issues/6216)
2. [vuejs/vue#9330](https://github.com/vuejs/vue/issues/9330)
3. [vuejs/vue#6327](https://github.com/vuejs/vue/pull/6327)
4. [bienvenidoY/blog#6](https://github.com/bienvenidoY/blog/issues/6)

## 总结

选择器类交互应尽量从页面层「摊开的点击处理」收束到独立组件中，用 `v-model` 暴露值；结构复杂时，再把「容器」与「选项」分层，由容器统一单多选逻辑，选项只负责展示与事件上报。这样数据边界清晰，后续替换布局或皮肤时改动面更小。

## 参考文献

1. [Vue 2 API：model](https://v2.cn.vuejs.org/v2/api/#model)
2. [Vue 2 指南：自定义组件的 v-model](https://v2.cn.vuejs.org/v2/guide/components-custom-events.html#自定义组件的-v-model)
3. [vuejs/vue#6216](https://github.com/vuejs/vue/issues/6216)
4. [vuejs/vue#9330](https://github.com/vuejs/vue/issues/9330)
5. [vuejs/vue#6327](https://github.com/vuejs/vue/pull/6327)
6. [bienvenidoY/blog#6](https://github.com/bienvenidoY/blog/issues/6)
