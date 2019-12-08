title: iview 源码分析之 Form 组件

---

### 前言

在 HTML 中，`<form>`元素表示了文档中的一个区域，此区域包含有交互控制元件，用来向 Web 服务器提交信息，其主要解决的问题就是三类问题：

- 数据赋值、数据获取：让*Form*内的表单组件获取到数据并展示出来，同时这些组件上的数据也能反过来让*Form*组件获取到
- 数据校验：对数据的有效性进行校验

经常使用 iview 的小伙伴应该对 form 组件非常熟悉，一般使用它长下面这个样子：

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

#### Form 展示

```html
<template>
  <form :class="classes" :autocomplete="autocomplete">
    <slot></slot>
  </form>
</template>
```
