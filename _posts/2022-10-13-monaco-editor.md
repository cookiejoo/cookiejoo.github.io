---
title: VUE代码编辑器
layout: post
author: 曹德高
tags: [java, vue]
categories: Java
---

### 前言

上一章大概的聊了一下我设想中的低代码模式原型，其中就有代码编辑器的功能，这一章来分享一下我的网页版代码编辑器如何实现。

### Monaco-Editor

前端使用的是VUE，微软开源了VSCode前端版插件，我们就用这个插件来做代码编辑器功能，虽然基础版只有几种前端的高级支持，但显示上已经包含很多语言的基础版支持了。

```bash
# 安装
npm install monaco-editor -S
```

我们需要自定义这个组件(类似java的封装)方便后续的组件引入使用，所以在src->components下新建了一个组件MonacoEditor.vue。

由于我的项目优先支持java所以我默认把组件的语言设置为java，还有一个注意事项的是props的定义，这个属性下面的子属性是可以在组件使用时传值过来动态设置绑定的，我也是跟前端大佬学的一招，我自己开发的时候不知道怎么根据文件后缀来动态设置语言苦恼了好几个小时。

```vue
<template>
  <div ref="editorContainer" class="editor"></div>
</template>

<script>
import * as monaco from 'monaco-editor';
export default {
  name: 'MonacoEditor',
  props: {
    code: '',
    language: 'java'
  },
  data() {
    return {
      editor: null
    }
  },
  mounted() {
    this.init()
  },
  methods: {
    init() {
      // 初始化编辑器
      if(this.language === 'properties'){
        this.language = 'yaml'
      }
      this.editor = monaco.editor.create(this.$refs.editorContainer, {
        value: this.code,
        language: this.language,
        tabSize: 2,
        scrollBeyondLastLine: false,
        automaticLayout: true, // 自动布局
        readOnly: false
      })
      console.log(this.editor)

      // 监听内容变化
      this.editor.onDidChangeModelContent(() => {

      })

      // 监听失去焦点事件
      this.editor.onDidBlurEditorText((e) => {
        console.log(e)
      });
    },
    // 获取编辑框内容
    getCodeContext() {
      return this.editor.getValue()
    },
    // 自动格式化代码
    format() {
      this.editor.trigger('anything', 'editor.action.formatDocument')
      // 或者
      // this.editor.getAction(['editor.action.formatDocument']).run()
    }
  }
}
</script>

<style scoped>
.editor {
  width: 100%;
  height: 100%;
}
</style>

```

### 使用

根据左边的项目文件树，选中文件后，右边的面板动态打开自定义的组件\<MonacoEditor\>，这里:code="item.content" :language="item.suffix"两个属性就是上面我说到的动态传值了。

```java
<el-tabs v-model="activeName" closable @tab-remove="removeFileTab">
  <el-tab-pane
  	v-for="item in fileTabList"
    	:key="item.path"
      :label="item.title"
      :name="item.name">
          <MonacoEditor :code="item.content" :language="item.suffix" ref="MonacoEditor" style="height: 830px"></MonacoEditor>
	</el-tab-pane>
</el-tabs>
```

读取文件路径下所有的文件，返回给前端生成文件树

```java
@Getter
@Setter
@ToString
public class FileTreeVO {
    private String filename;
    private String path;
    private String content;
    private Boolean directory;
    private String suffix;

    List<FileTreeVO> children;
}

@Slf4j
public class FileUtil {
		/**
     * 枚举所有文件，包括文件夹
     */
    public static void listFile(List<FileTreeVO> fileTree, File dir) {
        if (!dir.exists()) {
            return;
        }
        File[] fs = dir.listFiles();
        if (fs != null) {
            for (File f : fs) {
                //隐藏文件类型就不用显示
                if (f.isHidden()) {
                    continue;
                }
                FileTreeVO vo = new FileTreeVO();
                vo.setFilename(f.getName());
                vo.setPath(f.getPath());
                vo.setDirectory(f.isDirectory());
                fileTree.add(vo);
                if (f.isDirectory()) {
                    vo.setChildren(Lists.newLinkedList());
                    listFile(vo.getChildren(), f, accessSuffix);
                    vo.setSuffix(StringUtils.EMPTY);
                } else {
                    vo.setSuffix(FilenameUtils.getExtension(f.getName()));
                    
                  	try {
                    	vo.setContent(FileUtils.readFileToString(f, StandardCharsets.UTF_8));
                  	} catch (IOException e) {
                    	log.warn("{},读取失败！", f.getName());
                  	}
                }
            }
        }
    }
}
```

具体的效果如下：这个编辑器还是很漂亮的。

![image-20221013191911876](/images/2022-10-13-monaco-editor/image-20221013191911876.png)

![image-20221013192029604](/images/2022-10-13-monaco-editor/image-20221013192029604.png)

### 最后

编辑器的模型已经有了，虽然它只是个基础版，我看了有人已经把这个插件做了扩展能连接远程得到java的一些代码提示的功能，由于个人前端能力极其有限，所以这个高级功能留给专门的前端大佬去完善吧。

后续我把我实现的代码脚手架和Mysql连接到POJO的三层代码的做法分享出来。

