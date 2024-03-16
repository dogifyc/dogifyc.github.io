---
title: Layui 自定义打印
date: 2024-03-16 21:52:05
tags:
  - Layui
---

# 基础应用

Layui 的 table 模块提供了基础的打印功能，能够满足大部分的基本打印需求。只需要两行简单代码，即可开启Layui table模块提供的打印功能：

```javascript
layui.use('table', function() {
  const table = layui.table;
  
  table.render({
    elem: '#default_table',
    ...
    toolbar: true, // 开启表格头部工具栏
    defaultToolbar: ['filter', 'print'], // 指定工具栏内容为列配置，打印
    cols: ...
  })
})
```

以上配置渲染的列表：
{% asset_img table.png "表格示例图"%}

点击上图中1的按钮，可以选择只显示部分列表。

{% asset_img column.png "展示字段选择"%}

点击上图中的按钮2，Layui 构建一个新页面，仅包含内部的表格内容。

{% asset_img print.png "打印弹框"%}

至此 Layui table 模块介绍的简单打印功能说明完毕。

# 扩展 Layui table的打印功能

在打印时，经常会要在打印的表格前面或后面增加一些统计信息或签名信息，而Layui table提供的打印功能不支持添加自定义信息。为了实现这个需求，我们需要修改 Layui table 的源码。

1. 首先从码云上下载  Layui 的源码
```
git clone https://gitee.com/sentsin/layui.git
```
2. 安装依赖
```
yarn install
```
3. 修改代码(增加start-end之间内容)，修改思路为增加table打印参数，允许在初始化 table 时传入打印参数，自定义的打印参数最后介绍。
```javascript
文件：src/lay/modules/table.js

  ...
  //默认配置
  Class.prototype.config = {
    limit: 10 //每页显示的数量
    ,loading: true //请求数据时，是否显示loading
    ,cellMinWidth: 60 //所有单元格默认最小宽度
    ,defaultToolbar: ['filter', 'exports', 'print'] //工具栏右侧图标
    ,autoSort: true //是否前端自动排序。如果否，则需自主排序（通常为服务端处理好排序）
    ,text: {
      none: '无数据'
    },
    // start
    // 增加默认自定义打印参数
    print: {
      func: undefined,
      preHtml: '',
      suffixHtml: '',
    }
    // end
  };
  
  ...
  
  case 'LAYTABLE_PRINT': //打印
          // start
          // 若自定义打印函数存在，则直接调用
          if ((typeof options.print.func) === 'function') {
            options.print.func(that);
            break;
          }
          // end
          var printWin = window.open('打印窗口', '_blank')
          ,style = ['<style>'
            ,'body{font-size: 12px; color: #666;}'
            ,'table{width: 100%; border-collapse: collapse; border-spacing: 0;}'
            ,'th,td{line-height: 20px; padding: 9px 15px; border: 1px solid #ccc; text-align: left; font-size: 12px; color: #666;}'
            ,'a{color: #666; text-decoration:none;}'
            ,'*.layui-hide{display: none}'
          ,'</style>'].join('')
          ,html = $(that.layHeader.html()); //输出表头
          
          html.append(that.layMain.find('table').html()); //输出表体
          html.append(that.layTotal.find('table').html()) //输出合计行
          
          html.find('th.layui-table-patch').remove(); //移除补丁
          html.find('.layui-table-col-special').remove(); //移除特殊列

          // start
          // 若配置打印表格头部信息，则拼接
          if (options.print.preHtml) {
            html.prepend(options.print.preHtml instanceof $ ? options.print.preHtml.html() : options.print.preHtml);
          }
          // 若配置打印表格后部信息，则拼接
          if (options.print.suffixHtml) {
            html.prepend(options.print.suffixHtml instanceof $ ? options.print.suffixHtml.html() : options.print.suffixHtml);
          }
          // end
          printWin.document.write(style + html.prop('outerHTML'));
          printWin.document.close();
          printWin.print();
          printWin.close();
        break;
      }
```
4. 打包构建
```
gulp
```
你可以在 `release/zip/layui-v2.5.6/layui/lay/modules` 文件夹下找到新构建的 `table.js`。你可以使用此文档替换掉你所使用的 `table.js`。若你使用的是 `layui.all.js`，而不是模块化的引用方式，则可以使用 `release/zip/layui-v2.5.6/layui` 文件夹下的 `layui.all.js`。
若你不熟悉构建流程，或不想操作这么麻烦，可以直接使用我打包好的文件。下面这个压缩包里包含了魔改后的`table.js`和`layui.all.js` {% asset_link print.zip "print.zip" %}

5. 参数说明
以上修改为table的基本参数追加以下参数

|  参数 | 类型  | 说明  | 默认值  |
| ------------ | ------------ | ------------ | ------------ |
| print  | Object  | 打印参数配置对象  | {func: undefined,preHtml: '',suffixHtml: ''}  |
| print.func  | Function  | 自定义的打印函数，若配置此参数，则会替换掉Layui的打印行为  | undefined  |
| print.preHtml  | String/JqueryObject  | 表格前面的内容，可以是html字符串，也可以是$(selector)返回的jquery对象。因为打印页面的css样式传导比较麻烦，建议要打印的头部内容的样式采用行内样式（即style=""的形式）  | ''  |
| print.suffixHtml  | String/JqueryObject  | 表格后面的内容，与print.preHtml仅位置不同  | ''  |

如果你只需要在表格前后追加一些统计信息和签名信息，则使用print.preHtml和print.SuffixHtml即够使用，否则，你需要使用print.func模仿 layui的打印方式，完全自定义重写打印方法，为简化代码，调用print.func时传入了table的上下文this,你可以通过他引用layHeader，layMain和layTotal等内容。

# 打印统计页面

有时候可能要打印页面内容，如统计页。而这种打印与layui无关。但打印时，管理后台页面的菜单等内容往往是不需要打印的，但直接调用浏览器的打印功能，又会把这些打出来。

这时候就可以使用css的媒体查询符，在打印时隐藏不需要打印的内容，如：
```css
@media print {
    .layui-header {display:none}
    .layui-side {display:none}
    .layui-footer {display:none}
}
```

以上样式仅提供一个思路，具体实现需要自己想哦。