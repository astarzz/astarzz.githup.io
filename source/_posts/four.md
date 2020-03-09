---
title: "JavaScript多叉树的遍历方式"
date: 2018-11-23 18:38:39
categories: 
    - "数据结构"
    - "多叉树"
toc_number: true
tags:
	- 递归
	- 栈
	
---
## 数据结构
- 一个根节点，它下面有若干个(0-N)子节点，这些子节点下面又有属于各自的若干子节点，以此类推，构成一棵树。没有子节点的节点是叶子节点，有相同父节点的为兄弟节点。
- 多叉树在JavaScript中应用频繁，dom对象也是利用这种数据结构存储。还有数据库中的父子、上下级关系数据，在前台展示的时候需要体现层级效果，所以很多前端框架都有相关树形插件实现，比如easyui的tree、ztree。
- 树形结构的数据，在使用、处理过程中不免需要访问节点数据或转换成平级列表，这里就涉及到它的遍历。
<!--more-->

### 图形表示
![alt](tree.png)
### json格式

  ```json
  {
    "id": 1,
    "text": "Node 1",
    "children": [{
        "id": 11,
        "text": "Node 11"
    },{
        "id": 12,
        "text": "Node 12",
        "children":[{
            "id": 22,
            "text": "Node 22"
        }]
    }]
  }
  ```
  

## 遍历方式
### 广度优先: 
  从根节点开始，按照节点的层次，从上往下一级一级访问，每一级访问完所有节点后再往下一级访问，直至没有下一级。原则:有同级先访问同级，没有就找下一级
  ![alt](wide.png)
   A->B->C->D->E->F->G
- 非递归方式(队列实现):
    
    ```javascript
    function wideFirst(node,callback) {
        let nodes = [];//平级列表
        if (node) {
            let queue = [];//队列
            queue.push(node);//根节点处理
            while (queue.length > 0) {
                node = queue.shift();//队列删除第一个，先进先出
                nodes.push(node);
                if(callback){
                    callback(node);//回调函数
                }
                let childs = node.children;
                    if(childs){
                        for (let i = 0,len=childs.length; i<len ;i++){
                            queue.push(childs[i]);
                        }
                    }
            }
        }
        return nodes;
    }
    ```

### 深度优先:
从根节点开始，按照一条完整的分支线区分(从根节点到每一个叶子节点的路线)，从左往右(或从右往左)访问各个分支。**在节点不重复访问的前提下**，每一个分支从上往下访问各个节点。原则:有子节点优先访问子节点，没有子节点找兄弟节点,还是没有再找父节点的兄弟节点。
![alt](deep.png)
  A->B->E->C->F->G->D
- 递归方式:
    
    ```javascript
    function deepFirst(node,callback){
        let nodes = [];//平级列表
        if(node){
            deepFirstTo(node,nodes,callback);
        }
        return nodes;
    };
    function deepFirstTo(node,nodes,callback) {
        if (node) {
            nodes.push(node);
            if(callback){
                callback(node);//回调函数
            }
            let childs = node.children;
            if(childs){
                for (var i = 0,len = childs.length; i< len; i++){
                    deepFirstTo(childs[i],nodes,callback);  
                }
            }
        }
    }
    ```
    
- 非递归方式(栈实现):
    
    ```javascript
    function deepFirst(node,callback) {
        let nodes = [];//平级列表
        if (node) {
            var stack = [];//栈
            stack.push(node);//处理根节点
            while (stack.length > 0) {
                node = stack.pop();//栈删除最后一个，后进先出
                nodes.push(node);
                if(callback){
                    callback(node);//回调
                }
                let childs = node.children;
                if(childs){
                    for (let i = 0,len = childs.length; i < len; i++){
                        stack.push(childs[i]);
                    }
                }
            }
        }
        return nodes;
    }
    ```