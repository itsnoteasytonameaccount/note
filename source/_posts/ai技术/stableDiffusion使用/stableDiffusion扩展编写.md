---
title: stable diffusion webui拓展编写
tags:
- python
- stable diffusion webui
date: 2023/06/11
---

# 用户scripts

可以往scripts目录中添加py文件,此处添加的文件可以在界面如下位置嵌入用户脚本

![image-20230614010148165](.\assets\image-20230614010148165.png)

代码webui代码中对应的插入逻辑如下

![image-20230614010308383](.\assets\image-20230614010308383.png)

![image-20230614010429111](.\assets\image-20230614010429111.png)

webui会自动列出scripts下的文件并根据是否为`Script`类的子类来决定是否执行

script文件示例

```python
from ldm.modules.encoders.modules import FrozenCLIPEmbedder, FrozenOpenCLIPEmbedder
from modules import scripts

import gradio as gr

class Script(scripts.Script):
    def title(self):
        print('title')
        return 'mytitle'
    
    # 如果方法这个不实现，scripts下拉列表里就会出现一个选项，选项的文本由title方法定义，scripts下拉列表中对应文本被选中时，ui方法中生成的界面便会显示。若是此函数实现返回alwaysVisible,ui方法返回的组件就会直接显示在界面上
    def show(self, is_img2img):
        return scripts.AlwaysVisible

    def ui(self, is_img2img):
        checkbox_iterate = gr.Checkbox(label="我的内容", value=False)
        print('ui')
        return [checkbox_iterate]

    def run(self, checkbox_iterate):
        print('run')
        pass
```

![image-20230619011934643](D:\projects\hexo_blog\source\_posts\stableDiffusion使用\assets\image-20230619011934643.png)

# extensions

[文档地址](https://github.com/AUTOMATIC1111/stable-diffusion-webui/wiki/Developing-extensions)

## 文件路径及作用

首先是官方对各个文件夹的说明

- extension's `install.py` script, if it exists, is executed.

- extension's scripts in the

   

  ```
  scripts
  ```

   

  directory are executed as if they were just usual user scripts, except:

  - `sys.path` is extended to include the extension directory, so you can import anything in it without worrying
  - you can use `scripts.basedir()` to get the current extension's directory (since user can name it anything he wants)

- extension's javascript files in the `javascript` directory are added to the page

- extension's localization files in the `localizations` directory are added to settings; if there are two localizations with same name, they are not merged, one replaces another.

- extension's `style.css` file is added to the page

- if extension has `preload.py` file in its root directory, it is loaded before parsing commandline args

- if extension's `preload.py` has a `preload` function, it is called, and commandline args parser is passed to it as an argument. Here's an example of how to use it to add a command line argument:

## javascript

首先是javascript可以写在javascript目录，这个目录下的文件会自动添加到页面最后，无需多言

这里写的javascript脚本有个限制，就是没法该设置啥的，只能影响影响界面，想要进行一些设置上的更改还是需要配置python

## scripts

scripts目录，这个目录比较重要，一般python脚本写在这里，不过首先要知道这里可以写什么

### 界面扩展

#### gradio

想要写插件首先需要会用webui使用的界面组件，这个界面组件用的是一个叫做gradio的，可以在[官方文档](https://gradio.app/docs/#button)了解详细用法

最常用的用法以button为例

```python
# 如下写法需要注意的几点是
# 1. fn的入参一定要和inputs的长度匹配，否则点击按钮时会报错参数数量不匹配
# 2. 如果有_js参数，且inputs列表为空，那么此时_js必须返回数组，否则websocket调用时上传的数据后台会解析失败，按钮失效，inputs不为空时，会自动转化_js返回值为数组
# 3. fn中的返回值会通过websocket返回给界面，outputs中的组件会在websocket通信后将值设置为fn的返回值，故outputs需要和fn的返回值一一对应
btn = gr.Button(elem_id='switch_model_btn', visible=False, value='clickBtn')
btn.click(
    fn=justprint,
    _js='function(v, ...args) {console.log("test", v, args, this); return ["ceshi"];}',
    inputs=[],
    outputs=[btn]
)
```

![image-20230703004527783](D:\projects\hexo_blog\source\_posts\stableDiffusion使用\assets\image-20230703004527783.png)

<div style="text-size: 12px;text-align:center">1. inputs为空，但是__js返回值不为数组的情况，data数据不为数组，后台解析错误阻塞，无响应，js测的调用也一直处于pending无响应，之后再怎么点click都不执行代码了</div>

#### 用户scripts

界面上有很多插入的点，首先就是界面最下面的用户scripts列表，这个的实现方法和前面介绍到的用户scripts是一样的

#### 通过隐藏的按钮更新配置

在`shared.py`文件中定义了全局的配置，配置的主要操作方法在`shared.py`的`Options`中定义

![image-20230619014524467](D:\projects\hexo_blog\source\_posts\stableDiffusion使用\assets\image-20230619014524467.png)

在`ui.py`中有该类的使用方法示例如下，如下为实现修改选中模型的下拉框的实现

```python
# 回调函数定义
def run_settings_single(value, key):
        if not opts.same_type(value, opts.data_labels[key].default):
            return gr.update(visible=True), opts.dumpjson()

        if not opts.set(key, value):
            return gr.update(value=getattr(opts, key)), opts.dumpjson()

        opts.save(shared.config_filename)

        return get_value_for_setting(key), opts.dumpjson()

# ...

# 渲染界面输入元素
with gr.Blocks(css=css, analytics_enabled=False, title="Stable Diffusion") as demo:
    with gr.Row(elem_id="quicksettings", variant="compact"):
        for i, k, item in sorted(quicksettings_list, key=lambda x: quicksettings_names.get(x[1], x[0])):
            component = create_setting_component(k, is_quicksettings=True)
            component_dict[k] = component

    # ...
    # 设置输入框onChange回调
    for i, k, item in quicksettings_list:
        component = component_dict[k]
        info = opts.data_labels[k]
        component.change(
            fn=lambda value, k=k: run_settings_single(value, key=k),
            inputs=[component],
            outputs=[component, text_settings],
            show_progress=info.refresh is not None,
        )


# 设置其他切换按钮
button_set_checkpoint = gr.Button('Change checkpoint', elem_id='change_checkpoint', visible=False)
button_set_checkpoint.click(
	fn=lambda value, _: run_settings_single(value, key='sd_model_checkpoint'),
	_js="function(v){ var res = desiredCheckpointName; desiredCheckpointName = ''; return [res || v, null]; }"
    inputs=[component_dict['sd_model_checkpoint'], dummy_component],
    outputs=[component_dict['sd_model_checkpoint'], text_settings],
)

"""
关于gr.update的说明
(function) def update(**kwargs: Any) -> dict
Updates component properties. When a function passed into a Gradio Interface or a Blocks events returns a typical value, it updates the value of the output component. But it is also possible to update the properties of an output component (such as the number of lines of a Textbox or the visibility of an Image) by returning the component's update() function, which takes as parameters any of the constructor parameters for that component. This is a shorthand for using the update method on a component. For example, rather than using gr.Number.update(...) you can just use gr.update(...). Note that your editor's autocompletion will suggest proper parameters if you use the update method on the component.
Demos: blocks_essay, blocks_update, blocks_essay_update

Parameters:
    kwargs: Key-word arguments used to update the component's properties.
Example:
    # Blocks Example import gradio as gr with gr.Blocks() as demo:
        radio = gr.Radio([1, 2, 4], label="Set the value of the number") number = gr.Number(value=2, interactive=True)
        radio.change(fn=lambda value: gr.update(value=value), inputs=radio, outputs=number)
    demo.launch()

    # Interface example import gradio as gr def change_textbox(choice):
      if choice == "short":
          return gr.Textbox.update(lines=2, visible=True)
      elif choice == "long":
          return gr.Textbox.update(lines=8, visible=True)
      else:
          return gr.Textbox.update(visible=False)
    gr.Interface(
      change_textbox, gr.Radio(
          ["short", "long", "none"], label="What kind of essay would you like to write?"
      ), gr.Textbox(lines=2), live=True,
    ).launch()
"""
```





