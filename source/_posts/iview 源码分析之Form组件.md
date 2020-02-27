title: iview 源码分析之 Form 组件
date: {{ 2019-12-07 }}
---

### 前言

在 HTML 中，`<form>`元素表示了文档中的一个区域，此区域包含有交互控制元件，用来向 Web 服务器提交信息，其主要解决的问题就是三类问题：

- 数据赋值、数据获取：让*Form*内的表单组件获取到数据并展示出来，同时这些组件上的数据也能反过来让*Form*组件获取到
- 数据校验：对数据的有效性进行校验

经常使用 iview 的小伙伴应该对 form 组件非常熟悉，一般使用写成这个样子：

```html
<template>
  <Form
    inline
    ref="formInline"
    :model="formInline"
    :rules="ruleInline">
      <FormItem prop="user">
        <Input type="text" v-model="formInline.user" placeholder="Username">
          <Icon type="ios-person-outline" slot="prepend"></Icon>
        </Input>
      </FormItem>
      <FormItem prop="password">
        <Input type="password" v-model="formInline.password" placeholder="Password">
          <Icon type="ios-lock-outline" slot="prepend"></Icon>
        </Input>
      </FormItem>
      <FormItem>
        <Button type="primary" @click="handleSubmit('formInline')">Signin</Button>
      </FormItem>
  </Form>
</template>

<script>
export default {
  data () {
    return {
      formInline: {
        user: '',
        password: ''
      },
      ruleInline: {
        user: [
          { required: true, message: 'Please fill in the user name', trigger: 'blur' }
        ],
        password: [
          { required: true, message: 'Please fill in the password.', trigger: 'blur' },
          { type: 'string', min: 6, message: 'The password length cannot be less than 6 bits', trigger: 'blur' }
        ]
      }
  }
},
methods: {
  handleSubmit(name) {
    this.$refs[name].validate((valid) => {
      if (valid) {
        this.$Message.success('Success!');
      } else {
        this.$Message.error('Fail!');
      }
     })
    }
  }
}
</script>
```

了解了大概的使用之后，我看打开源码去一一分析，打开[传送门](https://github.com/iview/iview/tree/2.0/src/components/form)，大概可以看出：

- 整个Form组件利用了 HTML的`form`标签进行封装。
- 其中FormItem作为表单组件的容器，通过`Form`的slot获取。

#### Form

```html
<template>
  <form :class="classes" :autocomplete="autocomplete">
    <slot></slot>
  </form>
</template>

<!-- 
  autocomplete: 自动完成属性;
  浏览器会存储输入的内容会出现下拉框让用户便捷输入;
  有的表单项例如：验证码输入等及时性的，就不是很适合使用该属性;
  iview这里默认设置为off
-->

<script>
import { oneOf } from "../../utils/assist";

const prefixCls = "ivu-form";

export default {
  name: "iForm",
  props: { // 属性说明阅读文档：==> https://www.iviewui.com/components/form
    model: {
      type: Object
    },
    rules: {
      type: Object
    },
    labelWidth: {
      type: Number
    },
    labelPosition: {
      validator(value) {
        return oneOf(value, ["left", "right", "top"]);
      },
      default: "right"
    },
    inline: {
      type: Boolean,
      default: false
    },
    showMessage: {
      type: Boolean,
      default: true
    },
    autocomplete: {
      validator(value) {
        return oneOf(value, ["on", "off"]);
      },
      default: "off"
    }
  },
  provide() {
    return { form: this };
  },
  data() {
    return {
      fields: []
    };
  },
  computed: {
    classes() {
      return [
        `${prefixCls}`,
        `${prefixCls}-label-${this.labelPosition}`,
        {
          [`${prefixCls}-inline`]: this.inline
        }
      ];
    }
  },
  methods: {
    resetFields() {
      this.fields.forEach(field => {
        field.resetField();
      });
    },
    validate(callback) {
      return new Promise(resolve => {
        let valid = true;
        let count = 0;
        this.fields.forEach(field => {
          field.validate("", errors => {
            if (errors) {
              valid = false;
            }
            if (++count === this.fields.length) {
              // all finish
              resolve(valid);
              if (typeof callback === "function") {
                callback(valid);
              }
            }
          });
        });
      });
    },
    validateField(prop, cb) {
      const field = this.fields.filter(field => field.prop === prop)[0];
      if (!field) {
        throw new Error(
          "[iView warn]: must call validateField with valid prop string!"
        );
      }

      field.validate("", cb);
    }
  },
  watch: {
    rules() {
      this.validate();
    }
  },
  created() {
    this.$on("on-form-item-add", field => {
      if (field) this.fields.push(field);
      return false;
    });
    this.$on("on-form-item-remove", field => {
      if (field.prop) this.fields.splice(this.fields.indexOf(field), 1);
      return false;
    });
  }
};
</script>
```

##### data && computed 

- fields: FormItem组件mounted之后会传递到Form来，并收集到`this.fields`数组中
- classes: 不同样式的class名


#####  provide/inject

> 简单的来说就是在父组件中通过provider来提供变量，然后在子组件中通过inject来注入变量。需要注意的是这里不论子组件有多深，只要调用了inject那么就可以注入provider中的数据。

在这里Form组件将自身的实例传递出去，FormItem通过inject接受属性，所以我们在Form和FormItem中进行嵌套，并不会影响它们的通讯。

##### methods & watch

- resetFields: 遍历this.fields,并调用FormItem实例上的`resetField`方法
- validate: 表单校验，返回一个promise,并没有执行，返回valid是否通过
- validateField: 单独校验某一个字段
- watch-rules: 监听定义的rules，更新validate

##### created

- 监听on-form-item-add： 添加校验字段
- on-form-item-remove： 移除校验字段

#### FormItem

```html
<template>
  <div :class="classes">
    <label
      :class="[prefixCls + '-label']"
      :for="labelFor"
      :style="labelStyles"
      v-if="label || $slots.label">
      <slot name="label">{{ label }}</slot>
    </label>
    <div :class="[prefixCls + '-content']" :style="contentStyles">
      <slot></slot>
      <transition name="fade">
        <div
          :class="[prefixCls + '-error-tip']"
          v-if="validateState === 'error' && showMessage && form.showMessage"
        >{{ validateMessage }}</div>
      </transition>
    </div>
  </div>
</template>
<script>
import AsyncValidator from "async-validator";
import Emitter from "../../mixins/emitter";

const prefixCls = "ivu-form-item";

// 'items.' + index + '.value'  => "items.1.value"
// 根据prop嵌套查询的去获取model中的值

function getPropByPath(obj, path) {
  let tempObj = obj;
  path = path.replace(/\[(\w+)\]/g, ".$1");
  path = path.replace(/^\./, "");

  let keyArr = path.split(".");
  let i = 0;

  for (let len = keyArr.length; i < len - 1; ++i) {
    let key = keyArr[i];
    if (key in tempObj) {
      tempObj = tempObj[key];
    } else {
      throw new Error(
        "[iView warn]: please transfer a valid prop path to form item!"
      );
    }
  }
  return {
    o: tempObj, // ==> 数据源 model
    k: keyArr[i],
    v: tempObj[keyArr[i]]
  };
}

export default {
  name: "FormItem",
  mixins: [Emitter],
  props: { // 属性说明阅读文档：==> https://www.iviewui.com/components/form
    label: {
      type: String,
      default: ""
    },
    labelWidth: {
      type: Number
    },
    prop: {
      type: String
    },
    required: {
      type: Boolean,
      default: false
    },
    rules: {
      type: [Object, Array]
    },
    error: {
      type: String
    },
    validateStatus: {
      type: Boolean
    },
    showMessage: {
      type: Boolean,
      default: true
    },
    labelFor: {
      type: String
    }
  },
  data() {
    return {
      prefixCls: prefixCls,
      isRequired: false,
      validateState: "", // 校验状态： validating、error、success
      validateMessage: "",
      validateDisabled: false,
      validator: {}
    };
  },
  watch: {
    error: {
      handler(val) {
        this.validateMessage = val;
        this.validateState = val ? "error" : "";
      },
      immediate: true
    },
    validateStatus(val) {
      this.validateState = val;
    },
    rules() {
      this.setRules();
    }
  },
  inject: ["form"], // inject Form组件属性得到父组件实例
  computed: {
    classes() { // 样式变化
      return [
        `${prefixCls}`,
        {
          [`${prefixCls}-required`]: this.required || this.isRequired,
          [`${prefixCls}-error`]: this.validateState === "error",
          [`${prefixCls}-validating`]: this.validateState === "validating"
        }
      ];
    },
    fieldValue() {
      const model = this.form.model;
      if (!model || !this.prop) {
        return;
      }

      let path = this.prop;
      if (path.indexOf(":") !== -1) {
        path = path.replace(/:/, ".");
      }

      return getPropByPath(model, path).v;
    },
    labelStyles() {
      let style = {};
      const labelWidth =
        this.labelWidth === 0 || this.labelWidth
          ? this.labelWidth
          : this.form.labelWidth;

      if (labelWidth || labelWidth === 0) {
        style.width = `${labelWidth}px`;
      }
      return style;
    },
    contentStyles() {
      let style = {};
      const labelWidth =
        this.labelWidth === 0 || this.labelWidth
          ? this.labelWidth
          : this.form.labelWidth;

      if (labelWidth || labelWidth === 0) {
        style.marginLeft = `${labelWidth}px`;
      }
      return style;
    }
  },
  mounted() {
    if (this.prop) { // 添加字段，如果添加了prop属性
      this.dispatch("iForm", "on-form-item-add", this);

      Object.defineProperty(this, "initialValue", {
        value: this.fieldValue
      });

      this.setRules();
    }
  },
  beforeDestroy() { // 移除字段
    this.dispatch("iForm", "on-form-item-remove", this);
  },
  methods: {
    setRules() { // 触发rule收集并绑定对应的事件
      let rules = this.getRules();
      if (rules.length && this.required) {
        return;
      } else if (rules.length) {
        rules.every(rule => { // 遍历rule，确认组件是否required
          this.isRequired = rule.required;
        });
      } else if (this.required) { // 为真props 直接覆盖isRequired
        this.isRequired = this.required;
      }
      // 接受表单组件change、blur通知
      this.$off("on-form-blur", this.onFieldBlur);
      this.$off("on-form-change", this.onFieldChange);
      this.$on("on-form-blur", this.onFieldBlur);
      this.$on("on-form-change", this.onFieldChange);
    },
    getRules() { // 获取FormItem的rule，没有则取Form的rule
      let formRules = this.form.rules;
      const selfRules = this.rules;

      formRules = formRules ? formRules[this.prop] : [];

      return [].concat(selfRules || formRules || []);
    },
    getFilteredRule(trigger) {
      const rules = this.getRules();

      return rules.filter(
        rule => !rule.trigger || rule.trigger.indexOf(trigger) !== -1
      );
    },
    validate(trigger, callback = function() {}) {
      this.$nextTick(() => {
        let rules = this.getFilteredRule(trigger);
        if (!rules || rules.length === 0) {
          if (!this.required) {
            callback();
            return true;
          } else {
            rules = [{ required: true }];
          }
        }

        this.validateState = "validating";

        let descriptor = {};
        descriptor[this.prop] = rules;

        const validator = new AsyncValidator(descriptor);
        let model = {};

        model[this.prop] = this.fieldValue;

        validator.validate(model, { firstFields: true }, errors => {
          this.validateState = !errors ? "success" : "error";
          this.validateMessage = errors ? errors[0].message : "";

          callback(this.validateMessage);
        });
        this.validateDisabled = false;
      });
    },
    resetField() {
      this.validateState = "";
      this.validateMessage = "";

      let model = this.form.model;
      let value = this.fieldValue;
      let path = this.prop;
      if (path.indexOf(":") !== -1) {
        path = path.replace(/:/, ".");
      }

      let prop = getPropByPath(model, path);
      if (Array.isArray(value)) {
        this.validateDisabled = true;
        prop.o[prop.k] = [].concat(this.initialValue);
      } else {
        this.validateDisabled = true;
        prop.o[prop.k] = this.initialValue;
      }
    },
    onFieldBlur() {
      this.validate("blur");
    },
    onFieldChange() {
      if (this.validateDisabled) {
        this.validateDisabled = false;
        return;
      }
      this.validate("change");
    }
  }
};
</script>
```