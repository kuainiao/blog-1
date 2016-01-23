# React.js

## 简介

提供MVC中的“V”，

* 单向数据流

	数据一旦更新，就直接重新渲染整个app。
	
	一个react组件可以理解成一个独立的函数：接收参数（props），可复用，可传递，返回结果（渲染组件）	
* 虚拟DOM树

	* 重建DOM树
	* 找到上个版本中DOM的差异
	* 计算出最新的DOM更新操作
	* 从操作队列中批量执行DOM更新操作
	
	可以在Node中运行
	
react只关注两件事：

* 更新DOM
* 响应事件框架	

React本质上是一个“状态机”，可以帮助开发者管理复杂的随着时间而变化的状态。它以一个精简的模型实现了这一点。

## JSX

JSX（JavaScrip XML）：一种在react组件内部构建标签的类XML语法。React在不适用JSX的情况下一样可以工作，然而使用JSX可以提高组件的可读性。



	
## helloworld

## 组件
### 组件嵌套			

### 组件的状态	

### 组件参数props

## 事件events

## 指向ref

## 双向数据流


## 组件生命周期

* componentWillMount
* componentWillUpdate
* 

## mixins

## 投票小项目

### 环境配置

	mkdir reactQa
	cd reactQa
	npm init
	
	npm install react --save
	npm install -g gulp

开发工具：

	npm install --save-dev gulp gulp-browserify gulp-concat gulp-react gulp-connect lodash reactify
	
前端：

	bower init
	bower install bootstrap --save
	
创建app和dist目录

	reactQa/app, react/dist
	
创建gulpfile.js

					
	
 

