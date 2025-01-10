+++
title = 'Prosemirror自定义节点和视图说明'
date = 2024-11-20T10:14:13+08:00
draft = false
tags = ['ProseMirror']
categories = ['前端']
+++
在构建基于 ProseMirror 的在线编辑器时，自定义 schema 和 view 可以根据业务需求实现特定功能和样式。下面详细说明如何自定义 ProseMirror 的 schema 和 view，并提供相应的示例代码。
## 自定义 Schema

Schema 定义了编辑器中允许的文档结构和节点类型。通过自定义 schema，你可以控制哪些元素可以被编辑器处理，以及它们的属性和行为。
### 自定义 Schema 示例
假设我们希望在编辑器中支持自定义的标题和简单的段落格式：
```javascript
import { Schema } from "prosemirror-model";

// 定义自定义 schema
const mySchema = new Schema({
    nodes: {
        doc: { content: "block+" },
        paragraph: {
            content: "text*",
            group: "block",
            parseDOM: [{ tag: "p" }],
            toDOM() { return ["p", 0]; }
        },
        heading: {
            attrs: { level: { default: 1 } },
            content: "text*",
            group: "block",
            defining: true,
            parseDOM: [
                { tag: "h1", attrs: { level: 1 } },
                { tag: "h2", attrs: { level: 2 } }
            ],
            toDOM(node) { return ["h" + node.attrs.level, 0]; }
        },
        text: { group: "inline" }
    }
});
```
在这个 schema 中，我们定义了 paragraph 和 heading 节点，允许在文档中使用段落和标题。标题节点具有一个 level 属性，可以通过 DOM 解析和序列化来处理不同级别的标题。
## 自定义 View
自定义 view 允许你定义编辑器中节点的呈现方式和交互行为。通过自定义 view，你可以为节点添加特定的样式或交互逻辑。
### 自定义 View 示例
假设我们希望为标题节点添加一个自定义的样式：
```javascript
import { EditorView } from "prosemirror-view";

// 自定义标题节点的 view
function headingView(node, view, getPos) {
    let dom = document.createElement("h" + node.attrs.level);
    dom.className = "custom-heading";
    dom.textContent = node.textContent;
    return {
        dom,
        update(updatedNode) {
            if (updatedNode.type !== node.type) return false;
            dom.textContent = updatedNode.textContent;
            return true;
        }
    };
}

// 初始化编辑器视图
const view = new EditorView(document.querySelector("#editor"), {
    state,
    nodeViews: {
        heading(node, view, getPos) {
            return headingView(node, view, getPos);
        }
    }
});
```
在这个示例中，我们为 heading 节点创建了一个自定义的 view，使用 headingView 函数为标题节点添加了一个自定义的 CSS 类 custom-heading。通过 update 方法，我们可以更新节点的内容。
## 总结
通过自定义 schema 和 view，ProseMirror 提供了强大的扩展能力，使得开发者可以根据具体需求定制编辑器的功能和样式。你可以根据项目需求进一步扩展这些基础示例，以实现更复杂的文档结构和交互效果。这样做不仅提高了编辑器的灵活性，还能显著提升用户体验。