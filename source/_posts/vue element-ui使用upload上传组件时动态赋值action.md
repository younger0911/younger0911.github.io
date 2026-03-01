---
title:  vue+element使用upload组件动态action
date:  2019-01-11 15:06:28
tags: 
  - vue
  - element-ui
categories: 
  - 前端
---

vue+element-ui使用upload上传组件时动态赋值action问题

<!-- more -->

```html
<el-upload
           name="files"
           :action="upAct"
           :headers="upHeader"
           list-type="picture-card"
           :on-preview="handlePictureCardPreview"
           :on-remove="handleRemove"
           :before-remove="beforeRemove"
           :on-success="showSaveSuccess"
           :on-error="showSaveError"
           :auto-upload="false"
           :multiple="true"
           :limit="10"
           ref="uploadF">
    <i class="el-icon-plus"></i>
</el-upload>
```

```javascript
data(){
      return{
      	upHeader: {"Authorization": getCookie("token")},
        upAct: '',
      }
}
```

```javascript
methods:{
	...
	this.upAct = "/api/store/upFiles?storeId=" + res.data.msg;
    setTimeout(function(){
        this.$refs.uploadF.submit();
    }.bind(this),500)
    ...
```

**问题：**

​	配置action后上传时会报错，action一直是空值。这是因为element的上传方法先执行，action的动态响应后执行。

**现解决方法：**

​	给提交动作加个延时，不然action一直是初始的空。