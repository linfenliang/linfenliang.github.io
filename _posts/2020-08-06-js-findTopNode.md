---
layout: post
title:  "JS 从对象数组中JSONArray根据子节点ID找到最外层的节点"
subtitle: "JS根据子节点ID找到最顶层的节点"
author: "linfenliang"
date:   2020-08-06
header-img: "img/post-bg-tech.jpg"
catalog:    true
tags:
    - 学习
    - JS
categories: JavaScript
---


# 概述
老婆的需求，根据某个子节点的ID找到最顶层的父节点。
数据结构基本长这样：
   
```
   {id:"xx",name:"xx",children:[
        {id:"yyy",name:"",children:[
                {id:"zzz",name:"",children:[]}
        ]}
   ]}
   
```

如根据 zzz 节点 找到 xx 节点。

# 实现

## 实现思路
递归遍历整个数组对象，找到最内层的与给出ID相等的节点，并返回对应的父级节点，由于是递归调用，
那么一旦在内层发现有节点匹配则返回到外层取外层的节点再次返回，递归到最外层。

## 注意点
需要注意的地方有两个，一个是判断当前节点是JSON数组Array还是JSON对象Object，另一个是当为对象的时候可能存在两种情况：
当前已经是最内层节点了，不存在children，所以判断当前节点是否与给出节点匹配，匹配则返回当前节点即可，不匹配则继续；
另一种情况则当前节点处在中间位置，则需要继续遍历内部的子节点做判断，直到最内层节点或发现与给出节点匹配。


## 实现代码示例

```
function findTopNodeById(jsonArr,id){
	if(jsonArr instanceof Array){
		for(let i = 0;i < jsonArr.length;i++){
			let r = findTopNodeById(jsonArr[i],id);
			if(r!=undefined){
				return r;
			}
		}
		return undefined;
	}
	// console.log("jsonArr id:",jsonArr,jsonArr.id);
	if(jsonArr.id == id){
		return "1";
	}else if(jsonArr.children && jsonArr.children.length > 0){

		for(let i = 0;i < jsonArr.children.length; i++){
			let item = jsonArr.children[i];
			// console.log("curr:",item);
			if(item.id == id){
				// console.log('find it:',item);
				return jsonArr;
			}else if(item.children && item.children.length > 0){
				let r = findTopNodeById(item.children,id);
				if(r != undefined){
					return jsonArr;
				}
			}else{
				return undefined;
			}
		}
	}else{
		return undefined;
	}
}


var x = findTopNodeById(arr,'zzz');
console.log("the result is:",x);


```

